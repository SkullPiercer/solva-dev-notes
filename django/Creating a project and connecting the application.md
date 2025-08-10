## Что такое Django?

**Django** — это высокоуровневый фреймворк для веб-разработки на языке Python, созданный с целью **ускорить и упростить** создание сложных, безопасных и масштабируемых веб-приложений. Он следует философии **DRY** (Don't Repeat Yourself) и **«включён по умолчанию»** — то есть предлагает готовые решения "из коробки".

Фреймворк появился в 2005 году и изначально разрабатывался для нужд новостного портала. С тех пор стал одним из самых популярных инструментов для серверной разработки на Python.

## Основные особенности Django

- **Безопасность** — защита от SQL-инъекций, CSRF, XSS и других атак встроена по умолчанию.
    
- **Автоматическая админка** — мощная административная панель генерируется из моделей за считанные минуты.
    
- **Модульность** — Django построен на компонентах: ORM, шаблонизатор, маршрутизация, формы и т.д.
    
- **Скорость разработки** — создание MVP можно уложить в часы, а не недели.
    
-  **Хорошая документация и комьюнити** — один из самых документированных фреймворков в мире.
    

---

## Архитектура Django: MTV

Django следует паттерну **MTV**:

- **Model** – работа с базой данных через ORM
    
- **Template** – шаблоны (HTML) для отображения данных
    
- **View** – логика обработки запросов и подготовки ответов

По сути, это близко к MVC, просто названия немного другие.

---

## Где применяется Django?

Django подходит для **широкого круга веб-проектов** — от MVP и стартапов до крупных нагруженных систем:

### Примеры:

- Платформы онлайн-обучения (LMS)
    
- Новостные порталы и блоги
    
- Корпоративные CRM-системы
    
- Доски объявлений и каталоги
    
- REST API и backend под SPA или мобильные приложения
    
- Электронная коммерция (интернет-магазины)

Известные компании, использующие Django:  
Instagram (раньше), Disqus, Pinterest, Mozilla, NASA, Bitbucket

---

##  Когда лучше НЕ использовать Django

- Если нужен **легковесный API-only backend**, лучше взять FastAPI или Flask. Django будет избыточен.
    
- Если проект будет работать исключительно с **асинхронными задачами** — другие фреймворки могут подойти лучше.
    
- Если уже выбран стек Node.js или Go — нет смысла притягивать Django только ради «моды».

# Практика
Перейдем в рабочую директорию, создадим и активируем виртуальное окружение и выполним установку Django:
```
pip install Django
```

Для старта приложения используем следующую команду:
```
django-admin startproject <название проекта>
```

Наш проект будет посвящен ведению текстовых блогов, поэтому пропишем:
```
django-admin startproject blog
```

Автоматически создастся структура проекта
```
├── blog ← Корневая директория проекта
│├── blog ← Основной пакет конфигурации Django-проекта
││├── __init__.py ← Делает папку Python-пакетом, обычно пустой
││├── asgi.py ← Точка входа для ASGI-сервера (для асинхронных приложений, WebSocket)
││├── settings.py ← Главный конфигурационный файл проекта
││├── urls.py ← Главный роутинг проекта: подключение всех URL
││├── wsgi.py ← Точка входа для WSGI-сервера (используется при деплое)
│├── manage.py ← Скрипт для управления проектом: миграции, запуск сервера, создание суперпользователя и т.д.
```

Для проверки работоспособности проекта, в директории с файлом manage.py вводим команду:
```
python manage.py runserver
```

Мы должны увидеть следующий вывод:
```
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).

You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
August 06, 2025 - 14:31:30
Django version 5.2.4, using settings 'blog.settings' 
Starting development server at http://127.0.0.1:8000/
Quit the server with CTRL-BREAK.
```

Это значит что все сработало корректно и если мы перейдём на http://127.0.0.1:8000/ или localhost то увидим стартовую ракету джанго, которая сигнализирует о том, что все прошло успешно.

Так же в этом сообщении нас может смутить красный текст:
```
You have 18 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
Run 'python manage.py migrate' to apply them.
```

Когда ты запускаешь Django-проект в первый раз, он уже содержит встроенные приложения (как `admin`, `auth`, `contenttypes`, `sessions`). Эти приложения используют базы данных — у них есть свои таблицы (например, таблица пользователей, таблица сессий и т.д.).

**Но!** Эти таблицы ещё не созданы в твоей базе данных. Django хранит "инструкции по созданию таблиц" в виде **миграций** (это такие специальные файлы с шагами: "создай таблицу такую-то", "добавь поле такое-то").

И вот, при первом запуске Django говорит тебе:

> "У тебя есть 18 миграций, которые ещё не применены. Без них проект может не работать как надо. Прогони команду `python manage.py migrate`, чтобы создать нужные таблицы в базе."

Ниже он предлагает нам:
```
Run 'python manage.py migrate' to apply them.
```

Так и поступим, остановим наш сервер с помощью комбинации клавиш ctrl + c и напишем в терминал:
```
python manage.py migrate
```

Эта команда смотрит на все миграции (встроенные и твои) и применяет их — то есть **создаёт нужные таблицы в базе данных**, чтобы всё работало.

В ответ мы видим:
```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying auth.0012_alter_user_first_name_max_length... OK
  Applying sessions.0001_initial... OK
```

Этот ответ от Django — это **отчёт о том, что все миграции были успешно применены**, и теперь база данных готова к работе.

# Django app
Мы поставили основное приложение (грубо говоря конструктор), в который мы будем добавлять app's. App - это отдельный модуль, отвечающий за что-то конкретное: блог, магазин, комментарии, пользователи и т.д.

## Зачем создавать отдельные `app`

Чтобы **не лепить всё в одну кучу**, а **делить сайт на логичные части**. Так код:
- легче понимать и поддерживать;
    
- легче переиспользовать в других проектах;
    
- проще тестировать и разрабатывать независимо.

## Примеры из жизни

### Допустим, ты делаешь сайт университета:
```
|Функция сайта|App (приложение)|
|---|---|
|Работа со студентами|`students`|
|Работа с преподавателями|`teachers`|
|Управление расписанием|`schedule`|
|Авторизация пользователей|`users` или `accounts`|
```

Каждый `app` будет хранить:
- свои **модели** (таблицы в базе данных),
    
- **вьюхи** (что происходит при заходе на страницу),
    
- **шаблоны** (как выглядит страница),
    
- и даже свои **админки**, **url'ы** и т.п.

Создадим свой app:
```
python manage.py startapp posts
```

Это создаст папку `posts/` с таким содержимым:
```
├── posts
│   ├── __init__.py
│   ├── admin.py — регистрация моделей в админке
│   ├── apps.py — настройки приложения
│   ├── models.py — таблицы базы данных
│   ├── tests.py — тесты приложения
│   ├── views.py — логика обработки запросов
```
Файлы которые мы не планируем использовать можно удалить, например tests.py.
Исходя из этой логики мы можем и добавить свои файлы.
Добавьте в структуру приложения posts файл urls.py со следующим содержимым:
```
from django.urls import path

urlpatterns = [
    # Тут мы будем прописывать URL-адреса для приложения posts
]
```

На данный момент, django создал нам приложение, но чтобы полноценно вести с ним работу, нужно его подключить. Сделать это можно двумя способами.
Заходим в файл settings.py нашего головного приложения blog и находим переменную:
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```

Первый из вариантов, это просто добавить наше приложение по имени:
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'posts', <- новое приложение
]
```

Или мы можем указать непосредственно на конфиг приложения, которое создал сам django:
```
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'posts.apps.PostsConfig', <- новое приложение
]
```

