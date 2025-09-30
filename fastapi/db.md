**План действий:**

1. Установить драйвер для работы SQLite в асинхронном режиме.
    
2. Настроить файл `.env` и сохранить туда строку подключения к БД.
    
3. Подключить SQLAlchemy, создать асинхронный движок и сессии.
    
4. Подготовить базовый класс для моделей.
    
5. Создать первую модель базы данных.

---
#### 1) Установка драйвера

Для работы с SQLite в асинхронном режиме используется библиотека **aiosqlite**:
```
pip install aiosqlite
```

Или asyncpg для postgresql
```
pip install asyncpg
```
---
#### 2) Настройка переменных окружения

В файле `.env` пропишем параметры подключения к БД. Сам файл базы назовём **library.db** и разместим в корне проекта (рядом с `/app` и `/venv`). Путь будет относительным:
```
# .env
APP_TITLE=Электронная библиотека
DATABASE_URL=sqlite+aiosqlite:///./library.db
```

(вариант для постгрес)
```
APP_TITLE=Электронная библиотека
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/mydatabase
```

#### Конфигурация приложения
Для конфигурации приложения потребуется библиотека pydantic-settings
```
pip instal pydantic settings
```

Создадим файл `core/config.py` и опишем настройки:
```
# app/core/config.py
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    APP_TITLE: str = 'Default'
    DESCRIPTION: str = 'QWERTY'

    DATABASE_URL: str
  
    model_config = SettingsConfigDict(
        env_file='.env',
        extra='allow'
    )


@lru_cache
def get_settings() -> Settings:
    return Settings()

```

Тут мы с вами описываем поля необходимые для работы приложения. В документации такой подход описывают как **singleton-like**. Но на деле такой паттерн больше похож на **service-locator** с кешированием. Ведь мы все еще можем создать более одного экземпляра класса настроек, но при правильном использовании у нас будет одна единственная точка доступа.

---
### 3) #### Установка и настройка SQLAlchemy

Установим SQLAlchemy: 
```
pip install sqlalchemy
```

В каталоге `core` создаём файл `db.py`, где будет логика подключения к базе.

Структура проекта:
```
library_project/
├── app/
│   ├── api/
│   ├── core/
│   │   ├── config.py
│   │   └── db.py
│   ├── models/
│   ├── schemas/
│   └── main.py
├── venv/
└── .env
```

---
### Базовый класс и движок

Когда ты работаешь с **SQLAlchemy ORM**, ты описываешь таблицы через Python-классы.  
Чтобы SQLAlchemy понимал, что класс нужно связать с таблицей в БД, он должен **наследоваться от специального базового класса**.
```
# app/core/db.py
Base = declarative_base()
```

Движок отвечает за **подключение к базе данных** и работу с ней.  
Он хранит информацию:

- какой тип БД используется (PostgreSQL, SQLite, MySQL и т.д.),
    
- хост, порт, логин, пароль,
    
- драйвер для подключения.

Создаём асинхронный движок:
```
Base = declarative_base()
engine = create_async_engine(settings.database_url)
```

`sessionmaker` — это **фабрика**, которая создаёт новые сессии для работы с БД.

- Сессия (`Session` или `AsyncSession`) — это объект, через который мы:
    
    - выполняем SQL-запросы,
        
    - добавляем/удаляем объекты,
        
    - фиксируем изменения (`commit`),
        
    - откатываем транзакции (`rollback`).

То есть **сессия = «рабочее соединение с базой»**, обёрнутое в объект Python.

`AsyncSession`

В асинхронных приложениях (например, FastAPI с async-ручками) используется не обычная `Session`, а `AsyncSession`.  
Она позволяет писать асинхронный код:
```
Base = declarative_base()
engine = create_async_engine(settings.database_url)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession
)
```

#### Расширение базового класса

Чтобы в каждой таблице автоматически создавался идентификатор `id`, а имя таблицы соответствовало названию модели (в нижнем регистре), добавим промежуточный класс:
```
from sqlalchemy import Integer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base, declared_attr, sessionmaker, Mapped, mapped_column

from app.core.config import get_settings

settings = get_settings()


class PreBase:
    @declared_attr
    def __tablename__(cls) -> str:
        return cls.__name__.lower()

    id: Mapped[int] = mapped_column(Integer, primary_key=True)


Base = declarative_base(cls=PreBase)
engine = create_async_engine(settings.DATABASE_URL)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession)
```

---
#### 3) Первая модель — книга

Создадим каталог `models` и в нём файл `book.py`:
```
app/
├── core/
│   ├── config.py
│   └── db.py
├── models/
│   ├── author.py
│   ├── book.py
│   ├── user.py
│   └── comment.py
└── main.py
```

## Таблицы и модели

### Аннотации с `Mapped[...]`

SQLAlchemy 2.0 рекомендует писать типы так:
- `Mapped[str]` → колонка будет хранить строки.
    
- `Mapped[Optional[str]]` → колонка может хранить `str` или `NULL`.

Это удобно, потому что Python-аннотации совпадают с типами в базе.

---
### `mapped_column(...)`

Это **описание конкретной колонки**. Примеры:
- `nullable=False` → колонка обязательная (NOT NULL).
    
- `unique=True` → значение должно быть уникальным.
    
- `default=None` → если ничего не передали, будет `NULL`.

### Ассоциация для many-to-many (авторы ↔ книги)
```
# app/models/author.py
from typing import List
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import Table, Column, ForeignKey, Integer, String
from app.core.db import Base

# Ассоциативная таблица
author_book = Table(
    "author_book",
    Base.metadata,
    Column("author_id", ForeignKey("author.id"), primary_key=True),
    Column("book_id", ForeignKey("book.id"), primary_key=True),
)


class Author(Base):
    __tablename__ = "author"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)

    # связь many-to-many с книгами
    books: Mapped[List["Book"]] = relationship(
        back_populates="authors",
        secondary=author_book,
    )
```

```
# app/models/book.py
from typing import List
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import Integer, String
from app.core.db import Base
from app.models.author import author_book


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    title: Mapped[str] = mapped_column(String(150), unique=True, nullable=False)

    # связь many-to-many с авторами
    authors: Mapped[List["Author"]] = relationship(
        back_populates="books",
        secondary=author_book,
    )
```

---
Пользователи и комментарии (one-to-many)
```
# app/models/user.py
from typing import List
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import Integer, String
from app.core.db import Base


class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)

    # у пользователя может быть несколько комментариев
    comments: Mapped[List["Comment"]] = relationship(
        back_populates="user", cascade="all, delete-orphan"
    )
```

```
# app/models/comment.py
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy import Integer, String, ForeignKey
from app.core.db import Base


class Comment(Base):
    __tablename__ = "comment"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    text: Mapped[str] = mapped_column(String(300), nullable=False)

    # внешний ключ на пользователя
    user_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
    user: Mapped["User"] = relationship(back_populates="comments")

    # внешний ключ на книгу
    book_id: Mapped[int] = mapped_column(ForeignKey("book.id"))
    book: Mapped["Book"] = relationship(back_populates="comments")
```

И добавим связь в `Book`:
```
# дополнение в app/models/book.py
from typing import List
from app.models.comment import Comment

    comments: Mapped[List["Comment"]] = relationship(
        back_populates="book", cascade="all, delete-orphan"
    )
```

# Теперь все готово для создания миграций.
