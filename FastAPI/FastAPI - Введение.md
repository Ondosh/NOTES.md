#### FastAPI - фреймворк для создания API

Ключевые особенности:
- Асинхронность
- Автоматическая документация (http://127.0.0.1:8000/docs)
- Работа с HTTP-запросами: PUT, READ, POST, DELETE (тоже самое что CRUD)

#### Как создать типичный бэкэнд на FastAPI

##### 1. Установка

```bash
pip install fastapi uvicorn
```

##### 2. Код (`main.py`)

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello, world!"}

@app.get("/items/{item_id}")
def get_item(item_id: int):
    return {"item_id": item_id}
```

##### 3. Запуск в консоли

```bash
uvicorn main:app --reload
```

##### 4. Проверка

Откройте в браузере:

```
http://127.0.0.1:8000/docs
```

Готово — работающий API с автодокументацией (Swagger UI).

---

**Разбор:**

- `FastAPI()` — создаёт приложение
- `@app.get("/")` — декоратор, регистрирует функцию как обработчик GET-запроса на путь `/`
- `{item_id}` в пути + `item_id: int` в параметрах функции — FastAPI сам вытаскивает значение из URL и проверяет тип
- `--reload` — сервер перезапускается при изменении кода

Это минимум без базы данных и схем.
