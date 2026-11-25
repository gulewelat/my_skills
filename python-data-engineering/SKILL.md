---
name: python-data-engineering
description: >
  Design rules for Python data pipelines — ETL, batch and streaming jobs — whatever
  the engine or orchestrator: lazy evaluation, I/O at the edges behind ports,
  idempotent partitioned writes, data contracts, sync vs async I/O, and pipeline
  tests. Use when writing, reviewing or refactoring a pipeline, a transformation,
  a read/write connector or a scheduled job.
---

# Python data engineering

Conventions: Python 3.14+, `mypy --strict`, `pytest`, `ruff`. The rules hold
for any engine; examples use plain Python (typed records, generators,
`asyncio`).
Companion skills: **hexagonal-architecture-python** (ports, adapters,
composition root), **pytest-python**, **mypy-python**, **clean-code-python**.

## Reference files

| File | When |
|---|---|
| [references/engines.md](references/engines.md) | The project uses Polars, Spark or pandas, an async HTTP or cloud client, or a contract-validation library |

When existing code breaks a rule below, report it before changing anything.

---

## 1. Lazy by default, eager on purpose

- **Build the computation, execute it once**, at the edge of the path. Each
  execution re-reads and recomputes: one per path.
- **Reused intermediate results are materialised on purpose**, with a comment
  saying why — otherwise the engine silently recomputes them.
- **Filter and select early**, so readers skip data they don't need.
- **Larger than memory**: stream — iterators, chunks, or the engine's streaming
  mode — never a full in-memory list of records.
- **No execution for debugging** left in the code (counts, prints, previews).

In plain Python, a generator is the lazy form: the reader yields records, the
transformation consumes them once.

## 2. I/O behind ports

The pipeline depends on ports (`Protocol`), never on a concrete client
(storage SDK, database driver, message broker). Adapters are injected; wiring
happens at the entry point (`main`, orchestrator task, job) — the composition
root of **hexagonal-architecture-python**.

```python
class Reader[T](Protocol):
    def read(self, partition: date) -> T: ...


class Writer[T](Protocol):
    def write(self, data: T, partition: date) -> None: ...  # replaces the partition


@dataclass(frozen=True)
class Order:
    order_id: int
    amount: Decimal
    status: str


@dataclass(frozen=True)
class DailyRevenue:
    day: date
    revenue: Decimal
    orders: int


def daily_revenue(orders: Iterable[Order], day: date) -> DailyRevenue:
    paid = [order.amount for order in orders if order.status == "paid"]
    return DailyRevenue(day=day, revenue=sum(paid, Decimal(0)), orders=len(paid))


def run(
    reader: Reader[Iterable[Order]], writer: Writer[DailyRevenue], partition: date
) -> None:
    writer.write(daily_revenue(reader.read(partition), partition), partition)
```

- `T` is whatever the engine works with — records, a lazy frame, a dataset; the
  shape of `run` doesn't change.
- No client creation and no config reading inside a transformation.
- Every port has a production adapter and an in-memory fake used by the tests (§9).

## 3. Pure transformations, I/O at the edges

- A transformation takes data and returns data: no I/O, no side effect, no
  in-place mutation.
- Reading and writing happen only in adapters and in `run` — functional core,
  imperative shell.

## 4. Idempotence and determinism

- **Running a job twice on the same input gives the same final state**, with no
  duplicates: replace the partition, or `MERGE` / upsert on a natural key.
- **No clock, randomness or global state in transformations.** The run date or
  partition is an explicit parameter, so reruns and backfills reproduce the
  same result.
- **Atomic writes**:
  - local or HDFS: write to a temporary path, then rename;
  - object stores (S3, GCS, Azure): rename is copy + delete, not atomic — use a
    table format (Delta, Iceberg) or write to a new prefix and switch a
    pointer to it once complete.
- **Check what "overwrite" means in your engine**: some replace the whole
  table by default, not only the partitions written (engines.md).

## 5. Data contracts

- **Parse at the entry, don't just validate**: the reader turns raw rows into
  typed records and fails early with a clear message (mypy-python, anti-pattern 14).
- Explicit schemas for text formats (CSV, JSON); no inference in production.
- Quality checks on the data itself: nulls, key uniqueness, expected volume,
  freshness.

```python
class ContractError(ValueError): ...


def parse_order(raw: Mapping[str, object]) -> Order:
    try:
        order_id, amount, status = raw["order_id"], raw["amount"], raw["status"]
    except KeyError as error:
        raise ContractError(f"missing field {error} in {dict(raw)}") from error
    if (
        not isinstance(order_id, int)
        or not isinstance(amount, str)
        or not isinstance(status, str)
    ):
        raise ContractError(f"unexpected types in {dict(raw)}")
    return Order(order_id, Decimal(amount), status)
```

Frame engines validate a schema instead of each row (engines.md).

## 6. Volume and partitioning

- Partition by the read pattern — usually the date. Avoid many small files and
  data skew.
- Prefer incremental loads (watermark, CDC) to full reloads, and plan for late
  data (reprocess a window of recent partitions).

## 7. Robustness and operations

- Retries with backoff on **network I/O only**, never on data errors — a bad
  row fails the same way every time.
- Structured logs with partition, rows read and written, duration. No `print`
  (ruff rule `T201`).
- Configuration and secrets outside the code: environment variables or an
  injected config object.

## 8. Sync vs async

**Async for concurrent network I/O, never for computation.** Transformations
are CPU work: rely on the engine's own parallelism, or `ProcessPoolExecutor`
for CPU-bound Python.

- Async for many API calls, remote files or queries in flight at once.
- **No blocking call inside a coroutine** (sync HTTP client, `time.sleep`, sync
  SDK, sync file read, heavy computation): wrap it with `asyncio.to_thread(...)`
  or use the library's async API.
- **Bound concurrency** with `asyncio.Semaphore` to respect rate limits.
- **Handle every failure explicitly**: `gather(..., return_exceptions=True)` and
  sort successes from failures, or `TaskGroup` when one failure must cancel the rest.
- Transformations stay pure and synchronous; async lives in the adapters.
- **One `asyncio.run(...)`, at the entry point** — never nested in called code.
- An async port gets its own name (`AsyncReader`); no mix of sync and async
  methods in one `Protocol`.

```python
async def gather_bounded[K, V](
    keys: Iterable[K], fetch: Callable[[K], Awaitable[V]], max_concurrency: int = 10
) -> tuple[dict[K, V], dict[K, Exception]]:
    semaphore = asyncio.Semaphore(max_concurrency)

    async def bounded(key: K) -> V:
        async with semaphore:
            return await fetch(key)

    keys = list(keys)  # iterated twice
    outcomes = await asyncio.gather(
        *(bounded(key) for key in keys), return_exceptions=True
    )
    results: dict[K, V] = {}
    failures: dict[K, Exception] = {}
    for key, outcome in zip(keys, outcomes, strict=True):
        if isinstance(outcome, Exception):
            failures[key] = outcome
        elif isinstance(outcome, BaseException):
            raise outcome  # cancellation, KeyboardInterrupt: never swallowed
        else:
            results[key] = outcome
    return results, failures
```

`fetch` is any async call — an HTTP request, an object download, a query — so
the same helper serves every client, and tests pass a fake coroutine.

## 9. Tests

Test practice in general: **pytest-python**. Specific to pipelines:

- **Transformations**: small in-memory inputs, expected values as literals.
- **Orchestration**: in-memory adapters implementing the ports — no mocks of SDKs.
- **Idempotence**: run twice, compare the state.

```python
def test_running_twice_gives_the_same_state() -> None:
    reader, writer = InMemoryOrderReader({DAY: RAW}), InMemoryRevenueWriter()

    run(reader, writer, DAY)
    first = dict(writer.partitions)
    run(reader, writer, DAY)

    assert writer.partitions == first
```

The in-memory writer must replace the partition like the real one — a fake
that appends would hide a non-idempotent production writer, and vice versa.

---

## Review checklist

- [ ] One execution per path, or a commented materialisation
- [ ] Transformations pure: no I/O, no mutation, no clock or randomness
- [ ] I/O behind ports, adapters injected at the entry point
- [ ] Idempotent writes, parameterised by partition or run date
- [ ] Input parsed against an explicit contract; quality checks on key columns
- [ ] Async only for network I/O, no blocking call in a coroutine, bounded concurrency
- [ ] Unit tests without infrastructure, plus an idempotence test
