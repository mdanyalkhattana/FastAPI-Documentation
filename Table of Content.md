 
# **FastAPI Complete Notes**
#Table of Content
---

1. Introduction & Setup
2. Path Operations & Pydantic
3. Exception Handling
4. SQLAlchemy + CRUD + Alembic
5. Dependency Injection
6. Authentication & Authorization (JWT, OAuth2, Roles)
7. File Handling
8. Async Programming
9. Project Structure
10. Testing
11. Deployment (Uvicorn, Gunicorn, Docker, Nginx, Cloud)
12. Advanced Concepts (WebSockets, Streaming, Background Tasks, Rate Limiting, Caching)
 

## **Topic 1 — Introduction to FastAPI**

### What is FastAPI?

* Modern, fast (high-performance) Python framework for APIs (Python 3.7+).
* Built on **Starlette** (ASGI framework) + **Pydantic** (data validation).
* Supports **async programming**.

### Features

* Fast performance (Node.js/Go comparable)
* Automatic docs: `/docs` (Swagger), `/redoc`
* Type hints & validation
* Dependency injection
* Security support (OAuth2/JWT)
* Async-ready

### Why FastAPI?

* Quick, clean API development
* Auto-generated documentation
* Ideal for production-grade microservices

---

## **Topic 2 — Installation & Setup**

```bash
pip install fastapi
pip install "uvicorn[standard]"
```

* Uvicorn → ASGI server to run FastAPI

### Folder Setup (Small Project)

```
project/
│── main.py
│── requirements.txt
```

### First FastAPI App

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def home():
    return {"message": "Hello FastAPI"}
```

---

## **Topic 3 — Path Operations**

### HTTP Methods

| Method | Use Case       |
| ------ | -------------- |
| GET    | Retrieve data  |
| POST   | Create data    |
| PUT    | Update data    |
| DELETE | Delete data    |
| PATCH  | Partial update |

### Path Parameters

```python
@app.get("/users/{user_id}")
def get_user(user_id: int):
    return {"user_id": user_id}
```

### Query Parameters

```python
@app.get("/items/")
def get_items(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}
```

### Request Body (Pydantic)

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float

@app.post("/items/")
def create_item(item: Item):
    return {"item_name": item.name, "item_price": item.price}
```

### Response Model

```python
@app.get("/items/{item_id}", response_model=Item)
def read_item(item_id: int):
    return {"name": "Laptop", "price": 1500}
```

---

## **Topic 4 — Pydantic (Data Validation)**

### Basic Example

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int
    email: str

@app.post("/users/")
def create_user(user: User):
    return {"user": user}
```

### Field Validation

```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    name: str = Field(..., min_length=2, max_length=50)
    age: int = Field(..., gt=0, lt=100)
    email: EmailStr
```

### Nested Models

```python
class Address(BaseModel):
    city: str
    country: str

class User(BaseModel):
    name: str
    address: Address
```

### Optional Fields

```python
from typing import Optional

class User(BaseModel):
    name: str
    age: Optional[int] = None
```

---

## **Topic 5 — Exception Handling**

### HTTPException

```python
from fastapi import HTTPException

@app.get("/items/{item_id}")
def read_item(item_id: int):
    if item_id > 10:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item_id": item_id}
```

### Custom Exception Handler

```python
from fastapi.responses import JSONResponse
from fastapi.requests import Request

class MyCustomException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(MyCustomException)
async def custom_exception_handler(request: Request, exc: MyCustomException):
    return JSONResponse(status_code=418, content={"message": f"Oops! {exc.name} caused an error."})
```

### Common Status Codes

| Code | Meaning               |
| ---- | --------------------- |
| 200  | OK                    |
| 201  | Created               |
| 400  | Bad Request           |
| 401  | Unauthorized          |
| 403  | Forbidden             |
| 404  | Not Found             |
| 422  | Validation Error      |
| 500  | Internal Server Error |

---

## **Topic 6 — SQLAlchemy with FastAPI**

### Setup Engine & Session

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
```

### Models (ORM)

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

### DB Session Dependency

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### CRUD Operations

```python
# Create
@app.post("/users/")
def create_user(user: UserSchema, db: Session = Depends(get_db)):
    db_user = User(name=user.name, email=user.email)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user

# Read
@app.get("/users/")
def read_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    return db.query(User).offset(skip).limit(limit).all()

# Update
@app.put("/users/{user_id}")
def update_user(user_id: int, user: UserSchema, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    db_user.name = user.name
    db.commit()
    return db_user

# Delete
@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    db.delete(db_user)
    db.commit()
    return {"message": "User deleted"}
```

### Alembic Migrations

```bash
pip install alembic
alembic init migrations
alembic revision --autogenerate -m "create users table"
alembic upgrade head
```

---

## **Topic 7 — Dependency Injection**

```python
from fastapi import Depends

def common_parameters(q: str = None, skip: int = 0, limit: int = 10):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
def read_items(commons: dict = Depends(common_parameters)):
    return commons
```

* Dependencies can be used for DB sessions, auth, config, etc.
* Supports **dependency chaining**.

---

## **Topic 8 — Authentication & Authorization**

### JWT Login Example

```python
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt
from datetime import datetime, timedelta

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
SECRET_KEY = "supersecret"

@app.post("/token")
def login(form_data: OAuth2PasswordRequestForm = Depends()):
    if form_data.username != "admin" or form_data.password != "pass":
        raise HTTPException(status_code=401, detail="Invalid credentials")
    payload = {"sub": form_data.username, "exp": datetime.utcnow() + timedelta(minutes=30)}
    token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
    return {"access_token": token, "token_type": "bearer"}
```

### Protected Routes

```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    return payload["sub"]

@app.get("/profile")
def profile(user: str = Depends(get_current_user)):
    return {"user": user}
```

### Password Hashing

```python
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
hashed_password = pwd_context.hash("mypassword")
pwd_context.verify("mypassword", hashed_password)
```

---

## **Topic 9 — File Handling**

### Upload File

```python
from fastapi import UploadFile, File

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    content = await file.read()
    return {"filename": file.filename, "size": len(content)}
```

### Save File

```python
@app.post("/upload/save")
async def save_file(file: UploadFile = File(...)):
    path = f"uploads/{file.filename}"
    with open(path, "wb") as f:
        f.write(await file.read())
    return {"saved_to": path}
```

### Multiple Files

```python
@app.post("/upload/multiple")
async def upload_multiple(files: list[UploadFile] = File(...)):
    return {"file_count": len(files)}
```

### File Download

```python
from fastapi.responses import FileResponse
@app.get("/download/{filename}")
def download(filename: str):
    return FileResponse(f"uploads/{filename}")
```

### CSV Handling

```python
import csv
@app.post("/upload/csv")
async def upload_csv(file: UploadFile = File(...)):
    content = (await file.read()).decode("utf-8")
    rows = [row for row in csv.reader(content.splitlines())]
    return {"rows": rows}
```

---

## **Topic 10 — Asynchronous Programming**

### Async Route

```python
@app.get("/async")
async def async_route():
    return {"msg": "Hello Async"}
```

### Async DB (SQLAlchemy 2.0)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "mysql+asyncmy://root:password@localhost/db"
engine = create_async_engine(DATABASE_URL, echo=True)
SessionLocal = sessionmaker(bind=engine, class_=AsyncSession, expire_on_commit=False)
```

---

## **Topic 11 — Project Structure**

* Use **folders** for routers, schemas, services, DB models, middleware, tasks.
* Versioned APIs: `api/v1/routes`, `api/v1/schemas`, etc.
* Middleware → logging, CORS, auth.
* Background tasks → Celery, email, reports.

---

## **Topic 12 — Testing**

### TestClient

```python
from fastapi.testclient import TestClient
client = TestClient(app)
def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
```

### Async Test

```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/async")
    assert response.status_code == 200
```

---

## **Topic 13 — Deployment**

* **Uvicorn:** `uvicorn app.main:app --host 0.0.0.0 --port 8000`
* **Gunicorn:** `gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker`
* **Docker:** containerize app for portability
* **Nginx:** reverse proxy, SSL, load balancing
* **Cloud:** Railway, Render, AWS (EC2/ECS)

---

## **Topic 14 — Advanced Concepts**

### WebSockets

```python
@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    while True:
        data = await ws.receive_text()
        await ws.send_text(f"Message: {data}")
```

### Streaming Responses

```python
from fastapi.responses import StreamingResponse
def generate(): 
    for i in range(1000):
        yield f"Line {i}\n"
@app.get("/stream")
def stream_file():
    return StreamingResponse(generate(), media_type="text/plain")
```

### Background Tasks

```python
@app.post("/send-email")
def email(background_tasks: BackgroundTasks, email: str):
    background_tasks.add_task(send_email, email)
    return {"message": "Email will be sent in background"}
```

### Rate Limiting (Middleware)

```python
@app.middleware("http")
async def rate_limit(request: Request, call_next):
    ...
```

### Caching with Redis

```python
import aioredis
redis = aioredis.from_url("redis://localhost")
@app.get("/cached")
async def cached():
    data = await redis.get("key")
    if not data:
        data = "Expensive computation"
        await redis.set("key", data, ex=60)
    return {"data": data}
```

---
 

Do you want me to do that?
