# mypy - Static Type Checking for Python

## Overview

mypy is the standard static type checker for Python, enabling gradual typing
with type hints (PEP 484) and comprehensive type safety. It catches type errors
before runtime, improves code documentation, and enhances IDE support while
maintaining Python's dynamic nature through incremental adoption.

**Key Features**:
- Gradual typing: Add types incrementally to existing code
- Strict mode: Maximum type safety with --strict flag
- Type inference: Automatically infer types from context
- Protocol support: Structural typing (duck typing with types)
- Generic types: TypeVar, Generic, and advanced type patterns
- FastAPI integration: Full Pydantic compatibility
- Plugin system: Extend type checking for libraries
- Incremental checking: Fast type checking on large codebases

## Installation

```bash
# Basic mypy
uv add --dev mypy

# For FastAPI projects
uv add --dev mypy pydantic

# Development setup with pre-commit
uv add --dev mypy pre-commit
```

---

## Part 2 — Basic Type Annotations

### 2.1 — Variables

```python
# Basic types
name: str = "Alice"
age: int = 30
height: float = 5.9
is_active: bool = True

# Type inference (no need to annotate)
count = 10        # mypy infers: int
message = "Hello" # mypy infers: str

# Union — multiple possible types
from typing import Union
user_id: Union[int, str] = 123  # int OR str

# Optional — can be None (shorthand for Union[T, None])
from typing import Optional
user_email: Optional[str] = None  # str or None
```

### 2.2 — Functions

```python
# Basic function typing
def greet(name: str) -> str:
    return f"Hello, {name}"

# Multiple parameters
def add(a: int, b: int) -> int:
    return a + b

# Optional parameter with default value
def create_user(name: str, age: int = 18) -> dict:
    return {"name": name, "age": age}

# No return value
def log_message(message: str) -> None:
    print(message)

# Function that never returns (always raises an exception)
from typing import NoReturn

def raise_error(message: str) -> NoReturn:
    raise ValueError(message)
```

### 2.3 — Collections

```python
from typing import List, Dict, Set, Tuple

# List
numbers: List[int] = [1, 2, 3]
names: List[str] = ["Alice", "Bob"]

# Dict
user_ages: Dict[str, int] = {"Alice": 30, "Bob": 25}

# Set
unique_ids: Set[int] = {1, 2, 3}

# Fixed tuple — each position has its own type
coordinate: Tuple[float, float] = (10.5, 20.3)
user_record: Tuple[int, str, bool] = (1, "Alice", True)

# Variable-length tuple — homogeneous type
numbers: Tuple[int, ...] = (1, 2, 3, 4, 5)
```

**Modern syntax Python 3.9+** — no need to import from `typing`:

```python
numbers: list[int] = [1, 2, 3]
user_ages: dict[str, int] = {"Alice": 30}
unique_ids: set[int] = {1, 2, 3}
```

### 2.4 — Classes

```python
from typing import Optional, Dict, Union

class User:
    # Class attributes declared at the top
    name: str
    age: int
    email: Optional[str]

    def __init__(self, name: str, age: int, email: Optional[str] = None) -> None:
        self.name = name
        self.age = age
        self.email = email

    def get_info(self) -> Dict[str, Union[str, int]]:
        return {
            "name": self.name,
            "age": self.age,
            "email": self.email or "N/A"
        }

    # Forward reference — class not yet fully defined
    @classmethod
    def from_dict(cls, data: dict) -> "User":
        return cls(
            name=data["name"],
            age=data["age"],
            email=data.get("email")
        )
```

---

## Part 3 — Advanced Type Annotations

### 3.1 — Literal

```python
from typing import Literal

# Restrict to specific values
def set_log_level(level: Literal["debug", "info", "warning", "error"]) -> None:
    print(f"Log level: {level}")

# ✅ Valid
set_log_level("debug")

# ❌ mypy error: "verbose" not in allowed values
set_log_level("verbose")

# Business alias with Literal
Status = Literal["pending", "approved", "rejected"]

def update_status(status: Status) -> None:
    pass
```

### 3.2 — Type Aliases

```python
from typing import Union, Dict, List, NewType

# Simple alias — give a business name to a primitive type
UserId = int
UserName = str

def get_user(user_id: UserId) -> UserName:
    return f"User {user_id}"

# Complex alias — name a technical structure
Headers = Dict[str, str]
QueryParams = Dict[str, Union[str, int, List[str]]]

def make_request(url: str, headers: Headers, params: QueryParams) -> None:
    pass

# NewType — creates a truly distinct type, not just an alias
UserId = NewType("UserId", int)
ProductId = NewType("ProductId", int)

def get_user(user_id: UserId) -> str:
    return f"User {user_id}"

user = UserId(123)
product = ProductId(456)

get_user(user)    # ✅ Valid
get_user(product) # ❌ mypy error: ProductId not compatible with UserId
get_user(123)     # ❌ mypy error: raw int not accepted
```

> **Note — Primitive Obsession**: Type aliases are the main solution against
> primitive obsession. See Anti-patterns in Part 9.

### 3.3 — Generics and TypeVar

```python
from typing import TypeVar, Generic, List

# TypeVar — generic type parameter
T = TypeVar("T")

# Generic function — works with any type
def first_element(items: List[T]) -> T:
    return items[0]

# mypy infers type based on what we pass
num = first_element([1, 2, 3])         # mypy infers: int
name = first_element(["Alice", "Bob"]) # mypy infers: str

# Bounded TypeVar — restricted to certain types only
NumericType = TypeVar("NumericType", int, float)

def add_numbers(a: NumericType, b: NumericType) -> NumericType:
    return a + b

add_numbers(1, 2)      # ✅ int
add_numbers(1.5, 2.5)  # ✅ float
add_numbers("a", "b")  # ❌ mypy error: str not allowed

# Generic class
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: List[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

# Usage
int_stack: Stack[int] = Stack()
int_stack.push(42)      # ✅ Valid
int_stack.push("hello") # ❌ mypy error: expected int, got str
```

### 3.4 — Protocol vs ABC

Both define interfaces, but with different philosophies:

**ABC (Abstract Base Class)**
```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    @abstractmethod
    def draw(self) -> str:
        ...

class Circle(Drawable):  # must explicitly inherit
    def draw(self) -> str:
        return "Drawing circle"
```

- Explicit inheritance required
- Error raised **at runtime** if abstract method is missing
- Comes from classic OOP

**Protocol**
```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> str:
        ...

class Circle:  # no need to inherit
    def draw(self) -> str:
        return "Drawing circle"

class Triangle:
    def area(self) -> float:
        return 10.5

def render(obj: Drawable) -> str:
    return obj.draw()

render(Circle())   # ✅ Valid
render(Triangle()) # ❌ mypy error: Triangle has no draw()
```

- **Structural** compatibility — just need the right methods
- Error detected **by mypy** at static analysis
- Preferred with mypy — more Pythonic, no inheritance coupling

**Runtime checkable Protocol:**
```python
from typing import runtime_checkable, Protocol

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None:
        ...

f = File()
isinstance(f, Closeable)  # ✅ True — checked at runtime too
```

| Situation | Use |
|---|---|
| You control all classes | ABC or Protocol, either works |
| You need to type external classes | **Protocol** required |
| You want a runtime error | **ABC** |
| You want to stay decoupled | **Protocol** |

### 3.5 — Callable

```python
from typing import Callable

# Syntax: Callable[[parameter types], return type]

# Function that takes another function as parameter
def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

def double(x: int) -> int:
    return x * 2

apply_twice(double, 5)  # ✅ Returns 20
apply_twice(str, 5)     # ❌ mypy error: str -> str not compatible with int -> int

# Callable with multiple parameters
Validator = Callable[[str, int], bool]

def validate_user(name: str, age: int) -> bool:
    return len(name) > 0 and age >= 0

validator: Validator = validate_user  # ✅ Valid

# Generic Callable with TypeVar
from typing import TypeVar

T = TypeVar("T")
R = TypeVar("R")

def map_values(func: Callable[[T], R], values: list[T]) -> list[R]:
    return [func(v) for v in values]

map_values(double, [1, 2, 3])  # ✅ mypy infers: list[int]
map_values(str, [1, 2, 3])     # ✅ mypy infers: list[str]
```

---

## Part 4 — mypy Configuration

### 4.1 — pyproject.toml

```toml
[tool.mypy]
python_version = "3.11"
files = ["src", "tests"]
exclude = ["build", "dist", "venv"]

# Strictness
disallow_untyped_defs = true
no_implicit_optional = true
warn_return_any = true
warn_unused_ignores = true
warn_redundant_casts = true
strict_equality = true

# Error reporting
show_error_codes = true
show_column_numbers = true
pretty = true
color_output = true

# Incremental
incremental = true
cache_dir = ".mypy_cache"

# Per-module overrides
[[tool.mypy.overrides]]
module = "tests.*"
disallow_untyped_defs = false

[[tool.mypy.overrides]]
module = ["migrations.*"]
ignore_errors = true
```

### 4.2 — Strict Mode

```bash
# Enable strict mode via command line
uv run mypy --strict src/
```

```toml
# pyproject.toml — manual strict equivalent
[tool.mypy]
disallow_any_unimported = true
disallow_any_generics = true
disallow_subclassing_any = true
disallow_untyped_calls = true
disallow_untyped_defs = true
disallow_incomplete_defs = true
check_untyped_defs = true
disallow_untyped_decorators = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_return_any = true
strict_equality = true
```

**Progressive strict mode adoption:**

```toml
[tool.mypy]
# Global still lenient
disallow_untyped_defs = true

# Strict on new and critical code
[[tool.mypy.overrides]]
module = "app.core.*"
strict = true

# Being migrated
[[tool.mypy.overrides]]
module = "app.utils.*"
disallow_untyped_defs = true
warn_return_any = true

# Legacy, not yet touched
[[tool.mypy.overrides]]
module = "app.legacy.*"
ignore_errors = true
```

### 4.3 — Anti-pattern: Ignoring errors globally

```toml
# ❌ WRONG: disables mypy on the entire project
[tool.mypy]
ignore_errors = true
```

```toml
# ✅ CORRECT: ignore only what is justified
[tool.mypy]
strict = true

[[tool.mypy.overrides]]
module = "app.legacy.*"
ignore_errors = true        # Legacy pending migration

[[tool.mypy.overrides]]
module = "migrations.*"
ignore_errors = true        # Auto-generated migrations
```

---

## Part 5 — Incremental Adoption

### 5.1 — Start with Entry Points

The entry points depend on the project type:

| Project type | Entry point |
|---|---|
| Script / CLI | `main.py` |
| REST API | `app/routers/` |
| Worker / tasks | `app/tasks/` |
| Library | Public API — `__init__.py` |

```bash
# Always start with what is exposed to the outside
uv run mypy <entry_point>

# Then descend into the implementation
uv run mypy <next_layer>
```

### 5.2 — Strict Mode Per Module

```toml
[tool.mypy]
# Global still lenient
disallow_untyped_defs = true

# Strict on new and critical code
[[tool.mypy.overrides]]
module = "app.core.*"
strict = true

# Being migrated
[[tool.mypy.overrides]]
module = "app.utils.*"
disallow_untyped_defs = true
warn_return_any = true

# Legacy, not yet touched
[[tool.mypy.overrides]]
module = "app.legacy.*"
ignore_errors = true
```

### 5.3 — Strategic type: ignore

```python
# ✅ Ignore with error code + reason
import legacy_module  # type: ignore[import]  # TODO: migrate to typed lib

# ✅ Ignore specific error on a line
user_id = user_dict["id"]  # type: ignore[index]

# ❌ WRONG: ignore without explanation
import legacy_module  # type: ignore
```

### 5.4 — reveal_type

```python
# reveal_type is a mypy debug tool — no import needed
def process_user(user_id: int):
    user = get_user(user_id)
    reveal_type(user)  # mypy prints: Revealed type is "app.models.User"

    name = user.name
    reveal_type(name)  # mypy prints: Revealed type is "str"

# Useful for debugging generics
stack = Stack()
stack.push(42)
reveal_type(stack)  # mypy prints: Revealed type is "Stack[int]"
```

> **Warning**: `reveal_type` is a debug tool only — remove before committing.
> Available natively since Python 3.11.

---

## Part 6 — FastAPI Integration

### 6.1 — Typed Endpoints with Pydantic

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# Pydantic schemas — runtime validation + static typing
class User(BaseModel):
    id: int
    name: str
    email: str
    age: Optional[int] = None

class UserCreate(BaseModel):
    name: str
    email: str
    age: Optional[int] = None

# Typed endpoints
@app.get("/users/{user_id}")
def get_user(user_id: int) -> User:
    if user_id == 0:
        raise HTTPException(status_code=404, detail="User not found")
    return User(id=user_id, name=f"User {user_id}", email="user@example.com")

@app.get("/users")
def list_users(skip: int = 0, limit: int = 10) -> list[User]:
    return [
        User(id=i, name=f"User {i}", email=f"user{i}@example.com")
        for i in range(skip, skip + limit)
    ]

@app.post("/users")
def create_user(user: UserCreate) -> User:
    return User(id=1, name=user.name, email=user.email, age=user.age)
```

### 6.2 — Async Endpoints

```python
from typing import Optional
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

app = FastAPI()

async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db)
) -> User:
    user = await db.get(User, user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.get("/users")
async def list_users(
    skip: int = 0,
    limit: int = 10,
    db: AsyncSession = Depends(get_db)
) -> list[User]:
    result = await db.execute(
        select(User).offset(skip).limit(limit)
    )
    return result.scalars().all()
```

### 6.3 — Typed Dependency Injection

```python
from typing import Annotated, Optional
from fastapi import FastAPI, Depends, Header, HTTPException

app = FastAPI()

async def get_current_user(
    authorization: Annotated[Optional[str], Header()] = None
) -> User:
    if authorization is None:
        raise HTTPException(status_code=401, detail="Not authenticated")
    return User(id=1, name="Current User", email="user@example.com")

@app.get("/me")
async def get_me(
    current_user: Annotated[User, Depends(get_current_user)]
) -> User:
    return current_user

# Typed dependency chain
class UserService:
    def __init__(self, db: AsyncSession) -> None:
        self.db = db

    async def get_user(self, user_id: int) -> Optional[User]:
        return await self.db.get(User, user_id)

def get_user_service(
    db: Annotated[AsyncSession, Depends(get_db)]
) -> UserService:
    return UserService(db)

@app.get("/users/{user_id}")
async def get_user(
    user_id: int,
    service: Annotated[UserService, Depends(get_user_service)]
) -> User:
    user = await service.get_user(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

---

## Part 7 — Common Patterns

### 7.1 — TypedDict

```python
from typing import TypedDict, Optional

class UserDict(TypedDict):
    id: int
    name: str
    email: str
    age: Optional[int]

def create_user(data: UserDict) -> UserDict:
    return {
        "id": 1,
        "name": data["name"],
        "email": data["email"],
        "age": data.get("age"),
    }

# ✅ Valid
user: UserDict = {
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30
}

# ❌ mypy error: missing required key "email"
invalid_user: UserDict = {
    "id": 1,
    "name": "Bob",
}

# ❌ mypy error: "id" must be int not str
invalid_user2: UserDict = {
    "id": "1",
    "name": "Bob",
    "email": "bob@example.com",
    "age": None
}
```

> **TypedDict vs Pydantic**: TypedDict is purely static, no runtime validation.
> For a REST API, **Pydantic is preferred**. TypedDict is useful for internal
> structures that don't need runtime validation.

### 7.2 — Final

```python
from typing import Final

# Constants that should never change
API_VERSION: Final = "v1"
MAX_RETRIES: Final[int] = 3
DEFAULT_TIMEOUT: Final[float] = 30.0

# ❌ mypy error: Cannot assign to final name
API_VERSION = "v2"
MAX_RETRIES = 5

# Non-inheritable class
from typing import final

@final
class Config:
    DEBUG: Final[bool] = False
    DATABASE_URL: Final[str] = "postgresql://localhost/db"

# ❌ mypy error: Cannot inherit from final class
class AppConfig(Config):
    pass
```

### 7.3 — Self

```python
from typing import Self  # Python 3.11+

class QueryBuilder:
    def __init__(self) -> None:
        self._filters: list[str] = []
        self._limit: int = 100

    def filter(self, condition: str) -> Self:
        self._filters.append(condition)
        return self

    def limit(self, value: int) -> Self:
        self._limit = value
        return self

    def build(self) -> str:
        filters = " AND ".join(self._filters)
        return f"SELECT * FROM table WHERE {filters} LIMIT {self._limit}"

# Typed method chaining
query = (
    QueryBuilder()
    .filter("age > 18")
    .filter("is_active = true")
    .limit(50)
    .build()
)
```

**Why `Self` instead of the class name:**

```python
class Base:
    def clone(self) -> "Base":  # ❌ Returns Base, not the subclass
        ...

class Child(Base):
    pass

child = Child().clone()
reveal_type(child)  # mypy sees: Base — not Child!

# ✅ With Self
class Base:
    def clone(self) -> Self:  # Always returns the real type
        ...

child = Child().clone()
reveal_type(child)  # mypy sees: Child ✅
```

---

## Part 8 — Best Practices & Anti-patterns

### 8.1 — Best Practices

**1. Type public APIs first**
```python
# ✅ Always type what is exposed
def get_user(user_id: int) -> Optional[User]:
    return db.query(User).get(user_id)

# ✅ Acceptable — inference for internal helpers
def _format_name(first, last):
    return f"{first} {last}"
```

**2. Use business aliases**
```python
# ✅ Readable, clear business meaning
UserId = NewType("UserId", int)
OrderStatus = Literal["pending", "approved", "rejected"]

def get_order(user_id: UserId, status: OrderStatus) -> list[Order]:
    ...

# ❌ Primitive, no business meaning
def get_order(user_id: int, status: str) -> list:
    ...
```

**3. Document type: ignore**
```python
# ✅ Explicit and traceable
import legacy_module  # type: ignore[import]  # TODO: migrate to typed lib

# ❌ Silent and opaque
import legacy_module  # type: ignore
```

**4. Use reveal_type during development**
```python
# ✅ mypy debug — remove before committing
reveal_type(result)  # Revealed type is "list[User]"
```

### 8.2 — Anti-patterns

**1. Using Any everywhere**
```python
# ❌ WRONG: disables all verification
from typing import Any

def process(data: Any) -> Any:
    return data.transform()

# ✅ CORRECT: Protocol to type without constraining
from typing import Protocol

class Transformable(Protocol):
    def transform(self) -> dict: ...

def process(data: Transformable) -> dict:
    return data.transform()
```

**2. Ignoring type errors globally**
```toml
# ❌ WRONG
[tool.mypy]
ignore_errors = true

# ✅ CORRECT
[tool.mypy]
strict = true

[[tool.mypy.overrides]]
module = "app.legacy.*"
ignore_errors = true  # Legacy pending migration
```

**3. Not using Optional**
```python
# ❌ WRONG: can crash at runtime
def get_user(user_id: int) -> User:
    return db.get(user_id)  # Can return None!

# ✅ CORRECT: explicit and handled None
def get_user(user_id: int) -> Optional[User]:
    return db.get(user_id)

user = get_user(123)
if user is not None:
    print(user.name)
```

**4. Primitive Obsession**
```python
# ❌ WRONG: nested primitives, unreadable, no business meaning
def process_order(
    data: Dict[str, Union[List[Tuple[int, Optional[str]]], Dict[str, Any]]]
) -> Dict[str, Union[int, str]]:
    ...

# ✅ CORRECT: explicit business aliases
from typing import NewType

ProductId = NewType("ProductId", int)
ProductName = Optional[str]
OrderLine = Tuple[ProductId, ProductName]
OrderData = Dict[str, Union[List[OrderLine], Dict[str, Any]]]
OrderResult = Dict[str, Union[int, str]]

def process_order(data: OrderData) -> OrderResult:
    ...
```

---

## Part 9 — Quick Reference

### 9.1 — uv Commands

```bash
# Installation
uv add --dev mypy

# Run mypy
uv run mypy src/

# Strict mode
uv run mypy --strict src/

# Check a single file
uv run mypy app/main.py

# Show error codes
uv run mypy --show-error-codes src/

# Incremental mode (faster)
uv run mypy --incremental src/

# Generate HTML report
uv run mypy src/ --html-report mypy-report

# Verbose output
uv run mypy --verbose src/
```

### 9.2 — Error Code Reference

```python
# [attr-defined] — attribute does not exist
user.non_existent_field

# [arg-type] — wrong argument type
def greet(name: str) -> str: ...
greet(123)

# [return-value] — wrong return type
def get_id() -> int:
    return "123"

# [assignment] — wrong type on assignment
age: int = "thirty"

# [index] — invalid index
user_dict: Dict[str, int] = {}
user_dict[42]

# [operator] — unsupported operator
"hello" + 5

# [import] — import without stubs or not found
import unknown_lib

# [no-untyped-def] — function without annotations
def process(data):
    return data

# [var-annotated] — variable without inferable annotation
x = []        # ❌
x: list[int] = []  # ✅

# [misc] — miscellaneous error
# Often appears with untyped decorators
```

---

## Resources

- **Official documentation**: https://mypy.readthedocs.io/
- **PEP 484 — Type Hints**: https://peps.python.org/pep-0484/
- **typing module**: https://docs.python.org/3/library/typing.html
- **mypy GitHub**: https://github.com/python/mypy
- **FastAPI + mypy**: https://fastapi.tiangolo.com/tutorial/type-hints/

## Related Skills

- **pytest**: type-safe testing with mypy
- **fastapi**: endpoints with Pydantic runtime validation + mypy
- **ruff**: Python linter — works well in tandem with mypy
- **uv**: package manager — used throughout this skill
