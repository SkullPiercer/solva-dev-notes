В PostgreSQL для организации связей между таблицами используются внешние ключи (foreign keys). Они позволяют связать одну таблицу с другой, гарантируя целостность данных. Внешний ключ задаётся в подчинённой таблице (referencing table) и указывает на уникальное значение (обычно первичный ключ) в главной таблице (referenced table).

#### Синтаксис внешнего ключа

Внешний ключ можно задать:
1. На уровне столбца:
```
CustomerId INTEGER REFERENCES Customers(Id)
    ON DELETE CASCADE
    ON UPDATE RESTRICT
```
2. На уровне таблицы:
```
FOREIGN KEY (CustomerId) REFERENCES Customers(Id)
    ON DELETE CASCADE
    ON UPDATE RESTRICT
```
Оба варианта работают одинаково, выбор зависит от предпочтений разработчика и читаемости.

### Типы связей между таблицами
#### Один к одному (1:1)
Каждая запись в одной таблице соответствует ровно одной записи в другой.

**Пример:**
```
CREATE TABLE Users (
    Id SERIAL PRIMARY KEY,
    Email VARCHAR(100) UNIQUE
);

CREATE TABLE UserProfiles (
    Id INTEGER PRIMARY KEY REFERENCES Users(Id) ON DELETE CASCADE,
    Bio TEXT
);
```
Здесь у каждого пользователя может быть только один профиль, и наоборот. Связь реализуется через внешний ключ, совпадающий с первичным ключом таблицы Users.
#### Один ко многим (1:N)

Одна запись в главной таблице может быть связана с несколькими записями в подчинённой таблице.

**Пример:**
```
CREATE TABLE Authors (
    Id SERIAL PRIMARY KEY,
    Name VARCHAR(100)
);

CREATE TABLE Books (
    Id SERIAL PRIMARY KEY,
    Title VARCHAR(200),
    AuthorId INTEGER REFERENCES Authors(Id) ON DELETE SET NULL
);
```
Один автор может написать много книг, но каждая книга принадлежит только одному автору.
#### Многие ко многим (M:N)

Записи из одной таблицы могут быть связаны с несколькими записями из другой таблицы и наоборот. Для такой связи создаётся промежуточная таблица.

**Пример:**
```
CREATE TABLE Students (
    Id SERIAL PRIMARY KEY,
    Name VARCHAR(100)
);

CREATE TABLE Courses (
    Id SERIAL PRIMARY KEY,
    Title VARCHAR(100)
);

CREATE TABLE StudentCourses (
    StudentId INTEGER REFERENCES Students(Id) ON DELETE CASCADE,
    CourseId INTEGER REFERENCES Courses(Id) ON DELETE CASCADE,
    PRIMARY KEY (StudentId, CourseId)
);
```
Студент может записаться на несколько курсов, и каждый курс может содержать множество студентов.
### Поведение внешнего ключа при изменении данных

PostgreSQL позволяет настроить реакцию на удаление или обновление записей в главной таблице с помощью следующих опций:
- **CASCADE** — автоматически применяет изменения к связанным строкам.
    
- **RESTRICT** — запрещает изменение/удаление, если есть зависимости.
    
- **NO ACTION** — аналог RESTRICT, но проверка откладывается до конца транзакции.
    
- **SET NULL** — обнуляет внешний ключ, если запись в главной таблице удалена.
    
- **SET DEFAULT** — устанавливает значение по умолчанию, если оно задано.

Каскадное удаление:
```
CREATE TABLE Orders (
    Id SERIAL PRIMARY KEY,
    CustomerId INTEGER,
    Quantity INTEGER,
    FOREIGN KEY (CustomerId) REFERENCES Customers(Id) ON DELETE CASCADE
);
```
Если клиент удаляется, все его заказы также автоматически удаляются.
#### Установка NULL:
```
CREATE TABLE Orders (
    Id SERIAL PRIMARY KEY,
    CustomerId INTEGER,
    Quantity INTEGER,
    FOREIGN KEY (CustomerId) REFERENCES Customers(Id) ON DELETE SET NULL
);
```
В этом случае при удалении клиента поле `CustomerId` в таблице заказов становится `NULL`.
#### Значение по умолчанию:
```
CREATE TABLE Orders (
    Id SERIAL PRIMARY KEY,
    CustomerId INTEGER DEFAULT 1,
    Quantity INTEGER,
    FOREIGN KEY (CustomerId) REFERENCES Customers(Id) ON DELETE SET DEFAULT
);
```
Если заказ клиента удалён, его заказы будут указывать на клиента с `Id = 1`, если задано значение по умолчанию.