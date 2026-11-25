---
name: pull-requests
description: >
  Write pull request (merge request) titles and descriptions: scope of a PR, title
  format, description template, testing and impact sections, and how to gather
  context from the branch before writing. Use when opening a PR, drafting its
  description, or reviewing whether a PR is ready to submit.
---

# Pull requests

A PR description is the public record of a change: what changes, why, and what
a reviewer needs to judge it. It outlives the review — it is read again when
someone runs `git blame` months later.

Companion skill: **atomic-commits** — a PR is a series of atomic commits.

---

## 1. Principles

- **One PR = one goal**, small enough to review in one sitting. A title that
  needs "and" is two PRs.
- **Write for a reviewer without context**, and for whoever reads the history later.
- **Explain the why and the choices** — never paraphrase the diff.
- **Describe only what was actually done and tested.** Nothing announced that
  the branch doesn't contain; no test claimed that wasn't run.
- **Length follows the change**: three lines for a typo fix, the full template
  for a feature with a migration.
- **Language**: the repository's — English skill, French PRs if the team writes French.

---

## 2. Title

Same format as commits: `<type>(<scope>): <description>` (atomic-commits §2).

| ✅ | ❌ |
|---|---|
| `feat(auth): add password reset by email` | `Fix bug` |
| `fix(api): return 404 for orders of other customers` | `Updates` |
| `refactor(mail): extract template renderer` | `Phase 1`, `feature/password-reset` |

With **squash merge**, the PR title becomes the commit on the main branch — it
must stand on its own in `git log`.

---

## 3. Template

**Use the repository's template if there is one** (`.github/pull_request_template.md`,
`.gitlab/merge_request_templates/`). Otherwise:

```markdown
## Context
What problem, for whom. Link the ticket (`Closes #123`).

## Changes
- Main changes, grouped by intent — not file by file.

## Design decisions
Choices made, alternatives rejected and why, known limitations.

## How to test
Commands or steps, and the expected result.

## Impact
Breaking changes, migrations, configuration, rollback.

## Review notes
Where to look first; what is deliberately out of scope.
```

- **Delete empty sections** — an empty "Impact" reads as "no impact" only if
  that's true; then write "None".
- **UI change** → before/after screenshots.
- **Open questions** go in Review notes, explicitly: "409 or 422 for a
  shipped-order cancellation — no existing convention, 409 chosen; to confirm."
  (The convention choices of **rest-api-python** are settled here.)

### Example

```markdown
feat(auth): add password reset by email

## Context
Users locked out must contact support to reset their password. Closes #123.

## Changes
- Request and confirmation endpoints
- Single-use token, valid 30 minutes
- Reset email template

## Design decisions
Token stored hashed in the database so it can be revoked; a JWT was rejected
because it can't be invalidated before expiry.

## How to test
`uv run pytest tests/auth` — 14 passed.
Manual: `POST /password-resets` with a known email → email in the dev mailbox.

## Impact
One database migration (`password_reset_tokens` table). No API change for
existing endpoints — `openapi.json` diff only adds the two new paths.

## Review notes
Start with `auth/tokens.py`. Rate limiting is out of scope (#130).
```

---

## 4. Workflow

### Step 1 — Gather context from the branch

```bash
git log --oneline <base>..HEAD        # the commits of this PR
git diff --stat <base>...HEAD         # files touched since the branch point
git diff <base>...HEAD                # the full change
```

`<base>` is the target branch (`main`). Three dots in `diff` compare with the
branch point, ignoring what landed on `main` since.

Also look at: the linked ticket, the repository's PR template, recent merged
PRs for tone and length.

### Step 2 — Check the PR is ready

- The commits are atomic and clean (atomic-commits §4: fixups squashed before review).
- Tests, type check and lint pass on the branch — and the results are what
  "How to test" reports.
- No unrelated change hides in the diff. A refactor found mid-feature is
  either its own PR, or announced in "Changes" and "Review notes".

### Step 3 — Ask for what is missing

Don't invent context. Ask the user for: the business reason, the ticket number,
the alternatives considered, the deployment constraints.

### Step 4 — Propose, wait, create

Show the title and description; create the PR only after validation.

```bash
gh pr create --base main --title "feat(auth): add password reset by email" \
  --body-file - <<'EOF'
## Context
...
EOF
```

- `--draft` when asking for early feedback on an unfinished branch.
- GitLab: `glab mr create`, same principles.
- Push the branch first only if the user asked for it.

---

## 5. Anti-patterns

| ❌ | ✅ |
|---|---|
| Empty description, or the list of commit messages | Context, changes by intent, why |
| Long generic text that repeats the code | Short; the decisions the diff can't show |
| "Tests pass" without having run them | The command run and its result |
| Refactor or formatting hidden in a feature PR | Separate PR, or announced explicitly |
| Context invented to fill the template | Ask the user; delete what's unknown |
| Giant PR "to save review time" | Several PRs, each one goal |
| Breaking change mentioned only in the code | "Impact" section, `!` in the title |

---

## 6. Rules for the agent

- Use the repository's template and conventions when they exist.
- Read `git log <base>..HEAD` and `git diff <base>...HEAD` before writing.
- Ask for missing context instead of inventing it.
- **Propose the text and wait for validation** before creating the PR.
- Report only tests that were run, with their actual result.
- Stay concise: match the length to the size of the change.
