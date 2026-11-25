---
name: rest-api-python
description: >
  Design and write REST APIs in Python with FastAPI: endpoint naming, HTTP methods
  and status codes, request/response schemas, error format, pagination, versioning,
  and how routers fit a hexagonal architecture. Use when designing, implementing,
  or reviewing HTTP endpoints.
---

# REST APIs in Python

Conventions: Python 3.11+, FastAPI, Pydantic v2, `uv`, `mypy --strict`, `pytest`, `ruff`.

## Reference files

| File | When |
|---|---|
| [references/errors.md](references/errors.md) | Implementing the error handlers and their tests |
| [references/pagination.md](references/pagination.md) | Implementing cursor pagination (encoding, keyset query, tests) |
| [references/testing.md](references/testing.md) | Writing API tests: app fixture with in-memory adapters, contract tests, OpenAPI snapshot |
| [references/versioning.md](references/versioning.md) | Deciding whether a change is breaking; running and deprecating two versions |

---

## 1. Naming endpoints

### The rule: the HTTP method is the verb, the path is a noun

An endpoint is **method + path**. The method already says what happens, so the
path only names **the resource**. A verb in the path repeats the method — or
contradicts it.

| ❌ Verb in the path | ✅ Method + resource |
|---|---|
| `GET /getUsers` | `GET /users` |
| `GET /users/{user_id}/getOrders` | `GET /users/{user_id}/orders` |
| `POST /createUser` | `POST /users` |
| `POST /users/{user_id}/update` | `PATCH /users/{user_id}` |
| `POST /deleteUser/{user_id}` | `DELETE /users/{user_id}` |
| `GET /searchProducts?q=lamp` | `GET /products?q=lamp` |

**This is the opposite of function naming** (clean-code-python §2): a function
has no method to carry the action, so its name needs the verb. The handler
behind an endpoint is a function — it keeps its verb.

```python
from fastapi import APIRouter, status

router = APIRouter(prefix="/users", tags=["users"])


@router.get("/{user_id}")                                # path: noun
def get_user(user_id: int) -> UserOut: ...               # function: verb


@router.get("/{user_id}/orders")
def list_user_orders(user_id: int) -> Page[OrderOut]: ...      # section 5


@router.post("", status_code=status.HTTP_201_CREATED)
def create_user(payload: UserIn) -> UserOut: ...


@router.patch("/{user_id}")
def update_user(user_id: int, payload: UserPatch) -> UserOut: ...


@router.delete("/{user_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_user(user_id: int) -> None: ...
```

### Actions that are not CRUD

Some operations don't map to create/read/update/delete (cancel, approve,
send). Still no verb: **name the result of the action as a resource**.

| ❌ | ✅ |
|---|---|
| `POST /orders/{order_id}/cancel` | `POST /orders/{order_id}/cancellation` |
| `POST /invoices/{invoice_id}/send` | `POST /invoices/{invoice_id}/dispatches` |
| `POST /users/{user_id}/activate` | `PATCH /users/{user_id}` with `{"status": "active"}` |

- If the action only changes a field → `PATCH` the resource.
- If the action has its own data or history (who cancelled, when, why) → it is a
  resource of its own: `POST` creates it, `GET` reads it back.

### Path conventions

- **Plural nouns** for collections: `/users`, `/users/{user_id}`.
- **Lowercase, kebab-case** for multi-word segments: `/purchase-orders`.
- **Path parameters named after the resource**: `{user_id}`, not `{id}` — unambiguous once nested.
- **Nest one level at most**: `/users/{user_id}/orders` ✅; deeper → top-level
  resource with a filter: `/orders?user_id=42` instead of `/users/{user_id}/orders/{order_id}/lines`.
- **Filtering, sorting, searching go in the query string**, not in the path:
  `/products?category=lamps&sort=-price`.
- **No file extensions, no trailing slash**: `/users`, not `/users.json` or `/users/`.

---

## 2. HTTP methods and status codes

### Methods

| Method | Use | Safe | Idempotent | Success code |
|---|---|---|---|---|
| `GET` | Read a resource or a collection | ✅ | ✅ | `200` |
| `POST` | Create a resource in a collection | ❌ | ❌ | `201` + `Location` header |
| `PATCH` | Change some fields | ❌ | ❌* | `200` with the updated resource |
| `PUT` | Replace the whole resource | ❌ | ✅ | `200` |
| `DELETE` | Remove a resource | ❌ | ✅ | `204`, no body |

\* Idempotent in practice when the body sets values (`{"status": "active"}`), not
when it describes operations (`{"increment": 1}`).

- **Safe** = no state change: a `GET` never writes. No `GET /users/{user_id}?action=disable`.
- **Idempotent** = repeating the request leaves the server in the same state.
  Clients and proxies may retry idempotent requests on timeout.
- **`PATCH` by default for updates.** `PUT` only when the client really sends the
  full representation — a missing field then means "set to default", which
  surprises clients that only meant to change one field.
- A second `DELETE` on the same resource returns `404`: the state is unchanged
  (idempotent), the response may differ.

### Status codes

Use the most specific code; clients branch on it.

| Code | When |
|---|---|
| `200 OK` | Read or update succeeded, body returned |
| `201 Created` | Resource created — return it, with `Location: /users/{user_id}` |
| `202 Accepted` | Work queued, not done yet — return a way to follow it |
| `204 No Content` | Success with nothing to return (delete) |
| `400 Bad Request` | Request unusable for a reason other than validation (bad header, malformed cursor) |
| `401 Unauthorized` | Not authenticated — missing or invalid credentials |
| `403 Forbidden` | Authenticated, but not allowed |
| `404 Not Found` | Resource doesn't exist (hidden resources: see project conventions below) |
| `409 Conflict` | Request contradicts current state: duplicate key, concurrent modification |
| `422 Unprocessable Content` | Body or parameters fail validation (FastAPI's default) |
| `429 Too Many Requests` | Rate limit hit — add `Retry-After` |
| `500 Internal Server Error` | Unexpected bug — never raised on purpose |
| `503 Service Unavailable` | Dependency down or maintenance — add `Retry-After` |

Rules:
- **Never `200` with an error in the body.** The status code is the contract.
- **`401` vs `403`**: "who are you?" vs "I know who you are, and no".
- **Use `fastapi.status` constants**, not bare integers.

### Project conventions — don't pick, follow

Two cases have more than one defensible answer. The skill does not choose:

| Case | Options |
|---|---|
| Valid request refused by a business rule (cancel a shipped order) | `409 Conflict` or `422 Unprocessable Content` |
| Resource exists but the caller may not access it | `404` (hides existence) or `403` (explicit) |

1. **Follow the convention already used in the codebase** — look at existing
   routers and the exception-to-status mapping (section 4).
2. **No precedent** → pick one, apply it consistently, and **flag the choice in
   the PR description** so reviewers settle it. The merged PR becomes the convention.
3. Never mix both options within one API.

```python
from fastapi import APIRouter, Request, Response, status

router = APIRouter(prefix="/orders", tags=["orders"])


@router.post("", status_code=status.HTTP_201_CREATED)
def create_order(
    payload: OrderIn, request: Request, response: Response, orders: OrderServiceDep
) -> OrderOut:
    order = orders.place(payload.to_command())
    response.headers["Location"] = str(request.url_for("get_order", order_id=order.id))
    return OrderOut.from_domain(order)
```

`url_for` resolves the route by its function name, version prefix included —
never build URLs by hand.

Domain errors become status codes through a single mapping (section 4) —
routers never `try/except` them.

---

## 3. Request and response schemas

The schemas **are** the API contract. They live in the router layer, separate
from domain entities and ORM models.

### One schema per direction and operation

```python
from datetime import datetime
from typing import Self

from pydantic import BaseModel, ConfigDict, EmailStr, Field


class UserIn(BaseModel):                       # POST body
    model_config = ConfigDict(extra="forbid", frozen=True)

    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


class UserPatch(BaseModel):                    # PATCH body — every field optional
    model_config = ConfigDict(extra="forbid", frozen=True)

    name: str | None = Field(default=None, min_length=1, max_length=100)
    nickname: str | None = None


class UserOut(BaseModel):                      # response
    id: int
    email: str
    name: str
    nickname: str | None
    created_at: datetime

    @classmethod
    def from_domain(cls, user: User) -> Self:
        return cls(
            id=user.id,
            email=user.email,
            name=user.name,
            nickname=user.nickname,
            created_at=user.created_at,
        )
```

- **`<Resource>In` / `<Resource>Patch` / `<Resource>Out`**: input and output
  differ (no `id` or `created_at` in input, never `password` in output). One
  shared model leaks fields one way or the other.
- **Inputs forbid unknown fields** (`extra="forbid"`): a typo like `"nmae"`
  fails with `422` instead of being silently ignored.
- **Never return a domain entity or ORM model.** Map explicitly with
  `from_domain`: renaming a database column must not change the API.
- **The return annotation is the response model** — no `response_model=` needed.
- `EmailStr` needs the extra: `uv add "pydantic[email]"`.

### Validation: shape here, rules in the domain

| Schema validates | Domain validates |
|---|---|
| Types, required fields, formats (email, UUID, date) | Business rules: "email not already used" |
| Lengths, ranges, enums | Rules depending on state: "order not shipped yet" |

A schema validator never queries a database. Failing shape → `422` from
FastAPI; failing rule → domain exception → status from the mapping (section 4).

### PATCH: absent ≠ null

```python
@router.patch("/{user_id}")
def update_user(user_id: int, payload: UserPatch, users: UserServiceDep) -> UserOut:
    changes = payload.model_dump(exclude_unset=True)
    # {}                    → nothing to change
    # {"nickname": None}    → clear the nickname
    # {"name": "Bob"}       → change the name only
    return UserOut.from_domain(users.update(user_id, changes))
```

- Always `exclude_unset=True`: without it, every absent field becomes `None`
  and overwrites stored values.
- A field that cannot be cleared (`name`) must reject an explicit `null` —
  enforce it in the use case, or with a validator on the schema.

### Field formats

- **Dates**: ISO 8601 with timezone, UTC — `datetime` must be timezone-aware.
- **Money**: never `float`. `Decimal` serialized as a string (`"19.90"`), or an
  integer in minor units (`1990`) — plus the currency.
- **Enums**: lowercase string values (`"pending"`), not integers.
- **Field casing** (`snake_case` or `camelCase`) is a project convention, like
  in section 2: follow the existing API; if camelCase, use
  `alias_generator=to_camel` rather than camelCase Python attributes.

---

## 4. Errors

### One format for every error

Every error response — domain error, validation error, unknown route, crash —
has the **same shape**. Clients write one error parser.

**Follow the format already used by the API.** With no precedent, use
**RFC 9457 Problem Details** (`application/problem+json`):

```json
{
  "type": "/problems/order-already-shipped",
  "title": "Order already shipped",
  "status": 409,
  "detail": "Order 7 already shipped",
  "errors": [{"field": "body.email", "message": "value is not a valid email"}]
}
```

- **`type`** is the stable, machine-readable identifier: clients branch on it,
  never on `detail`, which is free text and may change.
- **`errors`** (extension) lists field errors for validation failures only.
- **Never leak internals**: no stack trace, SQL, file path or exception class
  name in a response. A `500` says "unexpected error"; details go to the logs.

### Domain exceptions → status: one mapping

The domain raises its own exceptions and knows nothing about HTTP. The API
translates them **in one place**; routers contain no `try/except` for this.

```python
@dataclass(frozen=True)
class ErrorSpec:
    status: int
    code: str


# 409 vs 422, 404 vs 403: project conventions (section 2) — they live here.
ERRORS: Final[Mapping[type[DomainError], ErrorSpec]] = {
    UserNotFoundError: ErrorSpec(status.HTTP_404_NOT_FOUND, "user-not-found"),
    OrderAlreadyShippedError: ErrorSpec(
        status.HTTP_409_CONFLICT, "order-already-shipped"
    ),
}
```

Exception handlers turn `DomainError`, validation errors, unknown routes and
unexpected crashes into the same Problem Details response. Full implementation
and tests: [references/errors.md](references/errors.md).

Rules:
- **Domain exceptions inherit from one `DomainError`** and are named `...Error`.
- **A new domain exception without a mapping is a bug**: the client gets a
  `500`. A test checks that every `DomainError` subclass is mapped.
- **No `HTTPException` outside routers.** Use cases raise domain exceptions only.
- **`detail` is safe to show to the client** — write domain exception messages
  with that in mind (no secrets, no internal ids that leak structure).

---

## 5. Pagination, filtering, sorting

### Every collection is paginated, from the first version

Adding pagination later changes the response shape — a breaking change.

```python
from typing import Annotated, Generic, Literal, TypeVar

from fastapi import Query
from pydantic import BaseModel, ConfigDict, Field

T = TypeVar("T")


class Page(BaseModel, Generic[T]):
    items: list[T]
    next_cursor: str | None          # None = last page


class OrderFilters(BaseModel):
    model_config = ConfigDict(extra="forbid")

    status: list[OrderStatus] = []                 # ?status=pending&status=paid
    customer_id: int | None = None
    q: str | None = Field(default=None, min_length=2)
    sort: Literal["created_at", "-created_at", "total", "-total"] = "-created_at"
    limit: int = Field(default=20, ge=1, le=100)
    cursor: str | None = None


@router.get("")
def list_orders(
    filters: Annotated[OrderFilters, Query()], orders: OrderServiceDep
) -> Page[OrderOut]: ...
```

- **Never return a bare list**: an object (`items` + metadata) can grow, a list can't.
- **Bounded page size**: a default, and a maximum enforced by validation —
  `limit=500` gets a `422`, not a silent cap.
- **No total count by default**: `COUNT(*)` on a large filtered table costs as
  much as the query. Add it only when a client needs it.

### Cursor or offset

**Follow what the API already does.** For a new API, **cursor** by default:
stable under concurrent inserts, constant cost on deep pages. Offset only when
clients need to jump to page N on a small dataset.

- **Cursors are opaque**: base64-encoded, clients never build or parse them.
- **Malformed cursor → `400`** (`InvalidCursorError` mapped in section 4).
- Comparison table and implementation (encoding, keyset query, `limit + 1`):
  [references/pagination.md](references/pagination.md).

### Filtering and sorting

- **Filters are query parameters named after the fields**: `?status=paid&customer_id=42`.
- **Several values → repeat the parameter**: `?status=pending&status=paid`.
- **Ranges → suffixed names**: `?created_after=2026-01-01&created_before=2026-02-01`.
- **Free-text search → `q`**: `?q=lamp`.
- **Sorting → `sort`, `-` for descending**: `?sort=-created_at`.
- **Sortable and filterable fields are whitelisted** (`Literal`): a raw field name
  never reaches the SQL query.
- **Query parameters live in one model** (`Annotated[Model, Query()]`) with
  `extra="forbid"`: an unknown filter is a `422`, not a silently ignored typo.

---

## 6. Versioning

**Follow the API's existing scheme.** With no precedent: major version as a path
prefix — `/v1/orders`. Visible in logs, links and docs; trivial to route.

```python
app.include_router(orders_v1.router, prefix="/v1")
app.include_router(orders_v2.router, prefix="/v2")   # only when v1 must break
```

- **A new major version only for breaking changes**: removing, renaming or
  retyping a field, making a field required, stricter validation, changing a
  status code, an error `type`, or a default (sort, page size). Adding endpoints,
  optional inputs or response fields is not breaking.
- **Versions share use cases and domain**; only routers and schemas are duplicated.
- **Deprecate before removing**: `deprecated=True` on the route, `Deprecation`
  and `Sunset` headers, then remove at the announced date.

Breaking-change table, side-by-side versions, deprecation headers:
[references/versioning.md](references/versioning.md).

---

## 7. Architecture — routers are driving adapters

In a hexagonal layout (**hexagonal-architecture-python**), a router is a
driving adapter: **parse → call one use case → map the result**.

```python
# adapters/inbound/api/orders.py
def get_cancel_order(request: Request) -> CancelOrder:
    container: Container = request.app.state.container   # set by create_app
    return container.cancel_order


CancelOrderDep = Annotated[CancelOrder, Depends(get_cancel_order)]


@router.post("/{order_id}/cancellation", status_code=status.HTTP_201_CREATED)
def cancel_order(order_id: int, cancel: CancelOrderDep) -> CancellationOut:
    return CancellationOut.from_domain(cancel(order_id))
```

- **No business logic in routers**, no repository or DB session: everything goes
  through a use case.
- **Schemas stay in the adapter**; use cases take primitives or value objects
  and return domain objects.
- **Dependencies as `Annotated[T, Depends(...)]` aliases**, resolved from the
  container built by the composition root — routers never import `main`.
- One router module per resource. A `create_app(container)` factory assembles
  the app (container, error handlers, routers); `main.py` calls it with the
  production container, tests with in-memory adapters.

---

## 8. Testing

Test the API **as a contract**: status codes, response shapes, error format,
validation, pagination. Business rules are tested on the domain and use cases,
without HTTP (**pytest-python**, **hexagonal-architecture-python**).

- **In-memory adapters**, not mocks: the HTTP stack runs for real, storage is fake.
- One test per status code an endpoint can return — the error paths are the contract too.
- **Guard against accidental breaking changes**: snapshot `app.openapi()` in a
  committed `openapi.json` and fail the test on any diff; the diff then shows up
  in the PR, where reviewers judge it against section 6.

Fixtures, contract tests and the OpenAPI snapshot test:
[references/testing.md](references/testing.md).

---

## 9. Anti-patterns

| ❌ | ✅ | § |
|---|---|---|
| Verb in the path: `POST /orders/{id}/cancel` | Resource: `POST /orders/{order_id}/cancellation` | 1 |
| `GET` that changes state | `POST` / `PATCH` / `DELETE` | 2 |
| `PUT` for a partial update | `PATCH` + `exclude_unset=True` | 2, 3 |
| `200` with `{"error": ...}` in the body | The status code that matches | 2 |
| One schema for input and output | `<Resource>In` / `Patch` / `Out` | 3 |
| Returning the ORM model or domain entity | `Out.from_domain(...)` | 3 |
| `try/except` → `HTTPException` in every route | One exception-to-status mapping | 4 |
| `HTTPException` raised in a use case | Domain exception, translated by the API | 4 |
| Stack trace or SQL in an error response | Generic `500`, details in logs | 4 |
| Unbounded list endpoint returning `list[T]` | `Page[T]` with a maximum `limit` | 5 |
| `sort` value passed straight to SQL | Whitelisted `Literal` | 5 |
| Renaming a response field in place | New major version, old one deprecated | 6 |
| Business logic or DB session in a router | One use case call | 7 |
