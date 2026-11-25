---
name: mypy-python
description: Static type checking of Python code with mypy in strict mode. Covers annotations (builtins, unions, Literal, NewType, Protocol, Callable), mypy configuration in pyproject.toml, type-ignore discipline, error codes, and typing anti-patterns. Use when adding type hints, configuring mypy, reviewing typed code, or fixing mypy errors.
---

# mypy — Static Type Checking for Python

Conventions: Python 3.11+, `uv`, `mypy --strict`, contracts via `typing.Protocol`.
Modern syntax only: builtin generics (`list[int]`) and `X | None` — never
`typing.List` / `Optional` / `Union`.

## Reference files

Load only when the task needs them:

| File | When |
|---|---|
| [references/advanced-types.md](references/advanced-types.md) | Generics / TypeVar, Protocol vs ABC in depth, Callable, TypedDict, Final, Self, overloads |
| [references/incremental-adoption.md](references/incremental-adoption.md) | Introducing mypy on an untyped or partially typed codebase |
| [references/fastapi.md](references/fastapi.md) | Typing FastAPI endpoints, dependencies, Pydantic models |

---

## 1 — Annotations

### 1.1 — Variables and functions

```python
name: str = "Alice"
count = 10                      # inferred: int — don't annotate the obvious

user_id: int | str = 123
email: str | None = None

def greet(name: str) -> str:
    return f"Hello, {name}"

def log_message(message: str) -> None:   # explicit None return, always
    print(message)

def fail(message: str) -> NoReturn:      # from typing; always raises
    raise ValueError(message)
```

### 1.2 — Collections

```python
numbers: list[int] = [1, 2, 3]
ages: dict[str, int] = {"Alice": 30}
ids: set[int] = {1, 2, 3}
point: tuple[float, float] = (10.5, 20.3)       # fixed length
values: tuple[int, ...] = (1, 2, 3)             # variable length

items: list[str] = []          # empty literal needs an annotation [var-annotated]
```

Parameters take the **narrowest abstract type** that works; return types are
concrete:

```python
from collections.abc import Iterable, Mapping, Sequence

def total(prices: Iterable[float]) -> float: ...          # any iterable
def first(items: Sequence[str]) -> str: ...               # indexable, read-only
def render(headers: Mapping[str, str]) -> list[str]: ...  # read-only mapping
```

### 1.3 — Classes

```python
from typing import Self

class User:
    def __init__(self, name: str, age: int, email: str | None = None) -> None:
        self.name = name
        self.age = age
        self.email = email

    @classmethod
    def from_row(cls, row: Mapping[str, str]) -> Self:
        return cls(name=row["name"], age=int(row["age"]), email=row.get("email"))
```

Prefer `@dataclass(frozen=True)` for value objects: fields are typed once,
`__init__` is generated and checked.

### 1.4 — Literal, NewType, aliases

```python
from typing import Literal, NewType, TypeAlias

Status = Literal["pending", "approved", "rejected"]
UserId = NewType("UserId", int)          # distinct type, zero runtime cost
Headers: TypeAlias = dict[str, str]      # 3.12+: type Headers = dict[str, str]

def update(user_id: UserId, status: Status) -> None: ...

update(UserId(1), "approved")   # ✅
update(1, "approved")           # ❌ [arg-type] int is not UserId
update(UserId(1), "done")       # ❌ [arg-type] not a valid Literal
```

| Need | Use |
|---|---|
| Closed set of string/int values | `Literal` (or `enum.Enum` if behaviour attached) |
| Same primitive, different meaning (ids) | `NewType` |
| Name a long type expression | type alias |
| Structured data with invariants | `@dataclass(frozen=True)` |

### 1.5 — Protocol (default for contracts)

```python
from typing import Protocol

class Notifier(Protocol):
    def send(self, to: str, body: str) -> None: ...

class SmtpNotifier:                      # no inheritance needed
    def send(self, to: str, body: str) -> None: ...

def welcome(notifier: Notifier, to: str) -> None:
    notifier.send(to, "Welcome!")

welcome(SmtpNotifier(), "a@b.c")         # ✅ checked structurally by mypy
```

Use `ABC` only when you need a runtime error on missing methods or shared
implementation. Details: [references/advanced-types.md](references/advanced-types.md).

### 1.6 — Narrowing

```python
def display(user: User | None) -> str:
    if user is None:
        return "anonymous"
    return user.name            # narrowed to User

def size(value: int | str) -> int:
    if isinstance(value, str):
        return len(value)
    return value                # narrowed to int
```

Narrow with `is None`, `isinstance`, early return, or `match`. Never with
`cast` (see anti-patterns).

---

## 2 — Configuration

### 2.1 — pyproject.toml (strict baseline)

```toml
[tool.mypy]
python_version = "3.11"
files = ["src", "tests"]
strict = true

# Not included in --strict, worth enabling
warn_unreachable = true
enable_error_code = ["ignore-without-code", "redundant-expr", "truthy-bool"]

# Reporting
pretty = true
show_column_numbers = true

# Third-party packages without stubs — per module, never globally
[[tool.mypy.overrides]]
module = ["some_untyped_lib.*"]
ignore_missing_imports = true
```

`strict = true` enables, among others: `disallow_untyped_defs`,
`disallow_any_generics`, `disallow_untyped_calls`, `warn_return_any`,
`warn_unused_ignores`, `no_implicit_reexport`, `strict_equality`.

Tests are type-checked too: annotate test functions `-> None` and fixtures
with their yielded type.

Before ignoring a missing import, look for stubs: `uv add --dev types-requests`
(or the package's `py.typed`).

### 2.2 — `# type: ignore` discipline

```python
# ✅ Error code + reason
value = legacy.load()  # type: ignore[no-untyped-call]  # legacy has no stubs, TICKET-42

# ❌ Bare ignore — hides every future error on that line
value = legacy.load()  # type: ignore
```

- Always scope to an error code; `ignore-without-code` makes this mandatory.
- `warn_unused_ignores` (in strict) flags ignores that are no longer needed.
- A function needing several ignores needs a typed wrapper instead.

### 2.3 — `reveal_type` for debugging

```python
reveal_type(result)   # mypy: Revealed type is "builtins.list[app.models.User]"
```

Recognised by mypy without import; at runtime it needs
`from typing import reveal_type` (3.11+). Remove before committing.

---

## 3 — Anti-patterns

### 3.1 — `Any` as an escape hatch

```python
# ❌ Disables checking for everything it touches — and it spreads
def process(data: Any) -> Any:
    return data.transform()

# ✅ Describe what you need
class Transformable(Protocol):
    def transform(self) -> dict[str, int]: ...

def process(data: Transformable) -> dict[str, int]:
    return data.transform()
```

Truly unknown input (parsed JSON): type it `object` and narrow, or validate
into a model at the boundary.

### 3.2 — `cast` to silence an error

```python
# ❌ cast is an unchecked assertion: lies to mypy if wrong
user = cast(User, repo.find(user_id))
print(user.name)                      # crashes at runtime when None

# ✅ Narrow and handle the case
user = repo.find(user_id)
if user is None:
    raise UserNotFound(user_id)
print(user.name)
```

`cast` is acceptable only where mypy cannot know what you provably know
(e.g. after a library call typed too loosely) — with a comment saying why.

### 3.3 — Hiding `None`

```python
# ❌ Return type lies: dict.get can return None
def find(user_id: int) -> User:
    return users.get(user_id)          # [return-value]

# ❌ Implicit Optional — rejected by mypy (no_implicit_optional)
def connect(timeout: float = None) -> None: ...

# ✅
def find(user_id: int) -> User | None:
    return users.get(user_id)

def connect(timeout: float | None = None) -> None: ...
```

### 3.4 — Bare generics and loose containers

```python
# ❌ [type-arg] under strict: dict means dict[Any, Any]
def load() -> dict: ...
def tags(items: list) -> list: ...

# ❌ Technically typed, still meaningless
def load() -> dict[str, Any]: ...

# ✅ Name the shape
@dataclass(frozen=True)
class Config:
    host: str
    port: int

def load() -> Config: ...
```

### 3.5 — Over-constrained parameters

```python
# ❌ Rejects tuples, generators, and list[bool] (list is invariant)
def total(prices: list[float]) -> float:
    return sum(prices)

# ✅ Accept the abstract type you actually use
def total(prices: Iterable[float]) -> float:
    return sum(prices)
```

### 3.6 — Primitive obsession

```python
# ❌ Unreadable, no meaning, `Any` inside
def process_order(
    data: dict[str, list[tuple[int, str | None]] | dict[str, Any]],
) -> dict[str, int | str]: ...

# ✅ Domain types
ProductId = NewType("ProductId", int)

@dataclass(frozen=True)
class OrderLine:
    product_id: ProductId
    label: str | None

@dataclass(frozen=True)
class OrderResult:
    order_id: int
    status: Status

def process_order(lines: Sequence[OrderLine]) -> OrderResult: ...
```

### 3.7 — Ignoring errors globally

```toml
# ❌ mypy runs and checks nothing
[tool.mypy]
ignore_errors = true
ignore_missing_imports = true

# ✅ Strict globally, exceptions scoped and justified
[tool.mypy]
strict = true

[[tool.mypy.overrides]]
module = "migrations.*"        # generated code
ignore_errors = true
```

### 3.8 — Legacy `typing` aliases

```python
# ❌ Deprecated aliases (PEP 585 / 604)
from typing import Dict, List, Optional, Union
def f(x: Optional[List[int]]) -> Union[int, str]: ...

# ✅
def f(x: list[int] | None) -> int | str: ...
```

Import abstract types from `collections.abc` (`Iterable`, `Callable`,
`Mapping`), not from `typing`. `ruff` rules `UP006`/`UP007`/`UP035` automate this.

### 3.9 — Runtime checks standing in for types

```python
# ❌ hasattr / runtime_checkable as a contract check
if hasattr(obj, "send"):
    obj.send(...)

@runtime_checkable
class Notifier(Protocol): ...
assert isinstance(notifier, Notifier)  # checks method names only, not signatures

# ✅ Type the parameter with the Protocol; mypy checks at every call site
def welcome(notifier: Notifier) -> None: ...
```

### 3.10 — Untyped decorators

```python
# ❌ Erases the decorated function's signature [misc] / [no-untyped-def]
def retry(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# ✅ ParamSpec preserves it
P = ParamSpec("P")
R = TypeVar("R")

def retry(func: Callable[P, R]) -> Callable[P, R]:   # 3.12+: def retry[**P, R](...)
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(*args, **kwargs)
    return wrapper
```


---

## 4 — Quick Reference

### 4.1 — Commands

```bash
uv run mypy                       # uses `files` from pyproject.toml
uv run mypy src/app/main.py       # single file
uv run mypy --strict src/         # strict without config
uv run mypy --html-report mypy-report src/   # needs lxml
```

### 4.2 — Frequent error codes

| Code | Meaning | Usual fix |
|---|---|---|
| `[arg-type]` | Wrong argument type | Fix the caller or widen the parameter |
| `[return-value]` | Wrong return type | Often a missing `| None` |
| `[assignment]` | Incompatible assignment | Annotate the variable's real type |
| `[union-attr]` | Attribute on a possibly-`None` value | Narrow first |
| `[attr-defined]` | Attribute doesn't exist | Typo, or missing Protocol member |
| `[type-arg]` | Bare generic (`dict`, `list`) | Add parameters |
| `[no-untyped-def]` | Function missing annotations | Annotate params and return |
| `[no-untyped-call]` | Calling an untyped function from typed code | Type the callee or wrap it |
| `[var-annotated]` | Empty container needs a type | `x: list[int] = []` |
| `[import-untyped]` | Package has no stubs | Install `types-*`, else scoped override |
| `[import-not-found]` | Module not found | Check environment / path |
| `[unused-ignore]` | Ignore no longer needed | Remove it |

## Related skills

- **pytest-python**: tests are type-checked with the same config
- **hexagonal-architecture-python**: ports as `Protocol`, mypy as conformance check
- **clean-code-python**: naming and function design behind good types
