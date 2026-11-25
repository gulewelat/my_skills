# Test doubles and mocking

## Vocabulary

| Double | Role | Typical tool |
|---|---|---|
| **Stub** | Returns canned answers — "what does it return?" | `Mock(return_value=...)`, small class |
| **Spy / Mock** | Records calls to verify interactions — "how was it called?" | `Mock`, recording fake |
| **Fake** | Working lightweight implementation | in-memory repository class |

`unittest.mock.Mock` can play every role; the usage defines it.

## Preference order

1. **Real object** when it is cheap and deterministic (value objects, pure functions).
2. **Hand-written fake** injected through a parameter — type-checked against
   the `Protocol` by mypy, readable, refactor-proof.
3. **`Mock(spec=...)` / `create_autospec`** injected through a parameter —
   when writing a fake is overkill.
4. **`monkeypatch` / `patch`** — only for code you don't control.

## Dependency injection

```python
# ❌ Dependency created inside — only patch can replace it
def send_welcome(user: User) -> None:
    EmailSender().send(user.email, f"Welcome {user.name}!")

# ✅ Dependency received
def send_welcome(user: User, sender: EmailSender) -> None:
    sender.send(user.email, f"Welcome {user.name}!")
```

A function that is hard to test usually instantiates its collaborators.

## Fakes

```python
class InMemoryUserRepository:
    def __init__(self) -> None:
        self._users: dict[int, User] = {}

    def save(self, user: User) -> None:
        self._users[user.id] = user

    def find(self, user_id: int) -> User | None:
        return self._users.get(user_id)

def test_registration_persists_user() -> None:
    repo = InMemoryUserRepository()
    register(repo, name="Alice")
    assert repo.find(1) == User(id=1, name="Alice")
```

Asserts on **state** (what is in the repository), not on calls.

## Mock with a spec

A bare `Mock()` accepts any attribute and any call — typos pass silently.

```python
from unittest.mock import create_autospec

def test_payment_is_charged() -> None:
    gateway = create_autospec(PaymentGateway, instance=True)
    checkout(order, gateway)
    gateway.charge.assert_called_once_with(order.total, "EUR")
```

- `create_autospec` enforces method names **and** signatures.
- Use the `assert_*` methods: `mock.called_with(...)` (no `assert_`) is just an
  auto-created attribute and always truthy.

## side_effect

```python
from unittest.mock import Mock

# Successive results — values or exceptions
api = Mock(side_effect=[ConnectionError("down"), ConnectionError("down"), {"status": "ok"}])

def test_retry_succeeds_on_third_attempt() -> None:
    result = call_with_retry(api, "https://api.example.com")
    assert result == {"status": "ok"}
    assert api.call_count == 3       # the retry count *is* the behaviour here

# Computed result
lookup = Mock(side_effect=lambda key: {"a": 1}.get(key))
```

Frequent `side_effect` gymnastics suggest a hand-written fake would read better.

## When patching is acceptable

```python
# ✅ Environment / global state you don't own
def test_debug_mode(monkeypatch: pytest.MonkeyPatch) -> None:
    monkeypatch.setenv("DEBUG", "1")
    assert load_settings().debug is True

# ✅ Third-party call with no seam — patch where it is *looked up*
def test_fetch_handles_timeout(monkeypatch: pytest.MonkeyPatch) -> None:
    def boom(*args: object, **kwargs: object) -> None:
        raise requests.Timeout
    monkeypatch.setattr("app.client.requests.get", boom)
    assert fetch_status() == "unavailable"
```

Prefer `monkeypatch` (fixture, auto-undone) over `unittest.mock.patch`
decorators. Patch the name **in the module that uses it**
(`app.client.requests.get`), not where it is defined.

For time, inject a clock (`Callable[[], datetime]`) rather than patching
`datetime.now` — patching C-implemented builtins fails.

## Why not patch your own code

```python
@patch("app.services.user.EmailSender")    # breaks on any move or rename
def test_welcome(mock_sender: Mock) -> None: ...
```

- The string path couples the test to file layout, not behaviour.
- Moving a module breaks the test, or worse, silently patches nothing useful.
- It hides the missing injection seam the design needs.

`pytest-mock` (`mocker` fixture) is a convenience over `patch` — same rules apply.
