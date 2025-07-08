
# Django Templates — система шаблонов для рендеринга HTML

Django Templates — это встроенная система шаблонов, которая позволяет удобно генерировать HTML, подставляя переменные и управляющие конструкции. Она безопасна (автоматически экранирует переменные) и расширяема.

---

## Основы: как работает шаблон

1. **Контекст** — данные, передаваемые из `views.py` в шаблон.
2. **Шаблон** — HTML-файл с конструкциями типа `{{ }}` и `{% %}`.
3. **Рендеринг** — Django подставляет переменные и возвращает итоговый HTML.

---

## Пример

### views.py

```python
from django.shortcuts import render

def home(request):
    context = {
        "name": "Леонид",
        "tasks": ["Написать лекцию", "Залить на GitHub", "Поспать"]
    }
    return render(request, "home.html", context)
```

### home.html

```html
<h1>Привет, {{ name }}!</h1>

<ul>
  {% for task in tasks %}
    <li>{{ task }}</li>
  {% endfor %}
</ul>
```

---

## Синтаксис шаблонов

### Вывод переменной

```html
{{ username }}
```

### Цикл for

```html
{% for item in items %}
  <p>{{ item }}</p>
{% endfor %}
```

### Условные конструкции

```html
{% if user.is_authenticated %}
  Привет, {{ user.username }}!
{% else %}
  Войдите в систему.
{% endif %}
```

### Комментарии

```html
{# Это комментарий и он не попадет в HTML #}
```

---

## Подключение шаблонов

1. Убедитесь, что в `settings.py` указано:

```python
TEMPLATES = [
    {
        ...
        'DIRS': [BASE_DIR / "templates"],
        ...
    },
]
```

2. Папка `templates/` на уровне проекта или приложения.

---

## Наследование шаблонов

### base.html

```html
<!DOCTYPE html>
<html>
<head>
  <title>{% block title %}Мой сайт{% endblock %}</title>
</head>
<body>
  <div class="content">
    {% block content %}
    {% endblock %}
  </div>
</body>
</html>
```

### child.html

```html
{% extends "base.html" %}

{% block title %}Главная{% endblock %}

{% block content %}
  <h1>Добро пожаловать!</h1>
{% endblock %}
```

---

## Встроенные фильтры

```html
{{ name|upper }}
{{ date|date:"d.m.Y" }}
{{ items|length }}
```

| Фильтр     | Назначение                       |
|------------|-----------------------------------|
| `upper`    | Преобразовать в верхний регистр   |
| `lower`    | Преобразовать в нижний регистр    |
| `date`     | Отформатировать дату              |
| `length`   | Количество элементов              |
| `default`  | Значение по умолчанию             |

---

## Подключение статических файлов

1. В `settings.py`:

```python
STATIC_URL = "static/"
```

2. В шаблоне:

```html
{% load static %}

<link rel="stylesheet" href="{% static 'css/style.css' %}">
<img src="{% static 'images/logo.png' %}">
```

---

## Подключение включаемых шаблонов (include)

```html
{% include "navbar.html" %}
```

---

## HTML-формы и данные

### views.py

```python
def contact(request):
    return render(request, "contact.html", {"email": "info@example.com"})
```

### contact.html

```html
<form method="post">
  {% csrf_token %}
  <input name="message">
  <button type="submit">Отправить</button>
</form>
```

---

## Пример структуры проекта

```
myproject/
├── manage.py
├── myapp/
│   └── views.py
├── templates/
│   ├── base.html
│   └── home.html
├── static/
│   └── css/
│       └── style.css
```

---

## Локализация и Переводы (i18n)

Django поддерживает многоязычные шаблоны через механизм перевода.

### В шаблоне

```html
{% load i18n %}

<h1>{% trans "Добро пожаловать" %}</h1>
<p>{% blocktrans %}Привет, {{ name }}!{% endblocktrans %}</p>
```

### Команда для создания `.po` файлов

```bash
django-admin makemessages -l ru
```

Затем в файле `locale/ru/LC_MESSAGES/django.po`:

```po
msgid "Добро пожаловать"
msgstr "Welcome"
```

Компилируем:

```bash
django-admin compilemessages
```

### Активация локали

В `settings.py`:

```python
LANGUAGE_CODE = 'ru'
USE_I18N = True
LOCALE_PATHS = [BASE_DIR / "locale"]
```

Или динамически:

```python
from django.utils.translation import activate

activate('ru')
```

---

## Заключение

Система шаблонов Django — это мощный инструмент для генерации HTML с логикой, повторным использованием и поддержкой локализации. Используя наследование, статику, переводы и фильтры — можно создавать масштабируемые и международные веб-приложения.
