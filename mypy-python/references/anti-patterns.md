# Typing anti-patterns

Full catalogue behind section 3 of the skill. Grouped by what the type does wrong.
Python 3.14 syntax; every ✅ example passes `mypy --strict`.

| Family | Anti-patterns |
|---|---|
| A. Types that say too little | 1 `Any` · 2 Catch-all dict · 3 Bare generics · 4 Primitive obsession · 5 Stringly typed · 6 Boolean blindness · 7 Anonymous tuples |
| B. Types that lie | 8 `cast` and `type: ignore` · 9 Hidden `None` · 10 Sentinel values · 11 Runtime checks instead of types · 12 Global `ignore_errors` |
| C. Types that allow invalid states | 13 Optional soup · 14 Validate instead of parse |
| D. Types that burden the caller | 15 Union return · 16 Over-constrained parameters |
| E. Outdated or erased types | 17 Legacy typing syntax · 18 Untyped decorators |

---

## A. Types that say too little

### 1. `Any` as an escape hatch

`Any` disables checking for everything it touches — and it spreads through
return values.

```python
# ❌
def process(data: Any) -> Any:
    return data.transform()

def first(items: list[Any]) -> Any: ...

# ✅ Describe the capability, or keep the type with a type parameter
class Transformable(Protocol):
    def transform(self) -> dict[str, int]: ...

def process(data: Transformable) -> dict[str, int]:
    return data.transform()

def first[T](items: Sequence[T]) -> T:
    return items[0]
```

Truly unknown input (parsed JSON): type it `object` and narrow, or parse it
into a model at the boundary (14).

### 2. Catch-all dict

```python
# ❌ Keys and value types known only by convention
def greet(user: dict[str, Any]) -> str:
    return f"Hello {user['name']}"

# ✅
@dataclass(frozen=True)
class User:
    name: str
    age: int

def greet(user: User) -> str:
    return f"Hello {user.name}"
```

`dataclass` for internal data, `TypedDict` for dicts crossing a boundary you
don't control (JSON payloads, kwargs), Pydantic for validated external input.

### 3. Bare generics

```python
# ❌ [type-arg] under strict: dict means dict[Any, Any]
def load() -> dict: ...
def tags(items: list) -> list: ...

# ✅
def load() -> Config: ...
def tags(items: Iterable[Item]) -> list[str]: ...
```

### 4. Primitive obsession

Domain concepts as `str` / `int` / `float`: easy to swap, no meaning.

```python
# ❌ from_id and to_id can be swapped silently; float for money
def transfer(from_id: str, to_id: str, amount: float) -> None: ...

# ✅
AccountId = NewType("AccountId", str)

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

def transfer(from_id: AccountId, to_id: AccountId, amount: Money) -> None: ...
```

`NewType` when only the identity of the type matters (ids); a value object when
there are invariants or several fields.

### 5. Stringly typed

`str` for a closed set of values.

```python
# ❌ "publihsed" type-checks
def set_status(status: str) -> None: ...

# ✅
type Status = Literal["draft", "published"]

def set_status(status: Status) -> None: ...
```

`Literal` for plain values; `enum.Enum` when the values carry behaviour or need
iteration.

### 6. Boolean blindness

Positional booleans are unreadable at the call site.

```python
# ❌ What are True and False?
export(doc, True, False)

# ✅ Keyword-only options
def export(doc: Document, *, compress: bool, encrypt: bool) -> None: ...

export(doc, compress=True, encrypt=False)
```

If the boolean switches between two **behaviours** rather than tuning one, split
the function instead (**clean-code-python** §3, flag arguments). An `Enum`
replaces a boolean that may later need a third value.

### 7. Anonymous tuples

```python
# ❌ Which str is the email?
def get_user() -> tuple[str, str, int]: ...

# ✅
class UserInfo(NamedTuple):
    name: str
    email: str
    age: int

def get_user() -> UserInfo: ...
```

Fixed-size tuples stay fine for obvious pairs (`tuple[float, float]` for a
point, `divmod`-like results); name them as soon as positions need a comment.

---

## B. Types that lie

### 8. `cast` and `# type: ignore` to silence an error

```python
# ❌ Unchecked assertion: crashes later when None
user = cast(User, repo.find(user_id))

# ✅ Narrow and handle the case
user = repo.find(user_id)
if user is None:
    raise UserNotFoundError(user_id)
```

- `cast` only where mypy cannot know what you provably know — with a comment
  saying why.
- `# type: ignore` always carries an error code and a reason (skill §2.2).

### 9. Hidden `None`

```python
# ❌ Return type lies: dict.get can return None
def find(user_id: int) -> User:
    return users.get(user_id)          # [return-value]

# ❌ Implicit Optional — rejected by mypy
def connect(timeout: float = None) -> None: ...

# ✅
def find(user_id: int) -> User | None:
    return users.get(user_id)

def connect(timeout: float | None = None) -> None: ...
```

### 10. Sentinel values

`-1`, `""` or `0` meaning "absent" — still typed `int` / `str`, so mypy can't
force the check.

```python
# ❌ -1 if absent: callers index with -1 and get the last element
def find_index(name: str) -> int: ...

# ✅ Absence is visible in the type
def find_index(name: str) -> int | None: ...
```

Raise instead when absence is an error for the caller (**clean-code-python** §7).

### 11. Runtime checks standing in for types

```python
# ❌ hasattr / runtime_checkable as a contract check
if hasattr(obj, "send"):
    obj.send(...)

@runtime_checkable
class Notifier(Protocol): ...
assert isinstance(notifier, Notifier)  # checks method names only, not signatures

# ✅ Type the parameter with the Protocol; mypy checks every call site
def welcome(notifier: Notifier) -> None: ...
```

### 12. Ignoring errors globally

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

---

## C. Types that allow invalid states

### 13. Optional soup

Interdependent optional fields allow combinations that must never exist.

```python
# ❌ status="failed" with a result, both None, both set...
@dataclass
class Job:
    status: str
    result: str | None = None
    error: str | None = None

# ✅ One type per state — invalid combinations can't be built
@dataclass(frozen=True)
class Pending:
    pass

@dataclass(frozen=True)
class Succeeded:
    result: str

@dataclass(frozen=True)
class Failed:
    error: str

Job = Pending | Succeeded | Failed

def describe(job: Job) -> str:
    match job:
        case Pending():
            return "waiting"
        case Succeeded(result=result):
            return f"done: {result}"
        case Failed(error=error):
            return f"failed: {error}"
        case _:
            assert_never(job)      # adding a state without handling it fails mypy
```

### 14. Validate instead of parse

A check returns `bool`, then the value travels on as a plain `str`: nothing
proves it was checked.

```python
# ❌
def is_valid_email(value: str) -> bool: ...
def send(email: str) -> None: ...          # was it validated? by whom?

# ✅ Parse once at the boundary into a type that can't be invalid
@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if "@" not in self.value:
            raise ValueError(f"Invalid email: {self.value!r}")

def send(email: Email) -> None: ...
```

Inside the system, functions take `Email`, never `str`. Same idea as the value
objects of **hexagonal-architecture-python**.

---

## D. Types that burden the caller

### 15. Union return

The caller must test which type came back.

```python
# ❌
def search(query: str) -> User | list[User] | None: ...

# ✅ One stable shape: empty list when nothing matches
def search(query: str) -> list[User]: ...
```

When the result type genuinely depends on an argument, use separate functions
(or `@overload`, see [advanced-types.md](advanced-types.md)).

### 16. Over-constrained parameters

```python
# ❌ Rejects tuples, generators, and list[bool] (list is invariant)
def total(prices: list[int]) -> int:
    return sum(prices)

# ✅ Accept the minimal abstract type you use
def total(prices: Iterable[int]) -> int:
    return sum(prices)
```

Parameters: `Iterable`, `Sequence`, `Mapping` (read-only, abstract). Return
types: concrete (`list`, `dict`).

---

## E. Outdated or erased types

### 17. Legacy typing syntax

```python
# ❌ Pre-3.9 aliases, pre-3.12 type variables, pre-3.14 quoted references
from __future__ import annotations
from typing import Generic, List, Optional, TypeAlias, TypeVar, Union

T = TypeVar("T")
Ids: TypeAlias = List[int]

class Box(Generic[T]):
    def merge(self, other: "Box[T]") -> Optional["Box[T]"]: ...

def f(x: Optional[List[int]]) -> Union[int, str]: ...

# ✅
type Ids = list[int]

class Box[T]:
    def merge(self, other: Box[T]) -> Box[T] | None: ...

def f(x: list[int] | None) -> int | str: ...
```

Abstract types come from `collections.abc`. With `target-version = "py314"`,
`ruff check --select UP --fix` rewrites aliases, unions and quoted references.
Converting to `class Box[T]` / `type X = ...` is an unsafe fix (`--unsafe-fixes`,
review the diff); delete the leftover `TypeVar` declarations and
`from __future__ import annotations` by hand.

### 18. Untyped decorators

```python
# ❌ Erases the decorated function's signature
def retry(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# ✅ A ParamSpec (**P) preserves it
def retry[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(*args, **kwargs)
    return wrapper
```
