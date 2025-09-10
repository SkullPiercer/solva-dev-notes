### Вывод полного списка книг на главной странице

Главная страница нашего проекта формируется с помощью view-функции `index()` в приложении `homepage`. Именно здесь мы будем запрашивать у базы данных все записи модели **Book**.

На SQL такой запрос выглядел бы так:
```
SELECT * 
FROM books;
```

Но мы используем Django ORM, поэтому запрос оформим на Python:
```
from django.shortcuts import render
from library.models import Book

def index(request):
    template_name = 'homepage/index.html'
    # Запросим все книги:
    book_list = Book.objects.all()
    # Передадим результат в контекст для шаблона:
    context = {'book_list': book_list}
    return render(request, template_name, context)

```

### Ленивая выборка
Если открыть главную страницу и посмотреть запросы через **Django Debug Toolbar**, то можно заметить, что никаких SQL-запросов не отправлено.

Это нормально: пока данные из `book_list` реально не используются в шаблоне или коде, Django не делает обращений к базе. Такой принцип называется _ленивыми запросами_: «не попросили — не дам».

Чтобы запрос выполнился, нужно вывести данные в HTML-шаблоне:
```
<h1>Список книг</h1>
{{ book_list }}
```

### Цикл по объектам

Дальше приведём данные в нормальный вид. Пройдемся циклом по QuerySet и выведем свойства каждой книги:
```
{% for book in book_list %}
  <ul>
    <li>ID: {{ book.id }}</li>
    <li>Название: {{ book.title }}</li>
    <li>Автор: {{ book.author }}</li>
    <li>Описание: {{ book.description }}</li>
    <li>Жанр: {{ book.genre }}</li>
  </ul>
{% endfor %}
```

Результатом будет HTML-список с информацией по каждой книге.

### Оптимизация с `.values()`
В QuerySet попадают все поля модели, включая те, что не нужны пользователю (например, `created_at` или `is_published`). Это лишняя нагрузка на сервер.

В SQL мы бы явно указали нужные столбцы:
```
SELECT id, title
FROM books;
```

В ORM то же самое делается через метод `.values()`:
```
# homepage/views.py
from django.shortcuts import render
from library.models import Book

def index(request):
    template_name = 'homepage/index.html'
    # Берём только нужные поля
    book_list = Book.objects.values('id', 'title')
    context = {'book_list': book_list}
    return render(request, template_name, context)
```

Теперь в QuerySet будут не объекты модели, а словари с парами ключ-значение:
```
<QuerySet [{'id': 1, 'title': 'Война и мир'}, {'id': 2, 'title': 'Преступление и наказание'} ...]>
```

И шаблон нужно немного изменить:
```
{% for book in book_list %}
  <ul>
    <li>ID: {{ book.id }}</li>
    <li>Название: {{ book.title }}</li>
  </ul>
{% endfor %}
```

Такой подход эффективнее: на страницу попадут только те данные, которые действительно нужны пользователю.

---
### Операторы фильтрации в Django ORM

В Django ORM есть специальные модификаторы, которые работают так же, как операторы в SQL, позволяя отбирать записи по заданным условиям. Перед модификатором всегда ставится двойное подчёркивание `__`.
```
|Задача сравнения|SQL-подобный оператор|Django ORM|
|---|---|---|
|Равно|`=`|`__exact`|
|Проверка на NULL|`IS NULL`|`__exact=None`|
|Больше|`>`|`__gt`|
|Больше или равно|`>=`|`__gte`|
|Меньше|`<`|`__lt`|
|Меньше или равно|`<=`|`__lte`|
|Поиск по фразе|`LIKE '%...'`|`__contains='...'`|
|Вхождение в список|`IN (..)`|`__in=[...]`|
|Проверка диапазона|`BETWEEN ... AND ...`|`__range=[..., ...]`|
```

Полный список можно найти в документации, но нам хватит этих, чтобы начать.

### Метод `.filter()`

Метод `.filter()` принимает условия выборки в виде аргументов `ключ=значение`. Внутри ключа мы указываем:
- имя поля модели,
    
- нужный модификатор,
    
- и значение для проверки.

Пример (SQL-запрос):
```
SELECT *
FROM library_book
WHERE title LIKE '%роман%';
```

Тот же запрос в Django ORM:
```
Book.objects.filter(title__contains='роман')
```

По умолчанию используется модификатор `__exact`. То есть:
```
Book.objects.filter(is_featured__exact=True)
# то же самое, что:
Book.objects.filter(is_featured=True)
```

### Пример во view-функции

Допустим, на главной странице сайта мы хотим отобразить только те книги, которые библиотекарь отметил как рекомендованные. Тогда запрос будет таким:
```
# homepage/views.py
from django.shortcuts import render
from library.models import Book

def index(request):
    template_name = 'homepage/index.html'
    book_list = (
        Book.objects.values('id', 'title', 'summary')
        .filter(is_featured=True)
    )
    context = {'book_list': book_list}
    return render(request, template_name, context)
```

Сгенерированный SQL будет выглядеть примерно так:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."summary"
FROM "library_book"
WHERE "library_book"."is_featured"
```

### Метод `.exclude()`

Иногда нужно убрать лишние записи. Для этого в Django ORM есть метод `.exclude()`, который работает как `WHERE NOT ...` в SQL.

Например, в модели `Book` есть флаг `is_available`, который показывает, доступна ли книга для выдачи. Если он равен `False`, книга не должна отображаться в списке:
```
def index(request):
    template_name = 'homepage/index.html'
    book_list = (
        Book.objects.values('id', 'title', 'summary')
        .exclude(is_available=False)
    )
    context = {'book_list': book_list}
    return render(request, template_name, context)
```

SQL-запрос будет примерно таким:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."summary"
FROM "library_book"
WHERE (NOT (NOT "library_book"."is_available"))
```

### Работа с датами

В проектах библиотек часто используется поле `published_date`, где хранится дата выхода книги.
```
class Book(models.Model):
    published_date = models.DateField('Дата публикации')
```

Задача: найти книги, изданные в определённом временном промежутке. Для этого используется модификатор `__range`.

Например, достанем книги, изданные с 1950 по 1960 год:
```
import datetime

def classics(request):
    start = datetime.date(1950, 1, 1)
    end = datetime.date(1960, 12, 31)
    Book.objects.filter(published_date__range=(start, end))
```

SQL-аналог:
```
WHERE published_date BETWEEN '1950-01-01' AND '1960-12-31';
```

Можно фильтровать не только по полной дате, но и по её частям:
```
# Книги, вышедшие 1 января любого года
Book.objects.filter(published_date__month=1, published_date__day=1)

# Все книги, изданные до 2000 года
Book.objects.filter(published_date__year__lt=2000)

# Все книги, вышедшие в первом квартале
Book.objects.filter(published_date__quarter=1)
```

Таким образом, с помощью фильтров Django ORM можно строить гибкие запросы к данным библиотеки.

---
### Объединение условий в ORM-запросах

При работе с Django ORM часто возникает необходимость наложить несколько условий одновременно. Самый простой способ — перечислить их внутри метода `.filter()`. Тогда в итоговый SQL-запрос условия будут объединены оператором `AND`.

Пример: пусть у нас есть модель **Book**, в которой указывается, опубликована ли книга (`is_published`) и должна ли она отображаться на главной странице сайта (`is_featured`). Если нужно выбрать только те книги, которые одновременно опубликованы **и** отмечены как «на главной», код будет таким:
```
# library/views.py
from django.shortcuts import render
from library.models import Book

def index(request):
    template = 'library/index.html'
    books = Book.objects.values(
        'id', 'title', 'description'
    ).filter(
        is_published=True, is_featured=True  # оба условия сразу
    )
    context = {'books': books}
    return render(request, template, context)
```

В SQL такой запрос превратится в:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."description"
FROM "library_book"
WHERE ("library_book"."is_featured" AND "library_book"."is_published");
```

То же самое можно записать иначе — несколько `.filter()` подряд или с использованием `Q`, но самый читаемый вариант остаётся через запятую.

### Q-объекты: сложные условия с логикой `NOT`, `AND`, `OR`

Когда обычного объединения условий мало, используют **Q-объекты**. Они позволяют строить более гибкие выражения, комбинируя их логическими операторами:

- `~` — отрицание (NOT),
    
- `&` — логическое И (AND),
    
- `|` — логическое ИЛИ (OR).

Например, выберем книги, которые опубликованы **и** при этом вынесены на главную:
```
from django.db.models import Q
from django.shortcuts import render
from library.models import Book

def index(request):
    template = 'library/index.html'
    books = Book.objects.values(
        'id', 'title', 'description'
    ).filter(
        Q(is_published=True) & Q(is_featured=True)
    )
    return render(request, template, {'books': books})
```

### Примеры запросов с логическими операторами

**AND**  
SQL-запрос, возвращающий книги, которые опубликованы и находятся на главной:
```
SELECT "library_book"."id"
FROM "library_book"
WHERE ("library_book"."is_featured" AND "library_book"."is_published");
```

В Django ORM это можно написать разными способами:
```
# Вариант 1: коротко
Book.objects.filter(is_published=True, is_featured=True)

# Вариант 2: через Q
Book.objects.filter(Q(is_published=True) & Q(is_featured=True))

# Вариант 3: цепочкой фильтров
Book.objects.filter(is_published=True).filter(is_featured=True)
```

**OR**  
Нужно выбрать книги, которые либо опубликованы, либо вынесены на главную:
```
SELECT "library_book"."id"
FROM "library_book"
WHERE ("library_book"."is_featured" OR "library_book"."is_published");
```

Django ORM:
```
# С использованием Q
Book.objects.filter(Q(is_published=True) | Q(is_featured=True))

# Или через объединение запросов (менее удобно):
Book.objects.filter(is_published=True) | Book.objects.filter(is_featured=True)
```

**NOT**  
Допустим, нам нужны только книги, которые опубликованы, но при этом **не отмечены как главные**:
```
SELECT "library_book"."id"
FROM "library_book"
WHERE ("library_book"."is_published" AND NOT "library_book"."is_featured");
```

Django ORM:
```
# Через отрицание в Q
Book.objects.filter(Q(is_published=True) & ~Q(is_featured=True))

# То же самое через exclude
Book.objects.filter(is_published=True).exclude(is_featured=True)
```

### Приоритет выполнения

Важно помнить:

1. Сначала обрабатывается `NOT`,
    
2. потом `AND`,
    
3. и только затем `OR`.

Чтобы избежать путаницы, условия часто заключают в скобки.

---
### Комбинированный запрос

Пусть нужно вывести книги, которые:

- опубликованы (`is_published=True`)
    
- и при этом либо вынесены на главную (`is_featured=True`),
    
- либо в названии содержат слово _«роман»_.

Django ORM-запрос:
```
from django.db.models import Q

Book.objects.values('id', 'title', 'description').filter(
    (Q(is_published=True) & Q(is_featured=True)) |
    (Q(is_published=True) & Q(title__contains='роман'))
)
```

SQL-версия будет выглядеть так:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."description"
FROM "library_book"
WHERE (("library_book"."is_published" AND "library_book"."is_featured")
       OR ("library_book"."is_published" AND "library_book"."title" LIKE '%роман%'));
```

Django автоматически добавит оператор `ESCAPE` в `LIKE`-запрос, даже если в нём нет спецсимволов — вручную этого указывать не требуется.

---
### Сортировка через `Meta` в модели

Если записи модели всегда должны возвращаться в одном и том же порядке, можно сразу указать правило сортировки прямо внутри модели. Для этого в классе модели используется вложенный класс `Meta`:
```
# library/models.py

class Book(models.Model):
    title = models.CharField(max_length=255)
    year = models.PositiveIntegerField()

    class Meta:
        ordering = ('title',)
```

Теперь каждый запрос к этой модели будет автоматически сортировать книги по названию в алфавитном порядке.

В сгенерированном SQL появится инструкция:
```
ORDER BY "library_book"."title" ASC
```

Если нужно отсортировать в обратном порядке (от новых к старым или от Я до А), достаточно поставить `-` перед полем:
```
class Meta:
    ordering = ('-year',)  
```

Что эквивалентно SQL:
```
ORDER BY "library_book"."year" DESC
```

Можно задать несколько критериев сортировки:
```
class Meta:
    ordering = ('author', 'title')
```

В таком случае сначала сортировка будет по автору, а при совпадении фамилии — по названию книги.

---
### Метод `.order_by()`
Необязательно задавать сортировку на уровне модели. Для конкретного запроса используется метод `.order_by()`:
```
Book.objects.order_by('year')
```

Если сортировка указана и в `Meta`, и в `.order_by()`, приоритет всегда у `.order_by()`.  
А если вызвать метод без аргументов:
```
Book.objects.order_by()
```

— сортировка из `Meta` будет отключена. Это полезно, когда порядок значений не имеет значения, и можно ускорить выполнение запроса.

Также `.order_by()` может принимать несколько аргументов:
```
Book.objects.order_by('author', '-year')
```

Сначала книги будут отсортированы по фамилии автора, а у одного и того же автора — от новых к старым.

---
### Практика: сортировка категорий
Предположим, у нас есть модель **Genre** (жанр книги):
```
class Genre(models.Model):
    title = models.CharField(max_length=128, verbose_name='Название')
    slug = models.SlugField(max_length=64, unique=True, verbose_name='Слаг')
    display_order = models.PositiveSmallIntegerField(
        default=100,
        verbose_name='Порядок отображения'
    )

    class Meta:
        verbose_name = 'жанр'
        verbose_name_plural = 'Жанры'

    def __str__(self):
        return self.title
```

Теперь во вьюхе можно отсортировать жанры по `display_order`, а при равных значениях — по алфавиту:
```
# library/views.py
from django.shortcuts import render
from library.models import Genre

def index(request):
    template_name = 'library/index.html'
    genres = Genre.objects.values(
        'id', 'display_order', 'title'
    ).order_by('display_order', 'title')
    return render(request, template_name, {'genres': genres})
```

SQL-запрос:
```
SELECT "library_genre"."id",
       "library_genre"."display_order",
       "library_genre"."title"
FROM "library_genre"
ORDER BY "library_genre"."display_order" ASC, "library_genre"."title" ASC
```

---
### Ограничение количества записей: `LIMIT` и `OFFSET`

Допустим, на главной странице нужно показать только несколько книг, а не весь список. Для этого используют **срезы** (`slice`) у QuerySet.

Например, выбрать книги с 2-й по 4-ю (3 записи всего):
```
books = Book.objects.values('id', 'title', 'year').filter(
    is_published=True, is_featured=True
).order_by('title')[1:4]
```

SQL-запрос будет таким:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."year"
FROM "library_book"
WHERE ("library_book"."is_featured" AND "library_book"."is_published")
ORDER BY "library_book"."title" ASC
LIMIT 3
OFFSET 1
```

- `LIMIT 3` — взять три записи,
    
- `OFFSET 1` — пропустить первую.

Результат всё так же вернётся в виде `QuerySet`, с которым можно работать в шаблоне:
```
{% for book in books %}
  <p>{{ book.title }} ({{ book.year }})</p>
{% endfor %}
```

---
### Метод get(): выбор одного объекта

В SQL, чтобы получить конкретную запись, обычно обращаются к её первичному ключу. Например, чтобы извлечь книгу с id = 1:
```
SELECT ...
FROM ...
WHERE id = 1;
```

В нашем приложении «Библиотека» вьюшка `book_detail()` должна принимать `pk` книги и передавать в шаблон её данные. Вместо `.filter()` здесь лучше использовать `.get()`:
```
# library/views.py
from django.shortcuts import render
from library.models import Book

def book_detail(request, pk):
    template_name = 'library/detail.html'
    book = Book.objects.get(pk=pk)  # ищем книгу по первичному ключу
    context = {'book': book}
    return render(request, template_name, context)
```

`Book.objects.get(pk=pk)` вернёт **не QuerySet**, а конкретный объект `Book`.  
Если открыть страницу по адресу `http://127.0.0.1:8000/books/1/`, в SQL это превратится в такой запрос:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."description",
       "library_book"."author_id",
       "library_book"."published_year"
FROM "library_book"
WHERE "library_book"."id" = '1'
LIMIT 21;
```

#### Почему в Python `pk`, а в SQL `id`?

Django использует псевдоним `pk` (primary key). В базе поле может называться `id`, `identifier` или как угодно — ORM сама подставит правильное имя.

#### Откуда берётся `LIMIT 21`?

Django автоматически ограничивает такие запросы 21 строкой. Логика простая: если вдруг совпадут несколько объектов, ORM всё равно должна выбросить ошибку, ведь `.get()` рассчитан на одну запись.

---
### Ошибки при .get()
Если по условию найдено несколько книг, ORM выдаст ошибку `MultipleObjectsReturned`.  
Например:
```
Book.objects.get(title__contains="История")
```

Если таких книг окажется семь — получите исключение с сообщением, что найдено более одного объекта.  
Если объектов больше 21 — Django даже не уточнит количество, а просто скажет «их больше 21».

---
### Безопасный вариант: get_object_or_404()
Если книга с указанным `pk` отсутствует, `.get()` вызовет исключение `DoesNotExist`. Для пользователя это будет некрасиво. Вместо этого можно использовать шорткат `get_object_or_404()`:
```
from django.shortcuts import get_object_or_404, render
from library.models import Book

def book_detail(request, pk):
    template_name = 'library/detail.html'
    book = get_object_or_404(Book, pk=pk)
    context = {'book': book}
    return render(request, template_name, context)
```

Теперь, если книги нет, пользователь увидит стандартную страницу ошибки 404.

---
### Передача только нужных полей
Функции `get_object_or_404()` можно передать не только модель, но и **QuerySet**. Это удобно, если нужны только определённые поля.

Например, для страницы книги требуется только название и описание:
```
def book_detail(request, pk):
    template_name = 'library/detail.html'
    book = get_object_or_404(
        Book.objects.values("title", "description"),
        pk=pk
    )
    context = {"book": book}
    return render(request, template_name, context)
```

SQL-запрос станет значительно короче:
```
SELECT "library_book"."title",
       "library_book"."description"
FROM "library_book"
WHERE "library_book"."id" = "1"
LIMIT 21;
```

---
### Получение списка: get_list_or_404()

Если нужно вернуть несколько объектов, используют `get_list_or_404()`.  
Если результат пуст — автоматически генерируется ошибка 404.

---

### Методы .first() и .last()

Чтобы взять первую или последнюю запись из набора, можно использовать `.first()` и `.last()`.
```
# первая опубликованная книга
Book.objects.filter(is_published=True).order_by("pk").first()

# последняя опубликованная книга
Book.objects.filter(is_published=True).order_by("pk").last()
```

Важно: применять только к **отсортированным QuerySet**, иначе порядок будет непредсказуемым.

---
### Итоговая задача
Сделайте страницу `/books/<pk>/`, на которой показываются:

- название книги,
    
- её описание.

Код вьюхи:
```
from django.shortcuts import get_object_or_404, render
from library.models import Book

def book_detail(request, pk):
    template_name = 'library/detail.html'
    book = get_object_or_404(
        Book.objects.values("title", "description").filter(is_published=True),
        pk=pk
    )
    context = {"book": book}
    return render(request, template_name, context)
```

А в шаблоне `detail.html`:
```
{% block content %}
  <h1>{{ book.title }}</h1>
  <p>{{ book.description }}</p>
{% endblock %}
```

Теперь при открытии страницы пользователь увидит детальную информацию о книге, а при отсутствии записи — получит корректную ошибку 404.

---
## Проблема лишних запросов
В Django ORM легко работать со связанными моделями. Можно в шаблоне обратиться к атрибуту, который указывает на другую модель, и сразу отобразить нужное поле через «точку».

Например, у нас есть модель `Book`, которая через `ForeignKey` связана с моделью `Author`. В списке книг мы хотим показывать имя автора.

Во view-функции получим список всех книг:
```
# homepage/views.py
from django.shortcuts import render
from library.models import Book

def index(request):
    template_name = "homepage/index.html"
    books = Book.objects.all()
    context = {"books": books}
    return render(request, template_name, context)
```

В шаблоне выведем название книги и имя автора:
```
{% for book in books %}
  <h3>{{ book.title }} (ID: {{ book.id }})</h3>
  <p>Автор: {{ book.author.name }}</p>
{% endfor %}
```

На первый взгляд всё работает, но если заглянуть в Django Debug Toolbar, окажется, что запросов к базе слишком много:

- один запрос получает список книг,
    
- затем для каждой книги отдельно идёт ещё один запрос за данными автора.

Если книг 10 — получится 11 запросов. А если книг тысяча? Такое решение перегружает базу.

---
## Оптимизация с помощью JOIN

Чтобы избежать «лавины» запросов, нужно сразу объединить таблицы через JOIN. Django позволяет сделать это двумя способами:

### 1. Метод `.values()`
Можно запросить не только поля модели `Book`, но и данные из связанной модели `Author`:
```
books = Book.objects.values("id", "title", "author__name")
```

Здесь синтаксис `author__name` означает: взять поле `name` из модели `Author`, связанной с книгой через внешний ключ.

В шаблоне обращаться нужно также с двойным подчёркиванием:
```
{% for book in books %}
  <h3>{{ book.title }} (ID: {{ book.id }})</h3>
  <p>Автор: {{ book.author__name }}</p>
{% endfor %}
```

SQL при этом будет уже один, с JOIN:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_author"."name"
FROM "library_book"
INNER JOIN "library_author"
    ON ("library_book"."author_id" = "library_author"."id");
```

Минус метода `.values()` — он возвращает список словарей, а не объектов модели.

---
### 2. Метод `.select_related()`

Альтернативный способ — использовать `.select_related()`:
```
books = Book.objects.select_related("author")
```

Теперь запрос объединяет таблицы `Book` и `Author`, но на выходе мы получаем привычный QuerySet с объектами. В шаблоне можно использовать точечную нотацию:
```
{% for book in books %}
  <h3>{{ book.title }} (ID: {{ book.id }})</h3>
  <p>Автор: {{ book.author.name }}</p>
  <p>Год публикации: {{ book.author.birth_year }}</p>
{% endfor %}
```

SQL-запрос:
```
SELECT "library_book"."id",
       "library_book"."title",
       "library_book"."author_id",
       "library_author"."id",
       "library_author"."name",
       "library_author"."birth_year"
FROM "library_book"
INNER JOIN "library_author"
    ON ("library_book"."author_id" = "library_author"."id");
```

## Фильтрация по связанным моделям
JOIN-запросы можно фильтровать по полям связанных таблиц. Например, хотим показывать только книги, у которых авторы ещё «активны» (поле `is_active=True` в таблице авторов):

Через `.values()`:
```
books = Book.objects.values("id", "title", "author__name").filter(
    author__is_active=True
)
```

Через `.select_related()`:
```
books = Book.objects.select_related("author").filter(
    author__is_active=True
)
```

SQL при этом:
```
SELECT ...
FROM "library_book"
INNER JOIN "library_author"
    ON ("library_book"."author_id" = "library_author"."id")
WHERE "library_author"."is_active" = True;
```
