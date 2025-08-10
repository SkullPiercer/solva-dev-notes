В FastAPI можно использовать **любой** шаблонизатор, который вам удобен. Это значит, что при рендеринге HTML-страниц вы не ограничены каким-то конкретным инструментом.

Наиболее популярный выбор — **Jinja2**.  
Это тот же шаблонизатор, который активно используется во Flask, а также во множестве других веб-фреймворков и приложений.

Почему Jinja2 так популярен:

- Простая и понятная синтаксис.
    
- Поддержка переменных, циклов, условий прямо в HTML.
    
- Возможность переиспользовать шаблоны (наследование).
    
- Хорошо интегрируется с Python и FastAPI.

FastAPI работает поверх **Starlette**, а Starlette уже предоставляет готовые утилиты для настройки Jinja2 в вашем приложении — их можно использовать напрямую, без лишнего кода.

---

### Установка зависимостей

Перед началом убедитесь, что:

1. Вы создали и активировали **виртуальное окружение** Python (это изолирует зависимости проекта).
    
2. Установили пакет `jinja2`.
```
pip install jinja2
```

---
## Подключение и использование Jinja2 в FastAPI

Когда мы хотим отдавать пользователю не просто JSON, а полноценные HTML-страницы с динамическим содержимым, нам нужен **шаблонизатор**. В FastAPI это легко делается с помощью **Jinja2**.

---

### 1. Импорт и настройка Jinja2

FastAPI (через Starlette) предоставляет утилиту `Jinja2Templates`.  
Пошагово процесс выглядит так:

**Импортируем** `Jinja2Templates`:
```
from fastapi.templating import Jinja2Templates
```

**Создаём объект шаблонов**, указывая папку, где они будут храниться:
```
templates = Jinja2Templates(directory="templates")
```

Если у вас есть **статические файлы** (CSS, изображения, JS), подключите их с помощью:
```
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="static"), name="static")
```

Теперь всё, что лежит в папке `static`, будет доступно по адресу `/static/...`

---
**Пример использования шаблона**
```
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")

templates = Jinja2Templates(directory="templates")

@app.get("/items/{id}", response_class=HTMLResponse)
async def read_item(request: Request, id: str):
    return templates.TemplateResponse(
        name="item.html",           # Имя HTML-файла шаблона
        request=request,            # Объект запроса (обязательно)
        context={"id": id}          # Данные для подстановки в шаблон
    )

```

- **`response_class=HTMLResponse`** — говорит документации (Swagger UI), что эндпоинт возвращает HTML, а не JSON.
    
- **`context`** — это словарь с переменными, которые будут доступны в шаблоне.  
    Например, `{"id": 42}` позволит в HTML написать `{{ id }}` и получить `42`.
    
- Параметр **`request`** обязателен, чтобы Jinja2 мог корректно работать с `url_for()` и другими функциями.

`app.mount('/static', StaticFiles(directory='static'), name='static')` — это строчка, которая говорит FastAPI:

> “Подключи папку **static** к приложению, чтобы все её файлы были доступны по URL, начинающимся с `/static`”.

### Разбор по частям:

1. **`app.mount()`**  
    Этот метод “подвешивает” дополнительный обработчик (подприложение) к определённому URL-префиксу.  
    В данном случае — всё, что начинается с `/static`, будет обрабатываться не самим FastAPI, а специальным классом `StaticFiles`.

2. **`'/static'`**  
    Это **URL-префикс**, по которому будут доступны статические файлы.  
    Например, файл `static/styles.css` станет доступен по адресу:
```  
   http://127.0.0.1:8000/static/styles.css	
```  
 
3. **`StaticFiles(directory='static')`**  
    Это класс из `fastapi.staticfiles`, который отвечает за отдачу статических файлов (CSS, JS, изображения, PDF и т.д.).  
    Параметр `directory='static'` говорит, из какой **папки проекта** брать файлы.

4. **`name='static'`**  
    Это имя монтируемого ресурса. Оно важно, если вы хотите использовать `url_for()` в шаблонах:
  ```
   <link href="{{ url_for('static', path='/styles.css') }}" rel="stylesheet">
   ```

Здесь `'static'` — это именно то имя, которое вы указали в `name='static'`.

---

Шаблон `item.html`
В папке `templates` создадим файл `item.html`
```
<html>
<head>
    <title>Item Details</title>
    <link href="{{ url_for('static', path='/styles.css') }}" rel="stylesheet">
</head>
<body>
    <h1>
        <a href="{{ url_for('read_item', id=id) }}">
            Item ID: {{ id }}
        </a>
    </h1>
</body>
</html>
```

## Как это работает

- **`{{ id }}`** — подставляет значение переменной `id` из `context`.
    
- **`url_for('read_item', id=id)`** — динамически генерирует URL для эндпоинта `read_item`, передавая в него аргумент `id`.  
    Например, если `id=42`, то получится `/items/42`.
    
- **`url_for('static', path='/styles.css')`** — генерирует ссылку на статический файл `styles.css` из папки `static`.  
    В итоге браузер загрузит `/static/styles.css`.

## Что такое `url_for` в шаблоне

В Jinja2-шаблонах FastAPI предоставляет функцию `url_for()` — это тот же самый метод, что в Starlette, и он:

- Берёт **имя эндпоинта** (не путь!),
    
- Подставляет параметры,
    
- Возвращает **полный URL**.

---

## Откуда берётся `'read_item'`

Когда мы объявляем эндпоинт:
```
@app.get("/items/{id}", response_class=HTMLResponse)
async def read_item(request: Request, id: int):
    ...
```

— FastAPI автоматически регистрирует у него **имя**, совпадающее с именем функции:
```
read_item
```
Это имя мы потом и передаём в `url_for`

---
## Как подставляется `id`

Маршрут `/items/{id}` содержит **параметр пути** `{id}`.  
`url_for()` требует, чтобы мы передали значение для всех таких параметров, иначе он не сможет построить ссылку.
```
url_for('read_item', id=42)
```

Результат:
```
/items/42
```

---
# Пример приложения с базовым шаблоном и наследованием

## Структура проекта
```
project/
│
├── main.py
├── static/
│   └── styles.css
└── templates/
    ├── base.html
    ├── header.html
    ├── footer.html
    ├── index.html
    └── item.html
```

## Код FastAPI (`main.py`)
```
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates

app = FastAPI()

# Подключаем папку static
app.mount("/static", StaticFiles(directory="static"), name="static")

# Указываем папку с шаблонами
templates = Jinja2Templates(directory="templates")


@app.get("/", response_class=HTMLResponse)
async def home(request: Request):
    items = [
        {"id": 1, "name": "Первый товар"},
        {"id": 2, "name": "Второй товар"},
        {"id": 3, "name": "Третий товар"},
    ]
    return templates.TemplateResponse(
        name="index.html",
        request=request,
        context={"items": items}
    )


@app.get("/item/{id}", response_class=HTMLResponse)
async def read_item(request: Request, id: int):
    return templates.TemplateResponse(
        name="item.html",
        request=request,
        context={"id": id}
    )
```

## Шаблон `base.html` (основной макет)
```
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Мой сайт{% endblock %}</title>
    <link rel="stylesheet" href="{{ url_for('static', path='/styles.css') }}">
</head>
<body>

    {% include "header.html" %}

    <main>
        {% block content %}{% endblock %}
    </main>

    {% include "footer.html" %}

</body>
</html>
```

## header.html
```
<header>
    <h1>Мой магазин</h1>
    <nav>
        <a href="{{ url_for('home') }}">Главная</a>
    </nav>
</header>
```

## footer.html
```
<footer>
    <p>&copy; 2025 Мой сайт</p>
</footer>
```

## `index.html` (главная страница со ссылками)
```
{% extends "base.html" %}

{% block title %}Главная{% endblock %}

{% block content %}
<h2>Список товаров</h2>
<ul>
    {% for item in items %}
        <li>
            <a href="{{ url_for('read_item', id=item.id) }}">
                {{ item.name }}
            </a>
        </li>
    {% endfor %}
</ul>
{% endblock %}
```

## `item.html` (страница конкретного товара)
```
{% extends "base.html" %}

{% block title %}Товар {{ id }}{% endblock %}

{% block content %}
<h2>Детали товара</h2>
<p>Вы смотрите товар с ID: {{ id }}</p>
<a href="{{ url_for('home') }}">Назад к списку</a>
{% endblock %}
```

## Пример стилей `static/styles.css`
```
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 0;
}

header, footer {
    background: #333;
    color: white;
    padding: 10px;
}

nav a {
    color: white;
    margin-right: 15px;
    text-decoration: none;
}

main {
    padding: 20px;
}
```