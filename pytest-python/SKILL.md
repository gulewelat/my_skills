---
name: pytest-python
description: Write and organize Python tests with pytest. Covers test structure (Given-When-Then), assertions and exceptions, suite layout, markers, skip and parametrize, fixtures, test doubles via dependency injection, and testing anti-patterns. Use when writing, structuring, reviewing, or debugging a pytest test suite.
---

# pytest — Testing Framework for Python

Conventions: `uv`, `pytest`, tests type-checked with `mypy --strict`
(test functions annotated `-> None`). Plain functions and native `assert` —
never `unittest.TestCase`.

## Reference files

Load only when the task needs them:

| File | When |
|---|---|
| [references/fixtures.md](references/fixtures.md) | Built-in fixtures, scopes, `conftest.py` hierarchy, factory fixtures |
| [references/mocking.md](references/mocking.md) | Stubs, mocks, fakes, `side_effect`, when `patch`/`monkeypatch` is acceptable |
| [references/cli-and-config.md](references/cli-and-config.md) | Command-line options, `pyproject.toml` config, coverage, output symbols |
| [references/anti-patterns.md](references/anti-patterns.md) | Full anti-pattern catalogue with fixes |

---

## 1 — Test structure

### 1.1 — Given-When-Then

```python
def test_add_numbers_returns_sum() -> None:
    # Given
    a, b = 5, 3

    # When
    result = add_numbers(a, b)

    # Then
    assert result == 8
```

- **Given** — context and data
- **When** — the action under test, ideally **one call**. Several actions =
  the test does too much.
- **Then** — checks only what follows from the When

### 1.2 — Assertions

Native `assert`; pytest rewrites it to show the exact values on failure, so a
custom message is rarely needed.

```python
assert user.name == "Alice"
assert "Hello" in text
assert {"name": "Alice", "age": 30} == actual      # diff shown item by item

# Floats: never ==
assert 0.1 + 0.2 == pytest.approx(0.3)
assert 10 / 3 == pytest.approx(3.33, rel=1e-2)
assert [0.1 + 0.2, 0.2 + 0.4] == pytest.approx([0.3, 0.6])
```

One **behaviour** per test — several asserts are fine if they describe the
same outcome.

### 1.3 — Exceptions

```python
def test_divide_by_zero_raises() -> None:
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)

def test_invalid_order_exposes_reason() -> None:
    with pytest.raises(InvalidOrder) as excinfo:
        place_order(empty_cart)
    assert excinfo.value.reason == "empty cart"     # inspect attributes
```

- `match` is a regex — escape special characters (`re.escape`).
- Error paths deserve as many tests as the nominal path.
- `ExceptionGroup` (pytest 8.4+): `with pytest.RaisesGroup(ValueError, TypeError): ...`

---

## 2 — Organization

```
project/
├── src/app/
│   ├── pricing.py
│   └── users/service.py
└── tests/
    ├── conftest.py
    ├── test_pricing.py
    └── users/test_service.py
```

- **Mirror structure**: one test file per source module, same relative path.
- **Classes are optional**: group tests sharing one business context; no
  `__init__` and no inheritance.
- **Names describe behaviour**: `test_<unit>_<condition>_<expected>`.

```python
class TestDiscount:
    def test_zero_discount_returns_original_price(self) -> None: ...
    def test_discount_above_100_percent_is_rejected(self) -> None: ...
```

---

## 3 — Markers

### 3.1 — Custom markers

```python
@pytest.mark.slow
def test_full_import() -> None: ...

@pytest.mark.integration
def test_repository_roundtrip(db_session: Session) -> None: ...
```

```bash
uv run pytest -m "not slow"
uv run pytest -m "integration and not slow"
```

Declare every marker in `pyproject.toml` and run with `--strict-markers`, so a
typo fails instead of silently selecting nothing (see
[references/cli-and-config.md](references/cli-and-config.md)).

### 3.2 — `skip`, `skipif`, `xfail`

```python
@pytest.mark.skipif(sys.platform == "win32", reason="POSIX permissions only")
def test_file_mode() -> None: ...

@pytest.mark.skip(reason="Bug #123 — fix planned sprint 42")
def test_refund() -> None: ...

@pytest.mark.xfail(reason="Known rounding bug #456", strict=True)
def test_rounding() -> None: ...
```

- `skipif` is for **environment** conditions (OS, Python version, service).
- `skip` is temporary and carries a ticket.
- `xfail(strict=True)` turns "fixed by accident" into a failure — prefer it
  to `skip` for known bugs.
- Watch the skipped count in CI: "all green, 30 skipped" is not green.

### 3.3 — `parametrize`

```python
@pytest.mark.parametrize(
    ("x", "y", "expected"),
    [
        pytest.param(5, 3, 8, id="positive"),
        pytest.param(0, 7, 7, id="zero"),
        pytest.param(-2, 5, 3, id="negative"),
    ],
)
def test_add(x: int, y: int, expected: int) -> None:
    assert add(x, y) == expected
```

```python
@pytest.mark.parametrize(
    ("x", "y", "error"),
    [(10, 0, ZeroDivisionError), ("a", 1, TypeError)],
)
def test_divide_errors(x: object, y: object, error: type[Exception]) -> None:
    with pytest.raises(error):
        divide(x, y)  # type: ignore[arg-type]  # invalid input on purpose
```

- One generated test per case; a failure names the case (`test_add[zero]`).
- Use `id=` when the values don't read well in the output.
- Ideal for edge cases: zero, negative, empty, `None`, boundaries.
- Don't parametrize over **different behaviours** — that is two tests.

---

## 4 — Fixtures (essentials)

```python
from collections.abc import Iterator

@pytest.fixture
def database() -> Iterator[Database]:
    db = Database()
    db.connect()
    yield db           # test runs here
    db.close()         # teardown runs even if the test fails

def test_query_returns_rows(database: Database) -> None:
    assert database.query("SELECT 1") is not None
```

- Injected by parameter name — no import.
- A fixture that yields is annotated `Iterator[T]`; one that returns, `T`.
- Default scope `function` = fresh instance per test. Keep it unless the
  resource is expensive **and** read-only.
- Shared fixtures go in `conftest.py`, never imported from test files.
- Golden rule: **every test runs alone and in any order.**

Built-ins (`tmp_path`, `monkeypatch`, `capsys`, `caplog`), scopes, factories,
`conftest` hierarchy: [references/fixtures.md](references/fixtures.md).

---

## 5 — Test doubles (essentials)

Inject dependencies; don't patch them.

```python
# Production code receives its collaborator
def send_welcome(user: User, sender: EmailSender) -> None:
    sender.send(user.email, f"Welcome {user.name}!")

# Test passes a fake
class FakeSender:
    def __init__(self) -> None:
        self.sent: list[tuple[str, str]] = []

    def send(self, to: str, body: str) -> None:
        self.sent.append((to, body))

def test_welcome_email_is_sent() -> None:
    sender = FakeSender()
    send_welcome(User(name="Alice", email="a@x.io"), sender)
    assert sender.sent == [("a@x.io", "Welcome Alice!")]
```

- Hand-written fakes are type-checked against the `Protocol`; `Mock()` is not.
- `patch` / `monkeypatch` only for what you don't control (clock, env vars,
  third-party modules).
- Assert on **outcomes**, not on how many times something was called.

Stubs vs mocks, `side_effect`, `autospec`, patch rules:
[references/mocking.md](references/mocking.md).

---

## 6 — Rules of thumb

| Do | Don't |
|---|---|
| One behaviour per test, named after it | `test_user()` checking creation, save and email |
| Each test builds its own data | Tests relying on state left by another |
| Fakes injected through parameters | `@patch("app.module.Class")` on your own code |
| Assert observable results | Assert call counts and internal queries |
| `parametrize` for input variations | Copy-pasted tests differing by one value |
| `skip` with ticket, or `xfail(strict=True)` | `skip(reason="fix later")` |
| `-> None` on tests, typed fixtures | Untyped tests excluded from mypy |

A test that is hard to write — long Given, many doubles, a name with "and" —
signals production code doing too much: fix the code, not the test (see
**clean-code-python** §8). Full catalogue:
[references/anti-patterns.md](references/anti-patterns.md).

---

## 7 — Quick reference

```bash
uv run pytest                                  # all tests
uv run pytest tests/test_pricing.py            # one file
uv run pytest tests/test_pricing.py::test_add  # one test
uv run pytest -k "discount and not zero"       # by name expression
uv run pytest -m "not slow"                    # by marker
uv run pytest -x                               # stop at first failure
uv run pytest --lf                             # rerun last failures
uv run pytest -v                               # verbose (-q quiet, -vv full diffs)
uv run pytest --cov=src --cov-report=term-missing
```

## Related skills

- **mypy-python**: same `pyproject.toml`; tests are type-checked
- **hexagonal-architecture-python**: fakes injected through ports
- **clean-code-python**: what hard-to-write tests reveal about production code
