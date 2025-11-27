 
# FastAPI Notes  
---

## **Topic 8 — Authentication & Authorization**

Authentication and Authorization are crucial for securing APIs.

### 1️⃣ **Authentication vs Authorization**

| Concept        | Definition      | Example                                    |
| -------------- | --------------- | ------------------------------------------ |
| Authentication | Verify identity | Login with username/password               |
| Authorization  | Access control  | Admin can delete user, regular user cannot |

---

### 2️⃣ **OAuth2 & JWT (JSON Web Token)**

* **OAuth2:** Standard for API authentication.
* **JWT:** Token-based authentication mechanism.

**JWT structure:**

```
header.payload.signature
```

* Header → Algorithm & type
* Payload → User data
* Signature → Secret key signature

---

### 3️⃣ Login API Example

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
import jwt
from datetime import datetime, timedelta

app = FastAPI()
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

---

### 4️⃣ Protected Routes

```python
from fastapi import Security

def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return payload["sub"]
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/profile")
def profile(user: str = Depends(get_current_user)):
    return {"user": user}
```

* Only valid token can access `/profile`.

---

### 5️⃣ Password Hashing

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
hashed_password = pwd_context.hash("mypassword")
pwd_context.verify("mypassword", hashed_password)
```

* Never store plain passwords.
* Use **bcrypt** for hashing.

---

### 6️⃣ Role-Based Access

```python
def check_admin(user: str = Depends(get_current_user)):
    if user != "admin":
        raise HTTPException(status_code=403, detail="Not authorized")
```

* You can protect endpoints by roles.

---

# **Topic 9 — File Handling**

File handling allows uploading and downloading images, documents, CSV, PDFs.

---

### 1️⃣ File Upload (Single File)

```python
from fastapi import UploadFile, File

@app.post("/upload")
async def upload(file: UploadFile = File(...)):
    content = await file.read()
    return {"filename": file.filename, "size": len(content)}
```

* `UploadFile` → recommended for large files
* Provides `filename`, `content_type`, and `read()`

---

### 2️⃣ Save File to Disk

```python
@app.post("/upload/save")
async def save_file(file: UploadFile = File(...)):
    path = f"uploads/{file.filename}"
    with open(path, "wb") as f:
        f.write(await file.read())
    return {"saved_to": path}
```

* Saves files for later download or processing.

---

### 3️⃣ Multiple File Upload

```python
@app.post("/upload/multiple")
async def upload_multiple(files: list[UploadFile] = File(...)):
    return {"file_count": len(files)}
```

---

### 4️⃣ File Download

```python
from fastapi.responses import FileResponse

@app.get("/download/{filename}")
def download(filename: str):
    return FileResponse(f"uploads/{filename}")
```

* Serves static files or user uploads.

---

### 5️⃣ CSV Example

**Read CSV**

```python
import csv

@app.post("/upload/csv")
async def upload_csv(file: UploadFile = File(...)):
    content = (await file.read()).decode("utf-8")
    rows = [row for row in csv.reader(content.splitlines())]
    return {"rows": rows}
```

**Export CSV**

```python
@app.get("/export/csv")
def export_csv():
    return FileResponse("exports/data.csv", media_type="text/csv")
```

---

# **Topic 10 — Asynchronous Programming (async/await)**

FastAPI is **async-ready**, so understanding async is key.

---

### 1️⃣ Sync vs Async

| Feature  | sync      | async                   |
| -------- | --------- | ----------------------- |
| Blocking | Yes       | No                      |
| Use case | CPU tasks | I/O tasks (DB, network) |
| Speed    | Slower    | Faster                  |

---

### 2️⃣ Async Route Example

```python
@app.get("/async")
async def async_route():
    return {"msg": "Hello Async"}
```

* `async def` → non-blocking route
* Can handle concurrent requests efficiently

---

### 3️⃣ Async Database (SQLAlchemy 2.0)

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

DATABASE_URL = "mysql+asyncmy://root:password@localhost/db"
engine = create_async_engine(DATABASE_URL, echo=True)
SessionLocal = sessionmaker(bind=engine, class_=AsyncSession, expire_on_commit=False)
```

* `AsyncSession` → supports async DB queries
* `await db.execute()` → non-blocking query

---

### 4️⃣ Mixing Sync + Async

**Bad:**

```python
async def route():
    time.sleep(5)  # blocks event loop
```

**Good:**

```python
import asyncio

async def route():
    await asyncio.to_thread(sync_function)
```

* Uses thread pool → keeps event loop free.

---

### 5️⃣ Real-World Async Examples

* Async API calls with `httpx`
* WebSocket communication
* High-traffic endpoints
* File streaming and background tasks

---

# ✅ **Summary —  

* **Authentication:** OAuth2, JWT, password hashing, role-based access
* **File Handling:** Upload, save, download, CSV, PDF, documents
* **Async Programming:** Non-blocking routes, async DB, async API calls, concurrency

 
