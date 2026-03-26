---
name: python-backend
description: "Guides Python backend development with Django and FastAPI, covering project structure, async patterns, Pydantic validation, SQLAlchemy ORM, Celery tasks, and testing. Enforces type hints, proper dependency injection, and framework-idiomatic patterns. Use when python, django, fastapi, flask, pydantic, sqlalchemy, celery, uvicorn, poetry, python api, python backend, async, REST API mentioned."
---

# Python Backend

## Principles

- **Type hints everywhere** -- use annotations on all function signatures and return types for IDE support, mypy checking, and Pydantic schema generation.
- **Don't fight the framework** -- follow Django's conventions in Django, FastAPI's in FastAPI. Django's ORM is not SQLAlchemy; use each idiomatically.
- **Async for I/O, sync for CPU** -- use `async def` for network/database I/O; use synchronous functions for CPU-bound computation. Never call blocking code (e.g., `requests.get`, `time.sleep`, Django ORM) inside an `async def` without wrapping it.
- **Pydantic for validation** -- define request/response schemas with Pydantic v2 models instead of manual validation.
- **Dependency injection in FastAPI** -- use `Depends()` for database sessions, auth, and services to keep endpoints testable.
- **Virtual environments always** -- use `poetry`, `uv`, or `venv`; never install packages globally. Use `pyproject.toml` for project metadata.

## Workflow

### Setting up a new FastAPI project

1. Initialize the project with `poetry init` or `uv init` and add `fastapi`, `uvicorn[standard]`, `pydantic-settings`, `sqlalchemy[asyncio]`, `alembic`.
2. Create the project layout (see example below).
3. Define settings in `config.py` using `pydantic_settings.BaseSettings` with an `.env` file.
4. Set up the async database engine in `database.py` with `create_async_engine` and a session dependency.
5. Define SQLAlchemy models in `models/`, Pydantic schemas in `schemas/`, and route handlers in `routers/`.
6. Extract business logic into `services/` classes that accept an `AsyncSession`.
7. Register routers in `main.py` with a `lifespan` context manager for startup/shutdown.
8. Write tests using `pytest`, `pytest-asyncio`, and `httpx.AsyncClient` with dependency overrides.

### FastAPI project structure example

```python
# main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager
from app.config import settings
from app.database import engine
from app.routers import users

@asynccontextmanager
async def lifespan(app: FastAPI):
    await engine.connect()
    yield
    await engine.dispose()

app = FastAPI(title=settings.app_name, lifespan=lifespan)
app.include_router(users.router, prefix="/users", tags=["users"])
```

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "My API"
    database_url: str
    redis_url: str = "redis://localhost:6379"
    secret_key: str

    class Config:
        env_file = ".env"

settings = Settings()
```

```python
# routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.schemas.user import UserCreate, UserResponse
from app.services.user_service import UserService

router = APIRouter()

@router.post("/", response_model=UserResponse)
async def create_user(
    user: UserCreate,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    service = UserService(db)
    return await service.create_user(user)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    service = UserService(db)
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

### Setting up a Django project with async views

1. Create the project with `django-admin startproject config .` and organize apps under `apps/`.
2. Split settings into `config/settings/base.py`, `local.py`, and `production.py`.
3. Use `AbstractUser` for custom user models from the start -- changing later requires migrations.
4. Add `select_related()` / `prefetch_related()` on every queryset that accesses relationships to avoid N+1 queries.
5. Wrap sync ORM calls with `sync_to_async` in async views, or use Django REST Framework serializers for API endpoints.
6. Run `python manage.py makemigrations` after every model change and check with `python manage.py migrate --check` in CI.

### Django model + API pattern example

```python
# apps/users/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser

class User(AbstractUser):
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    USERNAME_FIELD = "email"
    REQUIRED_FIELDS = ["username"]

    class Meta:
        db_table = "users"
        indexes = [
            models.Index(fields=["email"]),
            models.Index(fields=["created_at"]),
        ]
```

```python
# apps/users/views.py -- async view
from django.http import JsonResponse
from django.views import View
from asgiref.sync import sync_to_async
from .models import User

class UserListView(View):
    async def get(self, request) -> JsonResponse:
        users = await sync_to_async(list)(
            User.objects.values("id", "email", "created_at")[:100]
        )
        return JsonResponse({"users": users})
```

### Adding Pydantic validation (FastAPI)

1. Define a `Create` schema with field constraints and `@field_validator` methods.
2. Define an `Update` schema with all-optional fields and `model_config = ConfigDict(extra="forbid")`.
3. Define a `Response` schema with `model_config = ConfigDict(from_attributes=True)` for ORM compatibility.
4. Use Pydantic v2 syntax: `@field_validator` (not `@validator`), `.model_dump()` (not `.dict()`), `ConfigDict` (not inner `Config` class for models).

```python
from pydantic import BaseModel, Field, EmailStr, field_validator, ConfigDict
from datetime import datetime
from typing import Optional

class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8, max_length=100)
    name: str = Field(min_length=1, max_length=100)

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("must contain uppercase")
        if not any(c.isdigit() for c in v):
            raise ValueError("must contain digit")
        return v

class UserUpdate(BaseModel):
    name: Optional[str] = None
    model_config = ConfigDict(extra="forbid")

class UserResponse(BaseModel):
    id: int
    email: EmailStr
    name: str
    created_at: datetime
    model_config = ConfigDict(from_attributes=True)
```

### Adding background tasks with Celery

1. Configure `Celery` with JSON serialization, UTC timezone, and task time limits.
2. Use `bind=True` and `max_retries` on tasks that call external services.
3. Use `celery beat` for periodic scheduling.
4. Call tasks with `.delay()` or `.apply_async()` from endpoints -- never execute them synchronously in request handlers.

### Writing tests (pytest + httpx)

1. Create a `conftest.py` with an in-memory database fixture and an `AsyncClient` fixture that overrides `get_db`.
2. Mark async tests with `@pytest.mark.asyncio`.
3. Test happy paths, validation errors (expect 422), and not-found cases (expect 404).
4. For Django, use `@pytest.mark.django_db(transaction=True)` with `django.test.AsyncClient`.

```python
# conftest.py
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from app.main import app
from app.database import Base, get_db

@pytest.fixture
async def db_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with AsyncSession(engine) as session:
        yield session
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest.fixture
async def client(db_session):
    async def override_get_db():
        yield db_session
    app.dependency_overrides[get_db] = override_get_db
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

```python
# test_users.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient) -> None:
    response = await client.post(
        "/users/",
        json={"email": "test@example.com", "password": "Test1234"},
    )
    assert response.status_code == 200
    assert response.json()["email"] == "test@example.com"

@pytest.mark.asyncio
async def test_get_user_not_found(client: AsyncClient) -> None:
    response = await client.get("/users/999")
    assert response.status_code == 404
```

## Critical Sharp Edges

The agent must check for these issues and warn the developer when detected:

- **Blocking calls in async functions** -- calling `requests.get()`, `time.sleep()`, or Django ORM directly inside `async def` freezes the event loop. Use `httpx`, `asyncio.sleep`, or `sync_to_async` instead. See `references/sharp_edges.md` (async-blocking).
- **N+1 queries** -- iterating over a queryset and accessing relationships triggers one query per row. Always use `select_related()` / `prefetch_related()` (Django) or `joinedload()` / `selectinload()` (SQLAlchemy). See `references/sharp_edges.md` (n-plus-one-orm).
- **Hardcoded secrets** -- API keys or passwords in source code end up in Git history. Use `pydantic-settings` with `.env` files or a secrets manager. See `references/sharp_edges.md` (secret-in-code).
- **Mutable default arguments** -- `def f(items=[])` shares the list across calls. Use `None` sentinel and create inside the function body. See `references/sharp_edges.md` (mutable-defaults).
- **Pydantic v1/v2 confusion** -- mixing `@validator` (v1) with v2 models silently fails. Always use `@field_validator`, `.model_dump()`, and `ConfigDict`. See `references/sharp_edges.md` (pydantic-v1-v2).
- **Global mutable state** -- module-level dicts/lists are per-worker in production. Use Redis or database for shared state. See `references/sharp_edges.md` (global-state).

## Reference System Usage

Ground all responses in the provided reference files, treating them as the source of truth:

- **For creation tasks:** Consult `references/patterns.md` for project structures, FastAPI/Django patterns, Pydantic schemas, SQLAlchemy async setup, Celery configuration, and pytest fixtures.
- **For diagnosis tasks:** Consult `references/sharp_edges.md` for async blocking, N+1 queries, mutable defaults, circular imports, hardcoded secrets, global state, migration conflicts, and Pydantic v1/v2 issues.
- **For review tasks:** Consult `references/validations.md` for regex-based checks covering hardcoded secrets, SQL injection, debug mode, blocking async calls, N+1 queries, missing migrations, missing type hints, mutable defaults, bare except clauses, print statements, missing response models, and deprecated Pydantic syntax.

If a user's request conflicts with guidance in these files, explain the recommended approach using the reference material.
