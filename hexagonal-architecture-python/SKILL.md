---
name: hexagonal-architecture-python
description: Design, structure, and review Python applications using Hexagonal Architecture (Ports & Adapters). Covers project layout, the domain layer (entities, value objects, domain services), ports via typing.Protocol, use cases, driving and driven adapters, dependency injection at the composition root, and the testing strategy this architecture enables. Use when building or refactoring a Python app with a domain layer isolated from infrastructure, or when scaffolding a project with domain/application/adapters layers.
---

# Hexagonal Architecture in Python

Hexagonal Architecture — also called **Ports & Adapters**, near-synonymous with **Onion** and **Clean Architecture** — reduces to the **Dependency Inversion Principle**: business rules must not depend on infrastructure. The domain sits in the center; **ports** define *what* the application needs and offers as abstract contracts; **adapters** provide the concrete *how* (a Postgres adapter, a FastAPI adapter, an in-memory fake for tests). Swapping infrastructure means writing a new adapter and touching zero domain code.

> **The Dependency Rule:** source code dependencies point only inward. The domain knows nothing about use cases; use cases know nothing about adapters; nothing knows about the framework except the framework's own adapter. Every pattern below exists to uphold this rule.

> For **event-driven** patterns on top of this foundation — domain events, message bus, commands vs events, CQRS — see the companion skill `domain_events_python_SKILL.md`.

Conventions: Python 3.14+, `uv` as package manager, `mypy` strict, `pytest`, ports as `typing.Protocol` (structural typing).

**Vocabulary** — shared with Matt Pocock's `codebase-design` skill:

| Term | Meaning here |
|---|---|
| **Module** | Anything with an interface and an implementation — a function, a class, a package |
| **Seam** | Where a module's interface lives; behaviour changes there without editing the caller |
| **Port** | The interface at an application seam, written as a `Protocol` |
| **Adapter** | A concrete thing that satisfies a port: SQL repository, FastAPI router, in-memory fake |
| **Depth** | Behaviour per unit of interface — keep ports small and use cases deep |

A port is a **real** seam: every port has at least two adapters, the production
one and the in-memory one used by tests — so the indirection pays for itself
from day one. Two conditions keep it true:

- The in-memory adapter actually exists and the tests use it. Without it, the
  port has one adapter: a hypothetical seam, i.e. speculative abstraction.
- Ports sit only where something varies — the edges with infrastructure
  (database, email, clock, external APIs). Pure domain code has nothing to
  swap and gets no port.

"Boundary" is avoided (it collides with DDD's bounded context); say seam or interface.

---

## 1. Project structure

Organize the source tree by architectural layer. Each directory maps to one ring of the hexagon, and the import rules between them express the Dependency Rule.

```
src/
  <package>/
    domain/           # innermost — pure business logic, zero infra deps
      entities.py         # objects with identity
      value_objects.py    # immutable objects without identity
      services.py         # domain logic spanning multiple entities
      exceptions.py       # domain-specific errors
    ports/            # the contracts (interfaces) the app speaks through
      inbound.py          # what the app offers — use case interfaces (optional)
      outbound.py         # what the app needs — repositories, notifiers, clocks
    application/      # use cases — orchestrate domain + outbound ports
      use_cases.py        # one class/function per use case
    adapters/         # outermost — concrete implementations of ports
      inbound/            # driving: FastAPI routes, CLI, message consumers
      outbound/           # driven: SQL repositories, email senders, HTTP clients
    bootstrap.py      # composition root — wires adapters to ports
    config.py         # settings, env vars
tests/
  unit/               # domain + use cases against fakes — no I/O
  integration/        # real adapters against real infra (DB, broker)
  e2e/                # full stack through the driving adapter
```

### The dependency direction

Imports flow **inward only**:

- `domain/` imports nothing from other layers and no third-party infrastructure (`sqlalchemy`, `fastapi`, `httpx`, `redis`…). The standard library (`enum`, `datetime`, `decimal`, `dataclasses`, `uuid`, `typing`) is allowed — the rule is "no external *infrastructure*", not "no imports".
- `ports/` may import from `domain/` (signatures may reference domain objects), nothing else.
- `application/` imports from `domain/` and `ports/` — the abstractions, never a concrete adapter.
- `adapters/` may import from all inner layers plus infrastructure libraries. The only place `sqlalchemy` or `fastapi` may appear. Adapters import the port they implement.
- `bootstrap.py` sits outside the rings: the one place that knows every concrete adapter, because its job is to instantiate and inject them.

### Note — multiple bounded contexts (style B)

When a system has several independent bounded contexts, flip the top levels: split by context first, then by layer within each (`src/<package>/<context>/{domain,ports,application,adapters}/`, plus an optional `shared_kernel/`). Each context is its own hexagon with its own dependency rule. Contexts communicate only through published interfaces (a port, an API, or domain events — see companion skill), never by reaching into each other's `domain/`. Keep `shared_kernel/` tiny; a fat one re-couples what you separated.

---

## 2. Domain layer

The innermost ring: business rules in plain Python, written as if databases, frameworks, and APIs did not exist. Four kinds of thing live here.

### Value objects

No identity — defined entirely by attributes (one or several); two instances with equal attributes are interchangeable. **Immutable**: to "modify" one, build a new one. Idiomatic Python: a frozen dataclass, with invariants checked in `__post_init__` so an invalid instance can never exist — invariants may span several attributes.

```python
# domain/value_objects.py
from dataclasses import dataclass


@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self) -> None:
        if "@" not in self.value:  # illustrative invariant — any rule fits here
            raise ValueError(f"Invalid email: {self.value!r}")
```

`frozen=True` gives immutability plus value-based `__eq__`/`__hash__` for free.

### Entities

An **identity** that persists through change: attributes vary, the identifier does not. Equality is by identity, not attributes — deliberately opposite to value objects. Behaviour lives *on* the entity: expose methods that enforce the business rules, don't mutate public fields from outside.

```python
# domain/entities.py
from dataclasses import dataclass, field
from uuid import UUID, uuid4

from .value_objects import Email
from .exceptions import AccountAlreadyActiveError


@dataclass
class User:
    email: Email
    id: UUID = field(default_factory=uuid4)
    is_active: bool = False

    def activate(self) -> None:
        if self.is_active:
            raise AccountAlreadyActiveError(self.id)
        self.is_active = True

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, User):
            return NotImplemented
        return self.id == other.id

    def __hash__(self) -> int:
        return hash(self.id)
```

The rule "you cannot activate an already-active account" lives *inside* the entity, not in a use case or controller.

### Domain services

For the rare rule that spans **several** entities and belongs to none: a stateless function/class operating on domain objects, still pure domain code (no infrastructure, no orchestration — that's the application layer's job).

```python
# domain/services.py
from .entities import User


def is_duplicate(new_user: User, existing_users: list[User]) -> bool:
    """A user is a duplicate if the email already exists in the set."""
    return any(u.email == new_user.email for u in existing_users)
```

Last resort only: overusing services drains behaviour out of entities → **anemic domain model** (entities reduced to data holders). Keep behaviour on the objects that own it.

### Domain exceptions

When a business rule is violated, raise an error type from the domain vocabulary — never an infrastructure error (`HTTPException`, DB errors). Outer adapters translate each domain error into their own response (HTTP 409, CLI message); the domain never knows which.

```python
# domain/exceptions.py
from uuid import UUID


class DomainError(Exception):
    """Base class for all domain errors."""


class AccountAlreadyActiveError(DomainError):
    def __init__(self, user_id: UUID) -> None:
        super().__init__(f"Account {user_id} is already active")
        self.user_id = user_id


class UserNotFoundError(DomainError):
    def __init__(self, user_id: UUID) -> None:
        super().__init__(f"User {user_id} not found")
        self.user_id = user_id
```

A shared `DomainError` base lets each driving adapter catch the whole family in one place, at its seam.

---

## 3. Ports

A port is a contract, expressed as a `typing.Protocol`. Structural typing: an adapter conforms by having the right method signatures — no inheritance, no registration, no import of the port from the adapter side required. `mypy` verifies conformance wherever an adapter is passed to code expecting the port.

### Outbound ports — what the application needs

Declared from the **application's point of view** ("I need to store users, I need to notify someone"), named after the need, not the technology (`UserRepository`, not `PostgresClient`; `Notifier`, not `SmtpEmailSender`). Signatures speak the domain language: they take and return domain objects, never ORM rows, HTTP payloads, or driver types.

```python
# ports/outbound.py
from typing import Protocol
from uuid import UUID

from <package>.domain.entities import User


class UserRepository(Protocol):
    def get(self, user_id: UUID) -> User | None: ...
    def save(self, user: User) -> None: ...  # insert or update


class Notifier(Protocol):
    def send(self, recipient: User, message: str) -> None: ...
```

Methods are declared with `...` bodies only — a `Protocol` carries no implementation.

### Inbound ports — what the application offers

Optional in Python. The use case class/function itself already exposes a callable signature, which *is* the inbound contract; a separate `Protocol` for it adds value only when several driving adapters must be checked against one formal interface, or when you want to fake the use case itself in adapter tests. Default: skip them, keep `ports/inbound.py` empty or absent until proven needed.

The asymmetry with outbound is structural: a driving adapter already depends *inward* on the application — nothing to invert. An outbound call would depend *outward*; the outbound port is the inversion that flips it. Outbound ports are therefore never optional — without them the application imports concrete infrastructure and the hexagon collapses.

### Rules

- Ports live in `ports/`, not next to their adapters — the contract belongs to the application, the implementation to the edge.
- Keep ports **small and role-shaped** (interface segregation): a port per need, not one giant `Repository` with 15 methods. A use case that needs only `get` should depend on a port that only has `get`.
- Don't add `@runtime_checkable` by default. It enables `isinstance(x, UserRepository)` but only checks method *names*, not signatures — a false sense of safety. `mypy` is the conformance check.
- Async: if the adapter will be async, the port is async (`async def get(...)`). Sync and async are different contracts; don't mix them in one port.

---

## 4. Application layer — use cases

A use case is one application action. It **orchestrates**: fetch through ports, invoke domain behaviour, persist through ports, return. Business rules never live here — the `if` that encodes policy belongs to the entity; the use case only sequences the calls.

```python
# application/use_cases.py
from uuid import UUID

from <package>.domain.entities import User
from <package>.domain.exceptions import UserNotFoundError
from <package>.ports.outbound import Notifier, UserRepository


class ActivateUser:
    def __init__(self, users: UserRepository, notifier: Notifier) -> None:
        self._users = users
        self._notifier = notifier

    def execute(self, user_id: UUID) -> User:
        user = self._users.get(user_id)
        if user is None:
            raise UserNotFoundError(user_id)
        user.activate()          # business rule enforced by the entity
        self._users.save(user)
        self._notifier.send(user, "Your account is now active")
        return user
```

Ports arrive by **constructor injection**, typed as the `Protocol`s — `mypy` accepts any conforming adapter, real or fake.

### Rules

- Imports: `domain/` and `ports/` only. No adapters, no framework, no infrastructure.
- Inputs are primitives or value objects; outputs are domain objects. Transport models (request/response schemas, ORM rows) never cross this seam — the driving adapter translates on the way in and on the way out.
- Absence and failure become explicit domain errors (`UserNotFoundError`), raised for adapters to translate — a use case never returns framework responses or status codes.
- One use case = one action, named as a verb phrase. If it grows branches to serve different callers, split it.
- The use case delimits the transaction. If transactional grouping is needed, model it as an outbound port (a unit of work) — never import a DB session here.

---

## 5. Adapters

The only layer that touches infrastructure. Two kinds, mirroring the two sides of the hexagon.

### Driven (outbound) adapters — implement a port

An adapter conforms to its port **structurally**: right method signatures, no inheritance, no registration. The in-memory implementation is a first-class citizen — it doubles as the test fake and proves the port needs no infrastructure.

```python
# adapters/outbound/memory.py
from uuid import UUID

from <package>.domain.entities import User


class InMemoryUserRepository:  # conforms to UserRepository — no subclassing
    def __init__(self) -> None:
        self._users: dict[UUID, User] = {}

    def get(self, user_id: UUID) -> User | None:
        return self._users.get(user_id)

    def save(self, user: User) -> None:
        self._users[user.id] = user
```

A persistence adapter owns the **mapping** between its storage model and the domain, in both directions. The ORM/DB model lives in the adapter; the domain never sees it.

```python
# adapters/outbound/sql.py — sketch, any ORM
class SqlUserRepository:
    def __init__(self, session: Session) -> None:
        self._session = session

    def get(self, user_id: UUID) -> User | None:
        row = self._session.get(UserRow, user_id)
        return row.to_domain() if row else None

    def save(self, user: User) -> None:
        self._session.merge(UserRow.from_domain(user))
```

### Driving (inbound) adapters — call the use cases

Thin translators: parse transport input, call **one** use case, translate the result and the domain errors back into the transport's vocabulary.

```python
# adapters/inbound/cli.py
import argparse
import sys
from uuid import UUID

from <package>.application.use_cases import ActivateUser
from <package>.domain.exceptions import DomainError


def main(activate: ActivateUser, argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="activate-user")
    parser.add_argument("user_id", type=UUID)
    args = parser.parse_args(argv)
    try:
        user = activate.execute(args.user_id)
    except DomainError as error:          # this adapter's single translation point
        print(error, file=sys.stderr)
        return 1
    print(f"{user.email.value} is now active")
    return 0
```

The adapter receives its use case, parses the transport's input, calls the use
case once, and translates the result and domain errors into the transport's
vocabulary (output and exit code here). Transport models and domain objects stay
distinct on purpose: the contract and the domain model evolve independently.

**HTTP adapters** (FastAPI routers, schemas, dependency wiring, exception
handlers): **rest-api-python** — routers §7, error mapping §4.

### Rules

- One adapter per port implementation, named by technology (`InMemoryUserRepository`, `SqlUserRepository`).
- Driving adapters contain zero business logic and never bypass the use case to reach a repository directly.
- Error translation happens here and only here: the domain raises `DomainError`s, each driving adapter decides what they become (404/409, CLI exit code…) — in one place per adapter, not in every route.
- A CLI is just another driving adapter — argparse/typer parses in, exit codes translate errors out. Several driving adapters coexist, all wired by the same `bootstrap()`.
- Framework and driver imports (`fastapi`, `sqlalchemy`, `httpx`…) appear in `adapters/` and nowhere else.

---

## 6. Dependency injection — the composition root

All wiring lives in one place: `bootstrap.py`, the **composition root**. It is the only file that imports concrete adapters, decides which one backs each port (per environment/config), and injects them into the use cases. No DI framework needed — plain constructors are enough in Python; reach for a container library only if the object graph grows genuinely large.

```python
# bootstrap.py
from dataclasses import dataclass

from <package>.adapters.outbound.memory import InMemoryUserRepository
from <package>.adapters.outbound.sql import SqlUserRepository, make_session
from <package>.adapters.outbound.smtp import SmtpNotifier
from <package>.application.use_cases import ActivateUser
from <package>.config import Settings
from <package>.ports.outbound import Notifier, UserRepository


@dataclass(frozen=True)
class App:
    activate_user: ActivateUser
    # one field per use case


def bootstrap(settings: Settings) -> App:
    users: UserRepository = (          # annotate with the PORT, not the adapter
        SqlUserRepository(make_session(settings))
        if settings.storage == "sql"
        else InMemoryUserRepository()
    )
    notifier: Notifier = SmtpNotifier(settings)
    return App(activate_user=ActivateUser(users=users, notifier=notifier))
```

Annotating with the port matters: without it, `mypy` infers the first branch's concrete type and rejects the second. It also states the intent — this variable is a `UserRepository`, whichever adapter backs it.

This is also where `mypy` verifies the whole architecture: passing `users` to `ActivateUser(users=...)` is the injection site where each adapter's structural conformance to its port is checked.

Each driving adapter takes its use cases from `bootstrap()` at its entry point:

```python
# __main__.py
sys.exit(cli.main(bootstrap(Settings()).activate_user))
```

For FastAPI, a `create_app(container)` factory stores the container on the app
and routes read it through dependencies — see **rest-api-python** §7.

### Rules

- `bootstrap.py` is the only place that imports concrete adapters, and the only place environment/config branching (`if settings.storage == …`) appears.
- Constructor injection only. No service locator: a module never reaches out to a global registry to *fetch* its dependencies — dependencies are handed to it.
- Tests reuse the same mechanism: build use cases directly with fakes (or a `bootstrap`-for-tests). Swapping an adapter is a one-line change here and zero lines anywhere else.

---

## 7. Testing

The architecture's payoff. Because use cases depend on ports, tests inject fakes through the constructor — no patching, no infrastructure, no I/O.

```python
# tests/unit/test_activate_user.py
from uuid import uuid4

import pytest

from <package>.adapters.outbound.memory import InMemoryUserRepository
from <package>.application.use_cases import ActivateUser
from <package>.domain.entities import User
from <package>.domain.exceptions import UserNotFoundError
from <package>.domain.value_objects import Email


class FakeNotifier:  # conforms to Notifier; records instead of sending
    def __init__(self) -> None:
        self.sent: list[tuple[User, str]] = []

    def send(self, recipient: User, message: str) -> None:
        self.sent.append((recipient, message))


def test_activates_and_notifies() -> None:
    users, notifier = InMemoryUserRepository(), FakeNotifier()
    user = User(email=Email("a@b.com"))
    users.save(user)

    result = ActivateUser(users, notifier).execute(user.id)

    assert result.is_active
    assert len(notifier.sent) == 1


def test_unknown_user_raises() -> None:
    use_case = ActivateUser(InMemoryUserRepository(), FakeNotifier())
    with pytest.raises(UserNotFoundError):
        use_case.execute(uuid4())
```

Integration tests exercise a **real adapter against real infrastructure**, verifying the port's contract (round-trips, edge cases of the technology):

```python
# tests/integration/test_sql_user_repository.py
def test_round_trip(session: Session) -> None:  # real DB via fixture
    repo = SqlUserRepository(session)
    user = User(email=Email("a@b.com"))
    user.activate()
    repo.save(user)

    loaded = repo.get(user.id)

    assert loaded is not None
    assert loaded.email == user.email      # assert attributes, not the entity:
    assert loaded.is_active == user.is_active  # `==` on entities compares id only
```

### Rules

- **Unit tests** (domain + use cases) touch zero I/O and get fakes via the constructor. Never `mock.patch`/monkeypatch here — needing to patch means DI was bypassed.
- Prefer hand-rolled fakes and spies (like `FakeNotifier`) over `MagicMock`: they assert observable behaviour, not call internals, and `mypy` checks them against the port like any adapter.
- Run the same **contract test suite** against every adapter of a port — in-memory included — by parametrizing over implementations. Conformance proven, fake trusted.
- Pyramid: many unit tests, few integration tests, fewest end-to-end (full stack through a driving adapter).

---

## 8. Best practices & anti-patterns

✅ Do:

- Behaviour on entities and value objects; use cases orchestrate only.
- Ports named after needs, small and role-shaped; signatures in domain language.
- Domain vocabulary stays technology-free — a domain attribute is `key`, not `s3_key`; the adapter knows it maps to S3.
- One composition root; constructor injection everywhere.
- An in-memory adapter for every port, reused as the test fake, covered by the shared contract tests.
- Domain errors translated at each driving adapter, in one place per adapter, and nowhere else.

❌ Avoid — each has a name; naming it in review is half the fix:

- **Anemic domain model** — entities as data bags, all logic drained into services or use cases.
- **Leaky port** — a port exposing its technology: returning ORM rows, taking a `Session`, infra vocabulary in signatures.
- **Use case bypass** — a driving adapter reaching a repository directly.
- **God port** — one `Repository` with 15 methods serving every caller.
- **Service locator** — modules fetching dependencies from a global registry instead of receiving them.
- **Patch-based tests** — `mock.patch` in unit tests; the DI was bypassed.
- **Transport leak** — request/response schemas or ORM models crossing into `application/` or `domain/`.

---

## 9. When to use it — and when not

Use it when the application has **real business rules and invariants** that outlive technology choices, when several driving or driven technologies coexist or plausibly will, and when the domain must be testable without infrastructure.

Skip it for thin CRUD over a database, scripts, prototypes, and glue/ETL pipelines whose logic *is* the infrastructure interaction. The test: if every use case would merely forward calls with no domain behaviour in between, the hexagon adds indirection without benefit — a plain layered app is the better tool. Partial adoption is legitimate: hexagonize the sub-domains that carry rules, keep the trivial parts simple.
