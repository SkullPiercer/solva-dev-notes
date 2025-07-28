### Использование `pytest` в проектах на Django

Хотя Django поставляется с собственным механизмом для написания тестов (на основе `unittest`), многие разработчики предпочитают использовать `pytest` — гибкую и лаконичную альтернативу. Для интеграции с Django существует плагин `pytest-django`, который значительно упрощает настройку и написание тестов.

#### Установка `pytest-django`

Чтобы начать пользоваться этим плагином, достаточно установить его через `pip`:
```
pip install pytest-django==4.5.2
```

#### Подключение к проекту Django

Для того чтобы `pytest` знал, какие настройки использовать, необходимо создать файл `pytest.ini` в корне проекта и указать в нём путь к Django-конфигурации (настройкам):
```
[pytest]
DJANGO_SETTINGS_MODULE = myapp.settings
```

### Работа с утверждениями и проверками

Плагин предоставляет ряд вспомогательных функций для проверки результатов — аналогов методов из `TestCase`, но в функциональном стиле. Например:
```
from pytest_django.asserts import assertRedirects

def test_redirection(client):
    response = client.get('/old-url/')
    assertRedirects(response, '/new-url/')
```

### Полезные фикстуры в `pytest-django`

Плагин предлагает готовые фикстуры — специальные объекты, которые можно использовать напрямую в тестах без дополнительного импорта.

#### `client`
Создаёт неавторизованного клиента, аналог `Client()` из `django.test`.
```
def test_public_endpoint(client):
    response = client.get('/')
    assert response.status_code == 200
```

#### `admin_client`
Фикстура для получения клиента, от имени администратора (superuser). Не нужно создавать пользователя и вручную авторизовывать — всё уже готово:
```
def test_admin_access(admin_client):
    response = admin_client.get('/admin-area/')
    assert response.status_code == 200
```

#### `admin_user`
Если вам нужен именно пользователь, а не клиент, используйте `admin_user`. Он создаётся автоматически и может быть использован как автор какого-либо объекта:
```
def test_post_ownership(admin_user):
    post = Post.objects.create(title="Secret", author=admin_user)
    assert post.author.username == 'admin'
```

#### `django_user_model`
Иногда нужно создать своего пользователя с кастомными полями или ролями. Фикстура `django_user_model` возвращает модель пользователя, определённую в настройках (`AUTH_USER_MODEL`), с которой можно работать как обычно:
```
def test_custom_user(client, django_user_model):
    user = django_user_model.objects.create_user(username='alex')
    client.force_login(user)
    response = client.get('/profile/')
    assert response.status_code == 200
```

#### `db`
Некоторые тесты напрямую взаимодействуют с базой данных. Чтобы явно указать на это, используется фикстура `db`. Однако чаще всего она вызывается автоматически другими фикстурами.

Если ни одна из базовых фикстур не используется, но тест всё же взаимодействует с БД, следует добавить маркер:
```
import pytest

@pytest.mark.django_db
def test_direct_db_access():
    assert User.objects.count() == 0
```
Тестовая база данных создаётся единожды перед запуском всех тестов. Каждый тест изолирован и выполняется в своей транзакции, что позволяет избежать «перетекания» данных между проверками.

## Проверка маршрутов Django с использованием Pytest

В этом блоке мы начнём писать автоматические тесты для проекта Django с помощью Pytest. В частности, мы сфокусируемся на проверке маршрутов (routes) — то есть URL-адресов и прав доступа к ним.

### Структура проекта и начальная настройка

Чтобы организовать тесты в рамках одного приложения, создаём в папке `notes` отдельный пакет `tests_pytest` с файлом `__init__.py`. В нём будем хранить все наши проверки.

Чтобы избежать выполнения тестов, написанных ранее с использованием `unittest`, укажем Pytest’у, где именно искать тестовые файлы. Для этого добавим в `pytest.ini` следующее:
```
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
testpaths = notes/tests_pytest
```
Теперь все тесты будут выполняться только из указанной директории.

### Проверка доступа к маршрутам

#### Анонимный пользователь

Начнём с того, что убедимся: некоторые страницы должны быть доступны даже без авторизации. Например, главная, регистрация, вход и выход.
```
import pytest
from django.urls import reverse
from http import HTTPStatus

@pytest.mark.parametrize("url_name", [
    'notes:home',
    'users:login',
    'users:logout',
    'users:signup',
])
def test_public_pages_accessible_for_guests(client, url_name):
    url = reverse(url_name)
    response = client.get(url)
    assert response.status_code == HTTPStatus.OK
```
#### Авторизованный пользователь

Проверим, что после входа в систему пользователь может открыть определённые страницы:

- список заметок (`notes:list`)
    
- форма добавления (`notes:add`)
    
- страница с уведомлением об успешном добавлении (`notes:success`)

```
@pytest.mark.parametrize("url_name", [
    'notes:list',
    'notes:add',
    'notes:success',
])
def test_access_for_logged_user(not_author_client, url_name):
    url = reverse(url_name)
    response = not_author_client.get(url)
    assert response.status_code == HTTPStatus.OK
```
Для этого используется фикстура `not_author_client` — клиент, в который «вошёл» обычный пользователь (не админ).

Настройка фикстур (в `conftest.py`)
```
import pytest
from django.test.client import Client
from notes.models import Note

@pytest.fixture
def author(django_user_model):
    return django_user_model.objects.create(username='author')

@pytest.fixture
def stranger(django_user_model):
    return django_user_model.objects.create(username='stranger')

@pytest.fixture
def author_client(author):
    client = Client()
    client.force_login(author)
    return client

@pytest.fixture
def stranger_client(stranger):
    client = Client()
    client.force_login(stranger)
    return client

@pytest.fixture
def test_note(author):
    return Note.objects.create(
        title="Sample",
        text="Some text",
        slug="sample-slug",
        author=author,
    )

@pytest.fixture
def note_slug(test_note):
    return (test_note.slug,)  # Кортеж
```

### Автор против чужого пользователя

Теперь проверим, что автору заметки доступны определённые страницы (`detail`, `edit`, `delete`), а остальным — нет.

Для этого напишем универсальный тест с двойной параметризацией:
```
import pytest
from http import HTTPStatus
from django.urls import reverse

@pytest.mark.parametrize("url_name", [
    'notes:detail',
    'notes:edit',
    'notes:delete',
])
@pytest.mark.parametrize("client_fixture, expected_status", [
    (pytest.lazy_fixture('author_client'), HTTPStatus.OK),
    (pytest.lazy_fixture('stranger_client'), HTTPStatus.NOT_FOUND),
])
def test_note_permissions(url_name, client_fixture, expected_status, test_note):
    url = reverse(url_name, args=(test_note.slug,))
    response = client_fixture.get(url)
    assert response.status_code == expected_status
```

### Проверка редиректов на логин

Если анонимный пользователь попытается зайти на защищённые маршруты — его должно перенаправить на форму входа.

Чтобы избежать дублирования `if`, воспользуемся фикстурой `note_slug`.
```
from pytest_django.asserts import assertRedirects

@pytest.mark.parametrize("url_name, args", [
    ('notes:detail', pytest.lazy_fixture('note_slug')),
    ('notes:edit', pytest.lazy_fixture('note_slug')),
    ('notes:delete', pytest.lazy_fixture('note_slug')),
    ('notes:add', None),
    ('notes:success', None),
    ('notes:list', None),
])
def test_redirects_for_guest_users(client, url_name, args):
    login_url = reverse('users:login')
    url = reverse(url_name, args=args)
    expected = f'{login_url}?next={url}'
    response = client.get(url)
    assertRedirects(response, expected)
```
## Тестирование содержимого страниц

После проверки корректной работы маршрутов важно удостовериться, что страницы проекта получают именно тот контент, который мы ожидаем.

### Что проверяем:

- Заметка доступна в списке на странице, если её автор — текущий пользователь.
    
- Заметки других пользователей не отображаются в списке.
    
- Страницы создания и редактирования заметок содержат форму.
    

Создаём новый файл `test_content.py` в папке `pytest_tests/` — все тесты контента будем писать там, дополняя его по ходу работы и регулярно выполняя запуск тестов.

1. Заметка отображается автору
```
from django.urls import reverse

def test_note_visible_for_author(note, author_client):
    url = reverse('notes:list')
    response = author_client.get(url)
    object_list = response.context['object_list']
    assert note in object_list
```

2. Заметка скрыта от другого пользователя
```
def test_note_hidden_from_other_user(note, not_author_client):
    url = reverse('notes:list')
    response = not_author_client.get(url)
    object_list = response.context['object_list']
    assert note not in object_list
```
### Объединение с параметризацией

Чтобы избежать дублирования, объединим два теста в один, используя параметризацию:
```
import pytest
from django.urls import reverse

@pytest.mark.parametrize(
    'client_fixture, should_see_note',
    [
        (pytest.lazy_fixture('author_client'), True),
        (pytest.lazy_fixture('not_author_client'), False),
    ]
)
def test_note_visibility_based_on_user(note, client_fixture, should_see_note):
    url = reverse('notes:list')
    response = client_fixture.get(url)
    object_list = response.context['object_list']
    assert (note in object_list) is should_see_note
```

### 3. Наличие формы на страницах

Далее — тестирование наличия формы на страницах создания и редактирования заметок.

#### По отдельности:
```
from notes.forms import NoteForm

def test_add_note_page_has_form(author_client):
    url = reverse('notes:add')
    response = author_client.get(url)
    assert 'form' in response.context
    assert isinstance(response.context['form'], NoteForm)


def test_edit_note_page_has_form(slug_for_args, author_client):
    url = reverse('notes:edit', args=slug_for_args)
    response = author_client.get(url)
    assert 'form' in response.context
    assert isinstance(response.context['form'], NoteForm)
```

### Объединение тестов с параметризацией

Несмотря на разные URL-параметры, тесты можно объединить, если аккуратно задать аргументы:
```
@pytest.mark.parametrize(
    'page_name, url_args',
    [
        ('notes:add', None),
        ('notes:edit', pytest.lazy_fixture('slug_for_args'))
    ]
)
def test_note_pages_provide_form(author_client, page_name, url_args):
    url = reverse(page_name, args=url_args)
    response = author_client.get(url)
    assert 'form' in response.context
    assert isinstance(response.context['form'], NoteForm)
```

## Тестирование логики приложения

Завершающая стадия тестирования проекта — проверка бизнес-логики. Здесь важно убедиться, что приложение корректно обрабатывает действия пользователей в разных ситуациях.

### Цели:

- Только авторизованный пользователь может создать заметку.
    
- Система запрещает создание нескольких заметок с одинаковым `slug`.
    
- Если `slug` не указан — он генерируется автоматически.
    
- Пользователь может редактировать и удалять только свои заметки.
    

---

## Подготовка: фикстура с данными формы

Добавим в `conftest.py` фикстуру `form_data`, возвращающую словарь с тестовыми значениями для создания заметки:
```
@pytest.fixture
def form_data():
    return {
        'title': 'Новый заголовок',
        'text': 'Новый текст',
        'slug': 'new-slug',
    }
```
Эти данные будут использоваться во множестве тестов.

## Тестирование создания заметок

### 1. Авторизованный пользователь может создать заметку
```
from pytest_django.asserts import assertRedirects
from django.urls import reverse
from notes.models import Note

def test_logged_in_user_can_add_note(author_client, author, form_data):
    url = reverse('notes:add')
    response = author_client.post(url, data=form_data)
    assertRedirects(response, reverse('notes:success'))

    assert Note.objects.count() == 1
    note = Note.objects.get()
    assert note.title == form_data['title']
    assert note.text == form_data['text']
    assert note.slug == form_data['slug']
    assert note.author == author
```

2. Анонимный пользователь не может создать заметку
```
import pytest
from pytest_django.asserts import assertRedirects

@pytest.mark.django_db
def test_anonymous_user_cannot_add_note(client, form_data):
    url = reverse('notes:add')
    response = client.post(url, data=form_data)
    login_url = reverse('users:login')
    expected_redirect = f'{login_url}?next={url}'

    assertRedirects(response, expected_redirect)
    assert Note.objects.count() == 0
```
## Проверка уникальности slug

Если дважды использовать один и тот же `slug`, должна возникнуть ошибка валидации формы.
```
from pytest_django.asserts import assertFormError
from notes.forms import WARNING

def test_slug_must_be_unique(author_client, note, form_data):
    form_data['slug'] = note.slug
    url = reverse('notes:add')
    response = author_client.post(url, data=form_data)

    assertFormError(response, 'form', 'slug', errors=(note.slug + WARNING))
    assert Note.objects.count() == 1
```

## Автоматическая генерация slug

Если поле `slug` оставить пустым, оно должно быть сгенерировано из заголовка.
```
from pytils.translit import slugify

def test_slug_autogenerates_if_missing(author_client, form_data):
    form_data.pop('slug')
    url = reverse('notes:add')
    response = author_client.post(url, data=form_data)

    assertRedirects(response, reverse('notes:success'))
    note = Note.objects.get()
    assert note.slug == slugify(form_data['title'])
```
## Редактирование заметок

### 1. Автор может редактировать свою заметку
```
def test_author_can_edit_note(author_client, form_data, note):
    url = reverse('notes:edit', args=(note.slug,))
    response = author_client.post(url, data=form_data)

    assertRedirects(response, reverse('notes:success'))

    note.refresh_from_db()
    assert note.title == form_data['title']
    assert note.text == form_data['text']
    assert note.slug == form_data['slug']
```

2. Другой пользователь не может редактировать чужую заметку
```
from http import HTTPStatus

def test_non_author_cannot_edit_note(not_author_client, form_data, note):
    url = reverse('notes:edit', args=(note.slug,))
    response = not_author_client.post(url, data=form_data)

    assert response.status_code == HTTPStatus.NOT_FOUND

    note_from_db = Note.objects.get(id=note.id)
    assert note_from_db.title == note.title
    assert note_from_db.text == note.text
    assert note_from_db.slug == note.slug
```

## Удаление заметок

### 1. Автор может удалить свою заметку
```
def test_author_can_delete_note(author_client, slug_for_args):
    url = reverse('notes:delete', args=slug_for_args)
    response = author_client.post(url)

    assertRedirects(response, reverse('notes:success'))
    assert Note.objects.count() == 0
```

### 2. Удаление чужой заметки недоступно
```
def test_non_author_cannot_delete_note(not_author_client, slug_for_args):
    url = reverse('notes:delete', args=slug_for_args)
    response = not_author_client.post(url)

    assert response.status_code == HTTPStatus.NOT_FOUND
    assert Note.objects.count() == 1
```
