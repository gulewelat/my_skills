# Cursor pagination — implementation

Implements section 5 of the skill. Verified with FastAPI, `mypy --strict` and pytest.

## Cursor or offset

| | Cursor (default) | Offset (`?offset=40&limit=20`) |
|---|---|---|
| Inserts during browsing | Stable | Items skipped or repeated |
| Deep pages | Constant cost (keyset query) | Slower as offset grows |
| Jump to page N | ❌ | ✅ |

## Cursor encoding

The cursor carries the **sort value and id of the last item returned**. It is
opaque to clients: base64 of a small JSON array.

```python
import base64
import binascii
import json


class InvalidCursorError(DomainError): ...      # mapped to 400 (section 4)


def encode_cursor(sort_value: str | int, last_id: int) -> str:
    raw = json.dumps([sort_value, last_id]).encode()
    return base64.urlsafe_b64encode(raw).decode()


def decode_cursor(cursor: str) -> tuple[str | int, int]:
    try:
        sort_value, last_id = json.loads(base64.urlsafe_b64decode(cursor))
    except (binascii.Error, ValueError, TypeError) as error:
        raise InvalidCursorError("Malformed cursor") from error
    if not isinstance(sort_value, str | int) or not isinstance(last_id, int):
        raise InvalidCursorError("Malformed cursor")
    return sort_value, last_id
```

- Datetimes go in as ISO strings (`created_at.isoformat()`).
- A cursor is only valid with the `sort` it was produced for. Changing `sort`
  restarts from the first page; to enforce it, encode the sort key in the cursor
  and reject a mismatch with `InvalidCursorError`.
- Cursors are not a security boundary: filters and permissions are re-applied
  on every request.

## Keyset query

Sorting by a non-unique column needs a **tie-breaker** (`id`), otherwise rows
with equal values are skipped or repeated across pages.

```sql
-- sort=-created_at, page after cursor (created_at=:after_value, id=:after_id)
SELECT * FROM orders
WHERE status = ANY(:statuses)
  AND (created_at, id) < (:after_value, :after_id)
ORDER BY created_at DESC, id DESC
LIMIT :limit + 1
```

- Ascending sort → `>` instead of `<`, and `ASC` on both columns.
- Index on `(created_at, id)` — the query never scans skipped rows.

## The `limit + 1` trick

Fetch one row more than requested to know whether another page exists, without
a `COUNT`:

```python
def to_page(rows: Sequence[Order], limit: int) -> Page[OrderOut]:
    has_more = len(rows) > limit
    items = rows[:limit]
    next_cursor = (
        encode_cursor(items[-1].created_at.isoformat(), items[-1].id)
        if has_more
        else None
    )
    return Page[OrderOut](
        items=[OrderOut.from_domain(order) for order in items],
        next_cursor=next_cursor,
    )
```

The repository returns domain objects; building the `Page` happens in the
router layer.

## Tests

```python
def test_pages_cover_every_order_exactly_once(client: TestClient) -> None:
    seen: list[int] = []
    params: dict[str, str | int] = {"limit": 2}

    while True:
        page = client.get("/orders", params=params).json()
        seen += [order["id"] for order in page["items"]]
        if page["next_cursor"] is None:
            break
        params["cursor"] = page["next_cursor"]

    assert sorted(seen) == sorted(set(seen)) == ALL_ORDER_IDS


def test_limit_above_maximum_is_rejected(client: TestClient) -> None:
    assert client.get("/orders", params={"limit": 500}).status_code == 422


def test_malformed_cursor_is_rejected(client: TestClient) -> None:
    response = client.get("/orders", params={"cursor": "not-a-cursor"})
    assert response.status_code == 400
    assert response.json()["type"] == "/problems/invalid-cursor"
```

Seed orders with **equal `created_at` values** in the fixture: it is the case
that breaks pagination without a tie-breaker.
