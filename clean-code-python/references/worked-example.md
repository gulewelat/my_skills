# Worked example — a refactoring request

Shows the **approach**, not a catalogue: diagnose, rank, fix the top 2–3
smells, preserve behaviour, stop, and say what was left alone.

## Input

```python
def process_invoice(data, send=True):
    total = 0
    for l in data["lines"]:
        if l["type"] == "product":
            total += l["qty"] * l["price"]
        elif l["type"] == "service":
            total += l["hours"] * l["rate"]
    if data["country"] == "FR":
        total = total * 1.2
    else:
        total = total * 1.0
    if total > 10000:
        total = total * 0.95
    inv = {"customer": data["customer"], "total": round(total, 2)}
    if send:
        smtp = smtplib.SMTP("mail.example.com")
        smtp.sendmail("billing@example.com", data["email"], f"Total: {inv['total']}")
    return inv
```

## Step 1 — Diagnose and rank

List what you see, then rank by **cost to the reader and risk of bugs**:

| # | Smell | Why it hurts | Action |
|---|---|---|---|
| 1 | Does several things + flag argument + hidden I/O | Can't compute an invoice without risking an email; can't test without SMTP | Split pure computation from sending |
| 2 | Magic numbers, `float` for money | `1.2`, `10000`, `0.95` carry business rules with no name; float rounding on amounts | Named constants, `Decimal` |
| 3 | Untyped dicts in and out | Key typos fail at runtime; shape undocumented | Dataclasses for lines and invoice |
| – | `if type == "product" / "service"` | Two stable branches | **Keep as `if`** (§11) |
| – | Hard-coded SMTP host and sender | Real, but outside the computation | **Report, don't fix** (scope) |

Fix 1–3. Stop there.

## Step 2 — Refactor

```python
from collections.abc import Iterable, Mapping
from dataclasses import dataclass
from decimal import Decimal
from typing import Final

VAT_RATES: Final[Mapping[str, Decimal]] = {"FR": Decimal("0.20")}
BULK_THRESHOLD: Final = Decimal("10000")
BULK_DISCOUNT: Final = Decimal("0.05")
CENT: Final = Decimal("0.01")


@dataclass(frozen=True)
class ProductLine:
    quantity: int
    unit_price: Decimal


@dataclass(frozen=True)
class ServiceLine:
    hours: Decimal
    hourly_rate: Decimal


InvoiceLine = ProductLine | ServiceLine


@dataclass(frozen=True)
class Invoice:
    customer: str
    total: Decimal


def compute_invoice(
    customer: str, country: str, lines: Iterable[InvoiceLine]
) -> Invoice:
    total = apply_bulk_discount(apply_vat(subtotal(lines), country))
    return Invoice(customer=customer, total=total.quantize(CENT))


def subtotal(lines: Iterable[InvoiceLine]) -> Decimal:
    return sum((line_amount(line) for line in lines), start=Decimal(0))


def line_amount(line: InvoiceLine) -> Decimal:
    if isinstance(line, ProductLine):
        return line.quantity * line.unit_price
    return line.hours * line.hourly_rate


def apply_vat(amount: Decimal, country: str) -> Decimal:
    return amount * (1 + VAT_RATES.get(country, Decimal(0)))


def apply_bulk_discount(amount: Decimal) -> Decimal:
    if amount > BULK_THRESHOLD:
        return amount * (1 - BULK_DISCOUNT)
    return amount
```

The caller now decides about sending — the flag disappears:

```python
invoice = compute_invoice(customer, country, lines)
mailer.send(email, f"Total: {invoice.total}")
```

Notes on the choices:
- Read top-down: `compute_invoice` states the policy in one line; details follow.
- Order of operations kept: VAT, then discount on the VAT-inclusive amount —
  a refactoring preserves behaviour, even questionable behaviour.
- `ProductLine | ServiceLine` + `isinstance`: mypy narrows it, no class
  hierarchy needed for two variants.
- Converting the incoming dict into lines belongs to the boundary (API/JSON
  layer) and is not shown.

## Step 3 — Report

End the answer with what changed in behaviour and what was left alone:

- **Behaviour change**: amounts are now `Decimal` — results can differ from
  the float version in the last cent. Same rounding rule (half-even).
- **Signature change**: callers pass typed lines instead of a dict and send
  the email themselves.
- **Not changed, deliberately**: the product/service `if` (two stable
  variants); no `Mailer` abstraction introduced here.
- **Found, out of scope**: SMTP host and sender address are hard-coded; the
  VAT rule only knows France — worth a ticket.

## What not to do with this input

- Introduce a `LineCalculator` protocol with one class per line type.
- Add a `PricingStrategy` / `DiscountPolicy` hierarchy for one discount rule.
- Rewrite the email sending, logging and configuration in the same pass.
- Change the VAT/discount order "because it's more correct" without saying so.
