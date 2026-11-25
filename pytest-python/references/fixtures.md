# Fixtures

## Built-in fixtures

```python
import logging
from pathlib import Path

# tmp_path — isolated directory per test, cleaned up automatically
def test_process_file(tmp_path: Path) -> None:
    data = tmp_path / "data.txt"
    data.write_text("content")
    assert process_file(data) == "PROCESSED: content"

# monkeypatch — temporary changes to env, attributes, dict items, cwd
def test_reads_api_key(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("API_KEY", "test-key")
    assert load_settings().api_key == "test-key"

# capsys — captured stdout / stderr
def test_prints_report(capsys: pytest.CaptureFixture[str]) -> None:
    print_report([1, 2])
    captured = capsys.readouterr()
    assert "Total: 3" in captured.out

# caplog — captured log records
def test_logs_processing(caplog: pytest.LogCaptureFixture) -> None:
    caplog.set_level(logging.INFO)
    process({"id": 123})
    assert "Processing id 123" in caplog.text
    assert caplog.records[0].levelname == "INFO"
```

| Fixture | Type annotation |
|---|---|
| `tmp_path` | `pathlib.Path` |
| `tmp_path_factory` | `pytest.TempPathFactory` (session-scoped dirs) |
| `monkeypatch` | `pytest.MonkeyPatch` |
| `capsys` | `pytest.CaptureFixture[str]` |
| `caplog` | `pytest.LogCaptureFixture` |
| `request` | `pytest.FixtureRequest` |

## Scopes

| Scope | Created once per |
|---|---|
| `function` (default) | test |
| `class` | test class |
| `module` | test file |
| `package` | test package |
| `session` | pytest run |

```python
@pytest.fixture(scope="session")
def engine() -> Iterator[Engine]:
    engine = create_engine(TEST_DB_URL)     # expensive, shared read-only
    yield engine
    engine.dispose()

@pytest.fixture
def db_session(engine: Engine) -> Iterator[Session]:
    connection = engine.connect()
    transaction = connection.begin()
    session = Session(bind=connection)
    yield session                           # each test isolated
    session.close()
    transaction.rollback()
```

Pattern: **expensive resource at wide scope, mutable state at function scope**
(here a per-test transaction rolled back after the test).

A fixture can only depend on fixtures of equal or wider scope.

**Anti-pattern — mutable state at wide scope:**

```python
@pytest.fixture(scope="module")
def counter() -> dict[str, int]:
    return {"value": 0}

def test_a(counter: dict[str, int]) -> None:
    counter["value"] += 1
    assert counter["value"] == 1    # passes

def test_b(counter: dict[str, int]) -> None:
    counter["value"] += 1
    assert counter["value"] == 1    # fails: depends on test_a having run
```

## Factory fixtures

When tests need several instances or varied data, return a builder:

```python
from collections.abc import Callable

@pytest.fixture
def make_user() -> Callable[..., User]:
    def _make(name: str = "Alice", age: int = 30, active: bool = True) -> User:
        return User(name=name, age=age, active=active)
    return _make

def test_inactive_users_are_filtered(make_user: Callable[..., User]) -> None:
    users = [make_user(), make_user(name="Bob", active=False)]
    assert [u.name for u in active_only(users)] == ["Alice"]
```

Defaults express the "typical" object; each test overrides only what it is about.

## conftest.py

```
tests/
├── conftest.py            # fixtures for the whole suite
├── unit/
│   └── test_pricing.py
└── integration/
    ├── conftest.py        # db fixtures, only for integration/
    └── test_repository.py
```

- Discovered automatically; never imported.
- Nearest `conftest.py` wins when two define the same fixture name.
- Keep fixtures close to where they are used — a root `conftest.py` with 50
  fixtures is a smell.
- `pytest --fixtures` lists available fixtures with their docstrings.

## autouse

```python
@pytest.fixture(autouse=True)
def _freeze_env(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.delenv("APP_ENV", raising=False)
```

Reserve `autouse` for isolation guards (env, global state reset). Anything a
test **uses** must be requested explicitly, so readers can see it.
