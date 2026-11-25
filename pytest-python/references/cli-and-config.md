# Command line and configuration

## Selecting tests

```bash
uv run pytest                                        # everything under testpaths
uv run pytest tests/test_pricing.py                  # one file
uv run pytest tests/test_pricing.py::test_add        # one function
uv run pytest tests/test_pricing.py::TestDiscount    # one class
uv run pytest "tests/test_math.py::test_add[zero]"   # one parametrized case
uv run pytest -k "discount and not zero"             # name expression
uv run pytest -m "integration and not slow"          # marker expression
```

## Execution control

```bash
uv run pytest -x              # stop at first failure
uv run pytest --maxfail=2     # stop after 2 failures
uv run pytest --lf            # rerun only last failures
uv run pytest --ff            # failures first, then the rest
uv run pytest --sw            # stepwise: stop at failure, resume from it next run
uv run pytest --pdb           # debugger on failure
uv run pytest --durations=10  # 10 slowest tests
```

## Output

```bash
uv run pytest -q              # minimal — CI logs
uv run pytest -v              # one line per test — daily use
uv run pytest -vv             # full diffs on assertion failures
uv run pytest --tb=short      # shorter tracebacks (also: long, line, no)
uv run pytest -rA             # summary of all outcomes (-rs: skips, -rx: xfails)
```

| Normal | Verbose | Meaning |
|---|---|---|
| `.` | `PASSED` | passed |
| `F` | `FAILED` | assertion failed |
| `E` | `ERROR` | error in setup/teardown (fixture) |
| `s` | `SKIPPED` | skipped |
| `x` | `XFAIL` | expected failure, failed |
| `X` | `XPASS` | expected failure, passed |

`E` means the **fixture** broke, not the test assertion — look at setup.

## Coverage (pytest-cov)

```bash
uv add --dev pytest-cov
uv run pytest --cov=src --cov-report=term-missing
uv run pytest --cov=src --cov-report=html          # htmlcov/index.html
uv run pytest --cov=src --cov-fail-under=90
```

## pyproject.toml

```toml
[tool.pytest.ini_options]
minversion = "8.0"
testpaths = ["tests"]
addopts = [
    "--strict-markers",      # unknown marker = error
    "--strict-config",       # invalid config key = error
    "-ra",                   # summary of non-passing outcomes
    "--import-mode=importlib",
]
markers = [
    "slow: slow tests, excluded from the fast loop",
    "integration: tests touching real infrastructure",
]
xfail_strict = true
filterwarnings = ["error"]   # warnings fail the suite; whitelist explicitly

[tool.coverage.run]
source = ["src"]
branch = true

[tool.coverage.report]
show_missing = true
skip_covered = true
exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError", "\\.\\.\\."]
```

- Keep coverage **out of `addopts`**: it slows local runs and breaks `--pdb`.
  Pass `--cov` in CI only.
- `filterwarnings = ["error"]` surfaces deprecations early; add targeted
  `"ignore::DeprecationWarning:some_lib"` lines as needed.

## Shortcuts

uv has no script runner table: put recurring commands in a `Makefile`,
`justfile`, or CI config.

```makefile
test:      ; uv run pytest
test-fast: ; uv run pytest -m "not slow"
test-cov:  ; uv run pytest --cov=src --cov-report=term-missing
```

mypy settings live in the same `pyproject.toml` — see **mypy-python**.
