# FastAPI + Pydantic Cheatsheet

## Mental Model

FastAPI is an **async-first web framework** built on top of Starlette and Pydantic. The core idea: you define Python functions with type annotations and FastAPI automatically handles HTTP routing, request validation, serialization, and OpenAPI docs generation. Pydantic is the validation layer — it converts raw dicts/JSON into typed Python objects and raises structured errors when data doesn't match the schema. Together, they make the contract between your API and its clients explicit and machine-readable.

---

## Install & Minimal Setup

```bash
pip install fastapi "uvicorn[standard]" pydantic pydantic-settings

# Run development server
uvicorn main:app --reload --port 8000

# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
```

```python
# main.py — minimal working app
from fastapi import FastAPI

app = FastAPI(title="My API", version="1.0.0")

@app.get("/health")
def health_check():
    return {"status": "ok"}
```

```
http://localhost:8000/docs   → Swagger UI (interactive)
http://localhost:8000/redoc  → ReDoc
http://localhost:8000/openapi.json → raw schema
```

---

## Core Concepts

### 1. Path & Query Parameters

```python
from fastapi import FastAPI, Query, Path

app = FastAPI()

# Path parameter — part of the URL
@app.get("/users/{user_id}")
def get_user(user_id: int):   # FastAPI validates and converts to int
    return {"user_id": user_id}

# Query parameters — after the ?
@app.get("/items")
def list_items(
    skip: int = 0,
    limit: int = Query(default=10, ge=1, le=100),  # with validation
    search: str | None = None                       # optional
):
    return {"skip": skip, "limit": limit, "search": search}

# Path param with validation
@app.get("/items/{item_id}")
def get_item(
    item_id: int = Path(gt=0, description="The item ID, must be > 0")
):
    return {"item_id": item_id}
```

### 2. Pydantic Models — Request Bodies

```python
from pydantic import BaseModel, Field, EmailStr, field_validator
from typing import Optional
from datetime import datetime
from enum import Enum

class Role(str, Enum):
    admin = "admin"
    user  = "user"
    guest = "guest"

class UserCreate(BaseModel):
    name:     str       = Field(min_length=2, max_length=100)
    email:    EmailStr
    age:      int       = Field(ge=18, le=120)
    role:     Role      = Role.user
    bio:      str | None = None

    # Custom validator
    @field_validator("name")
    @classmethod
    def name_must_not_contain_numbers(cls, v: str) -> str:
        if any(c.isdigit() for c in v):
            raise ValueError("name must not contain numbers")
        return v.strip()

class UserResponse(BaseModel):
    id:         int
    name:       str
    email:      EmailStr
    role:       Role
    created_at: datetime

    model_config = {"from_attributes": True}  # allow ORM model → Pydantic

@app.post("/users", response_model=UserResponse, status_code=201)
def create_user(user: UserCreate):
    # user is already validated — access as typed object
    return create_user_in_db(user)
```

### 3. Response Models & Status Codes

```python
from fastapi import status

# Explicit response model — filters out fields not in the model
@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int):
    user = db.get(user_id)
    return user  # SQLAlchemy model or dict — Pydantic converts it

# Multiple response types documented in OpenAPI
@app.get(
    "/users/{user_id}",
    response_model=UserResponse,
    responses={
        404: {"description": "User not found"},
        403: {"description": "Not enough permissions"},
    }
)
def get_user(user_id: int): ...

# List response
@app.get("/users", response_model=list[UserResponse])
def list_users(): ...
```

### 4. Dependency Injection

```python
from fastapi import Depends

# Simple dependency
def get_db():
    db = SessionLocal()
    try:
        yield db        # yield makes it a context manager
    finally:
        db.close()

@app.get("/users")
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()

# Dependencies can depend on other dependencies
def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> User:
    user = verify_token(token, db)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return user

@app.get("/me", response_model=UserResponse)
def get_me(current_user: User = Depends(get_current_user)):
    return current_user

# Class-based dependency — useful for shared params
class PaginationParams:
    def __init__(self, skip: int = 0, limit: int = Query(default=10, le=100)):
        self.skip = skip
        self.limit = limit

@app.get("/items")
def list_items(pagination: PaginationParams = Depends()):
    return db.query(Item).offset(pagination.skip).limit(pagination.limit).all()
```

### 5. HTTPException & Error Handling

```python
from fastapi import HTTPException
from fastapi.responses import JSONResponse
from fastapi.requests import Request

# Raise HTTP errors
@app.get("/users/{user_id}")
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(
            status_code=404,
            detail=f"User {user_id} not found"
        )
    return user

# Custom exception class
class InsufficientFundsError(Exception):
    def __init__(self, balance: float, amount: float):
        self.balance = balance
        self.amount = amount

# Global exception handler
@app.exception_handler(InsufficientFundsError)
async def insufficient_funds_handler(request: Request, exc: InsufficientFundsError):
    return JSONResponse(
        status_code=402,
        content={
            "error": "insufficient_funds",
            "balance": exc.balance,
            "required": exc.amount
        }
    )
```

### 6. Lifespan (Startup & Shutdown)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup — runs before first request
    print("Starting up...")
    await init_db()
    load_ml_model()       # load heavy models once, not per request
    yield
    # Shutdown — runs after last request
    print("Shutting down...")
    await close_db()

app = FastAPI(lifespan=lifespan)
```

### 7. Async Routes

```python
import httpx

# Use async when doing I/O (DB, HTTP calls, file reads)
@app.get("/weather/{city}")
async def get_weather(city: str):
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.json()

# Use sync for CPU-bound work (FastAPI runs it in a thread pool)
@app.post("/process")
def process_data(data: ProcessRequest):
    result = cpu_intensive_computation(data)
    return result
```

### 8. Background Tasks

```python
from fastapi import BackgroundTasks

def send_email_task(email: str, message: str):
    # Runs after response is sent — don't use for critical operations
    send_email(email, message)

@app.post("/users", status_code=201)
def create_user(user: UserCreate, background_tasks: BackgroundTasks):
    new_user = save_to_db(user)
    background_tasks.add_task(send_email_task, user.email, "Welcome!")
    return new_user
```

### 9. Routers (Modular Structure)

```python
# routers/users.py
from fastapi import APIRouter

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
def list_users(): ...

@router.post("/", status_code=201)
def create_user(): ...

# main.py
from routers import users, items, auth

app = FastAPI()
app.include_router(users.router)
app.include_router(items.router)
app.include_router(auth.router, prefix="/auth")
```

---

## Pydantic Deep Dive

### Model Config & Validation

```python
from pydantic import BaseModel, ConfigDict, model_validator

class Settings(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,     # strip whitespace from strings
        str_to_lower=True,             # lowercase all strings
        validate_assignment=True,      # validate on attribute assignment
        populate_by_name=True,         # allow both alias and field name
        from_attributes=True           # ORM mode
    )

class Coordinates(BaseModel):
    lat: float = Field(ge=-90, le=90)
    lng: float = Field(ge=-180, le=180)

    # Cross-field validation
    @model_validator(mode="after")
    def check_not_null_island(self) -> "Coordinates":
        if self.lat == 0 and self.lng == 0:
            raise ValueError("Null Island is not a valid location")
        return self
```

### Nested Models

```python
class Address(BaseModel):
    street: str
    city:   str
    country: str = "Mexico"

class Company(BaseModel):
    name:    str
    address: Address              # nested model
    employees: list[UserCreate]   # list of models

# Usage
company = Company(
    name="Acme",
    address={"street": "Av. Reforma 1", "city": "CDMX"},
    employees=[{"name": "Joshua", "email": "j@acme.com", "age": 28}]
)
```

### Settings Management with pydantic-settings

```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    app_name:    str = "My API"
    debug:       bool = False
    database_url: str
    secret_key:  str
    openai_api_key: str | None = None

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

@lru_cache           # compute once, cache forever
def get_settings() -> Settings:
    return Settings()

# Inject as dependency
@app.get("/info")
def app_info(settings: Settings = Depends(get_settings)):
    return {"app_name": settings.app_name, "debug": settings.debug}
```

---

## Project Structure

```
myapi/
├── main.py               # app factory, lifespan, include routers
├── config.py             # Settings (pydantic-settings)
├── database.py           # engine, SessionLocal, Base
├── models/               # SQLAlchemy ORM models
│   └── user.py
├── schemas/              # Pydantic request/response models
│   └── user.py
├── routers/              # APIRouter per domain
│   ├── users.py
│   └── items.py
├── services/             # business logic (no HTTP concerns)
│   └── user_service.py
├── dependencies.py       # shared Depends (db, current_user)
└── tests/
    └── test_users.py
```

---

## Gotchas

- **`async def` vs `def`** — FastAPI runs `def` routes in a thread pool and `async def` on the event loop. Mixing them incorrectly causes performance issues. Use `async def` for I/O, `def` for CPU.
- **`response_model` filters output** — if a field is in your ORM model but not in the `response_model`, it won't appear in the response. Use this intentionally to hide `password_hash`, etc.
- **`Depends` is cached per request** — the same dependency called multiple times in a request chain returns the same instance. This is usually what you want for DB sessions.
- **`yield` in dependencies** — the cleanup code after `yield` runs even if the route raises an exception. Always use this pattern for resources that need cleanup (DB sessions, file handles).
- **Pydantic v1 vs v2** — LangChain still ships `langchain_core.pydantic_v1`. If you mix v1 and v2 models, use `from pydantic.v1` shim or keep them separate.
- **Startup errors** — if your `lifespan` raises, FastAPI won't start. Wrap risky startup code (DB connection, model loading) in try/except with a clear error message.

---

## Quick Links

- [FastAPI Docs](https://fastapi.tiangolo.com)
- [Pydantic v2 Docs](https://docs.pydantic.dev)
- [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
- [FastAPI Best Practices (GitHub)](https://github.com/zhanymkanov/fastapi-best-practices)
- [SQLModel](https://sqlmodel.tiangolo.com) — combines SQLAlchemy + Pydantic (same author as FastAPI)
