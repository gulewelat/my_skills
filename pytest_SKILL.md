# pytest - Testing Framework for Python

## Overview

pytest is the standard testing framework for Python. It offers simple and
intuitive syntax, detailed error messages, and a rich plugin ecosystem — while
remaining compatible with existing `unittest` tests.

**Key Features**:
- Simple assertions: uses Python's native `assert`, no specific methods needed
- Detailed error messages: shows exact values on failure
- Fixtures: elegant setup/teardown management with dependency injection
- Parametrize: reuse test logic with different values
- Markers: categorize and filter tests
- Plugins: rich ecosystem (`pytest-cov`, `pytest-mock`, etc.)
- unittest compatibility: progressive migration possible

## Installation

```bash
# Basic installation
uv add --dev pytest

# With common plugins
uv add --dev pytest pytest-cov pytest-mock

# Run tests
uv run pytest
```

> `--dev` ensures pytest is a development dependency only — no reason to
> install it in production.

---

## Part 2 — Test Structure

### 2.1 — Given-When-Then Pattern

```python
# math_operations.py
def add_numbers(a: int, b: int) -> int:
    return a + b

# test_math_operations.py
def test_add_numbers():
    # Given
    a = 5
    b = 3
    expected_result = 8

    # When
    result: int = add_numbers(a, b)

    # Then
    assert result == expected_result
```

**The 3 roles:**
- **Given** — prepares context and necessary data
- **When** — the action being tested, ideally **a single call**. If you need
  multiple actions, it often means your test is doing too many things
- **Then** — verifies only what is directly related to the When

### 2.2 — Assertions

pytest uses Python's native `assert` — no specific methods like `assertEqual()`.

```python
def test_string_operations():
    text = "Hello, World!"

    assert len(text) == 13
    assert text.upper() == "HELLO, WORLD!"
    assert "Hello" in text
    assert text.startswith("Hello")
```

**The power of pytest: error messages.**

On failure, pytest shows the exact values:

```python
def test_division():
    result = 10 / 3
    assert result == 3.33  # Fails
```

```
>       assert result == 3.33
E       assert 3.3333333333333335 == 3.33
E         +3.3333333333333335
E         -3.33
```

**Comparing complex structures:**

```python
def test_user_dict():
    expected = {"name": "Alice", "age": 30}
    actual = {"name": "Alice", "age": 25}
    assert expected == actual
```

```
E       AssertionError: assert {'name': 'Alice', 'age': 30} == {'name': 'Alice', 'age': 25}
E         Differing items:
E         {'age': 30} != {'age': 25}
```

**Floating point comparisons — use `pytest.approx`:**

```python
# ❌ Fragile: fails due to floating point precision
assert 0.1 + 0.2 == 0.3

# ✅ Correct: pytest.approx handles tolerance
assert 0.1 + 0.2 == pytest.approx(0.3)

# With explicit relative tolerance
assert 10 / 3 == pytest.approx(3.33, rel=1e-2)

# Works on lists too
assert [0.1 + 0.2, 0.2 + 0.4] == pytest.approx([0.3, 0.6])
```

**Key takeaways:**
- One `assert` per behavior tested — if you chain 10 assertions, you won't
  know which one failed
- pytest highlights **exactly** what differs — no custom message needed in
  most cases
- Never use `==` to compare floats directly — always use `pytest.approx`

### 2.3 — Exception Handling

```python
import pytest

def divide(a: int, b: int) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# Full method — inspect all exception attributes
def test_divide_by_zero_detailed():
    with pytest.raises(ValueError) as excinfo:
        divide(10, 0)
    assert "Cannot divide by zero" in str(excinfo.value)

# Simplified method — verify type + message in one line
def test_divide_by_zero_simple():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

**Key takeaways:**
- Use the **full** method (`excinfo`) when you need to inspect multiple
  exception attributes
- Use the **simplified** method (`match`) for the most common case — just
  verifying type and message
- Testing error cases is as important as testing the nominal case

**`pytest.RaisesGroup` — testing ExceptionGroup** *(pytest 8.4+, Python 3.11+)*

```python
# ExceptionGroup — multiple exceptions raised at once
with pytest.RaisesGroup(ValueError, TypeError):
    raise ExceptionGroup("errors", [
        ValueError("invalid value"),
        TypeError("wrong type")
    ])
```

---

## Part 3 — Test Organization

### 3.1 — Mirror Structure

```
project/
├── src/
│   └── math_operations.py
│   └── user_service.py
└── tests/
    └── src/
        └── test_math_operations.py
        └── test_user_service.py
```

The rule: **one test file per source file**, with the same relative path.
Tests for any module are immediately obvious.

### 3.2 — Grouping by Classes

Optional with pytest — unlike unittest — but useful for grouping related tests:

```python
class TestCalculator:
    def test_addition(self):
        assert add_numbers(2, 3) == 5

    def test_subtraction(self):
        assert subtract_numbers(5, 3) == 2

    def test_multiply(self):
        assert multiply_numbers(4, 3) == 12
```

**Key takeaways:**
- Mirror structure is **mandatory** — it's a universal convention
- Class grouping is **optional** — use it when multiple tests share the same
  business context
- Do not inherit from `unittest.TestCase` — stay in pure pytest syntax

---

## Part 4 — Decorators

### 4.1 — `mark`: Custom Markers

```python
@pytest.mark.critical
def test_user_authentication():
    pass

@pytest.mark.slow
def test_large_dataset_processing():
    pass

@pytest.mark.integration
def test_database_connection():
    pass
```

```bash
# Run only critical tests
uv run pytest -m critical

# Run everything except slow tests
uv run pytest -m "not slow"

# Combine markers
uv run pytest -m "critical and not slow"
```

**Declare markers in `pyproject.toml`:**

```toml
[tool.pytest.ini_options]
markers = [
    "critical: tests for critical features",
    "slow: slow tests",
    "integration: integration tests",
]
```

**Key takeaways:**
- Without declaration in `pyproject.toml`, pytest shows a warning for each
  unknown marker
- Markers are the key to **running targeted subsets** — useful in CI to
  separate fast and slow tests

### 4.2 — `skip` and `skipif`

```python
# skip — unconditionally ignore
@pytest.mark.skip(reason="Feature being refactored")
def test_new_feature():
    pass

# skipif — ignore based on condition
import sys

@pytest.mark.skipif(
    sys.version_info < (3, 11),
    reason="Requires Python 3.11 or higher"
)
def test_new_syntax():
    pass

# skipif on environment variable
import os

@pytest.mark.skipif(
    os.getenv("ENV") == "dev",
    reason="Does not run in development environment"
)
def test_production_only():
    pass
```

**⚠️ Anti-pattern: abusing skipif**

```python
# ❌ WRONG: skipif used to hide an unmaintained test
@pytest.mark.skipif(True, reason="To fix later")
def test_important_feature():
    pass
```

**Key takeaways:**
- `skip` is **temporary** — a skipped test should have an associated ticket
  and resolution date
- `skipif` is legitimate for **infrastructure dependencies** — environment,
  Python version, external service
- A CI pipeline showing "100% passed" with 30 skipped tests gives a
  **false sense of security** — monitor the number of ignored tests

### 4.3 — `parametrize`

```python
def add(x: int, y: int) -> int:
    return x + y

@pytest.mark.parametrize("x, y, expected", [
    (5, 3, 8),    # Simple case
    (0, 7, 7),    # Addition with zero
    (-2, 5, 3),   # Negative number
    (10, 20, 30)  # Large numbers
])
def test_add(x: int, y: int, expected: int):
    assert add(x, y) == expected
```

pytest generates **one distinct test per tuple** — on failure it indicates
precisely which dataset caused the problem:

```
PASSED test_math.py::test_add[5-3-8]
PASSED test_math.py::test_add[0-7-7]
FAILED test_math.py::test_add[-2-5-3]   <- failure here
PASSED test_math.py::test_add[10-20-30]
```

**Parametrize with error cases:**

```python
@pytest.mark.parametrize("x, y, expected_error", [
    (10, 0, ValueError),
    ("a", 1, TypeError),
])
def test_divide_errors(x, y, expected_error):
    with pytest.raises(expected_error):
        divide(x, y)
```

**Key takeaways:**
- `parametrize` avoids duplication — adding a test case = adding a tuple
- Each tuple is an **independent** test — one failure doesn't impact others
- Ideal for **edge cases**: zero, negative, None, empty string, maximum value

---

## Part 5 — Command Line Options

```bash
# Run all tests
uv run pytest

# Run a specific file
uv run pytest tests/src/test_math_operations.py

# Run a specific test function
uv run pytest tests/src/test_math_operations.py::test_add_numbers

# Run all tests in a class
uv run pytest tests/src/test_math_operations.py::TestCalculator
```

**Execution control:**

```bash
# Stop at first error
uv run pytest -x

# Stop after N errors
uv run pytest --maxfail=2

# Disable tracebacks
uv run pytest -tb=no
```

**Verbosity:**

```bash
# Silent mode — minimal summary, ideal for CI
uv run pytest -q

# Verbose mode — details each test, daily use
uv run pytest -v

# Full verbose mode — shows all differences
uv run pytest -vv
```

**Code coverage:**

```bash
# Coverage report in terminal
uv run pytest --cov=src

# Report with uncovered lines
uv run pytest --cov=src --cov-report=term-missing

# HTML report
uv run pytest --cov=src --cov-report=html
```

**Key takeaways:**
- `-v` is the most useful mode daily — detailed enough without being verbose
- `-q` is reserved for CI — concise output for logs
- `-x` is valuable during development — fix one error at a time
- `--cov` requires `pytest-cov`: `uv add --dev pytest-cov`

---

## Part 6 — Fixtures

### 6.1 — Fixtures with yield

```python
import pytest
from typing import Generator

@pytest.fixture
def database() -> Generator[Database, None, None]:
    # Setup
    db = Database()
    db.connect()

    yield db  # Test runs here

    # Automatic teardown — runs even if test fails
    db.close()

def test_database_query(database: Database):
    result = database.query("SELECT * FROM users")
    assert result is not None

def test_database_insert(database: Database):
    database.insert({"id": 1, "name": "Alice"})
    result = database.query("SELECT * FROM users WHERE id = 1")
    assert result is not None
```

**Key takeaways:**
- Everything **before** `yield` = setup
- Everything **after** `yield` = teardown — runs automatically even if the
  test crashes
- The fixture is **injected** into the test via its name as a parameter —
  no import needed
- Multiple tests can use the same fixture — each gets a fresh instance

### 6.2 — Built-in Fixtures

pytest provides ready-to-use fixtures for the most common needs:

```python
# tmp_path — isolated temporary directory per test
def test_file_processing(tmp_path):
    data_file = tmp_path / "data.txt"
    data_file.write_text("test content")
    result = process_file(data_file)
    assert result == "PROCESSED: test content"

# monkeypatch — temporarily modify the environment
def test_api_call(monkeypatch):
    monkeypatch.setenv("API_KEY", "test_key")
    monkeypatch.setattr("requests.get", lambda x: MockResponse())
    result = call_api()
    assert result.status_code == 200

# capsys — capture stdout/stderr
def test_print_function(capsys):
    print("Hello World")
    log_error("Error message")
    captured = capsys.readouterr()
    assert "Hello World" in captured.out
    assert "Error message" in captured.err

# caplog — capture logs
def test_logging_behavior(caplog):
    caplog.set_level(logging.INFO)
    process_data({"id": 123})
    assert "Processing data for id: 123" in caplog.text
    assert caplog.records[0].levelname == "INFO"
```

**Key takeaways:**
- `tmp_path` — preferable to manual temp file creation, cleaned up
  automatically after each test
- `monkeypatch` — lightweight alternative to mocking for environment
  variables and attributes
- `capsys` — essential for testing functions that write to stdout/stderr
- `caplog` — essential for testing logging behavior

### 6.3 — Fixture Scopes

The scope defines the **lifetime** of a fixture:

| Scope | Recreated per... |
|---|---|
| `function` | test function (default) |
| `class` | test class |
| `module` | test file |
| `session` | pytest run |

```python
# function (default) — fresh instance per test
@pytest.fixture(scope="function")
def database():
    db = Database()
    db.connect()
    yield db
    db.close()

# session — shared across the entire test suite
@pytest.fixture(scope="session")
def api_client():
    client = APIClient()
    client.authenticate()
    yield client
    client.logout()
```

**⚠️ Anti-pattern: shared state with a wider scope**

```python
@pytest.fixture(scope="module")
def shared_data():
    return {"counter": 0}

def test_increment(shared_data):
    shared_data["counter"] += 1
    assert shared_data["counter"] == 1  # ✅ Passes

def test_another_increment(shared_data):
    shared_data["counter"] += 1
    assert shared_data["counter"] == 1  # ❌ Fails: counter is already 1
```

Tests now depend on their **execution order** — one of the worst problems
in a test suite.

**Key takeaways:**
- `function` is the default scope — **don't change it without good reason**
- `session` is legitimate for expensive-to-initialize resources — HTTP
  client, read-only DB connection
- Golden rule: **a test must be able to run alone, in any order**
- If using a wider scope, shared data must be **read-only**

### 6.4 — conftest.py

`conftest.py` is the fixture sharing file between multiple test files —
pytest detects it automatically, no import needed.

```
tests/
├── conftest.py              # Fixtures available everywhere
├── src/
│   ├── conftest.py          # Fixtures available in src/ only
│   ├── test_user_service.py
│   └── test_math_operations.py
```

```python
# tests/conftest.py
import pytest
from typing import Generator

@pytest.fixture(scope="session")
def api_credentials():
    return {
        "api_key": os.getenv("TEST_API_KEY"),
        "api_secret": os.getenv("TEST_API_SECRET")
    }

@pytest.fixture
def database() -> Generator[Database, None, None]:
    db = Database()
    db.connect()
    yield db
    db.close()

@pytest.fixture
def temp_data_dir(tmp_path):
    data_dir = tmp_path / "data"
    data_dir.mkdir()
    yield data_dir
```

```python
# tests/src/test_user_service.py — no need to import conftest
def test_get_user(database: Database):
    result = database.query("SELECT * FROM users WHERE id = 1")
    assert result is not None
```

**Key takeaways:**
- `conftest.py` at the root of `tests/` = fixtures available **everywhere**
- `conftest.py` in a subfolder = fixtures available **only in that folder**
- It's the right place for shared fixtures — not in the test files themselves
- Multiple `conftest.py` can coexist — pytest resolves them automatically
  by hierarchy

---

## Part 7 — Mocking

### 7.1 — Stub vs Mock: What's the Difference?

Two types of test doubles serving different purposes:

**Stub** — provides predefined responses
> "What should this component return?"

```python
from unittest.mock import Mock

email_sender = Mock()
email_sender.send.return_value = True  # Configure the response

result: bool = email_sender.send("user@example.com")  # Returns True
```

**Mock (Spy)** — verifies how the component is used
> "How should this component be called?"

```python
payment_mock = Mock()
payment_mock.process_payment(100, "USD")

# Verify interactions
payment_mock.process_payment.assert_called_once()
payment_mock.process_payment.assert_called_with(100, "USD")
assert payment_mock.process_payment.call_count == 1
```

**Key takeaways:**
- **Stub** = I control what the dependency returns
- **Mock** = I verify that my dependency is called correctly
- In practice `unittest.mock.Mock` does both — usage determines the role

### 7.2 — Dependency Injection

The recommended approach for mocking — pass dependencies as parameters
rather than instantiating them inside.

```python
# ❌ WRONG: dependency instantiated inside — impossible to mock cleanly
def send_welcome_email(user: User) -> None:
    mailer = EmailSender()  # tight coupling
    mailer.send(f"Welcome {user.name}!")

# ✅ CORRECT: dependency injected as parameter
def send_welcome_email(user: User, email_sender: EmailSender) -> None:
    email_sender.send(f"Welcome {user.name}!")
```

**The test becomes trivial:**

```python
from unittest.mock import Mock

def test_welcome_email() -> None:
    # Given
    mock_sender = Mock()
    user = User(name="Alice")

    # When
    send_welcome_email(user, mock_sender)

    # Then
    mock_sender.send.assert_called_with("Welcome Alice!")
```

**Key takeaways:**
- Dependency injection makes code **testable by construction** — no need
  for `@patch`
- If a function is hard to test, it often means it instantiates too many
  things internally
- The mock is passed like any other parameter — simple, readable, refactor-proof

### 7.3 — `@patch` and Why to Avoid It

```python
# ❌ WRONG: patch
from unittest.mock import patch

@patch('user_manager.mailer.send')  # fragile path
def test_welcome_email(mock_send) -> None:
    user = User(name="Alice")
    send_welcome_email(user)
    mock_send.assert_called_with("Welcome Alice!")
```

**Problems with `@patch`:**

```python
# If you rename the module or move the file...
@patch('user_manager.mailer.send')      # ❌ Breaks
@patch('services.user.mailer.send')     # ❌ Must update manually
@patch('app.services.user.mailer.send') # ❌ Yet another path
```

- The import path is **fragile** — a refactoring silently breaks tests
- Python's import mechanism makes behavior **hard to understand**
- Tests become **coupled to file structure** rather than behavior

**✅ CORRECT: dependency injection instead**

```python
def send_welcome_email(user: User, email_sender: EmailSender) -> None:
    email_sender.send(f"Welcome {user.name}!")

def test_welcome_email() -> None:
    mock_sender = Mock()
    send_welcome_email(User(name="Alice"), mock_sender)
    mock_sender.send.assert_called_with("Welcome Alice!")
```

**Key takeaways:**
- `@patch` is tempting but **fragile** — avoid it in favor of dependency
  injection
- Legitimate exception: patching system modules you don't control
  (`datetime.now`, `os.path`, `requests.get`)
- The rule: if you control the code, use injection. If it's external,
  `@patch` is acceptable

### 7.4 — Side Effect

Allows varying mock behavior based on received arguments — instead of
always returning the same value.

```python
from unittest.mock import Mock

def dynamic_response(value: int) -> str:
    if value == 1:
        return "First"
    if value == 2:
        return "Second"
    raise ValueError(f"Unknown value: {value}")

mock_function = Mock(side_effect=dynamic_response)

mock_function(1)  # "First"
mock_function(2)  # "Second"
mock_function(3)  # ValueError: Unknown value: 3
```

**Concrete use case — simulating an unstable API:**

```python
def test_retry_on_failure() -> None:
    # Given
    call_count = 0

    def flaky_api(url: str) -> dict:
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ConnectionError("Service unavailable")
        return {"status": "ok"}

    mock_api = Mock(side_effect=flaky_api)

    # When
    result = call_with_retry(mock_api, "https://api.example.com")

    # Then
    assert result == {"status": "ok"}
    assert mock_api.call_count == 3
```

**Key takeaways:**
- `side_effect` is useful for simulating **dynamic behaviors** — intermittent
  errors, variable responses
- This is an advanced tool — if you need it often, it may be a sign your
  code could be better decomposed
- `side_effect` also accepts a **list** of successively returned values:
  `Mock(side_effect=["first", "second", ValueError()])`

---

## Part 8 — Configuration

Everything in `pyproject.toml` — consistent with our uv choice.

```toml
[tool.pytest.ini_options]
# Where to find tests
testpaths = ["tests"]
python_files = ["test_*.py"]

# Default options — equivalent to always passing these flags
addopts = "-v --cov=src --cov-report=term-missing"

# Declared markers
markers = [
    "critical: tests for critical features",
    "slow: slow tests",
    "integration: integration tests",
    "unit: unit tests",
]
```

**Utility scripts:**

```toml
[tool.uv.scripts]
test = "pytest"
test-cov = "pytest --cov=src --cov-report=html"
test-fast = "pytest -m 'not slow'"
test-critical = "pytest -m critical"
```

```bash
uv run test
uv run test-cov
uv run test-fast
uv run test-critical
```

> **mypy configuration**: `disallow_untyped_defs`, `warn_return_any`, etc.
> are managed in the same `pyproject.toml` — see the **mypy skill** for
> full details.

**Key takeaways:**
- `addopts` avoids retyping the same flags every time — verbose mode and
  coverage enabled by default
- uv scripts simplify daily commands — `uv run test` rather than
  `uv run pytest --cov=src -v`
- Everything is in `pyproject.toml` — a single config file for the entire
  project

---

## Part 9 — Best Practices & Anti-patterns

### 9.1 — Best Practices

**1. One test = one responsibility**
```python
# ❌ WRONG: tests too many things at once
def test_user():
    user = create_user("Alice", 30)
    assert user.name == "Alice"
    assert user.age == 30
    assert user.is_active == True
    assert user.email is None
    db.save(user)
    assert db.get(user.id) is not None

# ✅ CORRECT: each test has a single responsibility
def test_user_creation():
    user = create_user("Alice", 30)
    assert user.name == "Alice"
    assert user.age == 30

def test_user_saved_to_database(database):
    user = create_user("Alice", 30)
    database.save(user)
    assert database.get(user.id) is not None
```

**2. Explicit test names**
```python
# ❌ WRONG: vague name
def test_price():
    ...

# ✅ CORRECT: name describes the expected behavior
def test_price_with_discount_returns_reduced_amount():
    ...

def test_price_with_zero_discount_returns_original_amount():
    ...
```

**3. Prefer dependency injection over `@patch`**
```python
# ❌ WRONG
@patch('module.EmailSender.send')
def test_welcome_email(mock_send):
    ...

# ✅ CORRECT
def test_welcome_email():
    mock_sender = Mock()
    send_welcome_email(user, mock_sender)
    ...
```

**4. Use `parametrize` for edge cases**
```python
# ✅ All edge cases in one place
@pytest.mark.parametrize("value, expected", [
    (0, 0),       # zero
    (-1, 0),      # negative
    (None, None), # None
    (999, 999),   # maximum value
])
def test_process_value(value, expected):
    assert process(value) == expected
```

**5. Tests as a Single Responsibility signal**

A test that is hard to write often reveals a function that does too many
things. Watch for these signals:

```python
# Signal 1 — hard to name the test
def test_process_user_and_save_and_send_email_and_log():
    ...  # name reveals too many responsibilities

# Signal 2 — enormous Given
def test_process_order():
    user = create_user(...)
    product = create_product(...)
    inventory = setup_inventory(...)
    payment = setup_payment(...)
    email = setup_email(...)
    # 10 lines of setup before even the When

# Signal 3 — multiple unrelated assertions in Then
def test_process_order():
    result = process_order(order)
    assert result.status == "confirmed"   # responsibility 1
    assert email_sent == True             # responsibility 2
    assert inventory.stock == 9           # responsibility 3

# Signal 4 — too many mocks
@patch('module.database')
@patch('module.email_sender')
@patch('module.inventory')
@patch('module.payment_gateway')
def test_process_order(...):
    ...  # 4 mocks = function touches everything
```

### 9.2 — Anti-patterns

**1. Interdependent tests**
```python
# ❌ WRONG: tests depend on execution order
class TestUserFlow:
    def test_create_user(self):
        self.user = create_user("Alice")
        assert self.user.id is not None

    def test_update_user(self):
        # Depends on test_create_user!
        self.user.name = "Bob"
        assert self.user.name == "Bob"

# ✅ CORRECT: each test is autonomous
def test_create_user():
    user = create_user("Alice")
    assert user.id is not None

def test_update_user():
    user = create_user("Alice")  # Creates its own instance
    user.name = "Bob"
    assert user.name == "Bob"
```

**2. Abusing `skip`**
```python
# ❌ WRONG: tests skipped indefinitely
@pytest.mark.skip(reason="To fix later")
def test_payment_processing():
    pass

# ✅ CORRECT: a skip must have a deadline
@pytest.mark.skip(reason="Bug #123 — fix planned sprint 42")
def test_payment_processing():
    pass
```

**3. Too many mocks**
```python
# ❌ WRONG: 4 mocks = function under test does too many things
@patch('module.database')
@patch('module.email_sender')
@patch('module.inventory')
@patch('module.payment_gateway')
def test_process_order(mock_db, mock_email, mock_inv, mock_pay):
    ...

# ✅ CORRECT: decompose into single-responsibility functions
def test_order_confirmation():
    mock_db = Mock()
    confirm_order(order, mock_db)
    mock_db.save.assert_called_once()

def test_order_notification():
    mock_email = Mock()
    notify_user(order, mock_email)
    mock_email.send.assert_called_once()
```

**4. Tests too coupled to implementation**
```python
# ❌ WRONG: tests the how, not the what
def test_get_user():
    user = get_user(1)
    assert db.query.call_count == 1
    assert db.query.called_with("SELECT * FROM users WHERE id = 1")

# ✅ CORRECT: tests observable behavior
def test_get_user():
    user = get_user(1)
    assert user.id == 1
    assert user.name == "Alice"
```

**5. Test returning a value** *(error since pytest 8)*
```python
# ❌ WRONG: pytest 8 raises an explicit error
def test_something():
    return True  # Error: test must return None

# ✅ CORRECT
def test_something():
    assert something() is True
```

**6. yield inside a test function** *(error since pytest 8)*
```python
# ❌ WRONG: causes an explicit error since pytest 8
def test_something():
    yield 42  # Use a fixture with yield instead

# ✅ CORRECT: use a fixture if you need yield
@pytest.fixture
def my_resource():
    resource = setup()
    yield resource
    teardown(resource)
```

---

## Part 10 — Quick Reference

### 10.1 — uv Commands

```bash
# Installation
uv add --dev pytest pytest-cov pytest-mock

# Run all tests
uv run pytest

# Run a file
uv run pytest tests/src/test_math_operations.py

# Run a function
uv run pytest tests/src/test_math_operations.py::test_add_numbers

# Run a class
uv run pytest tests/src/test_math_operations.py::TestCalculator

# Filter by marker
uv run pytest -m critical
uv run pytest -m "not slow"
uv run pytest -m "critical and not slow"

# Verbosity
uv run pytest -q    # silent
uv run pytest -v    # verbose
uv run pytest -vv   # full verbose

# Execution control
uv run pytest -x            # stop at first error
uv run pytest --maxfail=2   # stop after 2 errors
uv run pytest -tb=no        # no tracebacks

# Coverage
uv run pytest --cov=src
uv run pytest --cov=src --cov-report=term-missing
uv run pytest --cov=src --cov-report=html
```

### 10.2 — Symbols and Statuses

**Normal mode:**

| Symbol | Status | Description |
|---|---|---|
| `.` | Success | Test passed |
| `F` | Failed | Assertion failed |
| `E` | Error | Runtime error |
| `s` | Skipped | Test ignored |

**Verbose mode (`-v`):**

| Symbol | Status |
|---|---|
| `PASSED` | Test passed |
| `FAILED` | Assertion failed |
| `ERROR` | Runtime error |
| `SKIPPED` | Test ignored |

**Final summary:**

```
========================= short test summary info ==========================
FAILED tests/src/test_math.py::test_division - AssertionError
ERROR tests/src/test_user.py::test_get_user - NameError
SKIPPED tests/src/test_payment.py::test_refund - reason: Bug #123
========================= 1 failed, 1 error, 1 skipped, 7 passed ==========
```

---

## Resources

- **Official documentation**: https://docs.pytest.org/
- **pytest-cov**: https://pytest-cov.readthedocs.io/
- **unittest.mock**: https://docs.python.org/3/library/unittest.mock.html

## Related Skills

- **mypy**: static type checking — configuration in same `pyproject.toml`
- **ruff**: Python linter — works well alongside pytest
- **uv**: package manager — used throughout this skill
- **tdd**: test strategy, test pyramid, contract tests — see TDD skill
