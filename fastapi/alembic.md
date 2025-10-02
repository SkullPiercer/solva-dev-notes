# Миграции в FastAPI — используем Alembic

Создание и изменение таблиц «вручную» через SQLAlchemy (вызов `Base.metadata.drop_all()` + `Base.metadata.create_all()`) приводит к потере данных и неудобен при эволюции моделей. Для аккуратного обновления схемы базы без утраты данных применяют миграции — они делают «переход» базы из одного состояния в другое, сохраняя содержимое таблиц. Первая миграция обычно отвечает за создание начальных таблиц.

FastAPI сам по себе не предоставляет инструмент для миграций (в отличие от Django), зато SQLAlchemy имеет отличное решение — **Alembic**. Оно умеет генерировать и применять миграции, а также может автоматически выявлять многие изменения в моделях.

Название «Alembic» исторически связано с аппаратом для дистилляции — отсюда метафора: инструмент преобразует (перегоняет) структуру базы.

---
## Установка и инициализация

Установим Alembic в виртуальное окружение:
```
pip install alembic
```

Инициализируем конфигурацию Alembic в корне проекта. Поскольку мы используем асинхронный код, применяем async-шаблон:
```
alembic init --template async alembic
```

В результате будут созданы каталоги и файлы:
```
library_project/
├── alembic/
│   ├── versions/       # здесь будут файлы миграций
│   ├── env.py          # служебный скрипт запуска Alembic
│   ├── README
│   └── script.py.mako
└── alembic.ini         # главный конфиг Alembic
```

Обязательно добавьте созданные файлы/папки в систему контроля версий (например, в Git) — это нужно, чтобы остальные разработчики могли применять те же миграции на своих машинах.

---
## Не храните креды в `alembic.ini`

В `alembic.ini` по умолчанию есть опция `sqlalchemy.url`. Если вписать туда строку подключения, она попадёт в репозиторий вместе с файлом — а в ней могут быть пароли или другие чувствительные данные. Чтобы избежать этого, лучше заставить Alembic читать URL из окружения (`.env`) — тогда параметры останутся вне репозитория и можно будет иметь разные строки подключения на разных окружениях.

---
## Подключение `DATABASE_URL` из `.env`

Добавим в `alembic/env.py` загрузку `.env` и установим значение `sqlalchemy.url` из переменной окружения. Ниже — минимально необходимая вставка; поместите её в начало `alembic/env.py` (в том месте, где объявлен `config = context.config`):
```
# alembic/env.py (фрагмент)

import os
from logging.config import fileConfig

from dotenv import load_dotenv  # pip install python-dotenv
from sqlalchemy import engine_from_config, pool
from sqlalchemy.ext.asyncio import AsyncEngine
from alembic import context

# Загрузим переменные окружения из .env
# (python-dotenv найдёт .env в родительских каталогах)
load_dotenv('.env') # В альтернативу dotenv можно использовать нашу точнку доступа get_settings

config = context.config

# Перезапишем параметр sqlalchemy.url значением из .env,
# чтобы не хранить его в alembic.ini
config.set_main_option('sqlalchemy.url', os.environ['DATABASE_URL'])

# ...дальше остаётся обычный код env.py, необходимый для async-режима
```

---
### Автоматическое создание миграций

Процесс изменений в базе через миграции состоит из двух шагов:

1. **Генерация файла миграции** — это инструкция с изменениями для базы (команда: `alembic revision`).
    
2. **Применение миграции** — выполнение этих изменений (команда: `alembic upgrade`) или их откат (`alembic downgrade`).

Сейчас у нас база пустая, и первая миграция создаст таблицы для моделей, например **книги и авторы**.

---
### Автогенерация

Миграции можно писать вручную, но удобнее — автоматически.  
Для этого используется команда:
```
alembic revision --autogenerate -m "initial migration"
```

Ключ `--autogenerate` говорит Alembic сверить текущее состояние моделей с таблицами в базе и записать разницу в новый файл миграции.

У каждой миграции есть свой уникальный **Revision ID**, а Alembic хранит информацию о применённых миграциях в специальной таблице `alembic_version`.

*Важно*: Alembic не всегда понимает изменения корректно. Например, если переименовать таблицу или колонку, он может предложить «удалить старую и создать новую», что приведёт к потере данных. В таких случаях файл миграции нужно вручную поправить. Или если в модели есть ENUM поля, миграция может сформироваться некорректно.

Если разработчику нужно полный контроль, он может создать пустой файл миграции командой:
```
alembic revision -m "manual migration"
```

Но можно вручную поправить сгенерированную миграцию.

---
### Подключение MetaData

Чтобы автогенерация работала, Alembic должен знать о всех моделях проекта.  
Для этого в `env.py` указываем объект `Base.metadata`, куда SQLAlchemy собирает все описания таблиц. Чтобы не редактировать `env.py` каждый раз, когда появляется новая модель, создаём специальный модуль `base.py`.
```
# app/core/base.py

"""Импорты Base и всех моделей для Alembic."""
from app.core.db import Base  # noqa
from app.models.book import Book  # noqa
from app.models.author import Author  # noqa
from app.models.user import User  # noqa
```

Теперь в `env.py` достаточно писать только:
```
from app.core.base import Base
target_metadata = Base.metadata
```

Таким образом, все модели будут доступны Alembic автоматически.

---
## Пример содержимого автоматически созданного файла:
```
def upgrade():
    op.create_table(
        'books',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('title', sa.String(length=200), nullable=False),
    )

    op.create_table(
        'authors',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(length=150), nullable=False),
    )

def downgrade():
    op.drop_table('authors')
    op.drop_table('books')
```

Применить все ещё неисполненные миграции можно так:
```
alembic upgrade head
```

Так как у нас пока что всего одна миграция, в логе появится что-то вроде:
```
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.runtime.migration] Running upgrade  -> 466f1da3d4b1, First migration
```

Здесь видно, что сработала команда `upgrade`, указан идентификатор ревизии (Revision ID) и её описание.  
В базе теперь есть служебная таблица `alembic_version` с этим ID, а также новая таблица `book` с полями из модели.

---
## Откат миграций

Если результат оказался неверным, изменения легко отменить:
```
alembic downgrade base
```

После этого таблица `book` исчезнет, а `alembic_version` снова станет пустой.

---
## Полезные команды Alembic

По умолчанию имена файлов миграций начинаются с случайного `Revision ID`, поэтому порядок в папке `/versions` может быть неочевидным.

Чтобы посмотреть историю миграций, используется:
```
alembic history
```

Результат:
```
466f1da3d4b1 -> befcaa650c3f (head), Add description to Book
<base> -> 466f1da3d4b1, First migration
```

Для более подробного вида:
```
alembic history -v
```

Актуальную миграцию можно отобразить с помощью ключа `-i`:
```
alembic history -i
```

В этом случае напротив текущей будет пометка `(current)`.

---
## Настройка имён файлов миграций

Можно задать собственные `Revision ID` или включить дату/время в имя файла. Например:
```
file_template = %%(year)d_%%(month).2d_%%(day).2d-%%(rev)s_%%(slug)s
```

Теперь миграции будут называться так:
```
2025_10_02-466f1da3d4b1_add_description_to_book.py
```

---
## Гибкий запуск миграций

Команды upgrade/downgrade позволяют указывать конкретные ID миграций, а также применять изменения относительно текущего состояния:
```
alembic upgrade befcaa650c3f   # применить все до указанного ID
alembic downgrade 466f1da3d4b1 # откатиться до конкретного состояния
alembic upgrade +2             # выполнить две следующие миграции
alembic downgrade -1           # отменить последнюю миграцию
```
