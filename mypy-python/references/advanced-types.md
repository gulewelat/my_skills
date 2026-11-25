# Advanced types

Python 3.14 syntax: PEP 695 type parameters, `type` aliases, unquoted forward
references. Every example passes `mypy --strict`.

## Generics

Type parameters are declared in brackets on the function, class or alias that
uses them — no module-level `TypeVar`.

```python
from collections.abc import Sequence

def first[T](items: Sequence[T]) -> T:
    return items[0]

first([1, 2, 3])          # int
first(["a", "b"])         # str
```

**Bounded** vs **constrained**:

```python
from typing import Protocol

class HasId(Protocol):
    @property
    def id(self) -> int: ...

# Bounded — any subtype of the bound; the caller's concrete type is preserved
def by_id[E: HasId](items: Sequence[E]) -> dict[int, E]:
    return {item.id: item for item in items}

# Constrained — exactly one of the listed types
def add[N: (int, float)](a: N, b: N) -> N:
    return a + b

add(1, 2)        # ✅ int
add(1.5, 2.5)    # ✅ float
add("a", "b")    # ❌ str not allowed
```

**Generic class and alias:**

```python
class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

type Pair[T] = tuple[T, T]

stack = Stack[int]()
stack.push(42)        # ✅
stack.push("hello")   # ❌ [arg-type]
```

**Defaults** — the parameter can be omitted by users of the type:

```python
class Response[T = bytes]:
    def __init__(self, body: T) -> None:
        self.body = body

def read(response: Response) -> bytes:   # Response means Response[bytes]
    return response.body
```

Variance is inferred from usage — no `covariant=True` to declare.

## Protocol vs ABC

```python
from abc import ABC, abstractmethod

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

## Callable and ParamSpec

```python
from collections.abc import Callable

def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

type Validator = Callable[[str, int], bool]

def map_values[T, R](func: Callable[[T], R], values: Sequence[T]) -> list[R]:
    return [func(v) for v in values]
```

Decorators keep the wrapped signature with a `ParamSpec` (`**P`):

```python
import functools

def logged[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(*args, **kwargs)
    return wrapper
```

When the callable has keyword arguments, defaults, or overloads, use a
callback Protocol instead:

```python
class Fetcher(Protocol):
    def __call__(self, url: str, *, timeout: float = 10.0) -> bytes: ...
```

## TypedDict

```python
from typing import NotRequired, ReadOnly, TypedDict

class UserPayload(TypedDict):
    id: ReadOnly[int]              # assignment is a type error
    name: str
    email: NotRequired[str]

payload: UserPayload = {"id": 1, "name": "Alice"}       # ✅
payload["id"] = 2                                       # ❌ read-only key
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

Returning `QueryBuilder` instead of `Self` loses the subclass type.

## `@override` and `@deprecated`

```python
from typing import override
from warnings import deprecated

class Exporter:
    def export(self) -> bytes: ...

class PdfExporter(Exporter):
    @override                      # renaming Exporter.export now fails mypy
    def export(self) -> bytes: ...

@deprecated("Use first() instead")
def head[T](items: Sequence[T]) -> T:
    return items[0]

head([1])                          # ❌ [deprecated] — and a DeprecationWarning at runtime
```

Both are reported only with `enable_error_code = ["explicit-override",
"deprecated"]` (included in the skill's baseline config).

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
