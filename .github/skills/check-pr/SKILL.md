---
name: check-pr
description: 'Review and merge Dependabot pull requests. Lists all open Dependabot PRs, verifies CI status and changes, runs local tests via git worktree, recommends merge, and auto-merges if all checks pass. Handles all PRs in a single run. Triggers on "sjekk pr", "check pr", "dependabot pr", "merge dependabot".'
---

# Check PR — Dependabot PR Reviewer

Automated review and merge of Dependabot pull requests via `gh` CLI.
Handles all open Dependabot PRs in a single run — no need to invoke the skill multiple times.

## When to Use This Skill

- User says "sjekk pr", "check pr", "dependabot pr", "merge dependabot"
- User wants to review and merge pending Dependabot updates
- User wants an overview of dependency update PRs

## Prerequisites

- `gh` CLI is installed and authenticated
- User has merge permissions on the repository
- Repository uses Dependabot for dependency updates
- `git` is available for worktree operations
- Build tool is available for running tests (e.g., `mvn`, `./mvnw`, `npm`, `./gradlew`)

---

## Workflow

Follow these steps in order. Execute each step as actual commands — do not just display them.

### Step 1: List all open Dependabot PRs

```bash
gh pr list --author "dependabot[bot]" --state open --json number,title,url,headRefName,statusCheckRollup,mergeable,labels,createdAt,updatedAt
```

If there are no open PRs, inform the user and stop.

### Step 2: Gather information for each PR

For each PR, run the following:

#### 2a. View PR details

```bash
gh pr view <PR_NUMBER> --json title,body,headRefName,baseRefName,files,additions,deletions,changedFiles,mergeable,mergeStateStatus,statusCheckRollup,reviews,labels
```

#### 2b. View changed files (diff)

```bash
gh pr diff <PR_NUMBER>
```

#### 2c. Check CI status

```bash
gh pr checks <PR_NUMBER>
```

### Step 3: Local testing via git worktree

For PRs that change locally testable files, delegate testing to the `@pr-test-runner` agent.

#### 3a. Determine which PRs are testable

<!-- TODO: Customize this table for your project's file structure and test strategy -->

| Changed files | Testable? | What to test |
|---|---|---|
| `pom.xml`, `src/**` | ✅ Yes | Build + unit tests |
| `package.json`, `src/**` | ✅ Yes | npm test |
| `build.gradle`, `src/**` | ✅ Yes | Gradle tests |
| `.github/workflows/**` only | ❌ No | Not locally testable |
| `Dockerfile` only | ❌ No | Not locally testable |

#### 3b. Run tests for each testable PR

For each testable PR, delegate to the `@pr-test-runner` agent with this task:

```
Test PR #<NUMBER> locally.
- Repository root: <REPO_ROOT_PATH>
- Branch: <HEAD_REF_NAME>
- Changed files: <LIST_OF_FILES>
Run the appropriate tests and report results.
```

The agent will:
1. Create a git worktree at `/tmp/{{PROJECT_NAME}}-pr-<NUMBER>`
2. Run the appropriate tests
3. Report PASS/FAIL/SKIP/ERROR
4. Clean up the worktree

#### 3c. Record results

Track local test results alongside CI status for the summary table:

- **✅ PASS** — local tests passed
- **❌ FAIL** — local tests failed (include brief error)
- **⏭️ SKIP** — not locally testable (e.g., workflow-only changes)
- **⚠️ ERROR** — test execution error (timeout, setup failure)

### Step 4: Analyze and assess each PR

For each PR, evaluate the following:

#### 4a. Detect already-applied versions

Before assessing a PR, check if the version it proposes is **already applied** in the
codebase on the `main` branch. This happens when a dependency was upgraded manually or
via another mechanism but the Dependabot PR is still open.

For each PR, extract the target version from the diff and compare it against the current
version in the codebase:

```bash
# For Maven dependencies — check current pom.xml on main
grep '<dependency.version>' pom.xml

# For npm dependencies — check current package.json on main
cat package.json | grep '"dependency-name"'

# For GitHub Actions — check current workflow files on main
grep 'actions/checkout@' .github/workflows/*.yml
```

If the version proposed by the PR **already matches** what's on `main`:
- Mark the PR as **🔄 Stale — already applied**
- Recommend closing the PR (not merging)
- In the summary, show it under "Closed (stale)" section

Close stale PRs with:

```bash
gh pr close <PR_NUMBER> --comment "Closing: this version is already applied in the codebase on main."
```

#### 4b. Special dependency handling

<!-- TODO: Add any dependencies that require special handling in your project -->
<!-- Examples: -->
<!-- - A core framework version referenced in multiple files (needs a dedicated upgrade skill) -->
<!-- - A dependency with known breaking change patterns -->
<!-- - Docker base images pinned in multiple locations -->

{{SPECIAL_DEPENDENCY_RULES}}

#### 4c. Approval checklist

- [ ] **Author is Dependabot**: PR was created by `dependabot[bot]`
- [ ] **Not stale**: The proposed version is not already applied in the codebase
- [ ] **No special handling needed**: Dependency is not flagged for special treatment (see 4b)
- [ ] **CI checks pass**: All required checks are green
- [ ] **Local tests pass**: Tests run via git worktree pass (or are skipped for non-testable PRs)
- [ ] **No merge conflicts**: PR is mergeable
- [ ] **Changes limited to dependencies**: Only dependency files are modified
- [ ] **Reasonable version bump**: Patch/minor is low risk; major requires caution
- [ ] **No known CVEs in new version**: Check if the update fixes security issues

#### 4d. Risk assessment

| Update type        | Risk       | Recommendation                       |
|--------------------|------------|--------------------------------------|
| Patch (x.x.PATCH)  | 🟢 Low    | Auto-merge recommended               |
| Minor (x.MINOR.x)  | 🟡 Medium | Merge recommended with review        |
| Major (MAJOR.x.x)  | 🔴 High   | Manual review recommended — ask user |

### Step 5: Present results

Display a summary table for all PRs. Include both CI and local test status:

```markdown
| # | PR | Type | CI | Local Test | Mergeable | Risk | Recommendation |
|---|---|---|---|---|---|---|---|
| 1 | #123 — Bump junit 5.11.4 → 5.11.5 | Patch | ✅ | ✅ PASS | ✅ | 🟢 | Merge |
| 2 | #124 — Bump mockito 5.17 → 5.23 | Minor | ✅ | ✅ PASS | ✅ | 🟡 | Merge |
| 3 | #125 — Bump jackson 2.x → 3.0.0 | Major | ✅ | ❌ FAIL | ✅ | 🔴 | Block — tests fail |
| 4 | #126 — Bump actions/checkout v4 → v5 | Major | ✅ | ⏭️ SKIP | ✅ | 🔴 | Ask user |
| 5 | #128 — Bump dep 1.2.3 → 1.2.4 | Patch | ✅ | — | ✅ | 🔄 | Close — already applied |
```

For PRs with 🔴 high risk (major updates): ask the user explicitly before merging.
For PRs where local tests fail: block merge and show error details.
For 🔄 stale PRs: close automatically — the version is already in the codebase.
For ⚠️ special dependency PRs: redirect to the appropriate upgrade skill/process.

### Step 6: Ask for user confirmation

Before merging anything, **stop and ask the user for confirmation**. Present the list of
PRs you recommend merging and wait for approval.

Example prompt:

```
I recommend merging the following PRs:
- #123 — Bump junit 5.11.4 → 5.11.5 (🟢 Patch, local tests ✅)
- #124 — Bump mockito 5.17 → 5.23 (🟡 Minor, local tests ✅)

The following PR requires your decision:
- #126 — Bump actions/checkout v4 → v5 (🔴 Major, not locally testable)

The following PRs are blocked:
- #125 — Bump jackson 2.x → 3.0.0 (❌ local tests fail)

The following PRs will be closed (stale — already applied):
- #128 — Bump dep 1.2.3 → 1.2.4 (🔄 version already in codebase)

Should I proceed? (all / select specific PRs / skip)
```

**Do NOT merge until the user responds.** If the user says:
- **"all"** or **"yes"** — merge all recommended PRs (and major PRs if explicitly included)
- **Specific PR numbers** — merge only those
- **"skip"** or **"no"** — do not merge, end with summary only

### Step 7: Merge approved PRs

For all approved PRs (🟢 and 🟡 that the user confirmed), merge with:

```bash
gh pr merge <PR_NUMBER> --squash --delete-branch
```

Use `--squash` to keep commit history clean.

Wait for each merge to complete before starting the next (avoid race conditions).

### Step 8: Summary

After all PRs are processed, display a final report:

```markdown
## Dependabot PR Report

**Date:** YYYY-MM-DD
**Total:** X PRs processed

### Merged ✅
- #123 — Bump junit 5.11.4 → 5.11.5 (local tests ✅)
- #124 — Bump mockito 5.17 → 5.23 (local tests ✅)

### Closed (stale) 🔄
- #128 — Bump dep 1.2.3 → 1.2.4 (already applied in codebase)

### Skipped / Deferred ⏸️
- #126 — Bump actions/checkout v4 → v5 (major — awaiting manual review)

### Blocked ❌
- #125 — Bump jackson 2.x → 3.0.0 (local tests failed)

### Failed ❌
- (none)
```

---

## Special Cases

### CI checks still in progress

If CI checks are running (pending/in_progress):

1. Inform the user that checks are still running
2. Suggest waiting and retrying later
3. Do NOT merge PRs with pending checks
4. Local tests CAN still run — this helps pre-validate while waiting for CI

### PR has merge conflicts

1. Comment on the PR with `@dependabot rebase`
2. Inform the user that a rebase was requested
3. Do NOT merge PRs with conflicts
4. Do NOT run local tests on PRs with conflicts (worktree checkout will fail)

```bash
gh pr comment <PR_NUMBER> --body "@dependabot rebase"
```

### Files beyond dependencies are changed

If files other than dependency files are modified, flag the PR for manual review.
Dependency files include: `pom.xml`, `package.json`, `package-lock.json`, `requirements.txt`,
`.github/workflows/*.yml`, `Dockerfile`, `go.sum`, `go.mod`, `Cargo.toml`, `Cargo.lock`,
`build.gradle`, `gradle.lockfile`.

### Local tests fail but CI passes

If local tests fail but CI checks pass, this may indicate environment differences.
Flag the PR for manual review and include the local test error output. Do NOT auto-merge.

### Local tests pass but CI fails

If local tests pass but CI fails, the CI failure may be environment-specific or flaky.
Report both results and let the user decide. Do NOT auto-merge.

### Stale PRs — version already applied

When a PR proposes a version that is already in the codebase on `main`:

1. Mark the PR as 🔄 Stale in the summary table
2. Close the PR automatically with:

   ```bash
   gh pr close <PR_NUMBER> --comment "Closing: this version is already applied in the codebase on main."
   ```

3. No user confirmation needed for closing stale PRs — this is automatic cleanup

---

## Important Rules

- **Dependabot only**: This skill processes ONLY PRs from `dependabot[bot]`. Ignore all others.
- **Never force-merge**: If checks fail, do NOT merge.
- **Never merge if local tests fail**: Even if CI passes, local test failure blocks merge.
- **Close stale PRs**: If the proposed version is already in the codebase, close the PR automatically.
- **Major = caution**: Major version updates always require the user's explicit approval.
- **Always clean up worktrees**: Ensure `/tmp/{{PROJECT_NAME}}-pr-*` directories are removed after testing.
- **Always report**: Always show a summary, even if there is only one PR.
