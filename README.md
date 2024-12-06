# FastAPI Boilerplate

[![Python](https://img.shields.io/badge/python-3.12-blue.svg)](https://www.python.org/downloads/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115.6-green.svg)](https://fastapi.tiangolo.com)
[![Poetry](https://img.shields.io/badge/poetry-1.7-orange.svg)](https://python-poetry.org/)

A modern, production-ready FastAPI boilerplate with async SQLAlchemy, authentication, caching, event dispatching, and more. Perfect for building scalable and maintainable web APIs.

## 🚀 Features

- ✨ Async SQLAlchemy with multiple database support
- 🔐 JWT Authentication with custom user class
- 🔒 Role-based access control with custom permissions
- 📦 Celery for background tasks
- 🐳 Docker support with hot reload
- 📡 Event dispatcher for decoupled architecture
- 💾 Redis caching system
- 📝 Alembic migrations
- 🧪 Pytest with async support
- 🎯 Pre-commit hooks for code quality

## 📋 Prerequisites

- Python 3.12+
- Poetry for dependency management
- Docker and Docker Compose (for local development)
- MySQL (or compatible database)
- Redis (for caching)

## 🛠️ Installation

### 1. Clone the repository
```bash
git clone <repository-url>
cd fastapi-boilerplate
```

### 2. Launch Docker services
```bash
docker-compose -f docker/docker-compose.yml up
```

### 3. Set up Python environment
```bash
poetry shell
poetry install
```

### 4. Apply database migrations
```bash
alembic upgrade head
```

## 🚀 Running the Application

### Development Server
```bash
python3 main.py --env local --debug
```

Available environments:
- local: Local development
- dev: Development server
- prod: Production server

### Running Tests
```bash
make test
```

### Generate Coverage Report
```bash
make cov
```

### Code Formatting
```bash
pre-commit
```

## 💡 Technical Guide

### SQLAlchemy for Asyncio Context

```python
from core.db import Transactional, session

@Transactional()
async def create_user(self):
    session.add(User(email="padocon@naver.com"))
```

Do not use explicit `commit()`. `Transactional` class automatically handles it.

#### Query with asyncio.gather()
When executing queries concurrently through `asyncio.gather()`, you must use the `session_factory` context manager rather than the globally used session.

```python
from core.db import session_factory

async def get_by_id(self, *, user_id) -> User:
    stmt = select(User)
    async with session_factory() as read_session:
        return await read_session.execute(query).scalars().first()

async def main() -> None:
    user_1, user_2 = await asyncio.gather(
        get_by_id(user_id=1),
        get_by_id(user_id=2),
    )
```

If you do not use a database connection like `session.add()`, it is recommended to use a globally provided session.

#### Multiple Databases

Go to `core/config.py` and edit `WRITER_DB_URL` and `READER_DB_URL` in the config class.
If you need additional logic to use the database, refer to the `get_bind()` method of `RoutingClass`.

### Authentication System

#### Custom User Class
```python
from fastapi import Request

@home_router.get("/")
def home(request: Request):
    return request.user.id
```

**Note: You must pass JWT token via header like `Authorization: Bearer 1234`**

The custom user class automatically decodes the header token and stores user information in `request.user`

To modify the custom user class, update these files:
1. `core/fastapi/schemas/current_user.py`
2. `core/fastapi/middlewares/authentication.py`

#### CurrentUser Schema
```python
class CurrentUser(BaseModel):
    id: int = Field(None, description="ID")
```

Simply add more fields based on your needs.

### Permission System

Permissions `IsAdmin`, `IsAuthenticated`, `AllowAll` are pre-implemented:

```python
from core.fastapi.dependencies import (
    PermissionDependency,
    IsAdmin,
)

user_router = APIRouter()

@user_router.get(
    "",
    response_model=List[GetUserListResponseSchema],
    response_model_exclude={"id"},
    responses={"400": {"model": ExceptionResponseSchema}},
    dependencies=[Depends(PermissionDependency([IsAdmin]))],  # HERE
)
async def get_user_list(
    limit: int = Query(10, description="Limit"),
    prev: int = Query(None, description="Prev ID"),
):
    pass
```

To create custom permissions, inherit `BasePermission` and implement `has_permission()`.

**Note: To use Swagger's authorize function, you must include `PermissionDependency` as a dependencies argument.**

### Caching System

#### Basic Caching
```python
from core.helpers.cache import Cache

@Cache.cached(prefix="get_user", ttl=60)
async def get_user():
    ...
```

#### Tag-based Caching
```python
from core.helpers.cache import Cache, CacheTag

@Cache.cached(tag=CacheTag.GET_USER_LIST, ttl=60)
async def get_user():
    ...
```

#### Custom Key Builder
```python
from core.helpers.cache.base import BaseKeyMaker

class CustomKeyMaker(BaseKeyMaker):
    async def make(self, function: Callable, prefix: str) -> str:
        ...
```

#### Custom Cache Backend
```python
from core.helpers.cache.base import BaseBackend

class RedisBackend(BaseBackend):
    async def get(self, key: str) -> Any:
        ...

    async def set(self, response: Any, key: str, ttl: int = 60) -> None:
        ...

    async def delete_startswith(self, value: str) -> None:
        ...
```

Initialize custom backend in `/app/server.py`:
```python
def init_cache() -> None:
    Cache.init(backend=RedisBackend(), key_maker=CustomKeyMaker())
```

#### Cache Management
```python
from core.helpers.cache import Cache, CacheTag

await Cache.remove_by_prefix(prefix="get_user_list")
await Cache.remove_by_tag(tag=CacheTag.GET_USER_LIST)
```

### Event Dispatcher
For event dispatcher documentation, refer to https://github.com/teamhide/fastapi-event

## 🔧 Configuration

The application can be configured through environment variables or config files. Key configuration options:

- `ENV`: Application environment (local/dev/prod)
- `DEBUG`: Enable debug mode
- `APP_HOST`: Application host
- `APP_PORT`: Application port
- `WRITER_DB_URL`: Writer database URL
- `READER_DB_URL`: Reader database URL
- `REDIS_HOST`: Redis host
- `REDIS_PORT`: Redis port

## 🚀 Deployment

### Docker Deployment
```bash
docker-compose -f docker/docker-compose.prod.yml up -d
```

### Manual Deployment
1. Set up your production environment
2. Configure your environment variables
3. Run database migrations
4. Start the application with gunicorn
```bash
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker
```

## 🔍 Troubleshooting

### Common Issues

1. **Database Connection Issues**
   - Verify database credentials
   - Check if database service is running
   - Ensure correct database URL format

2. **Redis Connection Issues**
   - Verify Redis is running
   - Check Redis connection settings

3. **Migration Issues**
   - Run `alembic history` to check migration status
   - Ensure database is accessible
   - Check for conflicting migrations

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## 📝 License

This project is licensed under the MIT License - see the LICENSE file for details.