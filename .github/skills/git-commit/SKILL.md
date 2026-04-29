---
name: git-commit
description: 'Execute git commit with conventional commit message analysis, intelligent staging, and message generation. Use when user asks to commit changes, create a git commit, or mentions "/commit". Supports: (1) Auto-detecting type and scope from changes, (2) Generating conventional commit messages from diff, (3) Interactive commit with optional type/scope/description overrides, (4) Intelligent file staging for logical grouping'
license: MIT
allowed-tools: Bash
---

# Git Commit with Conventional Commits

## Overview

Create standardized, semantic git commits using the Conventional Commits specification. Analyze the actual diff to determine appropriate type, scope, and message.

> **Authority**: `ref/commit.md` defines the canonical commit rules. This skill implements them. On conflict, `ref/commit.md` wins.

## Issue Reference (Required)

Every commit MUST include an issue reference as a prefix. Ask the user if not provided.

Supported formats — pick what the project uses (check recent `git log` if unsure):

| Source | Format | Example |
| --- | --- | --- |
| Jira | `<PROJECT>-<NUMBER>` | `SU-1234`, `ABC-42` |
| GitHub issue | `#<NUMBER>` or `gh-<NUMBER>` | `#128`, `gh-128` |
| GitLab issue | `#<NUMBER>` | `#57` |
| Other tracker | tracker-specific | `LIN-901` (Linear) |

If the work has no tracked issue, ask the user before committing without a reference. A `chore`/`docs` commit on internal tooling may legitimately have none — don't invent one.

## Conventional Commit Format

```
<ref> <type>[optional scope]: <subject in Norwegian>

[optional body — why, not what]
```

## Commit Types

| Type       | Purpose                        |
| ---------- | ------------------------------ |
| `feat`     | New feature                    |
| `fix`      | Bug fix                        |
| `docs`     | Documentation only             |
| `style`    | Formatting/style (no logic)    |
| `refactor` | Code refactor (no feature/fix) |
| `perf`     | Performance improvement        |
| `test`     | Add/update tests               |
| `build`    | Build system/dependencies      |
| `ci`       | CI/config changes              |
| `chore`    | Maintenance/misc               |
| `revert`   | Revert commit                  |

## Breaking Changes

```
# Exclamation mark after type/scope
feat!: remove deprecated endpoint

# BREAKING CHANGE footer
feat: allow config to extend other configs

BREAKING CHANGE: `extends` key behavior changed
```

## Workflow

### 1. Analyze Diff

```bash
# If files are staged, use staged diff
git --no-pager diff --staged

# If nothing staged, use working tree diff
git --no-pager diff

# Also check status
git status --porcelain
```

### 2. Stage Files (if needed)

If nothing is staged or you want to group changes differently:

```bash
# Stage specific files
git add path/to/file1 path/to/file2

# Stage by pattern
git add *.test.*
git add src/components/*

# Interactive staging
git add -p
```

**Never commit secrets** (.env, credentials.json, private keys).

### 3. Generate Commit Message

Analyze the diff to determine:

- **Type**: What kind of change is this?
- **Scope**: What area/module is affected?
- **Description**: One-line summary of what changed (present tense, imperative mood, <72 chars, **Norwegian**)

### 4. Execute Commit

```bash
# Single line
git commit -m "<ref> <type>[scope]: <subject in Norwegian>"

# Multi-line with body
git commit -m "$(cat <<'EOF'
<ref> <type>[scope]: <subject in Norwegian>

<optional body — why, not what>
EOF
)"
```

## Best Practices

- One logical change per commit
- Present tense: "legg til" not "la til"
- Imperative mood: "fiks feil" not "fikser feil"
- Reference issues as prefix: `SU-1234 fix(auth): ...`, `#128 fix(api): ...`
- Keep description under 72 characters

## Git Safety Protocol

- NEVER update git config
- NEVER run destructive commands (--force, hard reset) without explicit request
- NEVER skip hooks (--no-verify) unless user asks
- NEVER force push to main/master
- If commit fails due to hooks, fix and create NEW commit (don't amend)

