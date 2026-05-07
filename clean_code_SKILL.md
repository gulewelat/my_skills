---
name: clean-code-python
description: >
  Apply Clean Code principles (Robert C. Martin) to Python code. Use this skill
  whenever the user asks to review, refactor, clean up, or improve Python code quality.
  Also trigger when the user mentions: bad code, messy code, hard to read code, code smells,
  naming conventions, function design, class design, test quality, or any request to "make
  my code cleaner / more readable / more professional". Even for partial snippets or
  quick reviews, consult this skill first.
---

# Clean Code — Python

This skill applies the principles from *Clean Code* (Robert C. Martin) to Python code.
Its purpose: help Claude review, refactor, and explain code quality decisions with
precision and educational value.

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

### Python specifics
```python
# Bad
def calc(x, lst, f=False):
    res = []
    for i in lst:
        if i[0] == 4 or f:
            res.append(i)
    return res

# Good
FLAGGED_STATUS = 4

def get_flagged_cells(game_board: list[list], include_all: bool = False) -> list[list]:
    return [
        cell for cell in game_board
        if cell[STATUS_INDEX] == FLAGGED_STATUS or include_all
    ]
```

---

## 3. Functions

### Rules (in priority order)

1. **Do one thing** — if you can extract a meaningful sub-function, the function does more than one thing.
2. **Small** — aim for 5–15 lines; rarely exceed 20. No hard limit, but question anything longer.
3. **One level of abstraction per function** — don't mix high-level logic with low-level details in the same function.
4. **Limit arguments** — 0 is ideal, 1–2 is good, 3 is questionable, 4+ needs a dataclass/object.
5. **No side effects** — a function named `check_password` must NOT also initialize a session.
6. **Command-Query Separation** — a function either *does* something (command) or *returns* something (query), not both.
7. **Prefer exceptions over error codes** — avoid nested `if result == ERROR_CODE` chains.
8. **Don't Repeat Yourself (DRY)** — duplication is the root of all evil in software.

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
def normalize_inputs(raw_items: list[str]) -> list[str]:
    return [item.strip().lower() for item in raw_items if item]

def find_records(names: list[str], db: Database) -> list[Record]:
    return [record for name in names if (record := db.find_by_name(name))]

def mark_last_run(db: Database) -> None:
    db.update_last_run(datetime.utcnow())
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
# Bad
def get_kc(x):
    # multiply by 1.8 and add 32
    return x * 1.8 + 32  # formula for kelvin

# Good — name makes the comment unnecessary
def celsius_to_fahrenheit(celsius: float) -> float:
    return celsius * 1.8 + 32
```

---

## 5. Formatting

Consistent formatting signals professionalism. In Python, delegate all mechanical formatting to tools rather than doing it manually:

- **Black** — opinionated auto-formatter. Rewrites your file to a single canonical style (spacing, trailing commas, line wrapping). No arguments, no config to bikeshed.
- **isort** — sorts and groups `import` statements automatically (stdlib → third-party → local).

In practice you don't run them manually — you wire them as **pre-commit hooks**: they run automatically on every `git commit`, before the commit lands. Bad formatting never reaches the repo.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.3.0
    hooks:
      - id: black

  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort
        args: ["--profile", "black"]  # isort compatible with Black's style
```

```bash
uv add --dev pre-commit
uv run pre-commit install   # registers the hooks in .git/hooks/
```

After `pre-commit install`: on every `git commit`, Black and isort run automatically. If they modify files, the commit is blocked and you recommit the fixed files. Formatting never reaches the repo broken.

### Vertical formatting
- **Newspaper metaphor**: high-level concepts at the top, details below.
- Related functions should be close to each other (caller above callee).
- Empty lines between logical sections (like paragraphs).

### Horizontal formatting
- Max line length: **88 chars** (Black default) — never exceed 120.
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

# Good
class User:
    VALID_ROLES = {'admin', 'user', 'guest'}

    def __init__(self, name: str, email: str, role: str) -> None:
        self.name = name
        self.email = email
        self.role = role

    def is_valid(self) -> bool:
        return self._has_valid_email() and self._has_valid_role()

    def _has_valid_email(self) -> bool:
        return bool(self.email) and '@' in self.email

    def _has_valid_role(self) -> bool:
        return self.role in self.VALID_ROLES
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

---

## 7. Error Handling

- **Use exceptions, not return codes.** Return codes pollute the call chain.
- **Don't return `None`** when you can raise an exception or return a `Null Object`.
- **Don't pass `None`** as an argument (use defaults or Optional types explicitly).
- **Provide context** in exceptions: what operation failed, and why.
- **Exceptions are part of the interface** — document them.

```python
# Bad
def find_user(user_id):
    result = db.query(user_id)
    if not result:
        return None  # caller must remember to check
    return result

# Good
def find_user(user_id: int) -> User:
    user = db.query(user_id)
    if not user:
        raise UserNotFoundError(f"No user found with id={user_id}")
    return user
```

---

## 8. Unit Tests

Tests are **first-class code**. Dirty tests are worse than no tests: they rot and block refactoring.

The core Clean Code principle that applies here: **one concept per test**. A test that asserts five unrelated things tells you nothing useful when it fails.

> ⚠️ For everything else — fixtures, parametrize, mocking, pytest configuration — defer to the **pytest skill**, which covers this in full depth.

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
# Bad: Report knows how to format AND how to print
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
Open for extension, closed for modification. Use inheritance or composition.

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
| Duplicate Code | Same logic in 2+ places | Extract to shared function/mixin |
| Divergent Change | One class changes for multiple reasons | Apply SRP, split class |
| Feature Envy | Method uses another class's data heavily | Move method to that class |
| Data Clumps | Same 3 vars always together | Create a dataclass |
| Primitive Obsession | String/int for domain concepts | Create domain types (`Email`, `UserId`) |
| Switch Statements | `if type == 'X'...elif type == 'Y'` | Polymorphism or dict dispatch |
| Speculative Generality | Abstract class/hooks nobody uses | YAGNI — delete it |
| Dead Code | Commented-out or unreachable code | Delete it |
| Magic Numbers | `if status == 4` | `if status == FLAGGED_STATUS` |

---

## 11. Review Checklist

When reviewing or refactoring Python code, go through these in order:

- [ ] **Names**: Every name reveals intent without a comment
- [ ] **Functions**: Each function does one thing; no side effects
- [ ] **Arguments**: ≤ 3 args per function; use dataclasses for more
- [ ] **Comments**: No redundant/misleading comments; existing docstrings preserved
- [ ] **Formatting**: Black + isort applied; logical grouping with blank lines
- [ ] **Error handling**: Exceptions with context; no bare `except:` or silent `None` returns
- [ ] **Tests**: FIRST properties; one concept per test; AAA structure
- [ ] **Classes**: SRP applied; high cohesion; no hybrid data/behavior leaks
- [ ] **DRY**: No duplicated logic anywhere
- [ ] **Boy Scout Rule**: Is this code measurably cleaner than before?

---

## 12. How to Use This Skill

**For code review requests**: run through the checklist in §11, report findings by category, and suggest concrete rewrites.

**For refactoring requests**: identify the top 2–3 smells (§10), explain why each hurts, then provide a rewritten version with before/after comparison.

**For "write clean code" requests**: apply all applicable rules from §2–9 from the start, and call out design decisions explicitly.

**For test review**: use §8 to diagnose what the hard-to-test code reveals about production code quality. For pytest-specific guidance (fixtures, mocking, etc.), use the pytest skill.

Always explain the *why* behind each suggestion, grounded in the principles above.
