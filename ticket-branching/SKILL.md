---
name: ticket-branching
description: >
  Branching for ticket-driven work: one short-lived branch and PR per tracer-bullet
  ticket, stacked or sibling branches derived from the tickets' blocking edges,
  branch naming, commit and PR references to tickets, and keeping a stack up to
  date after merges. Use when creating the branch for a ticket, planning the
  branches of a set of tickets, rebasing a stack, or merging stacked PRs.
---

# Ticket branching

Conventions: Git ≥ 2.38 (`--update-refs`), tickets produced by `/to-tickets`
(Matt Pocock's skills) on the tracker chosen by `/setup-matt-pocock-skills`:
GitHub, GitLab, or local files under `.scratch/<feature>/issues/`.

Position in the flow: grilling → `/to-spec` → `/to-tickets` → **one branch per
ticket** → `/implement` → PR. The flow says what to build; this skill says
where it lives in Git.
Companion skills: **atomic-commits** (inside a branch), **pull-requests** (each PR).

---

## 1. Principles

- `main` is always deployable and protected: PR, review, green CI.
- **One ticket = one branch = one PR.** A tracer-bullet ticket is a vertical
  slice sized for one context window — its branch lives hours, not weeks.
- Start from an up-to-date `main`; resync often.
- Delete the branch after merge.
- Unfinished behaviour that must merge early hides behind a feature flag.

## 2. Mapping tickets to Git

| Flow element | Git |
|---|---|
| Spec (`/to-spec`) | Lives on the tracker — no branch |
| Prefactoring ticket | `refactor/` branch, first in the stack |
| Tracer-bullet ticket | One branch, one PR |
| Blocking edges ("Blocked by") | Where the branch starts (§4) |
| Wide refactor (expand–contract) | One branch per batch ticket; shared integration branch only when batches can't stay green alone |
| Prototype (`/prototype`) | `prototype/<name>`, kept unmerged as a primary source |

---

## 3. Naming

```
<type>/<ticket>-<slug>
```

| Tracker | `<ticket>` | Example |
|---|---|---|
| GitHub / GitLab | Issue number | `feat/142-request-reset` |
| Local files | `<feature>-<NN>` from `.scratch/<feature>/issues/<NN>-<slug>.md` | `feat/password-reset-02-request-reset` |

| Type | For |
|---|---|
| `feat` | A ticket delivering new behaviour |
| `fix` | Bug fix |
| `refactor` | Prefactoring, expand–contract batches |
| `chore` | Setup, configuration, dependencies |
| `docs` | Documentation only |
| `test` | Cross-cutting tests — a ticket's own tests stay in its branch |
| `prototype` | Throwaway prototype, never merged |

- `<slug>`: kebab-case, a few words from the ticket title.
- Branches of a local feature: `git branch --list "*/password-reset-*"`.

---

## 4. Where a branch starts — from the blocking edges

| The ticket's blockers | Start from |
|---|---|
| None, or all merged | `main` |
| One blocker, not merged yet | The blocker's branch (stacked) |
| Several blockers in one chain | The last one in the chain (stacked) |
| Several independent blockers, not merged | Wait until all but one are merged — don't merge branches into each other |

Independent tickets are **siblings**, never stacked on each other:

```
main
 └─ refactor/140-extract-token-store          # prefactor, no blocker
     ├─ feat/141-request-reset                # blocked by 140
     │   └─ feat/142-confirm-reset            # blocked by 141
     └─ feat/143-admin-reset-audit            # blocked by 140 only
```

- Each PR targets its parent branch; a branch started from `main` targets `main`.
- **Merge bottom-up, in order.** When a PR merges, retarget its children to
  `main` — GitHub does it automatically if the merged branch is deleted;
  otherwise `gh pr edit <number> --base main` (GitLab: change the MR's target branch).

---

## 5. Commits

Atomic commits (atomic-commits skill), referencing the ticket in the footer:

```
feat(auth): store reset token hashed

Refs: #141
```

Local tracker: `Refs: .scratch/password-reset/issues/02-request-reset.md`.

## 6. PR title and description

Title: a clean Conventional Commits line (pull-requests skill) — no ticket or
stack prefix, so it reads well as the squash commit on `main`.

Description: the pull-requests template, plus the ticket and the stack:

```markdown
Closes #141

## Stack
140 extract-token-store → **141 request-reset** → 142 confirm-reset
```

`Closes #141` closes the issue on merge (GitHub and GitLab). Local tracker:
link the ticket file and set its `Status` when the PR merges.

---

## 7. Workflow

```bash
# Ticket with no unmerged blocker
git switch main && git pull --ff-only
git switch -c feat/141-request-reset

# Ticket blocked by the current, unmerged branch
git switch -c feat/142-confirm-reset

# Publish (only when the user asked to push)
git push -u origin HEAD
```

Create the ticket's branch **before** running `/implement`, which commits to
the current branch. Before creating the next one: the current branch's tests
pass and its PR is open.

## 8. Updating a stack

Run from the **top** branch; `--update-refs` moves every branch of the stack
along (set it once with `git config --global rebase.updateRefs true`).

**`main` moved (hotfix, other merges):**

```bash
git fetch origin
git rebase --update-refs origin/main
```

**The bottom branch was merged** — use `--onto`, whatever the merge method:

```bash
git fetch origin
git rebase --update-refs --onto origin/main refactor/140-extract-token-store
```

The named branch is the **local** one, still at its pre-merge commit: `--onto`
drops exactly its commits. Don't delete or update it before this rebase. After
a squash merge, a plain `rebase origin/main` replays the merged commits on top
of their squashed copy and conflicts — `--onto` avoids it.

**Siblings**: `--update-refs` only moves the branches below the one you rebase.
Rebase the longest line first, then move each sibling onto its parent's new
position — `@{1}` is the parent's previous tip, from the reflog:

```bash
git switch feat/142-confirm-reset && git rebase --update-refs origin/main
git switch feat/143-admin-reset-audit
git rebase --onto refactor/140-extract-token-store 'refactor/140-extract-token-store@{1}'
```

Then publish every branch that moved:

```bash
git push --force-with-lease origin feat/141-request-reset feat/142-confirm-reset
```

Then delete the merged branch locally: `git branch -d <branch>` (`-D` after a
squash merge, which Git doesn't recognise as merged).

A rebase that stops on conflicts: `/resolving-merge-conflicts` finishes it —
once the user has approved the rebase itself.

## 9. Merge method

- **Rebase and merge** preferred: the atomic commits land on `main` as written.
- Squash merge is acceptable — PR titles are already clean commit messages (§6).
- **One method per repository**, applied to every PR.

---

## 10. Anti-patterns

| ❌ | ✅ |
|---|---|
| One branch for several tickets | One branch per ticket |
| Branches living for days | A ticket sized for one session; split it otherwise |
| Commits directly on `main` | Every change through a PR |
| Implementing a later ticket's scope early | Only the current ticket; note the rest on the tracker |
| Stacking independent tickets | Siblings on their common blocker |
| Merging two unmerged branches into a third | Wait for the blockers to land |
| `rebase origin/main` after a squash merge | `rebase --update-refs --onto origin/main <merged-branch>` |
| `--force` | `--force-with-lease` |

## 11. Rules for the agent

- Derive the branch plan from the tickets and their blocking edges; **present
  it (branch, base, ticket) and wait for validation.**
- Create the ticket's branch before implementing; never commit on `main`.
- Implement only the current ticket's scope.
- Check that tests pass before creating the next branch.
- A spec or ticket change is made on the tracker, not in the branch.
- **Ask for confirmation before any rebase, force push, merge or branch deletion.**
