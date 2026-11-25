# Engine-specific notes

How the rules of the skill translate per tool. Load the section for the tools
the project uses. Polars examples verified with Polars 2.2 on Python 3.14.

## Polars

**Lazy (§1)**
- `pl.scan_parquet(...)` / `scan_csv(...)` / `.lazy()`, then one `.collect()`.
- Several outputs of one plan: `pl.collect_all([...])` shares the common work.
- Larger than memory: `.collect(engine="streaming")`, or `sink_parquet(...)` to
  write without materialising.
- From async code: `await lf.collect_async()` — doesn't block the event loop.

**Ports (§2)** — `T` is `pl.LazyFrame` on the way in, `pl.DataFrame` (or a
`LazyFrame` the writer sinks) on the way out:

```python
class Reader(Protocol):
    def read(self, partition: date) -> pl.LazyFrame: ...


class Writer(Protocol):
    def write(self, df: pl.DataFrame, partition: date) -> None: ...


def transform(orders: pl.LazyFrame) -> pl.LazyFrame:
    return (
        orders.filter(pl.col("status") == "paid")
        .group_by("status")
        .agg(pl.col("amount").sum().alias("revenue"), pl.len().alias("orders"))
    )


def run(reader: Reader, writer: Writer, partition: date) -> None:
    orders = validate_schema(reader.read(partition), ORDERS_SCHEMA)
    writer.write(transform(orders).collect(), partition)
```

**Contracts (§5)** — check the schema once, not each row:

```python
ORDERS_SCHEMA = pl.Schema(
    {"order_id": pl.Int64, "amount": pl.Float64, "status": pl.String}
)


class SchemaMismatchError(ValueError): ...


def validate_schema(lf: pl.LazyFrame, expected: pl.Schema) -> pl.LazyFrame:
    actual = lf.collect_schema()          # resolves the schema, reads no data
    if actual != expected:
        raise SchemaMismatchError(f"expected {dict(expected)}, got {dict(actual)}")
    return lf
```

Explicit schema for text formats: `pl.scan_csv(path, schema=ORDERS_SCHEMA)`.

**Tests (§9)** — `from polars.testing import assert_frame_equal`.

## Spark (PySpark)

**Lazy (§1)**
- Transformations are lazy; actions (`count`, `collect`, `show`, `write`) run a
  job. No action inside a loop.
- Reused DataFrame: `.cache()` / `.persist()`, then `.unpersist()` when done.
  Caching is itself lazy — the first action fills it.

**Idempotence (§4)**
- Partition overwrite: set `spark.sql.sources.partitionOverwriteMode` to
  `dynamic`. With the default (`static`), `mode("overwrite")` on a partitioned
  table deletes every partition, not only the ones written.
- Upserts: `MERGE INTO` on Delta or Iceberg tables.

**Contracts (§5)** — explicit schema on read: `spark.read.schema(schema).csv(...)`;
no `inferSchema` in production.

**Tests (§9)** — a local `SparkSession` fixture with `scope="session"` (it is
slow to start); `pyspark.testing.assertDataFrameEqual`.

## pandas

- Everything is eager: load only what is needed — `columns=` (Parquet),
  `usecols=` and `dtype=` (CSV).
- Larger than memory: `chunksize=` on `read_csv`, processed chunk by chunk.
- No in-place mutation in transformations: return a new frame
  (`df.assign(...)`, not `df["col"] = ...` on the input).
- Tests: `pandas.testing.assert_frame_equal`.

## Async clients (§8)

| Need | Library |
|---|---|
| HTTP | `httpx.AsyncClient` — tests: `httpx.MockTransport` |
| AWS | `aiobotocore` (async botocore) |
| Sync SDK with no async API | `asyncio.to_thread(sdk_call, ...)` |

Pass the client's call as `fetch` to `gather_bounded`:

```python
async with httpx.AsyncClient(timeout=30) as client:
    async def fetch(url: str) -> object:
        response = await client.get(url)
        response.raise_for_status()
        payload: object = response.json()
        return payload

    results, failures = await gather_bounded(urls, fetch, max_concurrency=10)
```

## Contract validation libraries (§5)

| Library | Fits |
|---|---|
| Pydantic | Records, API payloads — parse into models |
| Pandera | DataFrame schemas and checks — Polars, pandas, PySpark |
| dbt tests | Warehouse tables, when transformations live in dbt |
