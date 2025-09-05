## Создание суперпользователя

В Django есть особая роль — **суперпользователь**. Такой аккаунт имеет полный доступ к админке и может управлять всеми объектами и пользователями.

Первого суперпользователя нельзя создать через веб-интерфейс — это делается только через консоль.

Активируй виртуальное окружение проекта и выполни команду:
```
python manage.py createsuperuser
```

Дальше появится пошаговый запрос:
```
Username (leave blank to use 'user'):   # логин (например, library_admin)
Email address:                          # почта
Password:                               # пароль (символы не видны)
Password (again):                       # повтор пароля
Superuser created successfully.
```

Теперь можно войти в админку под этой учётной записью.

Если пароль забылся — его можно сменить:
```
python manage.py changepassword <username>
```

либо создать нового суперпользователя той же командой `createsuperuser`.

---
### Проверка качества пароля

Django анализирует пароль с помощью валидаторов. Если пароль слишком простой, появится предупреждение:

- слишком похож на имя пользователя;
    
- короче 8 символов;
    
- слишком распространён;
    
- состоит только из цифр.

В этом случае можно либо ввести новый пароль, либо согласиться сохранить слабый (Django спросит `[y/N]`).

---

## Доступ в админ-зону

Запусти проект:
```
python manage.py runserver
```

Зайди по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/?utm_source=chatgpt.com) и авторизуйся как суперпользователь.

По умолчанию доступен только раздел **Authentication and Authorization** — управление пользователями и группами.

---
Никогда не создавай на реальном сервере суперпользователя с логином/паролем `admin/admin`. Такой пароль легко подбирается, и злоумышленник получит полный контроль над проектом.

---
## Подключение моделей к админке

По умолчанию админ-зона не отображает созданные модели. Чтобы они появились, нужно зарегистрировать их в `admin.py` конкретного приложения.

Например, добавим модель `Genre` (жанр книги) в админку:

**library/admin.py**
```
from django.contrib import admin
from .models import Genre

admin.site.register(Genre)
```
После сохранения и обновления страницы админки появится раздел **Genres**.

---
### Создание записи через админку

1. Перейди в раздел **Genres**.
    
2. Нажми **Add Genre**.
    
3. Заполни обязательные поля, например:
    
    - `title` — «Фантастика»
        
    - `slug` — «fantasy»
        
    - `order` — можно оставить по умолчанию.
        
4. Нажми **Save**.


После сохранения будет вызван метод:
```
Genre.objects.create(title="Фантастика", slug="fantasy", order=1)
```

И запись попадёт в таблицу `library_genre`.

---

### Поведение поля `id`

Каждая запись в таблице получает уникальный идентификатор (`id`). Если удалить жанр с id=1 и добавить новый, то новый объект получит id=2, а не 1. Django не переиспользует старые идентификаторы, а всегда назначает следующий по порядку.

---
## Локализация и перевод интерфейса

В Django есть встроенные инструменты для интернационализации — можно легко переключать интерфейс админки и стандартных страниц на разные языки.  
Чтобы сменить язык, нужно в файле `settings.py` изменить значение параметра `LANGUAGE_CODE` на код нужного языка:
```
# library/settings.py

# Включаем русский язык интерфейса
LANGUAGE_CODE = 'ru-RU'
```

После этого все встроенные надписи в админке и на стандартных страницах будут показаны по-русски. Также локализуются даты и время.

Никакого чуда тут нет — тексты переводов лежат в специальных файлах. При смене `LANGUAGE_CODE` Django подключает соответствующие ресурсы. Их можно найти в виртуальном окружении проекта:

- Windows: `venv\Lib\site-packages\django\contrib\admin\locale\ru\LC_MESSAGES`
    
- Linux/macOS: `venv/lib/python<версия>/site-packages/django/contrib/admin/locale/ru/LC_MESSAGES/`

Попробуйте переключить админку проекта «Библиотека» на русский язык и посмотрите, как изменятся страницы авторизации и разделы админки.

---

## Названия приложений и моделей

Переводы встроенных текстов Django уже есть, а вот названия приложений и моделей задаёт сам разработчик. Поэтому если мы хотим, чтобы админка выглядела понятно, их нужно переименовать вручную.

### Переводим название приложения

Чтобы в админке раздел приложения выглядел дружелюбнее, в файле `apps.py` можно задать свойство `verbose_name`:
```
# books/apps.py
from django.apps import AppConfig

class BooksConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'books'
    verbose_name = 'Каталог книг'
```

Теперь вместо «Books» админка будет показывать «Каталог книг».

---
### Переводим названия моделей

Каждая модель в админке отображается как отдельный раздел. Чтобы администратору было понятно, о чём речь, в классе `Meta` указывают человекочитаемые названия:
```
# books/models.py
from django.db import models

class Genre(models.Model):
    title = models.CharField(max_length=256)
    slug = models.SlugField(max_length=64, unique=True)

    class Meta:
        verbose_name = 'жанр'
        verbose_name_plural = 'Жанры'

    def __str__(self):
        return self.title
```

Аналогично стоит задать переводы для всех моделей:

- **Genre** → «Жанры»
    
- **Author** → «Авторы»
    
- **Book** → «Книги»
    
- **Publisher** → «Издательства»

---
### Переводим названия полей

По умолчанию Django выводит технические имена атрибутов моделей (например, `title`, `slug`). Чтобы они выглядели нормально, в аргументе `verbose_name` задают удобный текст:
```
# books/models.py
class Book(models.Model):
    title = models.CharField('Название', max_length=256)
    slug = models.SlugField('Слаг', max_length=64, unique=True)
    description = models.TextField('Описание')
    is_published = models.BooleanField('Опубликовано', default=True)
```

Для связей тоже можно указывать `verbose_name`:
```
class Book(models.Model):
    ...
    author = models.ForeignKey(
        'Author',
        on_delete=models.CASCADE,
        related_name='books',
        verbose_name='Автор'
    )
```

---
### Красивый вывод объектов

В админке объекты моделей по умолчанию отображаются как `Book object (5)`, что неудобно.  
Чтобы видеть название книги или имя автора, в модели переопределяют метод `__str__`:
```
class Author(models.Model):
    name = models.CharField('Имя', max_length=256)

    def __str__(self):
        return self.name
```
Теперь вместо «Author object (3)» админка покажет «Александр Пушкин».

То же самое стоит сделать для всех моделей — так работать с админкой будет гораздо приятнее.

---

После настройки локализации и перевода админка станет интуитивно понятной: разделы будут называться «Книги», «Авторы», «Жанры», «Издательства», а поля будут отображаться как «Название», «Описание», «Автор» и т.д.  
Администратору не придётся гадать, что скрывается за непонятными английскими словами или техническими идентификаторами.

---
# Настройка админ-зоны через `@admin.register`

По умолчанию Django отображает модели в админке с минимальными настройками. Чтобы сделать интерфейс удобнее, можно определить собственный класс, унаследованный от `admin.ModelAdmin`, и задать в нём нужные параметры.

После этого достаточно «подключить» этот класс к модели через декоратор `@admin.register`. Такой подход короче и нагляднее, чем использование `admin.site.register(...)`.

---
## Отображение всех нужных полей

По умолчанию в списке объектов показывается только строковое представление модели. Для модели **Book** удобнее видеть не только её название, но и описание, статус публикации, автора, жанр и издателя.
```
# books/admin.py
from django.contrib import admin
from .models import Book, Author, Genre, Publisher

@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = (
        'title',
        'description',
        'is_published',
        'author',
        'genre',
        'publisher',
    )
    list_editable = (
        'is_published',
        'author',
        'genre',
    )
    search_fields = ('title',)
    list_filter = ('genre',)
    list_display_links = ('title',)
```

- `list_display` — поля, отображаемые в таблице со списком книг.
    
- `list_editable` — поля, которые можно менять прямо из таблицы.
    
- `search_fields` — по каким полям будет работать поиск.
    
- `list_filter` — добавляет фильтры справа.
    
- `list_display_links` — задаёт, по каким полям можно перейти на страницу редактирования.
    

Теперь список книг в админке будет гораздо информативнее.

---
## Красивое отображение пустых полей

Если, например, у книги пока не указан издатель, в таблице отобразится дефис «-». Чтобы вместо дефиса показывался текст «Не указано», можно задать свойство:
```
@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    ...
    empty_value_display = 'Не указано'
```

А если хочется задать это правило сразу для всех моделей приложения:
```
# books/admin.py
admin.site.empty_value_display = 'Не указано'
```
---
## Удобный интерфейс для связей «многие ко многим»

У книги может быть несколько авторов. Django по умолчанию предлагает выбирать авторов через список с Ctrl/Command — неудобно, если их много.

Чтобы интерфейс стал удобнее, добавим `filter_horizontal`:
```
@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    ...
    filter_horizontal = ('authors',)
```

Теперь администратор сможет перетаскивать авторов между двумя окнами.

---
## Вставки (`Inline`) для связанных моделей

Предположим, мы хотим, чтобы на странице жанра отображались все книги этого жанра. Для этого в Django есть вставки (`Inline`).
```
class BookInline(admin.StackedInline):
    model = Book
    extra = 0

@admin.register(Genre)
class GenreAdmin(admin.ModelAdmin):
    inlines = (BookInline,)
    list_display = ('title',)
```

Теперь, открыв жанр в админке, можно прямо там управлять книгами, которые к нему относятся.  
Если заменить `admin.StackedInline` на `admin.TabularInline`, то книги будут отображаться компактнее — в таблице.

---
## Подсказки для полей (`help_text`)

Чтобы администратору было проще работать с формами, к каждому полю можно добавить подсказку.

Например, в модели издателя:
```
# books/models.py
class Publisher(models.Model):
    title = models.CharField(
        'Название',
        max_length=256,
        help_text='Введите уникальное название издательства (до 256 символов)'
    )

    class Meta:
        verbose_name = 'издательство'
        verbose_name_plural = 'Издательства'

    def __str__(self):
        return self.title
```

Теперь в форме под полем «Название» появится подсказка.

---
## Финальный вариант `admin.py`

В итоге админка для проекта «Библиотека» может выглядеть так:
```
# books/admin.py
from django.contrib import admin
from .models import Book, Author, Genre, Publisher

admin.site.empty_value_display = 'Не указано'


class BookInline(admin.StackedInline):
    model = Book
    extra = 0


@admin.register(Genre)
class GenreAdmin(admin.ModelAdmin):
    inlines = (BookInline,)
    list_display = ('title',)


@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = (
        'title',
        'description',
        'is_published',
        'author',
        'genre',
        'publisher',
    )
    list_editable = (
        'is_published',
        'author',
        'genre',
    )
    search_fields = ('title',)
    list_filter = ('is_published',)
    list_display_links = ('title',)
    filter_horizontal = ('authors',)


@admin.register(Author)
class AuthorAdmin(admin.ModelAdmin):
    list_display = ('name', 'birth_date')


@admin.register(Publisher)
class PublisherAdmin(admin.ModelAdmin):
    list_display = ('title',)
```

Таким образом, админ-зона из скучного интерфейса превратилась в удобный инструмент: список книг информативный, связанные записи легко редактировать, а пустые поля отображаются понятно.
