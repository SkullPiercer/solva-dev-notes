HTTP-запрос состоит из четырёх частей:

1. **Стартовая строка**
    
    - для запроса: указывается метод (`GET`, `POST` и т. д.), адрес и версия протокола;
        
    - для ответа: версия протокола и код состояния (`200 OK`, `404 Not Found`).
        
2. **Заголовки (headers)** — описывают запрос или тело (например, `Content-Type`).
    
3. **Пустая строка** — разделяет «метаинформацию» и тело.
    
4. **Тело запроса/ответа (body)** — содержит данные (JSON, форма, файл).

---

## Методы, которые используются в FastAPI

- **GET** — получить данные (без изменений).
    
- **POST** — создать объект или передать чувствительные данные.
    
- **PUT** — полная замена объекта.
    
- **PATCH** — частичное обновление.
    
- **DELETE** — удалить.
    
- **OPTIONS, HEAD, TRACE** — служебные.

> ⚠️ Ограничение: URL не должен превышать 2000 символов. Если данных больше (например, длинный список координат), то используется `POST`.

---

## Где хранится тело запроса

- JSON
    
- поля формы (`application/x-www-form-urlencoded`)
    
- файлы (`multipart/form-data`)

FastAPI рассматривает **JSON** как тело запроса, а формы и файлы вынесены в отдельные механизмы (`Form`, `File`).

---
## Работа с формами и файлами

Пример простого эндпоинта авторизации с загрузкой файла:
```
# auth_app.py

from fastapi import FastAPI, Form, File, UploadFile

app = FastAPI()

@app.post("/signin")
def signin(
    username: str = Form(...),
    password: str = Form(...),
    profile_pic: UploadFile = File(...)
):
    content = profile_pic.file.read().decode("utf-8", errors="ignore").splitlines()
    return {"username": username, "lines": content[:5]}  # показываем первые 5 строк
```

- `Form(...)` обрабатывает логин и пароль;
    
- `File(...)` позволяет прикрепить файл;
    
- Swagger автоматически изменит `Content-Type` на `multipart/form-data`.

---

## Приём JSON через Pydantic-модели

Для более сложных структур удобно использовать **модели Pydantic**.

Создадим модель для книги в библиотеке:
```
# library_app.py

from enum import Enum
from typing import Optional, Union

from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class BookFormat(str, Enum):
    PAPER = "Печатная версия"
    EBOOK = "Электронная книга"
    AUDIO = "Аудиокнига"

class Book(BaseModel):
    title: str
    authors: Union[str, list[str]]   # может быть одна строка или список
    year: Optional[int]
    in_stock: bool = True
    format: Optional[BookFormat]

@app.post("/books/add")
def add_book(book: Book) -> dict[str, str]:
    if isinstance(book.authors, list):
        authors = ", ".join(book.authors)
    else:
        authors = book.authors
    
    result = f"{book.title} — {authors}"
    if book.year:
        result += f", {book.year}"
    if book.format:
        result += f", {book.format.lower()}"
    if book.in_stock:
        result += " (в наличии)"
    return {"info": result}
```

### Что здесь происходит

- JSON из тела запроса автоматически валидируется и превращается в объект `Book`.
    
- `authors` принимает **или строку, или список строк** (гибкость для разных сценариев).
    
- Дополнительные параметры (`year`, `format`, `in_stock`) необязательные.
    
- Swagger покажет схему `Book` в разделе _Schemas_.

---
### Пример запроса
```
POST /books/add
Content-Type: application/json

{
  "title": "Преступление и наказание",
  "authors": ["Фёдор Достоевский"],
  "year": 1866,
  "in_stock": true,
  "format": "Печатная версия"
}
```

```
POST /books/add
Content-Type: application/json

{
  "title": "Преступление и наказание",
  "authors": ["Фёдор Достоевский"],
  "year": 1866,
  "in_stock": true,
  "format": "Печатная версия"
}
```

Ответ:
```
{
  "info": "Преступление и наказание — Фёдор Достоевский, 1866, печатная версия (в наличии)"
}
```

## Важные замечания
- В одном эндпоинте можно **комбинировать path-, query-параметры и Pydantic-модели**.
    
- Но **нельзя смешивать JSON и форму/файлы** в одном запросе (ограничение HTTP).
    
- Несколько моделей Pydantic в одном запросе — можно, FastAPI объединит их в один объект.

---
# Валидация в Pydantic

Файловая структура
```
fastapi_training/
├── main.py
├── schemas.py
└── venv/
```

schemas.py — модель и валидаторы (пример)
```
# schemas.py

import re
from enum import Enum
from typing import Optional, Union

from pydantic import BaseModel, Field, validator, root_validator


class MembershipTier(str, Enum):
    BASIC   = "Базовый"
    STUDENT = "Студенческий"
    PREMIUM = "Премиум"


class Member(BaseModel):
    """
    Pydantic-схема для данных участника библиотеки.
    Поддерживает либо одну фамилию (строку), либо несколько фамилий (список строк).
    """
    first_name: str = Field(
        ..., max_length=30,
        title="Имя участника",
        description="Можно вводить в любом регистре"
    )
    family_names: Union[str, list[str]] = Field(
        ..., max_length=60,
        description="Одна фамилия или список фамилий"
    )
    age: Optional[int] = Field(
        None, gt=5, le=120,
        description="Возраст участника (от 6 до 120 лет)"
    )
    is_librarian: bool = Field(
        False, alias="is-librarian",
        description="Флаг, указывающий, является ли участник сотрудником библиотеки"
    )
    membership: Optional[MembershipTier] = None

    class Config:
        title = "Схема участника библиотеки"
        # Общее минимальное ограничение длины для всех строковых полей
        min_anystr_length = 2
        # При использовании alias в JSON: allow_population_by_field_name управляет, 
        # разрешено ли заполнять модель по имени поля вместо alias
        allow_population_by_field_name = True

    # Валидатор для одного поля: имя не должно быть числом
    @validator("first_name")
    def first_name_cannot_be_digits(cls, v: str) -> str:
        if v.isnumeric():
            raise ValueError("Имя не может состоять только из цифр")
        return v

    # Корневой валидатор: не допускаем смешения кириллицы и латиницы между именем и фамилиями
    @root_validator(skip_on_failure=True)
    def check_alphabets_not_mixed(cls, values):
        first = values.get("first_name", "")
        family = values.get("family_names", "")
        # приводим фамилии к единой строке
        if isinstance(family, list):
            family_combined = "".join(family)
        else:
            family_combined = family

        combined = first + family_combined

        # Ищем буквы кириллицы и латиницы
        has_cyr = bool(re.search(r"[А-Яа-яЁё]", combined))
        has_lat = bool(re.search(r"[A-Za-z]", combined))

        if has_cyr and has_lat:
            raise ValueError("Пожалуйста, не смешивайте русские и латинские буквы в имени/фамилии")
        return values
```

- `MembershipTier` — строковый `Enum`, поэтому в Swagger будут понятные варианты.
    
- `Field(...)` задаёт ограничения/описания аналогично `Path`/`Query`.
    
- `Config.min_anystr_length` задаёт минимальную длину **для всех** строковых полей модели (в примере — 2). Это правило применяется при валидации, но иногда не полностью отображается в Swagger UI — тем не менее оно работает.
    
- `alias="is-librarian"` позволяет в JSON передавать поле с дефисом, а в коде использовать `is_librarian`.
    
- `@validator("first_name")` — простой проверяющий валидатор для одного поля.
    
- `@root_validator(skip_on_failure=True)` — проверяет зависимость между полями (здесь — запрет на смешение алфавитов). `skip_on_failure=True` предотвращает выполнение корневого валидатора, если отдельные полевые валидаторы уже провалились.

---
## main.py — как принимать модель в теле запроса
```
# main.py

from fastapi import FastAPI
from schemas import Member

app = FastAPI()

@app.post("/members/create", tags=["members"], summary="Создать участника")
def create_member(member: Member):
    # Пример обработки: приведение фамилий к строке
    if isinstance(member.family_names, list):
        fam = " ".join(member.family_names)
    else:
        fam = member.family_names

    result = f"{member.first_name} {fam}".title()
    if member.age:
        result += f", {member.age}"
    if member.membership:
        # Enum.value — текстовое значение (например, "Премиум")
        result += f", {member.membership.value.lower()}"
    if member.is_librarian:
        result += " (сотрудник)"
    return {"member_info": result}
```

Пример JSON-запроса:
```
{
  "first_name": "алексей",
  "family_names": ["Иванов", "Петров"],
  "age": 34,
  "is-librarian": false,
  "membership": "Студенческий"
}
```

Swagger автоматически покажет схему `Member` в разделе `Schemas`, и пользователю будут видны ограничения и возможные значения для `membership`.

---
## Несколько практических советов

- `Union[str, list[str]]` удобно, когда в реальности иногда приходит одна фамилия, а иногда — несколько. Pydantic корректно десериализует и строку, и список.
    
- Если у вас есть поля-псевдонимы (`alias`), при создании через python-объекты можно использовать либо alias, либо оригинальное имя поля (см. `allow_population_by_field_name` в `Config`).
    
- Порядок валидации: сначала выполняются валидаторы отдельных полей, затем `root_validator`. Если полевой валидатор упадёт, `root_validator` может не получить нужные значения — поэтому `skip_on_failure=True` полезен, чтобы не породить ошибку 500.
    
- Для сложных правил валидации (регулярки, cross-field rules) используйте `root_validator` и `re` — это гибко и прозрачно в ошибках.

---
# Примеры запросов в документации

Swagger генерирует базовые примеры автоматически, но они зачастую слишком формальные: строка `"string"`, число `99` и т.п. Из-за этого пользователям API приходится догадываться о возможных вариантах (например, передаётся ли фамилия всегда строкой или можно список).

Чтобы документация была наглядной, мы можем добавить **свои примеры** в схему модели или прямо в обработчик.

---
## Вариант 1. Один пример в `schema_extra`

В `schemas.py` в подклассе `Config` можно указать `schema_extra`. Туда помещается словарь, в котором ключ `example` содержит готовый JSON для Swagger:
```
# schemas.py

class Member(BaseModel):
    first_name: str
    family_names: Union[str, list[str]]
    age: Optional[int]
    is_librarian: bool = Field(False, alias="is-librarian")
    membership: Optional[MembershipTier]

    class Config:
        title = "Читатель библиотеки"
        schema_extra = {
            "example": {
                "first_name": "Анна",
                "family_names": ["Иванова", "Смирнова"],
                "age": 27,
                "is-librarian": False,
                "membership": "Студенческий"
            }
        }
```

Теперь в Swagger UI пример будет выглядеть осмысленно, а не как набор заглушек.

---
## Вариант 2. Несколько примеров прямо в `Body`

В обработчике запроса можно через `Body(..., examples={...})` перечислить разные варианты входных данных:
```
# main.py

from fastapi import FastAPI, Body
from schemas import Member

app = FastAPI()

@app.post("/members/register")
def register_member(
    member: Member = Body(
        ...,
        examples={
            "single_surname": {
                "summary": "Одна фамилия",
                "description": "Фамилия передаётся строкой",
                "value": {
                    "first_name": "Дмитрий",
                    "family_names": "Соколов",
                    "age": 30,
                    "is-librarian": False,
                    "membership": "Базовый"
                }
            },
            "multiple_surnames": {
                "summary": "Несколько фамилий",
                "description": "Фамилии можно передать списком",
                "value": {
                    "first_name": "Мария",
                    "family_names": ["Петрова", "Васильева"],
                    "age": 22,
                    "is-librarian": True,
                    "membership": "Премиум"
                }
            },
            "invalid": {
                "summary": "Некорректный пример",
                "description": "Возраст должен быть целым числом",
                "value": {
                    "first_name": "Алексей",
                    "family_names": ["Сидоров"],
                    "age": "forever young",
                    "is-librarian": False,
                    "membership": "Базовый"
                }
            }
        }
    )
):
    return {"msg": f"Зарегистрирован {member.first_name}"}
```

В Swagger появится выпадающий список, где можно выбрать любой из примеров.

---
## Вариант 3. Несколько примеров в `schemas.py`

Чтобы не захламлять сигнатуру функции длинным JSON, примеры можно описать один раз в `Config` модели:
```
# schemas.py

class Member(BaseModel):
    ...
    class Config:
        schema_extra = {
            "examples": {
                "single_surname": {
                    "summary": "Одна фамилия",
                    "value": {
                        "first_name": "Елена",
                        "family_names": "Орлова",
                        "age": 18,
                        "is-librarian": False,
                        "membership": "Студенческий"
                    }
                },
                "multiple_surnames": {
                    "summary": "Несколько фамилий",
                    "value": {
                        "first_name": "Сергей",
                        "family_names": ["Новиков", "Федоров"],
                        "age": 40,
                        "is-librarian": False,
                        "membership": "Премиум"
                    }
                }
            }
        }
```

А в `main.py` просто сослаться на них:
```
@app.post("/members/register")
def register_member(
    member: Member = Body(..., examples=Member.Config.schema_extra["examples"])
):
    return {"msg": f"Зарегистрирован {member.first_name}"}
```

Таким образом, можно наглядно показать пользователям API разные сценарии, которые заранее продумал разработчик: правильные, спорные и заведомо ошибочные запросы.

---
# Дополнительно

## 1. Формат ошибок (response_model и OpenAPI)

По умолчанию FastAPI сам генерирует описание ошибок, но часто нужно **задать единый формат ответа для ошибок**, чтобы документация выглядела аккуратно.

Пример схемы для ошибок:
```
# schemas.py
from pydantic import BaseModel, Field
from typing import Optional

class ErrorResponse(BaseModel):
    detail: str = Field(..., example="Некорректный запрос")
    code: Optional[int] = Field(None, example=400)
```

Использование в ручке:
```
# main.py
from fastapi import FastAPI, HTTPException
from schemas import ErrorResponse, Person

app = FastAPI()

@app.post(
    "/hello",
    responses={
        400: {"model": ErrorResponse, "description": "Ошибка валидации"},
        404: {"model": ErrorResponse, "description": "Не найдено"},
    },
)
def greetings(person: Person):
    if isinstance(person.surname, str) and len(person.surname) < 2:
        raise HTTPException(
            status_code=400,
            detail="Фамилия слишком короткая"
        )
    return {"message": f"Привет, {person.name}!"}
```

В Swagger теперь появятся **структурированные примеры ошибок**.

---
## 2. Преобразование данных из одной схемы в другую

Иногда нужно взять данные из одной схемы Pydantic и перегнать в другую (например, при переходе от DTO к модели ответа).
```
# schemas.py
from pydantic import BaseModel
from typing import Optional

class UserIn(BaseModel):
    username: str
    password: str

class UserOut(BaseModel):
    username: str
    is_active: bool = True

# main.py
@app.post("/user", response_model=UserOut)
def create_user(user: UserIn):
    # "перегоняем" UserIn -> UserOut через распаковку dict()
    user_out = UserOut(**user.dict(), is_active=True)
    return user_out
```

👉 В `Swagger` будет видно, что на входе `username + password`, а на выходе только `username + is_active`.

---
## 3. Вложенные схемы (одна схема параметром принимает другую)

Очень частый случай: одна модель включает другую. Например, у `Note` есть `Author`.
```
# schemas.py
from pydantic import BaseModel
from typing import List

class Author(BaseModel):
    name: str
    email: str

class Note(BaseModel):
    title: str
    text: str
    author: Author   # <-- вложенная схема
    tags: List[str] = []
```

Использование:
```
# main.py
@app.post("/notes", response_model=Note)
def create_note(note: Note):
    return note
```

Теперь в Swagger при создании заметки поле `author` будет отображаться как объект с полями `name` и `email`.

Пример запроса:
```
{
  "title": "Учеба FastAPI",
  "text": "Сегодня изучаем вложенные схемы",
  "author": {
    "name": "Александр",
    "email": "alex@example.com"
  },
  "tags": ["fastapi", "pydantic"]
}
```
