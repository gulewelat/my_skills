# Advanced types

Python 3.11 syntax; 3.12+ alternatives noted inline.

## Generics and TypeVar

```python
from collections.abc import Sequence
from typing import Generic, TypeVar

T = TypeVar("T")

def first(items: Sequence[T]) -> T:
    return items[0]

first([1, 2, 3])          # int
first(["a", "b"])         # str
```

**Constrained** vs **bounded** TypeVar:

```python
# Constrained — exactly one of the listed types
Num = TypeVar("Num", int, float)

def add(a: Num, b: Num) -> Num:
    return a + b

add(1, 2)        # ✅ int
add(1.5, 2.5)    # ✅ float
add("a", "b")    # ❌ str not allowed

# Bounded — any subtype of the bound
from typing import Protocol

class HasId(Protocol):
    @property
    def id(self) -> int: ...

E = TypeVar("E", bound=HasId)

def by_id(items: Sequence[E]) -> dict[int, E]:
    return {item.id: item for item in items}   # returns the caller's concrete type
```

**Generic class:**

```python
class Stack(Generic[T]):                       # 3.12+: class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

stack: Stack[int] = Stack()
stack.push(42)        # ✅
stack.push("hello")   # ❌ [arg-type]
```

## Protocol vs ABC

```python
from abc import ABC, abstractmethod
from typing import Protocol

class DrawableABC(ABC):
    @abstractmethod
    def draw(self) -> str: ...

class Circle(DrawableABC):             # explicit inheritance required
    def draw(self) -> str:
        return "circle"

class Drawable(Protocol):
    def draw(self) -> str: ...

class Square:                          # matches structurally
    def draw(self) -> str:
        return "square"

def render(obj: Drawable) -> str:
    return obj.draw()

render(Square())   # ✅
render(Circle())   # ✅ also structural match
```

| Situation | Use |
|---|---|
| Contract between layers (ports) | **Protocol** |
| Typing classes you don't own | **Protocol** (only option) |
| Need runtime error on missing method | ABC |
| Sharing implementation across subclasses | ABC (or composition) |

`@runtime_checkable` enables `isinstance` but checks method **names only**,
not signatures. Don't add it by default.

Protocol attributes: declare read-only members as `@property` so that
implementations using a plain attribute or a property both match.

## Callable

```python
from collections.abc import Callable

def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

Validator = Callable[[str, int], bool]

R = TypeVar("R")

def map_values(func: Callable[[T], R], values: Sequence[T]) -> list[R]:
    return [func(v) for v in values]
```

When the callable has keyword arguments, defaults, or overloads, use a
callback Protocol instead:

```python
class Fetcher(Protocol):
    def __call__(self, url: str, *, timeout: float = 10.0) -> bytes: ...
```

## TypedDict

```python
from typing import NotRequired, TypedDict

class UserPayload(TypedDict):
    id: int
    name: str
    email: NotRequired[str]

payload: UserPayload = {"id": 1, "name": "Alice"}       # ✅
bad: UserPayload = {"id": "1", "name": "Bob"}           # ❌ id must be int
```

Purely static — no runtime validation. Use it to type dicts crossing a boundary
you don't control (JSON payloads, kwargs). For internal data prefer
`@dataclass(frozen=True)`; for validated external input prefer Pydantic.

## Final

```python
from typing import Final, final

MAX_RETRIES: Final = 3
MAX_RETRIES = 5          # ❌ cannot assign to final name

@final
class Settings: ...

class Custom(Settings):  # ❌ cannot inherit from final class
    ...
```

## Self

```python
from typing import Self

class QueryBuilder:
    def __init__(self) -> None:
        self._filters: list[str] = []

    def where(self, condition: str) -> Self:
        self._filters.append(condition)
        return self

class AuditedQueryBuilder(QueryBuilder): ...

reveal_type(AuditedQueryBuilder().where("x"))   # AuditedQueryBuilder, not QueryBuilder
```

Returning `"QueryBuilder"` instead of `Self` loses the subclass type.

## Overloads

When the return type depends on the argument type/value:

```python
from typing import Literal, overload

@overload
def fetch(raw: Literal[True]) -> bytes: ...
@overload
def fetch(raw: Literal[False] = ...) -> str: ...
def fetch(raw: bool = False) -> bytes | str:
    data = b"..."
    return data if raw else data.decode()

reveal_type(fetch(raw=True))   # bytes
reveal_type(fetch())           # str
```

Prefer two separate functions when the overload set grows beyond two or three.
