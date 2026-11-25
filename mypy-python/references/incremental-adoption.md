# Incremental adoption

For an existing untyped codebase. New code starts directly at `strict = true`.

## 1 — Order of work

Start with what is exposed, then descend:

| Project type | Start with |
|---|---|
| Script / CLI | `main.py` |
| REST API | `app/routers/` |
| Worker / tasks | `app/tasks/` |
| Library | public API — `__init__.py` |

Typing the boundary first gives callers the benefit immediately and makes
mypy infer more inside.

## 2 — Per-module strictness

```toml
[tool.mypy]
python_version = "3.11"
files = ["src"]
# Global baseline: lenient but not blind
check_untyped_defs = true
warn_unused_ignores = true

# New and critical code: strict
[[tool.mypy.overrides]]
module = ["app.core.*", "app.domain.*"]
strict = true

# Being migrated
[[tool.mypy.overrides]]
module = "app.utils.*"
disallow_untyped_defs = true
warn_return_any = true

# Legacy, not touched yet — with a ticket
[[tool.mypy.overrides]]
module = "app.legacy.*"
ignore_errors = true
```

Move modules up one level at a time; never move one down.
The goal state is the strict baseline in the main skill (section 2.1).

## 3 — Baseline existing errors

On a large codebase, freeze current errors and block only new ones:

```bash
uv run mypy > mypy-baseline.txt || true
```

A tool such as `mypy-baseline` (`uv add --dev mypy-baseline`) filters CI output
against that file. Shrink the baseline with each PR touching a module.

## 4 — Ignore rules during migration

- Every ignore carries an error code and a reason (ticket when possible).
- Run with `warn_unused_ignores` so ignores disappear as code gets typed.
- Untyped third-party library: look for `types-<pkg>` stubs first, then a
  scoped `ignore_missing_imports` override, then a typed wrapper module.

## 5 — Signals of progress

```bash
uv run mypy --txt-report mypy-report src/   # per-module imprecision stats (needs lxml)
```

Track the share of modules under `strict = true`, and the count of
`# type: ignore` (`grep -rc "type: ignore" src`).
