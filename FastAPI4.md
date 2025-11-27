
# **FastAPI Notes  

---

## **Topic 11 — FastAPI Project Structure**

A professional FastAPI project must be **organized, scalable, and maintainable**.

### 1️⃣ Recommended Folder Layout

```
app/
│── main.py
│── api/
│     ├── v1/
│     │    ├── routes/
│     │    ├── schemas/
│     │    └── services/
│── core/
│     ├── config.py
│     └── utils.py
│── db/
│     ├── base.py
│     ├── session.py
│     └── models/
│── middleware/
│── tasks/
│── static/
│── uploads/
│── tests/
│── requirements.txt
│── Dockerfile
│── README.md
```

---

### 2️⃣ Folder Explanation

| Folder             | Purpose                                            |
| ------------------ | -------------------------------------------------- |
| `main.py`          | Entry point, app instance, load routers/middleware |
| `api/`             | API layer (versioned routes, schemas, services)    |
| `core/`            | App configuration, utilities, security             |
| `db/`              | Database models, session, Alembic migrations       |
| `middleware/`      | Custom middleware (logging, CORS)                  |
| `tasks/`           | Background tasks, Celery workers                   |
| `static/`          | Static files (images, CSS, JS)                     |
| `uploads/`         | Uploaded files                                     |
| `tests/`           | Test cases                                         |
| `requirements.txt` | Project dependencies                               |
| `Dockerfile`       | Containerization                                   |
| `README.md`        | Project documentation                              |

---

### 3️⃣ Key Concepts

* **Routers** → define API endpoints
* **Schemas** → request/response validation
* **Services** → business logic, clean separation
* **Models** → ORM mapping to database
* **Middleware** → logging, security, CORS
* **Tasks** → background jobs, Celery integration
* **Tests** → automated testing

---

## **Topic 12 — Testing**

Testing ensures APIs **work as expected** and prevents regressions.

### 1️⃣ Setup Pytest

```bash
pip install pytest pytest-asyncio httpx
```

* `pytest` → test framework
* `pytest-asyncio` → async test support
* `httpx` → async client for testing endpoints

---

### 2️⃣ TestClient Example

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello FastAPI"}
```

---

### 3️⃣ Async Testing

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.mark.asyncio
async def test_async_endpoint():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        response = await ac.get("/async")
    assert response.status_code == 200
```

---

### 4️⃣ Mock Database

* Use **SQLite in-memory** or **mock sessions**.
* Prevents tests from affecting production DB.

```python
SQLALCHEMY_DATABASE_URL = "sqlite:///:memory:"
```

---

### 5️⃣ CRUD Testing Example

```python
def test_create_user():
    response = client.post("/users/", json={"name":"Danyal","email":"test@test.com"})
    assert response.status_code == 200
    assert response.json()["name"] == "Danyal"
```

---

## **Topic 13 — Deployment**

### 1️⃣ Uvicorn

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

* Development: `--reload`
* Production: specify host and port

---

### 2️⃣ Gunicorn + Uvicorn Workers

```bash
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker
```

* `-w 4` → 4 workers
* Handles multiple requests concurrently

---

### 3️⃣ Docker Deployment

**Dockerfile Example**

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

### 4️⃣ Nginx as Reverse Proxy

* Handles HTTPS, static files, load balancing.

```
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

### 5️⃣ Cloud Deployment

* **Railway / Render:** Quick, auto-deploy from GitHub
* **AWS (EC2, ECS):** Full control for production-scale
* **Docker + Cloud:** Portable, consistent deployments

---

## **Topic 14 — Advanced Concepts**

### 1️⃣ WebSockets

```python
from fastapi import WebSocket

@app.websocket("/ws")
async def websocket_endpoint(ws: WebSocket):
    await ws.accept()
    while True:
        data = await ws.receive_text()
        await ws.send_text(f"Message: {data}")
```

* Real-time communication (chat, live dashboards)

---

### 2️⃣ Streaming Responses

```python
from fastapi.responses import StreamingResponse

def generate():
    for i in range(1000):
        yield f"Line {i}\n"

@app.get("/stream")
def stream_file():
    return StreamingResponse(generate(), media_type="text/plain")
```

* Efficient for **large files or logs**

---

### 3️⃣ Background Tasks

```python
from fastapi import BackgroundTasks

def send_email(email: str):
    print(f"Email sent to {email}")

@app.post("/send-email")
def email(background_tasks: BackgroundTasks, email: str):
    background_tasks.add_task(send_email, email)
    return {"message": "Email will be sent in background"}
```

* Async jobs without blocking request response

---

### 4️⃣ Rate Limiting (Middleware Example)

* Limits requests per user/IP to prevent abuse.

```python
from fastapi import Request, HTTPException
from time import time

request_counts = {}
LIMIT = 5
WINDOW = 60

@app.middleware("http")
async def rate_limit(request: Request, call_next):
    ip = request.client.host
    now = time()
    request_counts.setdefault(ip, [])
    request_counts[ip] = [t for t in request_counts[ip] if now - t < WINDOW]
    if len(request_counts[ip]) >= LIMIT:
        raise HTTPException(429, "Too many requests")
    request_counts[ip].append(now)
    return await call_next(request)
```

---

### 5️⃣ Caching with Redis

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

* Improves performance by **reducing DB hits**

---

# ✅ **Summary —  

* **Project Structure:** Clean folders, routers, services, DB, middleware
* **Testing:** Pytest, TestClient, async testing, mock DB
* **Deployment:** Uvicorn, Gunicorn, Docker, Nginx, Cloud
* **Advanced Concepts:** WebSockets, Streaming, Background Tasks, Rate Limiting, Caching
 
 
