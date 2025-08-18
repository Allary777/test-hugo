# Руководство по проекту на Hugo

> Актуально для: кастомной темы в репозитории `test-hugo` (без сторонних тем и библиотек).  
> Целевая аудитория: разработчики и контент-менеджеры.

## Оглавление

1. [Обзор](#обзор)
2. [Быстрый старт](#быстрый-старт)
3. [Структура проекта](#структура-проекта)
4. [Конфигурация (hugo.toml)](#конфигурация-hugotoml)
5. [Компоненты и шаблоны](#компоненты-и-шаблоны)
   - [Базовый шаблон: `_default/baseof.html`](#базовый-шаблон-_defaultbaseofhtml)
   - [Списки и страницы: `_default/list.html` и `_default/single.html`](#списки-и-страницы-_defaultlisthtml-и-_defaultsinglehtml)
   - [Страница «О компании»: `layouts/about/list.html`](#страница-о-компании-layoutsaboutlisthtml)
   - [Партилы (partials)](#партилы-partials)
6. [Стили и сборка SCSS](#стили-и-сборка-scss)
7. [Контент-менеджерам: добавление и правки](#контент-менеджерам-добавление-и-правки)
   - [Новая статья (карточка)](#новая-статья-карточка)
   - [Новый отзыв](#новый-отзыв)
   - [Правка страницы «О компании»](#правка-страницы-о-компании)
   - [Изображения](#изображения)
8. [Для разработчиков](#рецепты-для-разработчиков)
9. [Сборка и деплой](#сборка-и-деплой)
10. [Стандарты и договоренности](#стандарты-и-договоренности)
11. [Список источников](#список-источников)

---

## Обзор

Сайт компании на Hugo с самописной темой. Контент разделен на:
- **Блог** (`/content/posts`) — список статей, выводится сеткой карточек.
- **О компании** (`/content/about/_index.md`) — описание, иллюстрация и блок отзывов, загружаемых из отдельного файла.

Ключевые сущности:
- **Партилы**: `article-card.html`, `menu.html`, `footer.html`, `testimonial.html`, `testimonials-section.html`.
- **Стили**: SCSS в `assets/css/custom.scss` + компоненты, компиляция Hugo Pipes.
- **Меню**: настраивается в `hugo.toml` через `[menu.main]`.

## Быстрый старт

Требуется установленный **Hugo Extended** (для SCSS).

```bash
# локальный запуск (черновики видны)
hugo server -D

```

## Структура проекта

```
test-hugo/
├── archetypes/
│   └── default.md           # шаблон front matter (черновик по умолчанию)
├── assets/
│   └── css/
│       ├── components/_menu.scss # SCSS стили для меню
│       └── custom.scss      # основные SCSS стили
├── content/
│   ├── posts/               # статьи блога
│   │   ├── _index.md
│   │   ├── first-article.md
│   │   └── test-article.md
│   └── about/
│       ├── _index.md        # контент страницы "О компании"
│       └── testimonials.md  # данные для блока отзывов
├── layouts/
│   ├── _default/
│   │   ├── baseof.html      # общий каркас
│   │   ├── list.html        # список (карточки)
│   │   └── single.html      # одиночная статья
│   ├── about/
│   │   └── list.html        # шаблон страницы "О компании"
│   └── partials/
│       ├── article-card.html
│       ├── menu.html        # шаблон блока меню
│       ├── footer.html      # шаблон блока футер
│       ├── testimonial.html
│       └── testimonials-section.html
├── static/
│   └── images/              # изображения 
         └── articles/       # изображения для статей
├── hugo.toml                # конфигурация сайта, в котором задаются пункты меню, название сайта, путь к фавиконке и т д
└── README.md                # документация
```

## Конфигурация (hugo.toml)

```toml
baseURL = 'https://example.org/'
languageCode = 'en-us'
title = 'Сайт компании'

[menu]
  [[menu.main]]
    name = "Главная" # Название пункта меню
    url = "/"        # ссылка пункта меню
    weight = 1       # порядок пункта меню(1-й, 2-й и т д)
  [[menu.main]]
    name = "Статьи"
    url = "/posts/"
    weight = 2
  [[menu.main]]
    name = "О компании"
    url = "/about/"
    weight = 3

[params]
featured_image = '/images/posts.webp' # путь до картинки постов в на главной странице сайта
favicon = '/images/favicon.svg'  # путь до фавиконки 
```

### Что можно настроить в hugo.toml:

- **baseURL** —ваш продакшен-URL.
- **languageCode** — можно поменять в зависимости от языка сайта
- **[menu.main]** — пункты шапки (добавляются/сортируются по weight).
- **[params]** — значения по умолчанию для карточек/мета и путь к favicon.

## Компоненты и шаблоны

### Базовый шаблон: `_default/baseof.html`

- Подключает стили, собранные из `assets/css/custom.scss` (Hugo Pipes: `resources.Get | css.Sass | resources.Minify`).
- Вставляет партилы шапки (`partials/menu.html`) и подвала (`partials/footer.html`).
- Определяет блок `{{ block "main" . }}` для содержимого страниц/секций.

### Списки и страницы: `_default/list.html` и `_default/single.html`

- **list.html**: выводит сетку карточек статей.
  ```go
  {{ range .Pages }}
    {{ if not .Draft }}
      {{ partial "article-card.html" . }}
    {{ end }}
  {{ end }}
  ```
  Карточка рендерится из `.Params` текущей страницы.

- **single.html**: отображает заголовок, дату, featured_image (если задан), затем `.Content`.

### Страница «О компании»: `layouts/about/list.html`

- Читает контент из `/content/about/_index.md` (герой-блок, картинка/текст из `company_info`).
- Отдельно подхватывает отзывы:
  ```go
  {{ $t := .Site.GetPage "/about/testimonials" }}
  {{ if $t }}
    {{ partial "testimonials-section.html" $t.Params.testimonials }}
  {{ end }}
  ```
  То есть в партил передается объект `testimonials` из front matter файла `content/about/testimonials.md`.

### Партилы (partials)

- **partials/menu.html** — шапка и навигация. Источник пунктов меню — `[menu.main]` из `hugo.toml`. SCSS — `assets/css/components/_menu.scss`.

- **partials/article-card.html** — карточка статьи. Использует поля:
  - `.Params.title` (`.Title`)
  - `.Params.description` (или `site.Params.description` как fallback)
  - `.Params.featured_image` (или `site.Params.featured_image` как fallback)
  - `.Permalink`

- **partials/testimonial.html** — карточка отзыва. Поля элемента: `name`, `position`, `text`.

- **partials/testimonials-section.html** — секция отзывов с заголовком и гридом карточек.

- **partials/footer.html** — подвал с текущим годом и `site.Title`.

## Стили и сборка SCSS

### Точка входа: `assets/css/custom.scss`

Здесь:
- Импортируются компоненты (`@import 'components/menu';`).
- Определены переменные цветов, шрифтов, отступов и адаптивные медиазапросы.
`assets/css/components/_menu.scss`
- стили для меню, сюда же можно вынести стили для отдельных компонентов

### Сборка

Выполняется Hugo Extended автоматически по шаблону `baseof.html` (Pipes, минификация).

## Контент-менеджерам: добавление и правки

### Новая статья (карточка)

1. Создать файл в `content/posts/`:
   ```bash
   hugo new posts/my-article.md
   ```

2. Заполнить front matter (примеры на TOML/YAML — выбирайте единообразно):
   ```toml
   +++
   title = "Заголовок статьи"
   date = 2025-08-18
   description = "Краткое описание для карточки и превью"
   featured_image = "/images/articles/my-image.webp"
   draft = false
   +++
   ```
   Контент после разделителя — в Markdown.


#### Поля:
- **title** — заголовок.
- **date** — дата публикации.
- **description** — короткий текст под заголовком карточки.
- **featured_image** — путь к картинке (см. [Изображения](#изображения)).

### Новый отзыв

Открыть `content/about/testimonials.md` и добавить элемент в массив `testimonials.testimonials`:

```yaml
---
testimonials:
  title: "Что говорят о нас клиенты"
  testimonials:
    - name: "Имя Фамилия"
      position: "Должность, Компания"
      text: "Короткий текст отзыва."
    - name: "Имя 2"
      position: "Роль, Компания"
      text: "Еще один отзыв."
---
```

> **Обратите внимание:** структура двуслойная — корневой ключ `testimonials` содержит `title` и массив `testimonials`.

### Правка страницы «О компании»

Файл: `content/about/_index.md`.

```yaml
---
title: "О компании"
description: "Короткое описание раздела"
featured_image: "/images/about.webp"
company_info:
  image: "/images/office.webp"
  text: |
    Многострочный текст в markdown.
    Можно использовать **жирный**, списки и т.д.
---
```

- **title**, **description** формируют верхний герой-блок.
- **company_info.image** — иллюстрация в основной секции.
- **company_info.text** — поддерживает Markdown, разбивайте на короткие абзацы.

### Изображения

- **Расположение:** класть в `static/images/...`, затем использовать путь `/images/...` в контенте.
- **Формат:** предпочтительно WebP, для фонов можно JPEG высокого качества.
- **Именование:** `section/name.webp` (напр. `/images/articles/article-1.webp`).
- **Размер:** оптимизируйте до разумной ширины (1200–1600px для статейных превью).

## Для разработчиков

- **Новый пункт меню:** дописать `[[menu.main]]` в `hugo.toml` и задать `weight` для порядка.

- **Новая секция контента** (например, кейсы): создать `content/cases/`, `layouts/section/cases.html` (или использовать `_default/list.html`), при необходимости — свои партилы.

- **Новый партил/компонент:** добавить файл в `layouts/partials/` и подключить его из шаблонов (`{{ partial "my-partial.html" . }}`); стили — в `assets/css/...` и импортировать в `custom.scss`.

- **Archetypes:** в `archetypes/default.md` по умолчанию стоит `draft = true`. При необходимости создайте отдельный archetype для статей с базовым набором полей.

## Сборка и деплой

- **Dev:** `hugo server -D` — локально, автообновление.
- **Prod:** `hugo --minify` — результат в `public/`.

> **Важно:** Не храните `public/` в VCS

## Стандарты и договоренности

- Пути к картинкам — абсолютные (`/images/...`).
- Не править файлы в `resources/` и `public/` руками — это артефакты сборки.
- Именование SCSS переменных — на английском, BEM-подход в классах (как в текущих шаблонах).

## Список источников

По заданию — перечислить все использованные источники. 
В этом проекте сторонние темы и библиотеки не использовались. 
База — собственная разработка на основе опыта и внутренних проектов компании.

- **Внутренние проекты компании** (репозитории/макеты) — недоступны публично.
- **Официальная документация Hugo** — [https://gohugo.io/documentation/](https://gohugo.io/documentation/) (справочно).
- **Другие внешние материалы** — не использовались.