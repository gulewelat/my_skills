---
name: clean-code-python
description: >
  Clean Code principles (Robert C. Martin) applied to Python: naming, function and
  class design, comments, formatting and linting with ruff, error handling, code
  smells, and the limits of these rules. Use when reviewing or refactoring Python
  code for readability and design quality.
---

# Clean Code — Python

This skill applies the principles from *Clean Code* (Robert C. Martin) to Python code.
Its purpose: help Claude review, refactor, and explain code quality decisions with
precision and educational value.

Conventions: Python 3.11+, `mypy --strict` — every "Good" example below is fully typed.

## Reference files

| File | When |
|---|---|
| [references/worked-example.md](references/worked-example.md) | Refactoring request: the expected approach, from diagnosis to stopping point |

---

## 1. Core Philosophy

> "The only way to go fast is to keep the code clean at all times."

Clean code is **readable by humans first**. Every decision below serves that goal.

- Code is read far more than it is written.
- Bad code compounds: messy code slows teams asymptotically toward zero productivity.
- The **Boy Scout Rule**: always leave code a little cleaner than you found it.

---

## 2. Naming

**The single most important clean code skill.** A good name eliminates the need for a comment.

### Rules
| Rule | Bad | Good |
|------|-----|------|
| Reveal intent | `d`, `tmp`, `x` | `elapsed_days`, `user_cache`, `retry_count` |
| Avoid disinformation | `account_list` (if it's a set) | `accounts` |
| Avoid noise words | `get_data_info()` | `get_account()` |
| Use pronounceable names | `genymdhms` | `generation_timestamp` |
| Use searchable names | `86400` | `SECONDS_PER_DAY = 86400` |
| Avoid encodings | `str_name`, `m_name` | `name` |
| Classes = nouns | `ProcessData` | `DataProcessor` |
| Functions = verbs | `data()` | `fetch_data()` |
| Booleans = predicates | `flag`, `check` | `is_valid`, `has_permission` |
| One word per concept | `fetch_user`, `get_order`, `retrieve_invoice` | `get_user`, `get_order`, `get_invoice` |

### Python specifics
```python
# Bad
def calc(x, lst):
    res = []
    for i in lst:
        if i[2] == 4:
            res.append(i)
    return res

# Good
FLAGGED: Final = 4

@dataclass(frozen=True)
class Cell:
    row: int
    col: int
    status: int

def flagged_cells(board: Iterable[Cell]) -> list[Cell]:
    return [cell for cell in board if cell.status == FLAGGED]
```

Name length follows scope: `i` is fine in a 2-line comprehension, a module-level
constant needs a full name. Follow PEP 8 casing: `snake_case` functions and
variables, `PascalCase` classes, `UPPER_SNAKE` constants, `_leading` for private.

---

## 3. Functions

### Rules (in priority order)

1. **Do one thing** — if you can extract a meaningful sub-function, the function does more than one thing.
2. **Small** — aim for 5–15 lines; rarely exceed 20. No hard limit, but question anything longer.
3. **One level of abstraction per function** — don't mix high-level logic with low-level details in the same function.
4. **Limit arguments** — 0 is ideal, 1–2 is good, 3 is questionable, 4+ needs a dataclass/object.
5. **No flag arguments** — a `bool` parameter announces that the function does two things.
6. **No side effects** — a function named `check_password` must NOT also initialize a session.
7. **Command-Query Separation** — a function either *does* something (command) or *returns* something (query), not both.
8. **Prefer exceptions over error codes** — avoid nested `if result == ERROR_CODE` chains.
9. **Don't Repeat Yourself (DRY)** — duplication is the root of all evil in software.

### Python specifics
```python
# Bad: does multiple things, mixed abstraction levels, side effects
def process(data, db, flag=False):
    cleaned = [x.strip().lower() for x in data if x]
    result = []
    for item in cleaned:
        r = db.query(f"SELECT * FROM t WHERE name='{item}'")  # SQL injection!
        if r:
            result.append(r[0])
    if flag:
        db.update("SET last_run = NOW()")
    return result

# Good: separated concerns, no side effects, descriptive names
def normalize_names(raw_names: Iterable[str]) -> list[str]:
    return [name.strip().lower() for name in raw_names if name.strip()]

def find_records(names: Iterable[str], repo: RecordRepository) -> list[Record]:
    return [record for name in names if (record := repo.find_by_name(name))]

def mark_last_run(repo: RecordRepository, now: datetime) -> None:
    repo.set_last_run(now)
```

---

## 4. Comments

**A comment is a failure** — it means the code didn't speak for itself.

### Avoid these comment types
- Redundant comments (`# increment i by 1` above `i += 1`)
- Noise comments (`# Constructor`, `# Default constructor`)
- Commented-out code — delete it; version control remembers it
- Misleading or outdated comments

### Keep these comment types
- **Legal / license headers**
- **Intent explanations** for genuinely non-obvious decisions (e.g. a specific algorithm choice)
- **Warning comments** (`# NOT thread-safe, uses global state`)
- **TODO comments** — only if tracked (link to issue)
- **Docstrings**: don't add them if absent, but preserve and respect existing ones

```python
# Bad: misleading comment, cryptic name
def get_kc(x):
    # multiply by 1.8 and add 32
    return x * 1.8 + 32  # formula for kelvin

# Good — name makes the comment unnecessary
def celsius_to_fahrenheit(celsius: float) -> float:
    return celsius * 1.8 + 32
```

---

## 5. Formatting

Consistent formatting signals professionalism. Delegate everything mechanical to
**ruff**, a single tool for both:

- **`ruff format`** — Black-compatible formatter: one canonical style, nothing to debate.
- **`ruff check`** — linter; also sorts imports (rule set `I`, replaces isort) and
  turns several Clean Code rules into automatic checks.

```toml
# pyproject.toml
[tool.ruff]
line-length = 88
target-version = "py311"

[tool.ruff.lint]
select = [
    "E", "W", "F",  # pycodestyle, pyflakes
    "I",            # import sorting
    "N",            # PEP 8 naming
    "UP",           # modern syntax
    "B",            # likely bugs
    "SIM",          # simplifiable code
    "C90",          # cyclomatic complexity
    "PLR",          # too many args / branches / statements
    "ARG",          # unused arguments
    "ERA",          # commented-out code
    "BLE",          # blind `except Exception`
]

[tool.ruff.lint.mccabe]
max-complexity = 10
```

| Clean Code rule | Enforced by |
|---|---|
| No commented-out code (§4) | `ERA001` |
| Small functions, few branches (§3) | `C901`, `PLR0912`, `PLR0915` |
| Limit arguments (§3) | `PLR0913` (backstop — the review rule stays stricter) |
| Catch narrowly (§7) | `E722` bare `except`, `BLE001` |
| PEP 8 naming (§2) | `N` |
| No magic numbers | `PLR2004` |

Run it on every commit with **pre-commit**:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.17.6               # `pre-commit autoupdate` to refresh
    hooks:
      - id: ruff-check
        args: ["--fix"]
      - id: ruff-format
```

```bash
uv add --dev ruff pre-commit
uv run pre-commit install        # registers the hooks in .git/hooks/
uv run ruff check --fix . && uv run ruff format .   # manual run
```

`ruff-check` runs before `ruff-format`: fixes can change layout. If a hook
modifies files, the commit is blocked — review and recommit.

Tools handle layout; the rules below are what they can't decide.

### Vertical formatting
- **Newspaper metaphor**: high-level concepts at the top, details below.
- Related functions should be close to each other (caller above callee).
- Empty lines between logical sections (like paragraphs).

### Horizontal formatting
- Max line length: **88 chars** (ruff/Black default) — never exceed 120.
- No horizontal alignment (wastes time maintaining it).
- Use whitespace around operators.

### Python specifics
```python
# Bad — no separation of concerns, dense
class User:
    def __init__(self,name,email,role):
        self.name=name;self.email=email;self.role=role
    def validate(self):
        if not self.email or '@' not in self.email: return False
        if self.role not in ['admin','user','guest']: return False
        return True

# Good — reads top-down: public predicate first, details below
VALID_ROLES: Final = frozenset({"admin", "user", "guest"})

@dataclass(frozen=True)
class User:
    name: str
    email: str
    role: str

    def is_valid(self) -> bool:
        return self._has_valid_email() and self._has_valid_role()

    def _has_valid_email(self) -> bool:
        return "@" in self.email

    def _has_valid_role(self) -> bool:
        return self.role in VALID_ROLES
```

---

## 6. Objects and Data Structures

### The key distinction
| | Data Structure / NamedTuple / dataclass | Object (class with behavior) |
|---|---|---|
| Exposes | Data fields directly | Behavior via methods |
| Hides | Nothing | Internal representation |
| Easy to add | New functions | New types (polymorphism) |

- **Don't create hybrid classes** that half-expose data and half-hide it.
- **Law of Demeter**: a method should only call methods on: itself, its arguments, objects it creates, direct components. Avoid `a.b.c.do()`.

```python
# Violation of Law of Demeter
output_dir = ctx.options.output.path.parent

# Better (ask, don't dig)
output_dir = ctx.get_output_directory()
```

```python
# Bad: hybrid — public mutable fields AND invariants that callers can break
class Account:
    def __init__(self) -> None:
        self.balance = Decimal(0)        # anyone can write account.balance = -100

    def withdraw(self, amount: Decimal) -> None:
        if amount > self.balance:
            raise InsufficientFundsError(amount, self.balance)
        self.balance -= amount

# Good: an object hides its state behind behaviour
class Account:
    def __init__(self) -> None:
        self._balance = Decimal(0)

    @property
    def balance(self) -> Decimal:
        return self._balance

    def withdraw(self, amount: Decimal) -> None:
        if amount > self._balance:
            raise InsufficientFundsError(amount, self._balance)
        self._balance -= amount
```

Pure data (DTOs, rows, events) → `@dataclass(frozen=True)` with no logic.
Domain concepts with rules → objects with private state.

---

## 7. Error Handling

- **Use exceptions, not return codes.** Return codes pollute the call chain.
- **Raise when absence is an error**; return `T | None` only when absence is a
  normal outcome (a lookup, a search) — mypy then forces callers to handle it.
- **Don't pass `None`** as an argument to mean "default" — use a real default or a separate function.
- **Provide context** in exceptions: what operation failed, and why.
- **Chain the cause** with `raise ... from error`.
- **Catch narrowly**: never bare `except:` or `except Exception: pass`.
- **Exceptions are part of the interface** — document them.

```python
# Bad: None signals an error the caller will forget to check
def get_user(user_id: int) -> User | None:
    return db.get(user_id)

user = get_user(42)
send_invoice(user.email)    # AttributeError far from the cause

# Good: absence is an error here — fail at the source, with context
class UserNotFoundError(LookupError):
    def __init__(self, user_id: int) -> None:
        super().__init__(f"No user found with id={user_id}")
        self.user_id = user_id

def get_user(user_id: int) -> User:
    user = db.get(user_id)
    if user is None:
        raise UserNotFoundError(user_id)
    return user
```

```python
# Bad: swallows everything, including typos and KeyboardInterrupt; cause lost
try:
    config = json.loads(path.read_text())
except:
    config = {}

# Good: narrow, contextual, chained
class ConfigError(Exception): ...

def load_config(path: Path) -> Config:
    try:
        raw = json.loads(path.read_text())
    except FileNotFoundError as error:
        raise ConfigError(f"Config file missing: {path}") from error
    except json.JSONDecodeError as error:
        raise ConfigError(f"Invalid JSON in {path} at line {error.lineno}") from error
    return Config.from_dict(raw)
```

Keep `try` blocks small: wrap only the line that can fail, so the handler
can't catch an error from unrelated code.

---

## 8. Unit Tests

Tests are **first-class code**. Dirty tests are worse than no tests: they rot and block refactoring.

The core Clean Code principle that applies here: **one concept per test**. A test that asserts five unrelated things tells you nothing useful when it fails.

> ⚠️ For everything else — fixtures, parametrize, mocking, pytest configuration — defer to the **pytest-python** skill, which covers this in full depth.

The only thing to keep in mind from a clean code perspective when reviewing test code:

- A test that is **hard to name** reveals a function under test that does too many things.
- A test that requires **10 lines of setup** reveals excessive coupling in production code.
- A test that needs **4+ mocks** reveals a function that touches too many dependencies.

These are signals to fix the production code, not the test.

---

## 9. Classes

### Single Responsibility Principle (SRP)
A class should have **one reason to change**. If you need "and" to describe what it does, split it.

```python
# Bad: Report knows how to build, format AND deliver
class Report:
    def generate(self): ...
    def print_to_pdf(self): ...
    def send_by_email(self): ...

# Good: each class has one job
class ReportGenerator:
    def generate(self) -> ReportData: ...

class PdfExporter:
    def export(self, report: ReportData) -> bytes: ...

class ReportEmailer:
    def send(self, report: ReportData, recipient: str) -> None: ...
```

### Open/Closed Principle (OCP)
Open for extension, closed for modification. Prefer composition and `Protocol`s over inheritance —
once variation has actually appeared (see §11).

### Keep classes small
- Measure by **responsibilities**, not lines of code.
- Aim for high **cohesion**: all methods use most instance variables.
- Low cohesion → the class should be split.

---

## 10. Common Code Smells (with Python fixes)

| Smell | Symptom | Fix |
|-------|---------|-----|
| Long Method | Function > 20 lines | Extract sub-functions |
| Long Parameter List | 4+ args | Use dataclass or config object |
| Duplicate Code | Same logic in 2+ places | Extract to shared function |
| Divergent Change | One class changes for multiple reasons | Apply SRP, split class |
| Feature Envy | Method uses another class's data heavily | Move method to that class |
| Data Clumps | Same 3 vars always together | Create a dataclass |
| Primitive Obsession | String/int for domain concepts | Create domain types (`Email`, `UserId`) |
| Switch Statements | `if type == 'X'...elif type == 'Y'` | Polymorphism or dict dispatch |
| Speculative Generality | Abstract class/hooks nobody uses | YAGNI — delete it |
| Dead Code | Commented-out or unreachable code | Delete it |
| Magic Numbers | `if status == 4` | `if status == FLAGGED_STATUS` |

---

## 11. When Not to Apply

Clean Code rules are heuristics. Applied mechanically they produce
over-engineered code — which is not clean either.

- **No abstraction for a single case.** No `Protocol`, registry, base class or
  factory until a second real variant exists.
- **A short, stable `if/elif` stays an `if`.** Two or three branches that rarely
  change don't need polymorphism or dict dispatch.
- **Rule of three for duplication.** Two similar blocks may diverge; extract on
  the third occurrence, or when they must change together.
- **Group arguments only if they form a concept.** Four unrelated parameters
  don't become a dataclass just to reach "≤ 3 args".
- **Don't split a cohesive function to hit a line count.** A 25-line function
  at a single level of abstraction beats five 5-line fragments read in sequence.
- **Stay in scope.** In a review or a targeted refactor, fix what was asked;
  list other findings instead of rewriting neighbouring code.
- **Respect the codebase.** Follow existing conventions and patterns unless
  they are the problem being fixed.

When a rule and readability disagree, readability wins — say which rule you
are deliberately not applying and why.

---

## 12. Review Checklist

When reviewing or refactoring Python code, go through these in order:

- [ ] **Names**: Every name reveals intent without a comment
- [ ] **Functions**: Each function does one thing; no side effects; no flag arguments
- [ ] **Arguments**: ≤ 3 args per function; use dataclasses for more
- [ ] **Comments**: No redundant/misleading comments; existing docstrings preserved
- [ ] **Formatting**: `ruff format` and `ruff check` clean; logical grouping with blank lines
- [ ] **Error handling**: Exceptions with context and cause; no bare `except:`; `None` only for normal absence
- [ ] **Tests**: FIRST properties; one concept per test; Given-When-Then structure
- [ ] **Classes**: SRP applied; high cohesion; no hybrid data/behavior leaks
- [ ] **DRY**: No duplicated logic anywhere
- [ ] **Proportion**: No abstraction or indirection added without a present need (§11)
- [ ] **Boy Scout Rule**: Is this code measurably cleaner than before?

---

## 13. How to Use This Skill

**For code review requests**: run through the checklist in §12, report findings by category, and suggest concrete rewrites.

**For refactoring requests**: identify the top 2–3 smells (§10), explain why each hurts, then provide a rewritten version with before/after comparison. Follow the step-by-step format of [references/worked-example.md](references/worked-example.md).

**For "write clean code" requests**: apply the rules from §2–9 within the limits of §11 from the start, and call out design decisions explicitly.

**For test review**: use §8 to diagnose what the hard-to-test code reveals about production code quality. For pytest-specific guidance (fixtures, mocking, etc.), use the pytest-python skill.

Always explain the *why* behind each suggestion, grounded in the principles above.
