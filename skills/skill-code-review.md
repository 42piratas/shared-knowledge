# Skill: Code Review

Two-phase code review procedure: read-only audit, then targeted remediation. Applies to any codebase, any language.

---

## When to Use

When reviewing code changes for quality, correctness, security, and maintainability. Triggered by:

- Scheduled review sessions (periodic)
- Pre-merge review requests
- Post-implementation quality gates

---

## Phase 1: The Audit (Read-Only)

Understand first. Judge second. Change nothing yet.

### Scope Materialization (mandatory — before any review)

⛔ **No review may begin until the full file manifest is produced and every file on it is read.**

- [ ] 1. Produce the complete list of changed files in scope:
  - PR-scoped: `gh pr diff --name-only --repo {org}/{repo}`
  - Time-scoped: `git log --since="{timestamp}" --name-only --pretty=format: | sort -u` per repo
  - User-specified: use the provided list as-is
- [ ] 2. This list is the **file manifest** — the binding contract for the review. Every file on it must be read in full and appear in the audit report's `Files Reviewed` field.
- [ ] 3. Read every file on the manifest. Fully. No skimming. No skipping.
- [ ] 4. After the audit report is written, verify: every file in the manifest appears in `Files Reviewed`. Any missing file means the review is **incomplete** — go back and review it before finalizing.

### Audit Execution

- [ ] Read every file in scope. Fully. No skimming.
- [ ] Cross-reference against the project's review checklist (defined in project docs).
- [ ] Check contracts — do interfaces, schemas, and data formats still match between producers and consumers?
- [ ] Check for DRY violations — same logic in multiple places, same information in multiple docs.
- [ ] Cross-check: verify the engineer's change checklist was completed (see project's Engineer agent or `skill-change-tracking.md`).
- [ ] Create audit report in the project's code review log directory.

### Audit Report Format

```/dev/null/audit-report-template.md#L1-19
## Code Review: {Scope Description}

**Date:** YYYY-MM-DD
**Files Reviewed:** {complete list — must match file manifest exactly}
**Last Review:** {date of previous review, or "first review"}

### Findings

| # | Severity | File | Finding | Guideline |
|:--|:--|:--|:--|:--|
| 1 | BLOCKER | ... | ... | ... |
| 2 | HIGH | ... | ... | ... |
| 3 | MEDIUM | ... | ... | ... |

### Summary
{1-2 sentences: overall health, trend since last review}
```

---

## Phase 2: The Remediation (Action) — ENGINEER RESPONSIBILITY

**The reviewer does NOT perform remediation.** The reviewer is read-only. Phase 2 is the engineer's responsibility.

After receiving the audit report, the engineer:

- [ ] Reads the audit report in full
- [ ] Applies ALL fixes — every finding, every severity level
- [ ] Runs tests to verify no regressions
- [ ] For each fix: commits with a clear reference to the finding
- [ ] Documents what was fixed and how

The reviewer performs a follow-up validation audit after the engineer applies fixes, to confirm findings were addressed correctly.

### Remediation Report Format

```/dev/null/remediation-report-template.md#L1-14
## Code Review Remediation: {Scope Description}

### Fixed
| # | Severity | File | What Changed | Commit |
|:--|:--|:--|:--|:--|

### Deferred
| # | Severity | File | Reason | Debt ID |
|:--|:--|:--|:--|:--|

### Tests
- [ ] Existing tests still pass?
- [ ] New tests added for fixes?
```

---

## Deferral Policy

**All findings must be fixed.** There is no automatic deferral to a debt tracker based on severity level.

Exception path — the only route to deferral:

1. Engineer (applying fixes) determines that a fix would cause significant risk, downtime, cost, or requires architectural decisions outside the current scope
2. Engineer flags the concern explicitly to the user, describing the risk
3. User reviews and decides
4. If user approves deferral → item goes to pipeline/debt tracker with `[REVIEW-DEBT]` tag, severity, and origin (`code-review YYMMDD`)
5. If user does not explicitly approve → the fix must be made

Findings are never silently deferred. The pipeline debt tracker is not a dumping ground for avoided work.

---

## Severity Levels

| Level       | Meaning                                                    | Action                        |
| :---------- | :--------------------------------------------------------- | :---------------------------- |
| **BLOCKER** | Breaks core functionality or corrupts data/state           | Fix immediately               |
| **HIGH**    | Incorrect behavior, safety gap, contract violation         | Fix in this session           |
| **MEDIUM**  | Code quality issue, missing test, DRY violation            | Fix if quick, otherwise defer |
| **LOW**     | Style inconsistency, minor naming issue, documentation gap | Defer unless trivial          |

---

## Scope Definition

How to determine what to review:

1. **User specifies files** → review those.
2. **No scope given** → identify files changed since the last review.
   - Extract timestamp from the last review log's filename.
   - Find all files modified or created after that timestamp.
3. **No changes found** → report "No changes since last review" and stop.

Regardless of how scope is determined, the reviewer **must** produce a file manifest before starting the audit (see §Phase 1 — Scope Materialization). The manifest is the single source of truth for what gets reviewed. No file may be skipped; no file outside the manifest needs to be reviewed.

---

## Escalation Boundaries

Stay in scope. If a review finding falls outside the reviewer's domain, note it but do not act:

| Finding Type                  | Escalate To               |
| :---------------------------- | :------------------------ |
| Requires design/arch decision | Architect                 |
| Requires new feature work     | Engineer                  |
| Security-specific deep audit  | Dedicated security review |

Flag the finding in the audit report with a note: `[ESCALATE: {target agent}]`.

---

## Project-Specific Extensions

Each project extends this skill with:

1. **Review checklist** — language-specific, framework-specific, or domain-specific checks (e.g., Python type hints, parsing contract compliance, financial math rules).
2. **Contract checks** — project-specific interface/schema validation (e.g., API contracts, config schemas, prompt ↔ parser alignment).
3. **Log directory** — where audit and remediation reports are stored.
4. **Out-of-scope list** — what this reviewer explicitly does NOT cover in this project.

These extensions belong in the project's code review guidelines doc or the Reviewer agent file — not in this shared skill.

---

**Last Updated:** 2026-03-07

---

## Changelog

| Date       | Change                                                                                                               |
| :--------- | :------------------------------------------------------------------------------------------------------------------- |
| 2026-02-25 | Initial creation — extracted two-phase audit/remediation structure from 42Bros, Batcave, and TRON reviewers          |
| 2026-03-07 | Added mandatory Scope Materialization step — file manifest must be produced and fully covered before audit completes |
