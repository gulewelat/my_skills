---
name: sdd-branching
description: >
  Branching strategy for Spec-Driven Development: mapping a spec, its plan, tasks
  and user stories to short-lived stacked branches and PRs, branch naming,
  commit trailers, PR titles, and keeping the stack up to date after merges.
  Use when planning the branches of a spec, creating or rebasing a stack, or
  merging stacked PRs.
---

# Branching for Spec-Driven Development

Conventions: Git ≥ 2.38 (`--update-refs`), GitHub with `gh`, specs laid out as
`specs/<NNN>-<name>/{spec,plan,tasks}.md` (Spec Kit style), tasks numbered
`T001`, user stories `US1`.
Companion skills: **atomic-commits** (inside a branch), **pull-requests** (each PR).

---

## 1. Principles

- `main` is always deployable and protected: PR, review, green CI.
- **Short-lived branches**: aim for under a day, never more than a few days.
- **One branch = one goal = one PR.**
- Start from an up-to-date `main`; resync often.
- Delete the branch after merge.
- Unfinished behaviour that must merge early hides behind a feature flag.

## 2. Mapping SDD to Git

| SDD element | Git |
|---|---|
| Spec (`001-password-reset`) | Number shared by all its branches |
| Spec, plan, tasks | First branch and PR (`docs`) |
| Foundations (setup, prerequisites) | One branch (`chore`) |
| Each user story | One branch and one PR (`feat`) |
| Task (`T004`) | One or more atomic commits |

A story too big to review in one sitting is split into several branches.

---

## 3. Naming

```
<type>/<spec>-<order>-<description>

docs/001-00-password-reset-spec
chore/001-01-password-reset-setup
feat/001-02-request-reset
feat/001-03-confirm-reset
test/001-04-reset-e2e
```

| Type | For |
|---|---|
| `docs` | Spec, plan, tasks, documentation |
| `chore` | Setup, configuration, dependencies |
| `feat` | A user story, new behaviour |
| `fix` | Bug fix |
| `refactor` | Restructuring without behaviour change |
| `test` | Cross-cutting tests — a story's own tests stay in its branch |

- `<spec>`: three digits, the spec number. `<order>`: two digits, position in
  the stack. `<description>`: kebab-case, a few words.
- All branches of a spec: `git branch --list "*/001-*"`.
- **Spec Kit** may create a branch named `001-password-reset` when the spec is
  created: rename it (`git branch -m docs/001-00-password-reset-spec`). Its
  scripts locate the spec through `.specify/feature.json` or
  `SPECIFY_FEATURE_DIRECTORY`, not the branch name.

---

## 4. Stacking

```
main
 └─ docs/001-00-password-reset-spec
     └─ chore/001-01-password-reset-setup
         └─ feat/001-02-request-reset
             └─ feat/001-03-confirm-reset
                 └─ test/001-04-reset-e2e
```

- Each branch starts from the previous one; each PR targets the previous
  branch; the first targets `main`.
- **Merge bottom-up, in order.** When the bottom PR merges, retarget the next
  one to `main` — GitHub does it automatically if the merged branch is deleted;
  otherwise `gh pr edit <number> --base main`.
- **Independent stories are not stacked on each other**: they all start from
  the setup branch.

```
 └─ chore/001-01-password-reset-setup
     ├─ feat/001-02-request-reset
     └─ feat/001-03-admin-reset-audit      # does not need 001-02
```

---

## 5. Commits

Atomic commits (atomic-commits skill), with trailers linking to the spec:

```
feat(auth): store reset token hashed

Spec: 001-password-reset
Task: T005
```

## 6. PR title and description

```
[001][3/5] feat(auth): request password reset
```

```markdown
## Spec
specs/001-password-reset/spec.md (US1)

## Tasks covered
T004 to T007

## Stack
spec → setup → **request-reset** → confirm-reset → e2e
```

The rest of the description follows the pull-requests skill (context, changes,
how to test, impact).

**With squash merge, the PR title becomes the commit on `main`**: drop the
`[001][3/5]` prefix from the title (it breaks the Conventional Commits format)
and keep the stack position in the "Stack" section only.

---

## 7. Workflow

```bash
# First branch of the spec
git switch main && git pull --ff-only
git switch -c docs/001-00-password-reset-spec

# Next branch, from the current one
git switch -c chore/001-01-password-reset-setup

# Publish (only when the user asked to push)
git push -u origin HEAD
```

Before creating the next branch: the current one's tests pass and its PR is open.

## 8. Updating the stack

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
git rebase --update-refs --onto origin/main docs/001-00-password-reset-spec
```

`docs/001-00-…` here is the **local** branch, still at its pre-merge commit:
`--onto` drops exactly its commits. Don't delete or update it before this
rebase. After a squash merge, the plain `rebase origin/main` replays the
merged commits on top of their squashed copy and conflicts — `--onto` avoids it.

Then publish every branch that moved:

```bash
git push --force-with-lease origin chore/001-01-password-reset-setup feat/001-02-request-reset
```

Then delete the merged branch locally: `git branch -d docs/001-00-password-reset-spec`
(`-D` after a squash merge, which Git doesn't recognise as merged).

## 9. Merge method

- **Rebase and merge** preferred: the atomic commits land on `main` as written.
- Squash merge is acceptable — then PR titles must be clean commit messages (§6).
- **One method per repository**, applied to every PR.

---

## 10. Anti-patterns

| ❌ | ✅ |
|---|---|
| One branch for the whole spec | One branch per goal: spec, setup, each story |
| Branches living for weeks | Under a day; split the story |
| Commits directly on `main` | Every change through a PR |
| Implementing another branch's tasks ahead of time | Only the current branch's tasks |
| Stacking independent stories on each other | Siblings on the setup branch |
| `rebase origin/main` after a squash merge | `rebase --update-refs --onto origin/main <merged-branch>` |
| Rewriting a branch someone else works on | New commits on top, or agree first |
| `--force` | `--force-with-lease` |

## 11. Rules for the agent

- Read `spec.md`, `plan.md` and `tasks.md` before proposing the split.
- **Present the branch list with the tasks of each, then wait for validation.**
- Never commit on `main`.
- Implement only the tasks of the current branch.
- Check that tests pass before creating the next branch.
- If the spec changes: edit it in the `docs` branch (or a new PR if already
  merged), then propagate up the stack.
- **Ask for confirmation before any rebase, force push, merge or branch deletion.**
