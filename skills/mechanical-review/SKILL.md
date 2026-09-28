---
name: mechanical-review
description: >
  Objective analysis of a GitLab MR for bugs, regressions, security issues, and
  other concrete risks. Produces a structured report saved to a temp file. Does
  not post comments or offer design opinions — use design-review for design and
  post-review for posting. Use when you want a thorough mechanical check of an MR.
---

# Mechanical Review

Objective analysis of an MR for bugs, regressions, security issues, and other
concrete risks. Produces a structured report saved to a temp file.

This skill answers one question: does this code break anything?

It does not post comments, offer design opinions, or suggest simplifications.
Tone and response formatting are out of scope — a separate skill handles how
review comments are written and posted. The report is the raw material.

## Contract

Input: an MR reference — number, URL, branch name, or `group/project!number`.

Output: a structured report written to `/tmp/mechanical-review-<slug>.md` and
displayed. Each finding includes full context so a downstream agent can post
the review without re-analyzing the code.

Default mode is read-only. Do not post comments, modify code, or push anything.

## What To Flag

| Category | Hunt for | Typical severity |
|---|---|---|
| **Bugs** | Off-by-one, null/undefined deref, wrong operator, swapped args, incorrect conditionals, integer overflow, missing `await` on async return used synchronously, missing resource cleanup | BLOCKER / MAJOR |
| **Build/CI** | Syntax errors, type errors, missing imports, broken test assertions, circular imports | BLOCKER |
| **Regressions** | Behavior change in touched code paths, removed edge-case handling, altered response shapes that break callers | BLOCKER / MAJOR |
| **Security** | SQL/command/prompt injection, hardcoded secrets, auth bypass, unsafe deserialization, missing input validation, CSRF, XSS | BLOCKER |
| **Concurrency** | Race conditions, shared-state mutation, missing awaits, deadlock risk, stale closures over async state | BLOCKER / MAJOR |
| **Error handling** | Swallowed exceptions, unchecked errors, thrown-but-never-caught, retry-forever loops, missing timeouts on network calls | MAJOR |
| **API/contract breakage** | Changed signatures on exported symbols, removed exports, altered response shapes, semver violations | BLOCKER / MAJOR |
| **Resource leaks** | Unclosed files/connections, missing cleanup, dangling event listeners, promise leaks | MAJOR |
| **Test gaps** | MR adds logic but no test exercises it. Evaluate only functions this MR adds or modifies — skip passthrough, config, presentational code | MAJOR / MINOR |
| **Test anti-patterns** | Re-implements source logic, tests constants, tests mock behavior, tautological, no meaningful assertion. Flag as MAJOR when it's the only test for important new logic | MAJOR / MINOR |
| **Performance** | N+1 queries, hot-path bloat, unnecessary computations, sequential awaits that should be parallel, missing change-detection guards in polling/intervals | MAJOR / MINOR |
| **Backwards compat** | Database schema changes that break existing data, serialization format changes, config format changes, event/message shape changes consumed by other services | BLOCKER / MAJOR |
| **Migration safety** | Table locks on large tables, irreversible migrations, migrations unsafe on live systems, missing backfill for NOT NULL columns | BLOCKER / MAJOR |

## What NOT To Flag

| Skip | Why |
|---|---|
| Naming preferences | Not a defect unless genuinely misleading |
| Formatting / whitespace | Linter territory |
| "More idiomatic" restructures | Preference, not a risk |
| Simplification opportunities | Not review findings |
| Refactors of pre-existing code | Out of scope for this MR |
| Missing tests on untouched code | Not the author's job this MR |
| DRY / duplication cleanups | Not a risk unless it caused a bug |
| Micro-optimizations without evidence | Premature |
| Design opinions | Belongs in design-review |
| Architecture decisions | Belongs in design-review |
| Documentation for internal helpers | Only flag on exported API changes |
| Code quality / craft | Not review findings |

If unsure whether something is a concrete risk or a preference, it's a
preference. Drop it.

## Workflow

### 1. Gather Context

Run in parallel:

```bash
glab mr view <ref> -F json
glab mr diff <ref>
glab api projects/:fullpath/merge_requests/<iid>/approvals
glab api projects/:fullpath/merge_requests/<iid>/discussions --paginate
```

Then `glab mr checkout <ref>`. If checkout fails (fork, permissions), review from
diff only and note the limitation in the report.

Read the MR description for author intent, testing instructions, caveats, and "will
fix in follow-up" disclaimers. Respect stated scope.

### 2. File Triage

Classify each changed file:

| Category | Criteria | Depth |
|---|---|---|
| **New** | Didn't exist on base | Full read + full analysis |
| **Heavy mod** | Significant logic change, new funcs, altered control flow | Full read + full analysis |
| **Light mod** | Minor change (added a field, wired a call) | Hunks + ~20 lines context |
| **Refactor/move** | Renamed, mechanically restructured | Skim, verify no logic drift |
| **Deleted** | Removed | Skim, grep for leftover references |
| **Test** | Test file | Full read, check for anti-patterns |
| **Config/infra** | package.json, tsconfig, CI, lockfiles | Skim, flag only breaking changes |

Skip entirely: `*.lock`, `package-lock.json`, `yarn.lock`, `go.sum`, generated
code, vendored deps, snapshot files, build output.

For large diffs (>10 New + Heavy mod files), dispatch parallel sub-agents per
file group. Each agent gets the full diff for cross-file context and returns
findings with full bodies.

### 3. Analyse

Review the diff, not the codebase. Only flag pre-existing issues if this MR
makes them demonstrably worse — more callers, wider blast radius, or a removed
guard that was compensating.

For each candidate finding, record all fields from the report format (Step 5)
before running validation.

#### Validation checks (mandatory on every finding)

**Line-discipline.** Re-derive the line number from the diff. Find the nearest
`@@ -X,Y +Z,W @@` hunk header for the file, walk `+` lines from line `Z`,
verify the cited symbol is on a `+` line. If not, the finding is about
pre-existing code — drop it.

**Contradicting-guard.** Read ±3 lines around the cited line. If a guard
expression directly contradicts the claim (finding says "X is unhandled" but
the surrounding code contains `if (X) throw` / `if (X) return err`), drop as
code-state mismatch. Posting a finding the surrounding code already handles
undermines every other finding in the report.

**Function-scope.** For files >300 LOC or files with multiple functions sharing
common identifiers: verify the cited symbol is inside the correct function's
body, not a name-collision in a sibling scope.

**Paired-path completeness.** When the diff has symmetric logic (streaming vs
non-streaming, happy-path vs error-path, multiple enum consumer branches),
check that a bug found on one branch also exists on the paired branch. If you
find one finding, look for two.

### 4. Dedup Against Existing Reviews

If other reviewers (human or bot) have already posted comments on this MR,
read them. Do not re-flag anything already covered unless investigation adds
meaningful new insight. The report's value is what others missed.

If overlapping with a prior review, note it: "Also flagged by <reviewer> —
additional context: ..."

### 5. Write Report

Save to `/tmp/mechanical-review-<slug>.md` where `<slug>` is the MR ref with
special characters replaced by hyphens (e.g., `my-project-123`).

Display the report to the user and confirm the file path.

## Report Format

The report is the deliverable. Every finding must be self-contained — a reader
(human or agent) who has not seen the code should understand the issue, why it
matters, and what to do about it.

```
# Mechanical Review: <group>/<project>!<number>

**MR:** <title>
**Author:** <author>
**Diff:** +<additions> / -<deletions> across <changedFiles> files
**Reviewed at:** <head SHA> on <ISO date>

## MR Intent

<2-3 sentences: what the MR does and why, from the MR description and commits.
State when inferring vs quoting.>

## Findings

### BLOCKER

#### <one-line title>
- **Category:** <category from the table>
- **Confidence:** <high|med|low>
- **Location:** `<file>:<line>`
- **Issue:** <what the code does, what the concrete risk is, and why it
  matters. Plain English. A reader who hasn't seen the diff should
  understand this paragraph on its own.>
- **Evidence:** <specific code references — callers, data flow, guards
  checked, related code paths — that support the claim. Cite file:line.>
- **Suggested fix:** <concrete action, not "handle this properly">

### MAJOR

<same structure per finding>

### MINOR

<same structure per finding>

### QUESTION

<same structure per finding>

<omit any severity section with zero findings>

## Low Confidence

Findings below the confidence bar. Included for completeness — promote
any that look real on re-read.

<same structure per finding, all confidence: low>

<omit section if no low-confidence findings>

## Summary

- **Findings:** <N total> — <breakdown by severity>
- **Confidence spread:** high: <N>, med: <N>, low: <N>
- **Verdict:** <one of the three below>
  - "No blocking issues found"
  - "Non-blocking issues to address (<N> MAJOR, <N> MINOR)"
  - "Blockers found (<N> BLOCKER) — fix before merge"
```

## Quality Bar

The skill succeeds when:
- Every finding is objectively verifiable — a concrete risk, not an opinion
- Every finding body is self-contained enough that a downstream agent can
  post it without re-reading the code
- The don't-flag list is respected — no design opinions, no style preferences,
  no simplification suggestions
- Validation checks (line-discipline, contradicting-guard, function-scope)
  were applied to every finding
- The report file was written and its path displayed

The skill fails when:
- Findings include design opinions disguised as risks
- Findings suggest vague fixes ("handle this properly", "consider improving")
- The report requires re-analysis to act on
- A finding cites a line that doesn't exist on a `+` line in the diff
- A finding claims something is unhandled when surrounding code handles it

Proceed with the MR reference the user provided.
