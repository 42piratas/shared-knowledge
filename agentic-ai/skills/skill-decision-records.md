# Skill: Decision Records

Record architectural and design decisions so future sessions understand the _why_.

---

## When to Record

A decision needs a record if it meets **any** of:

- Changes a service boundary, data contract, or integration point
- Introduces or removes an external dependency
- Has cost implications (infrastructure, paid APIs)
- Constrains future options (hard to reverse)
- Overrides or supersedes a previous decision

When in doubt, record. A lightweight Y-Statement costs 30 seconds. A missing decision costs hours of archaeology.

---

## Format A: Y-Statement (Lightweight — Most Decisions)

For decisions that are straightforward and don't require deep analysis. One block, captured inline in session logs or block plans. Also used to summarise a full ADR.

```/dev/null/y-statement-template.md#L1-5
In the context of {use case or situation},
facing {concern or constraint},
we decided for {option chosen}
to achieve {desired quality or outcome},
accepting {downside or trade-off}.
```

**Example:**

```/dev/null/y-statement-example.md#L1-5
In the context of needing exchange precision data,
facing the choice between caching market info or fetching per-order,
we decided for startup-time caching with 24h refresh
to achieve sub-second gating without API rate pressure,
accepting stale precision data if an exchange changes lot sizes mid-day.
```

Inspired by the [Y-statement](https://medium.com/olzzio/y-statements-10eb07b5a177) from Zdun et al.

---

## Format B: Decision Record (Significant — Standalone File)

For decisions that are hard to reverse, have broad impact, or will be questioned by future sessions.

**File naming convention:** `{project-log-path}/adr-YYMMDD-{short-title}.md`

Each project defines where ADRs live (typically `meta/logs/architecture/`).

```/dev/null/adr-template.md#L1-18
# ADR: {Short Title}

**Status:** proposed | accepted | superseded by ADR-YYMMDD-{x} | deprecated
**Date:** YYYY-MM-DD

## Context
What forces are at play. What constraint or need is driving this. Facts, not opinions.

## Decision
What we are doing. Active voice: "We will..."

## Consequences
What becomes easier. What becomes harder. What risk remains.
All consequences — not just the positive ones.

## References
- Pipeline: {phase or debt ID, if applicable}
- System map: {section affected}
- Supersedes: {previous ADR, if applicable}
```

Inspired by [Nygard's ADR](http://thinkrelevance.com/blog/2011/11/15/documenting-architecture-decisions).

---

## Decision Lifecycle

Decisions are **never deleted** — only superseded or deprecated.

When context changes and a decision no longer holds:

1. Create a new ADR explaining the new decision.
2. Update the old ADR's status to `superseded by ADR-YYMMDD-{x}`.
3. This preserves the reasoning chain — future sessions can trace _why_ things changed.

---

## Choosing a Format

| Situation                                               | Format                                          |
| :------------------------------------------------------ | :---------------------------------------------- |
| Quick, reversible, low-impact                           | Y-Statement inline in session log or block plan |
| Hard to reverse, broad impact, will be questioned later | Full ADR as standalone file                     |
| Superseding a previous ADR                              | Full ADR (to update the chain)                  |

---

## Procedure

1. At the point a decision is made during a session, determine if it meets the recording criteria above.
2. Choose the appropriate format.
3. Write the record immediately — not at session end. Context decays fast.
4. For full ADRs: check if any existing ADRs in the project's architecture log are superseded by this decision. Update their status.

---

**Last Updated:** 2026-02-25

---

## Changelog

| Date       | Change                                                                                         |
| :--------- | :--------------------------------------------------------------------------------------------- |
| 2026-02-25 | Initial creation — extracted from 42Bros, Batcave, and TRON Architect agents into shared skill |
