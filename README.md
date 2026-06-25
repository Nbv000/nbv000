# nbv000.su

Сайт-визитка / портфолио для проекта Access Denied.

## Стек
- Jekyll 4.x
- GitHub Pages для хостинга
- Домен: nbv000.su

## Локальный запуск

```bash
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

## Структура

- `_posts/` — кейсы и технические разборы
- `_layouts/` — шаблоны страниц
- `assets/css/style.css` — стили
- `index.md`, `services.md`, `cases.md`, `contact.md` — статические страницы
- `CNAME` — кастомный домен для GitHub Pages

## Деплой

Push в `main` ветку — GitHub Pages пересобирает автоматически.

Для домена nbv000.su в DNS-провайдере нужны записи:
- A: 185.199.108.153
- A: 185.199.109.153
- A: 185.199.110.153
- A: 185.199.111.153

И в Settings репозитория → Pages → Custom domain: nbv000.su, Enforce HTTPS.
