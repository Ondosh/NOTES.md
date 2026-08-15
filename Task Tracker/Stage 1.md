
# Этап 1: Backend (FastAPI + PostgreSQL + SQLAlchemy) — конспект

Task Tracker — учебный fullstack-проект. Этап 1: рабочий CRUD API для задач, без пользователей.

Стек: Python, FastAPI, SQLAlchemy (ORM), PostgreSQL, Pydantic, python-dotenv, uvicorn.

---

## 1. Структура проекта

```
Task_Tracker_ed/
├── venv/              # виртуальное окружение
├── .env               # переменные окружения (пароли, секреты) — НЕ в git
├── .gitignore
└── core/
    ├── database.py     # подключение к БД
    ├── models.py       # ORM-модели (таблицы БД)
    ├── schemas.py      # Pydantic-схемы (формат данных в API)
    └── main.py         # приложение FastAPI, сами эндпоинты
```

**Важное разделение ответственности:**

- `models.py` — как данные **хранятся** в базе данных
- `schemas.py` — как данные **выглядят** в запросах/ответах API
- Это разные вещи! Модель — "внутреннее" представление, схема — "внешний" контракт.

---

## 2. `database.py` — подключение к БД

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from dotenv import load_dotenv
import os

load_dotenv()
DATABASE_URL = os.getenv("DATABASE_URL")

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Что тут происходит

| Элемент                         | Роль                                                                                      |
| ------------------------------- | ----------------------------------------------------------------------------------------- |
| `load_dotenv()` + `os.getenv()` | Читает переменные из `.env`, чтобы пароль не был захардкожен в коде                       |
| `engine`                        | Объект, который знает, как подключаться к PostgreSQL                                      |
| `SessionLocal`                  | Фабрика сессий — на каждый запрос создаётся новая сессия через неё                        |
| `Base`                          | Базовый класс, от которого наследуются все ORM-модели (таблицы)                           |
| `get_db()`                      | Функция-генератор, выдающая сессию БД и гарантированно закрывающая её после использования |

### `.env` файл

```
DATABASE_URL=postgresql://postgres:ПАРОЛЬ@localhost:5432/task_tracker
```

Важно: имя файла именно `.env` (без "db" или других префиксов) — `load_dotenv()` по умолчанию ищет файл с этим точным именем в текущей рабочей директории.

### `.gitignore`

```
venv/
.env
__pycache__/
```

Чтобы пароли и лишние файлы не попадали в публичный репозиторий на GitHub.

---

## 3. Новые концепции Python, разобранные на этом файле

### `yield` и генераторы

Обычная функция с `return` отдаёт результат один раз и завершается. Функция с `yield` — **генератор**: она "замораживается" на `yield`, отдав значение наружу, и может быть "разбужена" позже, чтобы продолжить выполнение с того же места.

```python
def simple_generator():
    print("Начало")
    yield 1
    print("Середина")
    yield 2
    print("Конец")
```

Генератор — это разновидность **итератора** (объекта с сохранённым состоянием, отдающего значения по одному), только удобный способ его создать без ручного написания класса с `__iter__`/`__next__`.

**Паттерн `try / yield / finally`:**

```python
def get_db():
    db = SessionLocal()
    try:
        yield db          # 1. отдать сессию наружу, "замереть"
    finally:
        db.close()        # 2. после использования — закрыть, ЧТО БЫ НИ СЛУЧИЛОСЬ
```

- До `yield` — код, выполняемый **перед** выдачей значения (создание сессии)
- `yield db` — сама выдача значения
- После `yield`, в `finally` — код, выполняемый **после** того, как значение использовали (закрытие сессии), причём `finally` сработает даже при ошибке внутри

FastAPI использует именно эту особенность через `Depends()` (см. ниже) — получает сессию через первый `yield`, а после ответа клиенту "будит" генератор для закрытия соединения.

### ORM (Object-Relational Mapping)

Способ работать с базой данных через обычные Python-объекты и классы, а не писать сырой SQL руками.

Без ORM:

```python
cursor.execute("INSERT INTO tasks (title) VALUES (%s)", (title,))
```

С ORM (SQLAlchemy сам генерирует SQL):

```python
new_task = models.Task(title="Купить молоко")
db.add(new_task)
db.commit()
```

**ORM-объект** — объект класса-модели (например, `models.Task`). Ведёт себя как обычный Python-объект (`task.title`, `task.id`), но SQLAlchemy отслеживает его состояние и знает, как сохранить/обновить/удалить в БД. У него также есть служебные атрибуты SQLAlchemy (например `_sa_instance_state`), не относящиеся к вашим полям — поэтому напрямую отдавать такой объект в JSON нельзя, нужна Pydantic-схема (см. `response_model` ниже).

---

## 4. `models.py` — ORM-модель таблицы

```python
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from database import Base

class Task(Base):
    __tablename__ = "tasks"

    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, nullable=False)
    description = Column(String, nullable=True)
    is_done = Column(Boolean, default=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

|Элемент|Значение|
|---|---|
|`class Task(Base)`|Наследование от `Base` — говорит SQLAlchemy "это описание таблицы"|
|`__tablename__`|Имя таблицы в БД (`tasks`, множественное число — конвенция)|
|`Column(...)`|Каждая обёрнутая переменная — колонка таблицы|
|`primary_key=True`|Главный уникальный ключ записи|
|`index=True`|Ускоряет поиск по этому полю|
|`nullable=False/True`|Обязательное / необязательное поле|
|`default=False`|Значение по умолчанию, если не передано|
|`server_default=func.now()`|Значение проставляется самой базой данных в момент вставки|

`Integer`, `String`, `Boolean`, `DateTime` — не обычные Python-типы, а объекты SQLAlchemy, транслируемые в типы столбцов PostgreSQL.

---

## 5. `schemas.py` — Pydantic-схемы для API

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class TaskCreate(BaseModel):
    title: str
    description: Optional[str] = None

class TaskUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    is_done: Optional[bool] = None

class TaskResponse(BaseModel):
    id: int
    title: str
    description: Optional[str]
    is_done: bool
    created_at: datetime

    class Config:
        from_attributes = True
```

**Зачем три разные схемы:**

- `TaskCreate` — что присылает клиент при создании. Без `id`/`created_at` — их генерирует сервер/БД.
- `TaskUpdate` — что присылает клиент при обновлении. Все поля опциональны — можно менять только часть.
- `TaskResponse` — что сервер отдаёт клиенту. Содержит все поля, включая сгенерированные.

`class Config: from_attributes = True` — позволяет Pydantic создавать объект схемы напрямую из ORM-объекта (читает через `task.title` и т.д., а не ждёт словарь).

---

## 6. `main.py` — приложение и эндпоинты

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from typing import List

from database import engine, get_db, Base
import models
import schemas

Base.metadata.create_all(bind=engine)

app = FastAPI()

@app.post("/tasks", response_model=schemas.TaskResponse)
def create_task(task: schemas.TaskCreate, db: Session = Depends(get_db)):
    new_task = models.Task(title=task.title, description=task.description)
    db.add(new_task)
    db.commit()
    db.refresh(new_task)
    return new_task

@app.get("/tasks", response_model=List[schemas.TaskResponse])
def get_tasks(db: Session = Depends(get_db)):
    return db.query(models.Task).all()

@app.get("/tasks/{task_id}", response_model=schemas.TaskResponse)
def get_task(task_id: int, db: Session = Depends(get_db)):
    task = db.query(models.Task).filter(models.Task.id == task_id).first()
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    return task

@app.put("/tasks/{task_id}", response_model=schemas.TaskResponse)
def update_task(task_id: int, task_update: schemas.TaskUpdate, db: Session = Depends(get_db)):
    task = db.query(models.Task).filter(models.Task.id == task_id).first()
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")

    update_data = task_update.model_dump(exclude_unset=True)
    for key, value in update_data.items():
        setattr(task, key, value)

    db.commit()
    db.refresh(task)
    return task

@app.delete("/tasks/{task_id}")
def delete_task(task_id: int, db: Session = Depends(get_db)):
    task = db.query(models.Task).filter(models.Task.id == task_id).first()
    if not task:
        raise HTTPException(status_code=404, detail="Task not found")
    db.delete(task)
    db.commit()
    return {"detail": "Task deleted"}
```

### `Base.metadata.create_all(bind=engine)`

При запуске приложения автоматически создаёт таблицы в PostgreSQL, если их ещё нет. В реальных проектах для этого используют миграции (Alembic), но для старта — ок.

### Декораторы `@app.post(...)`, `@app.get(...)` и т.д.

`@` — синтаксис **декоратора**: функция, которая "оборачивает" другую функцию, добавляя ей поведение без изменения её кода.

```python
@app.post("/tasks")
def create_task(...): ...
```

эквивалентно:

```python
def create_task(...): ...
create_task = app.post("/tasks")(create_task)
```

`app` — экземпляр `FastAPI()`. У него есть методы `.post()`, `.get()`, `.put()`, `.delete()` — по одному на HTTP-метод. `@app.post("/tasks")` регистрирует: "когда придёт POST-запрос на `/tasks` — вызови эту функцию". Это называется **роутингом**.

Простой пример декоратора вне FastAPI:

```python
def loud(func):
    def wrapper():
        print("Перед вызовом!")
        func()
        print("После вызова!")
    return wrapper

@loud
def say_hello():
    print("Привет")
# вывод: "Перед вызовом!" / "Привет" / "После вызова!"
```

### Аннотации типов — не просто подсказки, а инструкции для FastAPI

```python
def create_task(task: schemas.TaskCreate, db: Session = Depends(get_db)):
```

`task: schemas.TaskCreate` говорит FastAPI: провалидируй тело запроса по этой схеме. Если данные не соответствуют (нет обязательного поля, неверный тип) — FastAPI **сам** вернёт ошибку 422 ещё до выполнения тела функции.

```python
def get_task(task_id: int, ...):
```

`task_id: int` в пути (`/tasks/{task_id}`) — FastAPI сам преобразует часть URL в `int`, а если не получится (например, `/tasks/abc`) — сам вернёт ошибку.

### `Depends(get_db)` — dependency injection

```python
db: Session = Depends(get_db)
```

Говорит FastAPI: "прежде чем вызвать эту функцию, вызови `get_db()` и подставь результат в параметр `db`".

Последовательность при запросе:

1. Приходит запрос
2. FastAPI вызывает `get_db()`
3. Генератор доходит до `yield db` — отдаёт сессию, замирает
4. Сессия подставляется в параметр `db` вашего эндпоинта
5. Выполняется тело эндпоинта
6. После ответа — FastAPI "будит" генератор, тот доходит до `finally: db.close()`

Гарантирует: на каждый запрос — своя сессия БД, которая аккуратно закрывается всегда, даже при ошибке.

### `db.query(...)` — SQLAlchemy Query API

```python
db.query(models.Task).all()
```

→ `SELECT * FROM tasks;`

```python
db.query(models.Task).filter(models.Task.id == task_id).first()
```

→ `SELECT * FROM tasks WHERE id = ... LIMIT 1;`

- `.filter(...)` — условие `WHERE`
- `.first()` — первая подходящая запись или `None`
- `.all()` — список всех подходящих записей

Важный нюанс: `models.Task.id == task_id` — не обычное сравнение Python. `models.Task.id` — объект-колонка, а `==` для него перегружен так, чтобы возвращать объект-условие, который SQLAlchemy превращает в SQL, а не `True`/`False` сразу.

### `HTTPException` — управляемые ошибки

```python
if not task:
    raise HTTPException(status_code=404, detail="Task not found")
```

Без этого — обычная ошибка привела бы к невнятному 500 (Internal Server Error). `HTTPException` явно говорит: "это ожидаемая ситуация, верни клиенту именно этот код и сообщение". `404` — стандартный HTTP-код "не найдено".

### `response_model` — валидация и фильтрация ответа

```python
@app.post("/tasks", response_model=schemas.TaskResponse)
```

Функция возвращает ORM-объект (`new_task`), а не Pydantic-схему. FastAPI прогоняет его через `TaskResponse`:

- валидирует, что есть все нужные поля
- сериализует в чистый JSON строго по схеме
- **фильтрует** — даже если бы в модели было лишнее секретное поле, оно не попадёт в ответ, так как его нет в схеме

Без `response_model` (или с `None`) FastAPI не знает, как сериализовать произвольный ORM-объект (у него есть служебные атрибуты типа `_sa_instance_state`) — это, скорее всего, приведёт к ошибке. Также `response_model` — источник документации в Swagger UI.

Для списков используется `List[schemas.TaskResponse]` — `.all()` возвращает список ORM-объектов, и FastAPI должен провалидировать/сериализовать каждый элемент отдельно:

```python
@app.get("/tasks", response_model=List[schemas.TaskResponse])
```

### `model_dump(exclude_unset=True)` — частичное обновление

Pydantic-модель отличает поле, которое клиент **явно передал**, от поля, которое просто получило значение по умолчанию.

Схема:

```python
class TaskUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    is_done: Optional[bool] = None
```

Клиент прислал: `{"is_done": true}`

```python
task_update.model_dump()
# {"title": None, "description": None, "is_done": True}  ← ВСЕ поля

task_update.model_dump(exclude_unset=True)
# {"is_done": True}  ← только реально переданные
```

Без `exclude_unset=True` цикл `setattr` затёр бы `title` и `description` значением `None`, даже если клиент их не трогал. `exclude_unset=True` защищает от этого — обновляются только явно присланные поля.

### `setattr(task, key, value)`

Динамическая установка атрибута объекта, когда заранее неизвестно, какое именно поле нужно менять:

```python
for key, value in update_data.items():
    setattr(task, key, value)
```

Эквивалент `task.is_done = True`, но с именем поля как переменной (строкой), а не жёстко прописанным в коде.

### `db.add()`, `db.commit()`, `db.refresh()`

```python
db.add(new_task)      # запомнить: эту запись нужно сохранить (ничего ещё не отправлено в БД)
db.commit()            # реально отправить INSERT в PostgreSQL, подтвердить транзакцию
db.refresh(new_task)   # перечитать объект из БД — подтянуть id, created_at, сгенерированные базой
```

Без `db.refresh()` поля вроде `id` и `created_at` (генерируются базой — автоинкремент и `server_default=func.now()`) остались бы пустыми в Python-объекте после `commit()`, и клиент получил бы, например, `id: null`.

---

## 7. Запуск и проверка

```bash
uvicorn main:app --reload
```

`--reload` — автоматический перезапуск сервера при изменении кода (удобно для разработки).

Swagger UI (интерактивная документация, генерируется автоматически):

```
http://127.0.0.1:8000/docs
```

---

## 8. Разобранные проблемы окружения (Windows-специфика)

Отдельная секция — чтобы не забыть, как решались проблемы с окружением, вдруг пригодится снова.

- **Несколько версий Python в системе** → конфликт через PATH. Решение: `where python` показывает все найденные пути по приоритету; нужная версия должна стоять выше остальных в `Path` (Переменные среды → User variables и/или System variables).
- **App execution aliases** (Windows Store заглушки для `python`/`py`) могут перехватывать команду — проверяются и включаются в Settings → Apps → Advanced app settings → App execution aliases.
- **venv привязан к абсолютному пути** создания — при переносе папки проекта venv ломается (`Fatal error in launcher`). Решение: пересоздать `venv` на новом месте, переустановить пакеты.
- **`load_dotenv()`** ищет файл с точным именем `.env` в текущей рабочей директории — не `db.env`, не в родительской папке относительно того, откуда запущена команда.

---

## 9. Инцидент: `.env` случайно попал в git

Произошло, когда `.gitignore` ещё не подхватил `.env` до первого коммита.

**Что делать в таком случае:**

1. **Сразу сменить пароль** от БД, попавший в публичный репозиторий (`ALTER USER postgres WITH PASSWORD 'новый_пароль';` через `psql -U postgres`) и обновить его в `.env`. Это важнее любых технических шагов ниже — файл мог быть уже виден.
2. Убедиться, что в `.gitignore` есть строка `.env`.
3. Убрать файл из отслеживания git (сам файл на диске остаётся):
    
    ```bash
    git rm --cached .envgit commit -m "Remove .env from repo"
    git push  
    ```
    
1. Если репозиторий уже был запушен публично и коммитов мало — можно пересоздать историю целиком (`git checkout --orphan` + новый commit + `push -f`), либо для длинной истории — `git filter-repo --path .env --invert-paths`.

**Вывод на будущее:** создавать `.gitignore` (с `.env` внутри) **до** первого `git add`/`git commit`, а не после.

---

## Что дальше — Этап 2

Пользователи и авторизация:

- Модель `User` (email, hashed_password)
- `POST /register`, `POST /login` (JWT)
- Защита эндпоинтов `/tasks` — только для авторизованных
- Привязка задач к `user_id`