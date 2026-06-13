# worktree-development

OpenCode skill that forces every dev task (feature, bugfix, refactor) into an isolated git worktree on a dedicated branch. The agent works autonomously: creates the worktree, writes code, runs tests/lint/build, pushes, opens a PR via `gh`, merges it, and cleans up — all without asking the user for confirmation.

## Install

Clone this repo directly into your OpenCode skills directory:

```bash
git clone https://github.com/Anhydrite/worktree-development-skills.git ~/.config/opencode/skills/worktree-development
```

Restart OpenCode. The skill triggers automatically whenever you ask the agent to implement something.

## What it enforces

- **One task = one worktree = one branch.** Never work on `main`.
- **Questions only before Phase 1.** Once work starts, the agent decides alone.
- **Three gates before merge:** tests, lint, build — all must pass.
- **Auto-merge via `gh`:** PR created, merged, branch deleted, worktree removed. No user confirmation.
- **Fallback:** if `gh` is not installed, falls back to local merge.

## Workflow phases

1. **Verify & Setup** — detects git state, creates worktree if on `main`
2. **Develop** — writes code, runs tests (red-test-first), commits incrementally
3. **PR & Merge** — pushes branch, opens PR, merges, cleans up
