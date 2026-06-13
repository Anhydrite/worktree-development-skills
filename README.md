# worktree-development

An [OpenCode](https://github.com/sst/opencode) skill that enforces an autonomous git worktree workflow for AI coding agents.

Every development task happens in an isolated git worktree on a dedicated branch. The agent creates the worktree, develops, runs tests/lint/build, creates a PR, and merges — all without asking the user for confirmation.

## Installation

### Option 1 — Copy the skill file

```bash
# From this repo's root:
cp SKILL.md ~/.config/opencode/skills/worktree-development/SKILL.md
```

### Option 2 — Clone and symlink

```bash
git clone https://github.com/Anhydrite/worktree-development-skills.git ~/.local/share/opencode/worktree-development-skills

mkdir -p ~/.config/opencode/skills/worktree-development
ln -s ~/.local/share/opencode/worktree-development-skills/SKILL.md \
      ~/.config/opencode/skills/worktree-development/SKILL.md
```

### Option 3 — One-liner

```bash
mkdir -p ~/.config/opencode/skills/worktree-development && \
curl -sL https://raw.githubusercontent.com/Anhydrite/worktree-development-skills/main/SKILL.md \
  -o ~/.config/opencode/skills/worktree-development/SKILL.md
```

## How It Works

When you ask OpenCode to implement a feature, fix a bug, or start any development task, this skill activates automatically and enforces:

1. **Phase 1 — Verify & Setup**: Detects current git state. Creates a worktree on a new branch if you're on `main`.
2. **Phase 2 — Develop**: Writes code, runs tests, lint, and build in the worktree. Uses red-test-first.
3. **Phase 3 — PR & Merge**: Pushes the branch, creates a PR via `gh`, merges it, cleans up the worktree and branch.

No user confirmation is asked at any point after Phase 1 begins.

## Configuration

The skill lives at:

```
~/.config/opencode/skills/worktree-development/SKILL.md
```

No additional configuration is needed. OpenCode discovers it automatically on next session start.

## License

MIT
