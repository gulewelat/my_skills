# Versioning — details

Complements section 6 of the skill.

## A new version only for breaking changes

| Non-breaking — same version | Breaking — new major version |
|---|---|
| New endpoint | Endpoint removed or path renamed |
| New optional request field or query parameter | New required field, or optional → required |
| New response field | Response field removed, renamed or retyped |
| Looser validation | Stricter validation (shorter max length, new format) |
| | Status code or error `type` changed for the same case |
| | Default `sort` or page size changed |

- **New enum value in a response** is technically non-breaking but breaks
  clients that match exhaustively — announce it.
- Clients must ignore unknown response fields; the API must not rely on it
  for request fields (`extra="forbid"`, section 3).

## Two versions side by side

- A new version is a new **router and schema set**; use cases and domain are
  shared. If the domain must change, the old version's router adapts to it.
- **Deprecate before removing**: mark the old routes, announce a date, keep them
  running until then.

```python
@router.get("/orders", deprecated=True)          # flagged in OpenAPI
def list_orders(response: Response, orders: OrderServiceDep) -> Page[OrderOutV1]:
    response.headers["Deprecation"] = "@1782864000"  # RFC 9745: deprecated since 2026-07-01
    response.headers["Sunset"] = "Fri, 01 Jan 2027 00:00:00 GMT"  # RFC 8594: removal
    ...
```
