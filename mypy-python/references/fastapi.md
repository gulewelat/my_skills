# FastAPI typing

FastAPI uses annotations at runtime (validation, OpenAPI) — so the same types
serve mypy and the framework. Enable the Pydantic mypy plugin:

```toml
[tool.mypy]
plugins = ["pydantic.mypy"]

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
warn_required_dynamic_aliases = true
```

## Typed endpoints

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class UserOut(BaseModel):
    id: int
    name: str
    email: str
    age: int | None = None

class UserIn(BaseModel):
    name: str
    email: str
    age: int | None = None

@app.get("/users/{user_id}")
def get_user(user_id: int) -> UserOut:
    if user_id == 0:
        raise HTTPException(status_code=404, detail="User not found")
    return UserOut(id=user_id, name=f"User {user_id}", email="user@example.com")

@app.post("/users")
def create_user(payload: UserIn) -> UserOut:
    return UserOut(id=1, **payload.model_dump())
```

The return annotation is also the `response_model`: no need to repeat it.

## Dependencies with `Annotated`

```python
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends, Header
from sqlalchemy.ext.asyncio import AsyncSession

async def get_session() -> AsyncIterator[AsyncSession]:   # yield → AsyncIterator
    async with session_factory() as session:
        yield session

SessionDep = Annotated[AsyncSession, Depends(get_session)]

async def get_current_user(
    authorization: Annotated[str | None, Header()] = None,
) -> UserOut:
    if authorization is None:
        raise HTTPException(status_code=401, detail="Not authenticated")
    return UserOut(id=1, name="Current User", email="user@example.com")

CurrentUser = Annotated[UserOut, Depends(get_current_user)]

@app.get("/me")
async def get_me(user: CurrentUser) -> UserOut:
    return user
```

Rules:
- A dependency that `yield`s is annotated `Iterator[T]` / `AsyncIterator[T]`,
  never `T`.
- Prefer `Annotated[T, Depends(...)]` aliases over `x: T = Depends(...)` — the
  default-value form lies to mypy about the parameter having a default.
- Endpoints return API schemas, not ORM entities: map explicitly.

## Service dependencies

```python
class UserService:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def find(self, user_id: int) -> UserOut | None: ...

def get_user_service(session: SessionDep) -> UserService:
    return UserService(session)

UserServiceDep = Annotated[UserService, Depends(get_user_service)]

@app.get("/users/{user_id}")
async def read_user(user_id: int, service: UserServiceDep) -> UserOut:
    user = await service.find(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

## Common mypy errors with SQLAlchemy results

```python
result = await session.execute(select(UserRow).limit(10))
rows = result.scalars().all()     # Sequence[UserRow], not list[UserRow]
return [to_schema(r) for r in rows]
```

For a hexagonal layout (endpoint → use case → port), see the
**hexagonal-architecture-python** skill.
