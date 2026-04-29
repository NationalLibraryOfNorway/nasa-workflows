# Agent Instructions

Global rules for AI coding agents. Goal: consistent, minimal, verifiable changes.

> Bias: caution over speed. For trivial tasks (see *Scope* below), use judgment.
>
> Conflict precedence: `AGENTS.md` > `.github/ref/*.md`.

---

## TL;DR

- **Plan before coding** — for non-trivial work: answer the four intake questions (§ 1), present plan, wait for consent (explicit "go" / "kjør", or affirmative reply on the plan itself).
- **Minimal change** — no speculative code, no abstractions for single use, no error handling for impossible cases.
- **Surgical edits** — change only what is needed. Don't "improve" neighbors. Match existing style.
- **Verify** — run tests after code changes (see language ref).
- **Ask when uncertain** — surface confusion early.
- **Hand off with a manual test plan** — detail scales with triviality. See § *Hand-off & Verification*.
- **Commit only on explicit request** — messages in Norwegian per [`.github/ref/commit.md`](.github/ref/commit.md). Issue reference required.
- **Chat language** — match the user's language in their most recent message.

---

## Scope: trivial vs. non-trivial

Triviality gates the plan requirement and hand-off detail.

**Trivial** (skip plan step, light hand-off):

- ≤ 10 lines changed, AND
- No behavior change in production code paths, AND
- No new dependency, no new exported/public API surface.
- Examples: typo fix, comment edit, formatting, local rename (not exported), single-value config tweak, doc clarification.

**Non-trivial** (plan + wait for consent + full hand-off):

- Behavior change, new endpoint/feature, refactor across files, new dependency, schema/migration, build/CI change, security-sensitive code, anything touching auth or data handling.

If unsure which side a task falls on, treat as non-trivial.

---

## 1. Think Before Coding

Don't assume. Don't hide confusion. Surface tradeoffs.

### Task intake — answer four questions before touching code

1. **What problem does this solve?** State it in one sentence. If you can't, ask.
2. **Do we need this?** Is there a simpler fix, an existing feature, or a way to skip the change entirely?
3. **What does "done" look like?** Concrete acceptance criteria — the state to reach, not the method.
4. **How do we verify it?** The specific check: command to run, assertion, manual step, metric to read. If you can't define one, the task isn't ready.

Present the four answers and a brief plan (numbered steps). **Wait for approval before implementing.** This is the cheapest moment to refine scope — much cheaper than rewriting code.

For trivial tasks (typo, one-line fix), name it briefly and proceed. Use judgment.

### General principles

- State assumptions explicitly. If uncertain, ask.
- Multiple interpretations exist? Present them — don't pick silently.
- Simpler approach exists? Say so. Push back when warranted.
- Unclear? Stop. Name what's confusing. Ask.

## 2. Minimal & Surgical Changes

Minimum code that solves the problem. Touch only what you must.

**Scope of code:**

- No features beyond what was asked.
- No abstractions for single-use code.
- No flexibility/configurability that wasn't requested.
- No error handling for impossible scenarios.
- 200 lines that could be 50? Rewrite.

**Scope of diff:**

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- Notice unrelated dead code? Mention it — don't delete it.
- Your changes orphan an import/var/function? Remove it.

Tests:

- *"Would a senior engineer call this overcomplicated?"* If yes, simplify.
- Every changed line traces directly to the user's request.

## 3. Goal-Driven Execution

Define success criteria. Loop until verified.

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

Multi-step tasks: state plan as `step → verify` pairs.

## 4. Hand-off & Verification

Every completed task ends with a manual test plan. Detail scales with triviality.

**Trivial change:** one line stating what to re-read / re-render / re-run and what to look for. No formal sections.

**Non-trivial change** must include:

- **Concrete steps**: commands to run, URLs to open, buttons to click, inputs to try. Copy-pasteable.
- **Happy path**: the main scenario the change enables.
- **The motivating case**: for a bug fix, the exact reproduction; for a feature, the example from the request.
- **Likely regressions**: the one or two adjacent things most likely to break — what to glance at to confirm they didn't.
- **Expected vs. observed**: what the user should *see* (status code, log line, UI state, file content). "It should work" is not a verification step.
- **Already verified vs. needs human eyes**: distinguish what tests/CI cover from what only a human can confirm (visual, UX, real data).

Format: short numbered steps or a checklist. Not prose. Group by area if many steps.

If a change genuinely cannot be verified before merge/deploy (CI workflow changes, scheduled jobs, infra), say so explicitly and describe what to watch *after* deploy.

For pure docs / config changes with no behavior: still say what the user should re-read or re-render (Markdown preview, `mkdocs serve`, regenerated config) and what to look for.

---

## Language References

Pick the file(s) matching the stack you're working in. Combine when relevant (e.g. Kotlin + Spring WebFlux + Backend API).

Languages:

- [`.github/ref/java.md`](.github/ref/java.md) — Java, JUnit, Maven, SLF4J.
- [`.github/ref/kotlin.md`](.github/ref/kotlin.md) — Kotlin idioms, null-safety, coroutines, Maven.
- [`.github/ref/python.md`](.github/ref/python.md) — Python, pytest, uv (always), ruff, type hints.
- [`.github/ref/typescript.md`](.github/ref/typescript.md) — TypeScript/JS on Bun (runtime, install, test), eslint, strict types.

Frameworks & domains:

- [`.github/ref/spring-mvc.md`](.github/ref/spring-mvc.md) — Spring Boot MVC (servlet, blocking).
- [`.github/ref/spring-webflux.md`](.github/ref/spring-webflux.md) — Spring Boot WebFlux (Reactor, non-blocking).
- [`.github/ref/spring-error-handling.md`](.github/ref/spring-error-handling.md) — Spring error responses with `ProblemDetail` per RFC 9457.
- [`.github/ref/backend-api.md`](.github/ref/backend-api.md) — REST/HTTP API design, status codes, errors, versioning.
- [`.github/ref/frontend.md`](.github/ref/frontend.md) — Browser-facing code, a11y, performance, state.
- [`.github/ref/structured-logging.md`](.github/ref/structured-logging.md) — JSON logs, correlation IDs, what (not) to log.
- [`.github/ref/security.md`](.github/ref/security.md) — OWASP Top 10, CVE handling, SBOM (CycloneDX), Dependency Track, `dependency-check-maven`.

Other rules:

- [`.github/ref/commit.md`](.github/ref/commit.md) — Norwegian commit messages, issue references.
- [`.github/ref/skills.md`](.github/ref/skills.md) — Agent skill format and conventions.

---

## Security (universal)

Highlights below. See [`.github/ref/security.md`](.github/ref/security.md) for full guidance — OWASP Top 10, CVE/SBOM workflow, suppression rules, language-specific tooling.

- **Never** hardcode secrets, passwords, API keys, or tokens.
- Secrets via environment variables only.
- Spot a secret in existing code? Flag it immediately — rotate before committing on top of it.
- Don't introduce OWASP-class vulnerabilities (injection, XSS, SSRF, path traversal, BOLA). If you wrote insecure code, fix it before declaring done.
- **Security finding during unrelated work:** stop, surface the finding to the user (severity + file/line), and ask before continuing or scoping a fix. Never silently fix a security issue inside an unrelated commit.
- Dependencies: scan + SBOM on every build. CVSS 8.0+ blocks the build (see [`.github/ref/security.md`](.github/ref/security.md)).

---

## Communication & Response Compression

Goal: concrete answers with minimal noise. Active by default in every response unless the user asks for a detailed explanation. Stay active across long sessions — don't drift back to prose.

**Formatting:**

- Plain text by default. No decorative emoji.
- 🔴 🟡 🟢 only in lists with multiple prioritized items (critical / important / suggestion). Not in prose.
- ✅ 🔄 ⏭️ ❌ in nasa-library command output (status indicators in tables and summaries).

**Compression rules:**

- Start with direct answer in first line.
- Drop fillers (`just`, `really`, `basically`, `actually`, `simply`), pleasantries (`sure`, `of course`, `happy to`), and hedging. Sentence fragments are fine.
- Prefer short words and concrete verbs.
- Preserve verbatim: code blocks, error messages, log lines, identifiers, file paths, commands. Don't paraphrase or truncate them.
- Max 5 bullets unless user asks for depth.
- One bullet = one action/point.
- Ask at most 1 clarifying question, only if blocking.

**Default response pattern:**

1. Result / diagnosis
2. What changed / what to do
3. Verification / next step

**Auto-clarity exceptions** (override compression):

- Security warnings
- Destructive or irreversible operations
- Multi-step instructions where terseness may cause mistakes

Resume compression after the exception part is delivered.

**Out of scope** — write normally, never compressed:

- Commit messages and PR bodies — see [`.github/ref/commit.md`](.github/ref/commit.md).
- Code, code comments, and docstrings.

---

## Practical Workflow

1. **Read** project docs and any relevant `.github/ref/*.md`.
2. **Plan** — for non-trivial work, answer the four intake questions (§ 1), present plan, wait for consent. Skip for trivial tasks.
3. **Implement** smallest change that solves the problem.
4. **Verify** by running tests (see language ref).
5. **Update** docs if behavior or APIs changed.
6. **Hand off** with a manual test plan (§ *Hand-off & Verification*).
7. **Commit** only when explicitly requested, per [`.github/ref/commit.md`](.github/ref/commit.md).

---

**Working if:** fewer unnecessary changes in diffs, fewer rewrites from overcomplication, clarifying questions arrive *before* implementation, not *after* mistakes.
