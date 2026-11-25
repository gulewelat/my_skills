---
name: mypy-python
description: Static type checking of Python 3.14+ code with mypy in strict mode. Covers annotations (builtins, unions, PEP 695 generics, Literal, NewType, Protocol, TypeIs), mypy configuration in pyproject.toml, type-ignore discipline, error codes, and typing anti-patterns. Use when adding type hints, configuring mypy, reviewing typed code, or fixing mypy errors.
---

# mypy — Static Type Checking for Python

Conventions: Python 3.14+, `uv`, `mypy --strict`, contracts via `typing.Protocol`.
Modern syntax only:
- builtin generics (`list[int]`) and `X | None` — never `typing.List` / `Optional` / `Union`;
- PEP 695 type parameters (`def f[T]`, `class C[T]`, `type X = ...`) — never
  `TypeVar(...)`, `Generic[T]`, `TypeAlias`;
- forward references without quotes (`-> User`, annotations are lazy since 3.14) —
  no `from __future__ import annotations`.

## Reference files

Load only when the task needs them:

| File | When |
|---|---|
| [references/advanced-types.md](references/advanced-types.md) | Bounded/constrained generics, defaults, Protocol vs ABC, Callable, TypedDict, Final, Self, `@override`, `@deprecated`, overloads |
| [references/incremental-adoption.md](references/incremental-adoption.md) | Introducing mypy on an untyped or partially typed codebase |
| [references/fastapi.md](references/fastapi.md) | Typing FastAPI endpoints, dependencies, Pydantic models |
| [references/anti-patterns.md](references/anti-patterns.md) | Reviewing typed code: 18 anti-patterns with before/after |

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
`__init__` is generated and checked. A class can reference itself or a class
defined later without quotes: `def parent(self) -> Node | None`.

### 1.4 — Literal, NewType, aliases

```python
from typing import Literal, NewType

type Status = Literal["pending", "approved", "rejected"]
type Headers = dict[str, str]
UserId = NewType("UserId", int)          # distinct type, zero runtime cost

def update(user_id: UserId, status: Status) -> None: ...

update(UserId(1), "approved")   # ✅
update(1, "approved")           # ❌ [arg-type] int is not UserId
update(UserId(1), "done")       # ❌ [arg-type] not a valid Literal
```

| Need | Use |
|---|---|
| Closed set of string/int values | `Literal` (or `enum.Enum` if behaviour attached) |
| Same primitive, different meaning (ids) | `NewType` |
| Name a long type expression | `type` alias |
| Structured data with invariants | `@dataclass(frozen=True)` |

### 1.5 — Generics

```python
def first[T](items: Sequence[T]) -> T:
    return items[0]

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

first([1, 2, 3])                # int
stack = Stack[str]()
stack.push(42)                  # ❌ [arg-type]
```

Type parameters are declared in brackets and scoped to the function or class.
Bounds, constraints, defaults, `ParamSpec`:
[references/advanced-types.md](references/advanced-types.md).

### 1.6 — Protocol (default for contracts)

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

### 1.7 — Narrowing

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
`cast` (see anti-patterns). A reusable check becomes a `TypeIs` function:

```python
from typing import TypeIs

def is_user(value: object) -> TypeIs[User]:
    return isinstance(value, User)
```

The narrowed type must be a subtype of the parameter type: `list[object]` →
`list[str]` is rejected (`list` is invariant); use `Sequence[object]`.

---

## 2 — Configuration

### 2.1 — pyproject.toml (strict baseline)

```toml
[tool.mypy]
python_version = "3.14"
files = ["src", "tests"]
strict = true

# Not included in --strict, worth enabling
warn_unreachable = true
enable_error_code = [
    "ignore-without-code",
    "redundant-expr",
    "truthy-bool",
    "deprecated",          # calls to @deprecated APIs
    "explicit-override",   # overriding methods must use @override
]

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
`from typing import reveal_type`. Remove before committing.

---

## 3 — Anti-patterns

Spot the symptom, apply the fix. Examples and nuances for each:
[references/anti-patterns.md](references/anti-patterns.md).

| # | Symptom | Fix |
|---|---|---|
| 1 | `Any` in a signature | `Protocol`, type parameter `[T]`, or `object` + narrowing |
| 2 | `dict[str, Any]` carrying a known shape | `dataclass` · `TypedDict` at boundaries · Pydantic for input |
| 3 | Bare `dict` / `list` | Type parameters, or a named type |
| 4 | `str` / `int` / `float` for ids, money, emails | `NewType` or value object; `Decimal` for money |
| 5 | `str` for a closed set of values | `Literal` or `Enum` |
| 6 | `f(doc, True, False)` | Keyword-only flags, `Enum`, or two functions |
| 7 | `tuple[str, str, int]` returned | `NamedTuple` or `dataclass` |
| 8 | `cast(...)`, bare `# type: ignore` | Narrow; fix the type at its source; scoped ignore with code |
| 9 | `-> User` that can return `None`; `x: T = None` | `-> User \| None`; `x: T \| None = None` |
| 10 | `-1` / `""` meaning "absent" | `T \| None`, or raise |
| 11 | `hasattr`, `isinstance(x, SomeProtocol)` as contract | Parameter typed with the `Protocol` |
| 12 | `ignore_errors = true` globally | `strict = true` + scoped overrides |
| 13 | Optional fields valid only in some combinations | Union of state types + `match` / `assert_never` |
| 14 | `is_valid_x(value) -> bool`, then `value: str` travels on | Parse at the boundary into a type that can't be invalid |
| 15 | `-> A \| list[A] \| None` | One stable return type, or separate functions |
| 16 | `list[T]` parameter only iterated | `Iterable[T]` / `Sequence[T]` / `Mapping[K, V]` |
| 17 | `List`, `Optional`, `TypeVar(...)`, `Generic[T]`, quoted forward refs | `list[int]`, `X \| None`, `def f[T]`, `class C[T]`, unquoted names |
| 18 | Untyped decorator | `def deco[**P, R](func: Callable[P, R]) -> Callable[P, R]` |

Rule of thumb: **a type should make wrong code fail mypy** — if a bug type-checks,
the type says too little (A), lies (B), or allows invalid states (C).

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
