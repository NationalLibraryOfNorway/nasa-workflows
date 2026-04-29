# Agent Skills Reference

Rules for authoring agent skills (Claude Code, Copilot, etc.).

## Location

- Project-scoped: `.github/skills/<skill-name>/SKILL.md`
- User-scoped (Claude Code): `~/.claude/skills/<skill-name>/SKILL.md`

`<skill-name>` is **lowercase, hyphen-separated**, and matches the folder name exactly.

## Language

- Skill content (`SKILL.md`) is always **English**.
- The `description` field in frontmatter is always **English**.
- This applies even in repos where commits / user-facing text are in another language.

## Frontmatter

```yaml
---
name: skill-name-matches-folder
description: Keyword-rich one-liner. State WHAT the skill does and WHEN it triggers.
---
```

The `description` is what the agent matches against to decide whether to invoke the skill — make it specific and trigger-oriented. Bad: *"Helps with code"*. Good: *"Use when reviewing pull requests to check for missing tests, unsafe SQL, or hardcoded secrets."*

## Body

- Lead with the trigger / when-to-use.
- Then the procedure or rules.
- Keep it short. If it grows large, split into sub-skills or reference docs in the same folder.
- Examples beat prose. Show input → output.

## Don'ts

- Don't duplicate global rules already in `AGENTS.md` — link instead.
- Don't make the skill describe itself ("This skill is..."). Say what to do.
- Don't pad with filler. Every line should change behavior.
