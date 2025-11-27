 
# **FastAPI Notes — 

---

## **Topic 5 — Exception Handling**

### 1️⃣ Why Exception Handling?

* APIs may receive **invalid input** or face **runtime errors**.
* Without handling → server crashes or returns unhelpful errors.
* Proper handling improves **user experience** and **debugging**.

### 2️⃣ FastAPI’s HTTPException

FastAPI provides **`HTTPException`** to raise errors with HTTP status codes.

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int):
    if item_id > 10:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"item_id": item_id}
```

* `status_code` → HTTP status (404, 401, 500, etc.)
* `detail` → error message

### 3️⃣ Global Exception Handling

FastAPI allows **custom exception handlers**:

```python
from fastapi.responses import JSONResponse
from fastapi.requests import Request

class MyCustomException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(MyCustomException)
async def custom_exception_handler(request: Request, exc: MyCustomException):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} caused an error."}
    )

@app.get("/custom-error")
def error_demo():
    raise MyCustomException("Test")
```

* Returns custom JSON for any exception.
* Helps **centralize error handling**.

### 4️⃣ Common Status Codes

| Status Code | Meaning               |
| ----------- | --------------------- |
| 200         | OK                    |
| 201         | Created               |
| 400         | Bad Request           |
| 401         | Unauthorized          |
| 403         | Forbidden             |
| 404         | Not Found             |
| 422         | Validation Error      |
| 500         | Internal Server Error |

### 5️⃣ Best Practices

* Use **HTTPException** for predictable errors.
* Use **custom handlers** for repeated error patterns.
* Always return **JSON with meaningful messages**.

---

## **Topic 6 — SQLAlchemy with FastAPI**

FastAPI + SQLAlchemy = **real-world database-backed APIs**.

### 1️⃣ Setting Up SQLAlchemy Engine

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "sqlite:///./test.db"

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(bind=engine, autocommit=False, autoflush=False)
```

* `create_engine` → connects to DB
* `SessionLocal` → create sessions for DB operations

### 2️⃣ Creating Models (ORM)

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
```

* ORM maps **Python class → DB table**
* Columns define table fields

### 3️⃣ Creating DB Session Dependency

```python
from fastapi import Depends

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

* Use `Depends(get_db)` in routes to get a session.
* Ensures **DB session is properly closed**.

### 4️⃣ CRUD Operations

#### **Create**

```python
@app.post("/users/")
def create_user(user: UserSchema, db: Session = Depends(get_db)):
    db_user = User(name=user.name, email=user.email)
    db.add(db_user)
    db.commit()
    db.refresh(db_user)
    return db_user
```

#### **Read**

```python
@app.get("/users/")
def read_users(skip: int = 0, limit: int = 10, db: Session = Depends(get_db)):
    return db.query(User).offset(skip).limit(limit).all()
```

#### **Update**

```python
@app.put("/users/{user_id}")
def update_user(user_id: int, user: UserSchema, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    db_user.name = user.name
    db.commit()
    return db_user
```

#### **Delete**

```python
@app.delete("/users/{user_id}")
def delete_user(user_id: int, db: Session = Depends(get_db)):
    db_user = db.query(User).filter(User.id == user_id).first()
    db.delete(db_user)
    db.commit()
    return {"message": "User deleted"}
```

### 5️⃣ Alembic Migrations

* Alembic handles **DB schema changes**.
* Install:

```bash
pip install alembic
```

* Initialize:

```bash
alembic init migrations
```

* Create migration:

```bash
alembic revision --autogenerate -m "create users table"
```

* Apply migration:

```bash
alembic upgrade head
```

---

## **Topic 7 — Dependency Injection**

Dependency Injection = **inject reusable resources** into routes.

### 1️⃣ Why Dependency Injection?

* Reuse code (DB sessions, auth)
* Avoid global variables
* Cleaner code

### 2️⃣ FastAPI `Depends`

```python
from fastapi import Depends

def common_parameters(q: str = None, skip: int = 0, limit: int = 10):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
def read_items(commons: dict = Depends(common_parameters)):
    return commons
```

* `Depends` calls function and injects return value into route.

### 3️⃣ DB Example

```python
@app.get("/users/")
def read_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

* `get_db()` returns **DB session** → injected into route.

### 4️⃣ Advanced Dependency

Dependencies can depend on other dependencies:

```python
def get_current_user(token: str = Depends(oauth2_scheme)):
    return decode_token(token)

@app.get("/profile")
def read_profile(user=Depends(get_current_user)):
    return {"user_id": user.id}
```

* `get_current_user` depends on `oauth2_scheme` → shows **dependency chaining**.

### 5️⃣ Key Notes

* Dependencies can be used for:

  * DB session
  * Authentication
  * Rate limiting
  * Settings/config
* Improves **modularity and testability**.

---

# ✅ **Summary —  
* **Exception Handling:** HTTPException, custom handlers, error codes.
* **SQLAlchemy + FastAPI:**

  * Engine, Session, Models
  * CRUD operations
  * Alembic migrations
* **Dependency Injection:**

  * `Depends` for DB, auth, services
  * Chaining dependencies
  * Makes code modular and testable

---
 
