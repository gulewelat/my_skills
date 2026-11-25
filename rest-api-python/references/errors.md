# Error handling — full implementation

Implements section 4 of the skill: RFC 9457 Problem Details for every error,
and a single mapping from domain exceptions to HTTP status codes.
Verified with FastAPI, `mypy --strict` (pydantic plugin) and ruff.

## Error handlers

Call `install_error_handlers(app)` once, where the app is assembled.

```python
import logging
from collections.abc import Mapping
from dataclasses import dataclass
from http import HTTPStatus
from typing import Final

from fastapi import FastAPI, Request, status
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

logger = logging.getLogger(__name__)


@dataclass(frozen=True)
class ErrorSpec:
    status: int
    code: str


# 409 vs 422, 404 vs 403: project conventions (section 2) — they live here.
ERRORS: Final[Mapping[type[DomainError], ErrorSpec]] = {
    UserNotFoundError: ErrorSpec(status.HTTP_404_NOT_FOUND, "user-not-found"),
    EmailAlreadyUsedError: ErrorSpec(status.HTTP_409_CONFLICT, "email-already-used"),
    OrderAlreadyShippedError: ErrorSpec(
        status.HTTP_409_CONFLICT, "order-already-shipped"
    ),
}


def problem(status_code: int, code: str, detail: str, **extra: object) -> JSONResponse:
    return JSONResponse(
        status_code=status_code,
        media_type="application/problem+json",
        content={
            "type": f"/problems/{code}",
            "title": code.replace("-", " ").capitalize(),
            "status": status_code,
            "detail": detail,
            **extra,
        },
    )


def spec_for(error: DomainError) -> ErrorSpec:
    for cls in type(error).__mro__:          # subclasses inherit their parent's mapping
        if cls in ERRORS:
            return ERRORS[cls]
    raise LookupError(f"No HTTP mapping for {type(error).__name__}")


def install_error_handlers(app: FastAPI) -> None:
    @app.exception_handler(DomainError)
    async def on_domain_error(_request: Request, error: DomainError) -> JSONResponse:
        spec = spec_for(error)
        return problem(spec.status, spec.code, str(error))

    @app.exception_handler(RequestValidationError)
    async def on_validation_error(
        _request: Request, error: RequestValidationError
    ) -> JSONResponse:
        errors = [
            {"field": ".".join(map(str, e["loc"])), "message": e["msg"]}
            for e in error.errors()
        ]
        return problem(
            status.HTTP_422_UNPROCESSABLE_CONTENT,
            "validation-error",
            "The request is invalid.",
            errors=errors,
        )

    @app.exception_handler(StarletteHTTPException)   # unknown routes, 405, ...
    async def on_http_error(
        _request: Request, error: StarletteHTTPException
    ) -> JSONResponse:
        code = HTTPStatus(error.status_code).phrase.lower().replace(" ", "-")
        return problem(error.status_code, code, str(error.detail))

    @app.exception_handler(Exception)
    async def on_unexpected_error(request: Request, _error: Exception) -> JSONResponse:
        logger.exception("Unhandled error on %s %s", request.method, request.url.path)
        return problem(
            status.HTTP_500_INTERNAL_SERVER_ERROR,
            "internal-error",
            "An unexpected error occurred.",
        )
```

## What each handler covers

| Handler | Catches | Status |
|---|---|---|
| `on_domain_error` | Any `DomainError` subclass | From `ERRORS` (walks the class hierarchy) |
| `on_validation_error` | Invalid body, path or query parameters | `422`, with `errors` list |
| `on_http_error` | Unknown route, wrong method, any `HTTPException` | Its own status |
| `on_unexpected_error` | Everything else | `500`, logged with traceback |

## Test that every domain exception is mapped

```python
def all_subclasses(cls: type[DomainError]) -> set[type[DomainError]]:
    direct = set(cls.__subclasses__())
    return direct.union(*(all_subclasses(sub) for sub in direct))


def test_every_domain_error_has_an_http_mapping() -> None:
    for error_type in all_subclasses(DomainError):
        assert spec_for(error_type("probe")).code
```

`spec_for` raises `LookupError` for an unmapped exception, so the test names the
missing class. Import every module that defines domain exceptions before running
it (importing the application does that).

## Testing the format

```python
def test_unknown_order_returns_problem_details(client: TestClient) -> None:
    response = client.post("/orders/999/cancellation")

    assert response.status_code == status.HTTP_404_NOT_FOUND
    assert response.headers["content-type"] == "application/problem+json"
    assert response.json()["type"] == "/problems/order-not-found"
```
