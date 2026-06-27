---
name: worktree-development
description: Use when starting any new feature, bug fix, refactor, or development task in a git repository. Triggers on requests to create a branch, implement a change, start coding, or begin work. Do NOT use for read-only exploration, documentation-only changes, or trivial single-line commits.
---

# Worktree Development

## Overview

Every development task must happen in an isolated git worktree on a dedicated branch. Never work directly on `main` or in someone else's worktree. Each task = its own worktree = its own branch.

## Core Rules

**One task = one worktree = one branch. No sharing. No work on `main`.**

**Ask everything you need at the START. Then go autonomous.**

The agent may ask clarifying questions BEFORE entering Phase 1, when the
request is ambiguous (e.g. ambiguous scope, unknown library choice, several
valid implementation strategies). Once Phase 1 begins, the agent MUST be
fully autonomous: no questions, no "should I do X?" prompts, no "is this
OK?" pauses. Decisions on the fly are the agent's responsibility. The merge
at the end of Phase 3 is also automatic.

Rationale: a fragmented turn ("ask → answer → do → ask → answer → do") is
slower and noisier than a single autonomous run. Asking up front is fine;
asking mid-run is a workflow violation.

## When to Use

Use this skill when:
- User asks to implement a feature, fix a bug, or refactor code
- User asks to create a new branch
- User wants to start any development work

Do NOT use when:
- Reading or exploring code without making changes
- Documentation-only changes (no code affected)
- Trivial single-line commits (typo fixes, version bumps)
- Operations that don't modify code (e.g., git status, git log)

## Workflow

## Question Policy

**BEFORE Phase 1 (the only window for questions):**
- Ambiguous scope ("is this a refactor or a feature?")
- Genuine multi-valid choice that changes the work significantly
- Missing information the user can supply cheaply (paths, names, target version)
- Library or tool choice the user should make (e.g. "use Bun or keep Node?")

**DURING Phase 1 / Phase 2 / Phase 3 (no questions allowed):**
- Implementation details ("should I use a map or a slice here?")
- Code style / naming / organization choices
- Test layout or assertion style
- Merge timing (always auto-merge when gates are green)
- Anything the agent can decide faster than asking

**If a real blocker appears mid-run** (e.g. a test fails for an environmental
reason, a dependency is missing, a conflict cannot be auto-resolved):
- Try the obvious fixes first (re-run, re-install, rebase).
- If still blocked, document the blocker in the worktree commit message and
  the project's known-issues log, and proceed with the partial work
  (commit what works, leave a clear follow-up).
- Only ask the user if the work is fundamentally impossible to ship without
  their input (e.g. "which API key should I use?" with no default available).

### Phase 1 — Verify & Setup

Check current state:

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
BRANCH=$(git branch --show-current)
```

**Already in a worktree** (`GIT_DIR != GIT_COMMON` and not in a submodule)?
→ Continue working in the existing worktree. Skip to Phase 2.

**On `main` and not in a worktree**?
→ STOP. Create a worktree first:

```bash
# 1. Ensure .worktrees is gitignored
git check-ignore -q .worktrees || echo ".worktrees/" >> .gitignore

# 2. Create worktree on a new branch
#    Branch name: <type>/<short-description> (e.g. feat/checkbox-fix, fix/click-zone)
BRANCH="<type>/<short-description>"
WORKTREE_DIR=".worktrees/$(echo "$BRANCH" | cut -d/ -f2)"
git worktree add "$WORKTREE_DIR" -b "$BRANCH"

# 3. Enter the worktree
cd "$WORKTREE_DIR"

# 4. Install dependencies (auto-detect)
[ -f package.json ] && npm install
[ -f go.mod ] && go mod download
[ -f Cargo.toml ] && cargo build
[ -f requirements.txt ] && pip install -r requirements.txt
[ -f pyproject.toml ] && poetry install
```

**Detached HEAD or externally managed worktree**?
→ Note the branch name, create your own worktree from it, then proceed.

### Phase 2 — Develop

All work happens in the worktree:
- Write code, run tests, run lint, build
- Use red-test-first: write a failing test, then implement
- Commit incrementally on the feature branch
- **Never** push to `main` directly

**Required gates before merge** (run all, all must pass):
1. Tests: project test suite (e.g. `bash tests/run-guard.sh` if it exists)
2. Lint: project's linter (e.g. `npm run lint`, `golangci-lint run`)
3. Build: project's build command (e.g. `npm run build`, `go build`)

### Phase 3 — PR & Merge to `main` (AUTOMATIC — do not ask)

**The agent is fully autonomous from worktree creation through merge.** Once
all Phase 2 gates are green, the agent MUST create a PR and merge it without
asking the user for confirmation. The user does not want a review pause — the
work is shipped as soon as the gates pass.

Before starting Phase 3, check that `gh` is available and authenticated:
```bash
gh auth status 2>/dev/null || echo "FALLBACK"
```
If the check fails, use the local merge fallback.

The PR+merge is the agent's responsibility, not the user's. Skipping it is a
workflow violation (see Red Flags below). This is also the canonical example
of the "no questions mid-run" rule from the Question Policy above.

The flow uses `gh` CLI to create a Pull Request and merge it, so GitHub
history is preserved. If `gh` is not installed or not authenticated, fall
back to the local merge (see fallback section below).

From the worktree (all `gh` commands are API calls, not local git ops):

```bash
# 1. Push the feature branch to origin
git push origin <branch-name>

# 2. Create a Pull Request (title/body from commits via --fill)
gh pr create --base main --head <branch-name> --fill

# 3. Merge the PR (--merge creates a merge commit, like --no-ff)
#    --delete-branch removes the remote branch after merge
gh pr merge --merge --delete-branch

# 4. Switch to main and pull the merged commits
git checkout main
git pull origin main

# 5. Delete the local feature branch
git branch -d <branch-name>

# 6. Remove the worktree
git worktree remove .worktrees/<short-name>
```

**Fallback — local merge (when `gh` is unavailable):**

If `gh` is not installed, not authenticated, or the repo has no remote
`origin`, fall back to the traditional local merge:

```bash
# From the main repo checkout:
git checkout main
git merge --no-ff <branch-name>
git branch -d <branch-name>
git worktree remove .worktrees/<short-name>
git push origin main
```

**Pre-merge verification (mandatory, no exceptions):**

Before running the merge commands, the agent MUST re-verify the three Phase 2
gates one last time. If any gate fails, the merge MUST NOT happen — fix in
the worktree and re-verify. Do not merge red code.

```bash
# Re-verify gates in the worktree
cd .worktrees/<short-name>
<test-cmd>          # e.g. bash tests/run-guard.sh
<lint-cmd>          # e.g. bun run lint
<build-cmd>         # e.g. go build ./...
```

**When merge is blocked or risky:**

- **Gates fail on the worktree** → fix in the worktree, re-verify, then merge.
- **Conflict on merge** → resolve in the worktree (cherry-pick / rebase),
  re-verify gates, then merge. **CRITICAL: Never delete code that exists in
  `main` but not in your worktree.** Before resolving any conflict, read the
  git log to understand WHY that code was added. If the added code is not in
  your scope, KEEP IT. Your conflict resolution must preserve all recent
  `main` additions — you only resolve conflicts in YOUR code. If unsure, keep
  both and let the gates catch issues.
- **Merge would touch a guarded file (e.g. `.github/workflows`, lockfiles
  shared across features)** → still merge automatically; the next feature
  will deal with any fallout via its own gates.
- **User explicitly said "don't merge" in the current turn** → respect that
  one-off override, but the default remains automatic for all other turns.

**Never do these:**

- ❌ Ask "should I merge now?" — the answer is yes, execute it.
- ❌ Stop after committing in the worktree without merging.
- ❌ Leave the worktree and branch in place after a successful merge.
- ❌ Delete code from `main` during conflict resolution — even if it seems
  unrelated. You are not the owner of `main`'s history.
- ❌ Merge when a gate is red, even by one test.
- ❌ Push to `main` directly (bypassing the merge commit).
- ❌ Ask the user any question once Phase 1 has started.

## Quick Reference

| Situation | Action |
|---|---|
| On `main`, no worktree | Create worktree first, then work |
| Already in a worktree | Continue working there |
| Work complete, gates green | **Push branch → create PR → merge PR via `gh`**, delete branch, remove worktree — no user confirmation needed |
| Gates fail (test/lint/build) | Fix in the worktree, do not merge |
| Conflict on merge | Resolve in the worktree, then merge |
| User said "don't merge" in this turn | One-off override — skip merge this turn only; default is still auto-merge next time |

## Common Mistakes

- **Working directly on `main`** — always create a worktree first
- **Reusing someone else's worktree** — each task gets its own
- **Merging before tests pass** — red tests = no merge
- **Forgetting `git worktree remove`** — leaves stale worktrees
- **Forgetting to delete the branch** — leaves stale branches (local and remote)
- **Forgetting to push the branch before creating the PR** — `gh pr create` needs the branch on remote
- **Forgetting `.worktrees/` in `.gitignore`** — risk of committing worktree contents
- **Asking the user before merging** — the merge is automatic; the user does not want a confirmation step
- **Stopping after the worktree commit** — leaving the branch unmerged is a workflow violation; the merge is part of "done"
- **Asking questions mid-run** — once Phase 1 starts, the agent decides alone; questions are only allowed before Phase 1

## Red Flags — STOP and Recreate

- You started editing files in the main checkout
- You're inside a worktree that wasn't created for your task
- You ran `git checkout -b` instead of `git worktree add`
- You merged a branch without running the full test/lint/build gate
- `.worktrees/` is not in `.gitignore`
- You finished the worktree work and **did not create a PR / merge** — this is the most common violation under the new autonomous workflow
- You asked the user "should I merge now?" — autonomous workflow means execute, do not ask
- You pushed the branch to `origin` but did not create a PR — the PR is how GitHub history is preserved
- You asked the user ANY question after Phase 1 started (e.g. "is this implementation OK?", "should I refactor X?") — the agent is supposed to decide and proceed

**All of these mean: stop, undo, restart with this skill.**
