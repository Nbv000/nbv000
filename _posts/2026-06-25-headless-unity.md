---
title: "Запуск Unity-игры без рендера"
date: 2026-06-25
description: "Как через LD_PRELOAD и IL2CPP-методы Unity отключить графику клиента, оставив логику живой. Подводные камни OnDemandRendering, VFXManager и SIGSEGV в ToggleAllVfx."
---

Стандартная задача для сбора данных, автоматизации и фоновых клиентов: поднять онлайн-игру на сервере без GPU. Логика крутится, сеть жива, графика выключена.

Эмулировать протокол — долго и ломается на каждом патче. Запускать клиент как есть — упирается в видеокарту. Решение: подгрузить через `LD_PRELOAD` свою библиотеку и из неё дёрнуть IL2CPP-методы самого Unity.

Базовый набор:

```cpp
SetStaticInt(core, "UnityEngine", "Application",
             "set_targetFrameRate", 1);
SetStaticInt(core, "UnityEngine", "QualitySettings",
             "set_shadows", 0);
SetStaticInt(core, "UnityEngine", "QualitySettings",
             "set_pixelLightCount", 0);
SetStaticInt(core, "UnityEngine", "QualitySettings",
             "set_vSyncCount", 0);
SetStaticBool(core, "UnityEngine", "QualitySettings",
              "set_realtimeReflectionProbes", false);
```

Каждый из этих вызовов — обращение к статическому setter'у в IL2CPP-метаданных конкретной версии Unity. Это значит: список рабочих setters придётся сверять с дампом для каждой сборки игры отдельно. В моей версии Albion, например, в `QualitySettings` нет `antiAliasing`, `lodBias`, `maximumLODLevel`, `masterTextureLimit` — их пришлось убрать из конфига.

Дальше начинаются неочевидные места.

**OnDemandRendering.renderFrameInterval** — официальный механизм пропуска кадров. В этой версии IL2CPP экспортирован только getter, setter отсутствует. Обходится через `targetFrameRate = 1` — на практике даёт то же самое.

**VFXManager** — класс существует, но в дампе у него только `.cctor`. Никаких публичных методов выключения визуальных эффектов. Решается через статическое поле `Vfx.bToggle` — пишем напрямую в память.

**ToggleAllVfx()** — единственный «правильный» способ глобально вырубить VFX. И он стабильно падает в SIGSEGV (`addr=0x134`) после `force_reinit + GC`. Причина: метод итерирует объекты VFX на сцене, а часть из них уже освобождена сборщиком мусора — managed pointer ведёт в никуда, IL2CPP dispatch валится. Safe-версия: только статика, никакой итерации.

В итоге: клиент работает на VPS без видеокарты, потребление CPU ~5%, GPU не задействован. Подходит для market data collection, фоновых ботов, автоматизации и любых задач, где нужна живая логика игры без графики.

Этот стек переиспользуется почти для любой Unity-игры на IL2CPP. Делаю под заказ — детали [в разделе услуг](/services/).
