# Testing anti-patterns

## 1 — Testing several behaviours at once

```python
# ❌ Fails for 4 different reasons; name can't describe it
def test_user() -> None:
    user = create_user("Alice", 30)
    assert user.name == "Alice"
    assert user.is_active is True
    db.save(user)
    assert db.get(user.id) is not None
    assert email_outbox

# ✅ One behaviour each
def test_new_user_is_active() -> None:
    assert create_user("Alice", 30).is_active is True

def test_saved_user_can_be_retrieved(repo: InMemoryUserRepository) -> None:
    user = create_user("Alice", 30)
    repo.save(user)
    assert repo.find(user.id) == user
```

## 2 — Interdependent tests

```python
# ❌ test_update relies on test_create having run first
class TestUserFlow:
    user: User

    def test_create(self) -> None:
        TestUserFlow.user = create_user("Alice")

    def test_update(self) -> None:
        TestUserFlow.user.rename("Bob")
        assert TestUserFlow.user.name == "Bob"

# ✅ Each test builds its own state
def test_rename_changes_name() -> None:
    user = create_user("Alice")
    user.rename("Bob")
    assert user.name == "Bob"
```

Detect with `pytest-randomly` (random order) or by running each test alone
(`pytest path::test_name`).

## 3 — Vague names

```python
# ❌
def test_price() -> None: ...
def test_price_2() -> None: ...

# ✅ The name is the specification
def test_price_with_discount_returns_reduced_amount() -> None: ...
def test_price_with_zero_discount_returns_original_amount() -> None: ...
```

## 4 — Patching your own code

```python
# ❌ String paths coupled to file layout
@patch("app.orders.database")
@patch("app.orders.email_sender")
@patch("app.orders.inventory")
@patch("app.orders.payment_gateway")
def test_process_order(m_db: Mock, m_mail: Mock, m_inv: Mock, m_pay: Mock) -> None: ...

# ✅ Split responsibilities, inject collaborators
def test_confirmed_order_is_saved(repo: InMemoryOrderRepository) -> None:
    confirm_order(order, repo)
    saved = repo.find(order.id)
    assert saved is not None
    assert saved.status == "confirmed"

def test_confirmed_order_notifies_customer() -> None:
    notifier = FakeNotifier()
    notify_confirmation(order, notifier)
    assert notifier.sent_to == [order.customer_email]
```

Four patches means the function under test touches four subsystems.

## 5 — Testing implementation instead of behaviour

```python
# ❌ Breaks on any refactor; also `called_with` is not an assertion
def test_get_user() -> None:
    get_user(1)
    assert db.query.call_count == 1
    assert db.query.called_with("SELECT * FROM users WHERE id = 1")   # always truthy

# ✅ Observable result
def test_get_user_returns_stored_user(repo: InMemoryUserRepository) -> None:
    repo.save(User(id=1, name="Alice"))
    assert get_user(repo, 1) == User(id=1, name="Alice")
```

Verify interactions only when the interaction **is** the behaviour (an email
sent, a payment charged) — and then on the boundary, not on internals.

## 6 — Unspecced mocks

```python
# ❌ EmailSender.send was renamed deliver(); production now calls deliver().
# A bare Mock still has `send` — the test keeps passing on a stale contract.
sender = Mock()
send_welcome(user, sender)
sender.deliver.assert_called_once()   # passes
# ...while any test still configuring sender.send.return_value lies silently.

# ✅ autospec: unknown attributes and wrong signatures raise
sender = create_autospec(EmailSender, instance=True)
```

Better still: a hand-written fake checked by mypy against the `Protocol`.

## 7 — Skips without exit

```python
# ❌ Hidden forever
@pytest.mark.skip(reason="To fix later")
def test_payment_processing() -> None: ...

@pytest.mark.skipif(True, reason="flaky")
def test_sync() -> None: ...

# ✅ Ticket, or xfail that alerts when fixed
@pytest.mark.xfail(reason="Bug #123 — rounding", strict=True)
def test_payment_processing() -> None: ...
```

## 8 — Logic in tests

```python
# ❌ Loops and conditionals: the test needs its own tests
def test_discounts() -> None:
    for price, rate in [(100, 0.1), (50, 0)]:
        if rate:
            assert apply(price, rate) < price
        else:
            assert apply(price, rate) == price

# ✅ parametrize with literal expectations
@pytest.mark.parametrize(("price", "rate", "expected"), [(100, 0.1, 90), (50, 0, 50)])
def test_apply_discount(price: int, rate: float, expected: int) -> None:
    assert apply(price, rate) == expected
```

Expected values are **literals**, not recomputed with the production formula.

## 9 — Non-determinism

```python
# ❌ Depends on the clock, randomness, network, or dict/set order
def test_token_expiry() -> None:
    token = issue_token()
    assert token.expires_at > datetime.now()

# ✅ Inject the source of variability
def test_token_expires_after_one_hour() -> None:
    now = datetime(2026, 1, 1, 12, 0)
    token = issue_token(clock=lambda: now)
    assert token.expires_at == datetime(2026, 1, 1, 13, 0)
```

No `time.sleep` in tests: inject the clock, or poll with a timeout.

## 10 — Errors since pytest 8

```python
# ❌ Test returning a value → error
def test_something() -> bool:
    return compute() is True

# ❌ yield in a test function → error
def test_resource() -> Iterator[int]:
    yield 42

# ✅ assert in tests; yield only in fixtures
def test_something() -> None:
    assert compute() is True
```

## 11 — Hard-to-write test = production smell

| Signal in the test | What it reveals |
|---|---|
| Name needs "and" | Function has several responsibilities |
| 10+ lines of Given | Too many collaborators / hidden coupling |
| 3+ doubles | Function touches too many boundaries |
| Needs `patch` on own module | Missing injection seam |
| Asserts on private attributes | Behaviour not observable through the API |

Fix the production code (see **clean-code-python**), not the test.
