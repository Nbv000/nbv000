---
title: "IL2CPP bridge: что это и зачем"
date: 2026-06-24
description: "Как устроен IL2CPP runtime, зачем нужен bridge-слой между нативным кодом и managed-методами Unity, и как это работает на практике."
---

Большинство Unity-игр на мобайле и в вебе собраны через IL2CPP — Intermediate Language to C++. Вместо JIT-компиляции Mono весь managed C# транслируется в нативный код при билде. Результат: один большой `GameAssembly.so` / `GameAssembly.dll` вместо набора `.dll`-файлов.

Это меняет правила игры при реверсе и при инъекции.

## Что такое IL2CPP runtime

Il2CppObject — базовый тип для всех managed объектов. Каждый `MonoBehaviour`, каждый `string`, каждый `List<T>` в памяти — это нативная структура с заголовком из klass-pointer и полями данных.

```
Il2CppObject {
    Il2CppClass* klass;    // указатель на метаданные типа
    MonitorData*  monitor; // для lock/Monitor в C#
    // затем поля объекта
}
```

Вызов метода — это не виртуальная таблица в привычном смысле. IL2CPP генерирует конкретные функции для каждого метода, адреса хранятся в метаданных (`global-metadata.dat`). При AOT всё это фиксировано.

## Зачем нужен bridge

Из нативного кода (своя `.so`, injected library) нельзя просто вызвать managed метод как обычную функцию. Нужно:

1. Найти `Il2CppClass*` для нужного типа
2. Найти `MethodInfo*` для нужного метода внутри класса
3. Вызвать через `il2cpp_runtime_invoke` или напрямую по указателю из `MethodInfo`

Bridge — это обёртка, которая скрывает эту механику и даёт удобный интерфейс:

```cpp
// Без bridge
Il2CppClass* klass = il2cpp_class_from_name(
    il2cpp_domain_get(), "UnityEngine", "Application");
MethodInfo* method = il2cpp_class_get_method_from_name(
    klass, "set_targetFrameRate", 1);
il2cpp_runtime_invoke(method, nullptr, args, &exc);

// С bridge
SetStaticInt("UnityEngine", "Application",
             "set_targetFrameRate", 30);
```

В реальных проектах bridge ещё кэширует `MethodInfo*` — поиск по имени дорогой, делать его каждый кадр нельзя.

## Поиск методов в runtime

`il2cpp_class_from_name` принимает domain, namespace и class name. Namespace берётся из дампа — il2cppdumper или Cpp2IL дают полную картину.

Проблема возникает с generic-типами и nested классами. `List<int>` — это `System.Collections.Generic.List\`1` с параметром. Имя в метаданных выглядит иначе чем в исходнике C#. Nested class (`OuterClass/InnerClass`) в метаданных часто хранится как `OuterClass_InnerClass` или через отдельный поиск по enclosing type.

Для статических методов `object` в `il2cpp_runtime_invoke` — `nullptr`. Для instance-методов — указатель на `Il2CppObject`.

## Строки

`System.String` в IL2CPP — не null-terminated C string. Это `Il2CppString` со своим заголовком и UTF-16 данными. Передавать в метод как `char*` нельзя — нужно конструировать через `il2cpp_string_new` или `il2cpp_string_new_utf16`.

Обратно: получить C-строку из `Il2CppString*` — через `il2cpp_string_chars` (даёт UTF-16) с последующей конвертацией.

## Где брать адреса API

В `libil2cpp.so` (Linux) или `il2cpp.dll` (Windows) экспортированы все нужные функции — `il2cpp_class_from_name`, `il2cpp_class_get_method_from_name`, `il2cpp_runtime_invoke` и т.д. Получаем через `dlsym` / `GetProcAddress` после `dlopen` на библиотеку.

Конкретный набор экспортов зависит от версии Unity. Начиная с Unity 2019 API более-менее стабилен. В старых версиях часть функций может отсутствовать или иметь другую сигнатуру — проверяется через `nm -D libil2cpp.so | grep il2cpp_`.

## Итог

IL2CPP bridge — базовый строительный блок для любой инъекции в Unity-игры на IL2CPP. Без него каждое взаимодействие с managed кодом требует ручной работы с метаданными. С ним — вызов любого метода сводится к одной строке после первоначальной настройки.

Делаю под заказ — детали [в разделе услуг](/services/).
