### Подключение базы данных

В Django-проектах настройка СУБД выполняется в файле **settings.py** через параметр `DATABASES`.

Пример (по умолчанию используется SQLite):
```
# library/settings.py

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',   # Движок для SQLite
        'NAME': BASE_DIR / 'db.sqlite3',          # Файл БД хранится рядом с manage.py
    }
}
```
Особенность Django в том, что при смене базы данных достаточно изменить настройки подключения в `settings.py`. В остальном проект продолжит работать без доработок.

---
### Создание приложения и структура

В проекте **library_project** заведём отдельное приложение для работы с книгами:
```
python manage.py startapp books
```

После этого появится директория:
```
books/
├── migrations/
│   └── __init__.py
├── __init__.py
├── admin.py
├── apps.py
├── models.py        # тут будут классы для таблиц БД
├── tests.py
└── views.py
```

Чтобы приложение заработало, нужно добавить `'books'` в список **INSTALLED_APPS** в `settings.py`.

---
### Первая модель

Таблицы в Django формируются через ORM. Процесс состоит из двух шагов:

1. Описываем структуру в виде Python-классов (моделей).
    
2. Генерируем миграции и применяем их (`makemigrations`, `migrate`).

Создадим модель для книг:
```
# books/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=128)
```

---
### Что произойдёт при миграции?

Django сгенерирует SQL-запрос:
```
CREATE TABLE "books_book" (
  "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
  "title" varchar(128) NOT NULL
);
```

- Имя таблицы = название приложения + название модели → `books_book`.
    
- Django автоматически создаёт поле **id** (целое число, автоинкремент, PRIMARY KEY).
    
- Поле **title** будет строкой длиной до 128 символов (`VARCHAR(128) NOT NULL`).

---
### Основные типы полей

При проектировании моделей можно использовать разные типы:

- `models.IntegerField()` → целое число (INTEGER)
    
- `models.FloatField()` → число с плавающей точкой (REAL)
    
- `models.BooleanField()` → булево значение True/False (BOOL)
    
- `models.CharField(max_length=N)` → строка ограниченной длины (VARCHAR)
    
- `models.TextField()` → длинный текст (TEXT)
    
- `models.DateField()` → дата (DATE)
    
- `models.DateTimeField()` → дата и время (DATETIME)
    
- `models.SlugField()` → «чистая» строка для URL (латиница, цифры, -, _)
    
- `models.ImageField()` → поле для изображений (например, обложка книги)

---
## Связи между моделями в Django ORM

В Django не только таблицы, но и их взаимосвязи описываются внутри моделей.  
В реляционных базах данных встречаются три основных типа связей:

- **Один к одному (1:1)** — каждой записи одной таблицы соответствует ровно одна запись другой.
    
- **Многие к одному (N:1)** — несколько записей одной таблицы могут быть связаны с одной записью другой.
    
- **Многие ко многим (N:M)** — у каждой записи обеих таблиц может быть несколько связанных элементов.

---
### Пример связи 1:1

В библиотеке может храниться как переводное, так и оригинальное название книги. Для этого создадим отдельную модель `OriginalTitle` и соединим её с `Book` связью «один к одному»:

```
# books/models.py
from django.db import models

class OriginalTitle(models.Model):
    title = models.CharField(max_length=128)

class Book(models.Model):
    title = models.CharField(max_length=128)
    original_title = models.OneToOneField(
        OriginalTitle,
        on_delete=models.CASCADE
    )
```

Django ORM автоматически добавляет к названию поля суффикс `_id`. В БД появится поле `original_title_id`, которое будет хранить внешний ключ.

Если при удалении оригинального названия мы хотим оставить книгу, то вместо `CASCADE` укажем:
```
original_title = models.OneToOneField(
    OriginalTitle,
    on_delete=models.SET_NULL,
    null=True
)
```

Вместо удаления книга сохранится, а ссылка обнулится (`NULL`).

---
### Пример связи N:1

Много книг могут принадлежать одному жанру. Для этого создадим модель `Genre`, а в `Book` добавим поле `ForeignKey`:
```
class Genre(models.Model):
    title = models.CharField(max_length=128)

class Book(models.Model):
    title = models.CharField(max_length=128)
    genre = models.ForeignKey(
        Genre,
        on_delete=models.CASCADE
    )
```

Здесь:

- Каждая книга ссылается на жанр.
    
- Если жанр удалить, связанные книги тоже будут удалены (`CASCADE`).
    
- Django автоматически создаст индекс по полю `genre_id` для ускорения поиска.

---
### Пример связи N:M

У книги может быть несколько авторов, а один автор может писать множество книг. Это классическая связь «многие ко многим».

#### Вариант 1. Явная промежуточная таблица
```
class Author(models.Model):
    full_name = models.CharField(max_length=128)

class Book(models.Model):
    title = models.CharField(max_length=128)

class AuthorBook(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
```

Таблица `AuthorBook` связывает книги и авторов.

#### Вариант 2. Упрощённый ManyToManyField
```
class Author(models.Model):
    full_name = models.CharField(max_length=128)

class Book(models.Model):
    title = models.CharField(max_length=128)
    authors = models.ManyToManyField(Author)
```

В этом случае Django сам создаст промежуточную таблицу.

#### Вариант 3. ManyToManyField с дополнительными полями

Иногда нужно хранить доп. информацию о связи — например, дату выхода книги в соавторстве.
```
class Author(models.Model):
    full_name = models.CharField(max_length=128)

class Book(models.Model):
    title = models.CharField(max_length=128)
    authors = models.ManyToManyField(Author, through='CoAuthoring')

class CoAuthoring(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    book = models.ForeignKey(Book, on_delete=models.CASCADE)
    date_joined = models.DateField()
    comment = models.CharField(max_length=300)
```

---
### related_name: удобный доступ к связям

Поле `ForeignKey` может быть дополнено параметром `related_name`, который задаёт имя для обратной связи.
```
class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.ForeignKey(
        Author,
        on_delete=models.CASCADE,
        related_name='books'
    )
```

Теперь к книгам автора можно обратиться так:
```
pushkin = Author.objects.get(name="Александр Пушкин")
pushkin_books = pushkin.books.all(
```

Без `related_name` пришлось бы писать `pushkin.book_set.all()`.

---
Итого:
- **OneToOneField** — уникальная связь (например, книга ↔ оригинальное название).
    
- **ForeignKey** — «многие к одному» (например, книга ↔ жанр).
    
- **ManyToManyField** — «многие ко многим» (например, книги ↔ авторы).


---
## Абстрактные модели и добавление временных меток

Во многих приложениях требуется отслеживать момент создания записи и время её изменения. Например, полезно хранить, когда книга была добавлена в каталог или когда сведения о ней редактировались в последний раз.

Для этого в каждую модель можно добавить два одинаковых поля:

- `created_at` — время создания записи;
    
- `modified_at` — время последнего изменения.

Простейшая реализация:
```
class Book(models.Model):
    title = models.CharField(max_length=128)
    created_at = models.DateTimeField(auto_now_add=True)
    modified_at = models.DateTimeField(auto_now=True)
```

Но если такие поля нужны в десятках моделей (например, и в `Author`, и в `Genre`, и в `Reader`), придётся постоянно копировать один и тот же код. Это нарушает принцип **DRY** («don’t repeat yourself»).

---
### Абстрактные модели

Чтобы избавиться от дублирования, в Django используют **абстрактные модели**.  
Это такие классы, которые сами таблицу в базе не создают, но могут передавать свои поля и поведение дочерним моделям.

Пример:
```
from django.db import models

class BaseModel(models.Model):
    """Абстрактная модель: добавляет даты создания и изменения"""
    created_at = models.DateTimeField(auto_now_add=True)
    modified_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True
```

Теперь достаточно унаследовать любую другую модель от `BaseModel`:
```
class Book(BaseModel):
    title = models.CharField(max_length=128)
```

После миграции Django создаст таблицу `library_book` с полями:
```
CREATE TABLE "library_book" (
    "id" integer NOT NULL PRIMARY KEY AUTOINCREMENT,
    "created_at" datetime NOT NULL,
    "modified_at" datetime NOT NULL,
    "title" varchar(128) NOT NULL
);
```

---
### Несколько абстрактных классов и множественное наследование

В Python можно наследоваться сразу от нескольких базовых классов, в том числе абстрактных. Это удобно, если у моделей есть разные повторяющиеся наборы полей.
```
class CommonInfoBaseModel(models.Model):
    name = models.CharField(max_length=128)
    description = models.TextField()

    class Meta:
        abstract = True
```

Теперь можно комбинировать:
```
# Книга: содержит даты + базовую информацию
class Book(BaseModel, CommonInfoBaseModel):
    pages = models.IntegerField()

# Жанр: тоже использует общее описание и временные метки
class Genre(BaseModel, CommonInfoBaseModel):
    popularity = models.IntegerField()

# Автор: наследует только от BaseModel
class Author(BaseModel):
    first_name = models.CharField(max_length=128)
    last_name = models.CharField(max_length=128)
```

Здесь:

- `Book` унаследует `created_at`, `modified_at`, `name`, `description` и собственное поле `pages`;
    
- `Genre` — аналогично, но с полем `popularity`;
    
- `Author` — только даты и свои ФИО.

---

### Наследование абстрактных моделей

Абстрактную модель можно унаследовать от другой абстрактной:
```
class BaseModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    modified_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True

class CommonInfoBaseModel(BaseModel):
    name = models.CharField(max_length=128)
    description = models.TextField()

    class Meta:
        abstract = True
```

В этом случае `CommonInfoBaseModel` расширяет `BaseModel` и передаёт наследникам полный набор полей.

Абстрактные модели — мощный инструмент для сокращения повторов и упрощения поддержки кода. Но использовать их стоит только тогда, когда реально есть дублирующийся код. Создание десятков абстрактных классов «про запас» — избыточно (YAGNI).

---
## Работа с миграциями в Django

Миграции — это обязательная часть любого Django-проекта, их нужно хранить в репозитории вместе с кодом.
### Этап 1. Создание файлов миграций

Если в проекте есть новые модели, для них нужно сгенерировать миграции:

`python manage.py makemigrations`

После выполнения появится сообщение о создании файла, например:

`Migrations for 'library':   library/migrations/0001_initial.py`

Внутри этого файла Django фиксирует операции, которые нужно выполнить в базе данных.

Название файла состоит из порядкового номера (`0001`) и описания (`initial`). Для последующих изменений Django создаст новые файлы с номерами `0002`, `0003` и так далее.

Миграцию можно «подсмотреть» в виде SQL-запроса:
```
python manage.py sqlmigrate library 0001
```

Эта команда покажет SQL-код, который Django применит к базе данных на основании миграции.

---
### Этап 2. Применение миграций

Чтобы выполнить миграции и создать таблицы, используем:
```
python manage.py migrate
```

Django применит:

- миграции нашего приложения `library` (например, таблицы для книг, авторов и жанров);
    
- встроенные миграции для служебных приложений (`auth`, `sessions`, `admin`, `contenttypes`).

Если всё прошло успешно, в консоли появятся отметки `OK`.

---

### Что происходит в итоге?

- В каталоге проекта появится файл базы данных `db.sqlite3`.
    
- Внутри него будут созданы таблицы для всех моделей приложения `library`, а также служебные таблицы для аутентификации, сессий и админки.
    
- Миграции (`library/migrations/0001_initial.py` и последующие) останутся в репозитории, чтобы их могли применить все участники команды.

---

### Важный момент

Файл самой базы данных (`db.sqlite3`) в Git добавлять не нужно. Его расширение (`.sqlite` или `.sqlite3`) стоит указать в `.gitignore`.

Таким образом, в репозитории хранятся **миграции**, а не сама база.

---

### Классическая последовательность команд при запуске Django-проекта

1. `git clone`
    
2. `python -m venv venv`
    
3. `source venv/bin/activate`
    
4. `pip install -r requirements.txt`
    
5. `python manage.py migrate`

Обрати внимание: команды `makemigrations` здесь нет. Если миграции уже сохранены в проекте, повторно их создавать не нужно — достаточно просто применить готовые.
