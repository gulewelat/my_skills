# API testing — details

Complements section 8 of the skill. General pytest practice: **pytest-python**.

## App fixture with in-memory adapters

The app is built by a factory, so tests assemble it with fake adapters — the
HTTP stack (routing, validation, error handlers) runs for real.

```python
@pytest.fixture
def client() -> Iterator[TestClient]:
    app = create_app(container=build_container(orders=InMemoryOrderRepository()))
    with TestClient(app) as client:
        yield client


def test_cancelling_a_shipped_order_returns_problem_details(client: TestClient) -> None:
    client.post("/orders", json=SHIPPED_ORDER)

    response = client.post("/orders/1/cancellation")

    assert response.status_code == status.HTTP_409_CONFLICT
    assert response.json()["type"] == "/problems/order-already-shipped"
```


`create_app` builds the `FastAPI` instance, stores the container in
`app.state.container`, installs the error handlers and includes the routers;
`main.py` calls it with the production container.

## What to test per endpoint

- The success status and response shape.
- **Each error status the endpoint can return** — with the Problem Details `type`.
- Validation: one missing field, one unknown field (`extra="forbid"`) → `422`.
- Collections: pagination walk, `limit` above maximum, malformed cursor
  (see [pagination.md](pagination.md)).

Business rules themselves are tested on the domain and use cases, without HTTP.

## OpenAPI snapshot — catch accidental breaking changes

```python
import json
from pathlib import Path

SNAPSHOT = Path(__file__).parent / "openapi.json"


def test_openapi_contract_is_unchanged() -> None:
    current = create_app(container=build_container()).openapi()

    if not SNAPSHOT.exists():
        SNAPSHOT.write_text(json.dumps(current, indent=2, sort_keys=True) + "\n")
        pytest.fail("Snapshot created — review and commit openapi.json")

    assert current == json.loads(SNAPSHOT.read_text()), (
        "API contract changed: if intended, delete openapi.json, rerun, "
        "and review the diff in the PR against the versioning rules"
    )
```

- The committed `openapi.json` makes every contract change visible in the PR diff.
- An intended change regenerates the file; reviewers decide whether it is
  breaking (section 6 and [versioning.md](versioning.md)).
