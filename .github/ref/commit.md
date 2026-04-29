# Commit Guidelines

Commit messages are written in **Norwegian**. Everything else (code, comments, docs) stays English.

## Rules

- One logical change per commit.
- Short, clear subject line (≤ 72 chars).
- **Conventional Commits** format.
- **Issue reference required** — see formats below. Ask the user if not provided.
- Body (optional) explains *why*, not *what*. Wrap at ~72 chars.

## Issue references

Any of these are valid. Pick what the project uses — check recent `git log` if unsure.

| Source | Format | Example |
| --- | --- | --- |
| Jira | `<PROJECT>-<NUMBER>` | `SU-1234`, `ABC-42` |
| GitHub issue | `#<NUMBER>` or `gh-<NUMBER>` | `#128`, `gh-128` |
| GitLab issue | `#<NUMBER>` | `#57` |
| Other tracker | tracker-specific | `LIN-901` (Linear), `T-1234` (Phabricator) |

If the work has no tracked issue, ask the user before committing without a reference. A `chore`/`docs` commit on internal tooling may legitimately have none — don't invent one.

## Format

```text
<ref> <type>(<scope>): <subject in Norwegian>

<optional body — why, not what>
```

Common types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `build`, `ci`.

## Examples

```text
SU-1234 feat(modul): implementer ny funksjonalitet
ABC-42 fix(auth): håndter utløpt token
#128 fix(api): returner 404 når ressurs mangler
gh-57 docs(readme): oppdater installasjonsinstruksjoner
LIN-901 refactor(db): trekk ut tilkoblingspool
SU-1234 test(modul): legg til test for edge case
SU-1234 chore(deps): oppdater avhengigheter
```

## Don'ts

- Don't skip hooks (`--no-verify`) unless explicitly told to. Pre-commit hooks (gitleaks, formatters) catch real problems.
- Don't amend pushed commits without coordination.
- Don't commit without a reference unless the user has confirmed none exists.
- Don't bundle unrelated changes — split into separate commits.
- Don't mix reference styles in one repo. Match the project convention.

## Pull Requests

- PR title can mirror the lead commit subject. Body explains scope and test plan.
- Auto-review: workflows like `copilot-review.yml` (if present in the project) ask `@copilot` to review.
- Manual review: comment `@copilot do a thorough review of this PR`.
