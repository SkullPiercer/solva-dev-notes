# Django ORM — Работа с базой данных без SQL

Django ORM (Object-Relational Mapping) — это инструмент, встроенный в Django, который позволяет взаимодействовать с базой данных с помощью Python-кода, а не SQL-запросов. Он абстрагирует таблицы как Python-классы, а строки в этих таблицах — как объекты.

---

Представим, что мы пишем блог. У нас есть посты и комментарии. Обычно это выглядело бы в SQL так:

## Пример структуры таблиц в SQL и Django ORM

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    content TEXT
);

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INTEGER REFERENCES posts(id),
    text TEXT
);
```

В Django ORM та же структура будет описана как Python-классы:

```python
from django.db import models

class Post(models.Model):
    title = models.CharField(max_length=255)
    content = models.TextField()

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    text = models.TextField()
```

После создания моделей, Django сам создаёт SQL-таблицы и управляет ими через миграции.

---

## Основные типы полей

Django предоставляет множество типов полей для описания структуры таблиц:

| Тип поля        | Назначение                              |
|------------------|------------------------------------------|
| `CharField`      | Строка фиксированной длины               |
| `TextField`      | Длинный текст                            |
| `IntegerField`   | Целое число                              |
| `DateTimeField`  | Дата и время                             |
| `BooleanField`   | Истина/ложь                              |
| `ForeignKey`     | Связь с другой моделью (один ко многим)  |
| `ManyToManyField`| Много ко многим                          |


Пример модели с разными типами:

```python
class Article(models.Model):
    title = models.CharField(max_length=100)
    body = models.TextField()
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
```

---

## Миграции

Когда вы создаёте или меняете модели, вы должны применить изменения к базе данных. Django делает это через миграции:

```bash
python manage.py makemigrations
python manage.py migrate
```

Django отслеживает изменения моделей и обновляет структуру БД автоматически.

---

## Основные операции ORM

После настройки моделей можно работать с данными.

Создание, получение, обновление и удаление объектов

```python
post = Post.objects.create(title="Привет ORM", content="Это пример")
all_posts = Post.objects.all()
first_post = Post.objects.first()
filtered = Post.objects.filter(title__contains="ORM")

post = Post.objects.get(id=1)
post.title = "Обновленный заголовок"
post.save()

post = Post.objects.get(id=1)
post.delete()
```

---

## Связи между моделями

### ForeignKey (Один ко многим)

```python
comment = Comment.objects.create(post=post, text="Комментарий")
comments = post.comment_set.all()
```

### ManyToManyField (Многие ко многим)

```python
class Tag(models.Model):
    name = models.CharField(max_length=50)

class Article(models.Model):
    title = models.CharField(max_length=100)
    tags = models.ManyToManyField(Tag)
```

Пример использования:

```python
tag = Tag.objects.create(name="Django")
article = Article.objects.create(title="Про ORM")
article.tags.add(tag)
```

---

## Фильтрация и QuerySet

Django ORM позволяет гибко фильтровать данные:

```python
Post.objects.filter(title__startswith="Привет")
Post.objects.exclude(content__icontains="устарело")
Post.objects.filter(id__in=[1, 2, 3])
```

Также можно делать сортировки:

```python
Post.objects.order_by('-id')
```

И агрегаты:

```python
from django.db.models import Count
Post.objects.annotate(comment_count=Count('comment'))
```

---

## Жизненный цикл модели

В Django можно выполнять действия при сохранении модели:

```python
class Post(models.Model):
    title = models.CharField(max_length=255)

    def save(self, *args, **kwargs):
        self.title = self.title.capitalize()
        super().save(*args, **kwargs)
```

Также есть сигналы `pre_save`, `post_delete` для разделения логики.

---

## Админка

Django позволяет быстро подключить административную панель:

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

Теперь модель Post будет доступна по адресу `/admin`

---

## Пример структуры проекта

Представим, у нас проект blog, и в нём:

```
blog/
├── blog/
│   ├── models.py
│   ├── views.py
│   └── admin.py
├── manage.py
└── db.sqlite3
```

### models.py

```python
class Post(models.Model):
    title = models.CharField(max_length=255)
    content = models.TextField()

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE)
    text = models.TextField()
```

### views.py

```python
from .models import Post
from django.shortcuts import render

def index(request):
    posts = Post.objects.all()
    return render(request, "index.html", {"posts": posts})
```

---

## Что нужно уметь

- Создавать модели и миграции
- Работать с QuerySet: фильтровать, сортировать, аггрегировать
- Настраивать связи моделей
- Добавлять модели в админку
- Использовать методы модели
- Использовать `select_related` и `prefetch_related` для решения N+1 проблемы

---

## Заключение

Django ORM — мощный инструмент, позволяющий работать с базой данных на языке Python. Он экономит время и упрощает сопровождение кода. Правильное использование ORM делает приложение надёжным, читаемым и легко расширяемым.
