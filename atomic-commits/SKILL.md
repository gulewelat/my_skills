---
name: atomic-commits
description: >
  Split work into atomic commits and write Conventional Commits messages: grouping
  changes by intent, planning commits, staging whole files or single hunks
  non-interactively, testing each commit, and cleaning history with fixups before
  sharing. Use when committing, planning commits, or tidying a branch before review.
---

# Atomic commits

Conventions: Git ≥ 2.44, Conventional Commits, pre-commit hooks (ruff, mypy).
Commands below are non-interactive: an agent can run all of them.

---

## 1. Principles

- **One commit = one logical change**, describable in one sentence without "and".
- **Every commit leaves the project working**: build, type check and tests pass.
  `git bisect` and `git revert` rely on it.
- **Tests and docs go with the code they cover** — same commit.
- **Separate by nature**: refactoring, formatting, bug fix and new feature never
  share a commit. A reviewer reads a refactor for "nothing changed", a feature
  for "what changed" — mixing them defeats both.
- **Slice vertically**: one behaviour end to end (model + endpoint + test), not
  "all models", then "all endpoints", then "all tests".
- **A feature is a branch of several small commits**, not one big commit.

### Ordering a feature

1. Preparatory refactors first — they make the feature commits small and are
   reviewable on their own.
2. Then the feature, one working slice per commit.
3. Docs last only if they describe the feature as a whole.

```
refactor(mail): extract template renderer
feat(auth): add password reset token model
feat(auth): add password reset endpoint
feat(auth): send password reset email
docs(auth): document password reset flow
```

---

## 2. Message format — Conventional Commits

**Follow the repository's convention first** (`git log --oneline -20`,
`CONTRIBUTING.md`, commitlint config). With no convention, use:

```
<type>(<scope>): <description>

<body: why this change>

<footer: Refs, BREAKING CHANGE, trailers>
```

| Type | For |
|---|---|
| `feat` | New behaviour for users of the code |
| `fix` | Bug fix |
| `refactor` | Code change with no behaviour change |
| `perf` | Performance improvement |
| `test` | Tests only (added or fixed alone) |
| `docs` | Documentation only |
| `style` | Formatting only — no code meaning changes |
| `build` | Build system, dependencies |
| `ci` | CI configuration |
| `chore` | Maintenance that fits nothing above |
| `revert` | Reverting a previous commit |

The spec defines `feat` and `fix`; the rest is the common
`@commitlint/config-conventional` set.

**Description**
- Imperative mood, lowercase, no trailing period: `add`, not `added` / `adds`.
- ~50 characters, 72 at most — it is what `git log --oneline` and PR lists show.
- Says *what* changes for the reader, not which files were touched.

**Scope** — the module or area (`auth`, `api`, `mail`); omit it rather than invent one.

**Body** — wrapped at 72 columns. Explains **why**: the problem, the decision,
the alternative rejected. The diff already shows how.

**Footer**
- `Refs: #123` / `Closes: #123` — follow the tracker's syntax.
- Breaking change: `!` before the colon (`feat(api)!: ...`) and/or a
  `BREAKING CHANGE: <what breaks and how to migrate>` footer.
- Trailers (`Co-Authored-By: ...`) go last.

```
fix(api): return 404 for orders of other customers

Returning 403 revealed that the order exists. The project convention
(see #87) is to hide resources the caller may not access.

Refs: #91
```

---

## 3. Workflow

### Step 1 — Inspect

```bash
git status --short
git diff                 # unstaged
git diff --staged        # already staged
git log --oneline -10    # conventions and context
```

### Step 2 — Plan, explain, wait

Group the changes **by intent**, then present the plan before committing:

```
Plan — 3 commits
1. refactor(mail): extract template renderer
   mail/renderer.py (new), mail/sender.py (hunks 1-2)
   → separate: pure move, no behaviour change; reviewable as "nothing changed"
2. feat(auth): send password reset email
   auth/reset.py, mail/templates/reset.html, tests/test_reset.py
   → the feature, with its test
3. style: apply ruff format to auth/
   auth/models.py
   → formatting noise kept out of the feature diff
```

- For **each group, say why it is separate** — the plan is also a lesson.
- **Wait for validation** before the first commit.
- A file holding changes for two groups is split by hunk (step 3).

### Step 3 — Stage one group

Whole files:

```bash
git add auth/reset.py auth/templates/reset.html tests/test_reset.py
```

Part of a file — `git add -p` is interactive; stage a hunk with a patch instead:

```bash
git diff -U0 mail/sender.py > /tmp/sender.patch
# keep only the hunks for this commit in the patch (edit the file)
git apply --cached --unidiff-zero /tmp/sender.patch
```

Or leave the hunk split to the user with `git add -p` — a good exercise in
reading one's own diff.

Never `git add .` / `git add -A` without having read `git status` first.

### Step 4 — Review what will be committed

```bash
git diff --staged --stat
git diff --staged
```

Check: only this group's changes; no secrets (`.env`, keys, tokens); no
generated files (build output, caches, coverage); no debug leftovers
(`print`, `reveal_type`, `breakpoint()`).

### Step 5 — Test the index, not the working tree

The working tree still holds the next groups; test what the commit contains:

```bash
git stash push --keep-index --include-untracked -m "next groups"
uv run pytest && uv run mypy && uv run ruff check
git stash pop
```

`--keep-index` keeps the staged changes in place; after `pop`, staged and
unstaged parts are restored as they were.

### Step 6 — Commit, then repeat

```bash
git commit -F - <<'EOF'
feat(auth): send password reset email

Users locked out had to contact support. The email carries a
single-use token valid for 30 minutes.

Refs: #42
EOF
```

**Pre-commit hooks**
- Never `--no-verify`.
- A hook that fails means **no commit was created**: fix, `git add` again,
  commit again — do not `--amend` (that would rewrite the *previous* commit).
- A hook that modified files (formatter): review, re-stage, commit again.

---

## 4. Fixing history before sharing

On a branch **not yet pushed or shared**, fix earlier commits instead of adding
"fix typo" commits:

```bash
# Change content of an earlier commit
git add auth/reset.py
git commit --fixup <sha>

# Change only the message of an earlier commit
GIT_EDITOR="printf '%s\n' 'amend! <original subject>' '' '<new message>' >" \
  git commit --fixup=reword:<sha>

# Fold fixups into their targets — non-interactive since Git 2.44
git rebase --autosquash <base>
```

- `<base>` is the commit the branch started from (`main`, or `git merge-base main HEAD`).
- **Never rewrite a shared branch.** Once pushed, fix with a new commit.
- If a rewritten branch must be pushed (own PR branch, with explicit
  agreement): `git push --force-with-lease`, never `--force`.

---

## 5. Anti-patterns

| ❌ | ✅ |
|---|---|
| `wip`, `fix`, `update`, `changes` as message | Type, scope, imperative description, why in the body |
| `git add .` without reading the status | Stage one group, review `git diff --staged` |
| Catch-all commit: feature + refactor + formatting | One commit per intent |
| Commit that breaks tests, fixed by the next | Test the index before each commit |
| "fix typo" commits on an unshared branch | `--fixup` + `rebase --autosquash` |
| Horizontal slices: all models, then all tests | Vertical slices, tests with code |
| Secrets, `.env`, build output committed | Review staged files; `.gitignore` |
| `--no-verify` to get past a hook | Fix what the hook reports |
| `--amend` after a failed hook | New commit — the failed one never existed |
| Force-pushing a shared branch | New commit on top |

---

## 6. Rules for the agent

- Follow the repository's convention when one exists.
- **Present the commit plan and wait for validation** before committing.
- **Explain why each group is separate** — the goal is also to learn.
- Run the tests on the staged content before each commit.
- **Never push** unless the user asked for it — a standing instruction for a
  given repository counts.
- Never rewrite pushed history, never `--no-verify`, never `--force`.
