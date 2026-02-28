# Skill: Architect Operating Modes

The Architect operates in one of four modes per task. Clarify which mode applies before starting. If ambiguous, ask.

---

## Mode 1: Design — "How should we build X?"

Produce a design for a new feature, service, or capability.

- [ ] Clarify the problem being solved (not the solution being requested)
- [ ] Identify what already exists that could be extended before proposing anything new
- [ ] Define boundaries, data flows, and integration points
- [ ] Answer the failure question: what breaks, what degrades, how do you detect it
- [ ] Call out infrastructure implications (new components, memory, cost)
- [ ] Call out operational implications (who monitors this, how is it restarted, what's the runbook)
- [ ] Apply project-specific design checks (defined in the project's Architect agent file)
- [ ] Produce output as a **Design Brief** (format below)

## Mode 2: Scope — "What exactly is the work for X?"

Break a vague idea into concrete, sequenced, implementable tasks.

- [ ] Decompose into the smallest shippable increments
- [ ] For each increment: what changes, what's affected, what's the acceptance test
- [ ] Identify dependencies and ordering constraints
- [ ] Flag anything that requires user decisions before engineering can start
- [ ] **⛔ Block sizing self-check** — before finalizing, verify every block passes the sizing guardrails below. If any block fails, split it before producing output.
- [ ] Produce output as a **Scope Breakdown** (format below)

### Block Sizing Guardrails

A block is a unit of work an engineer agent can complete, validate, and close in a **single session**. Oversized blocks degrade agent attention, increase handover errors, and block parallel work. Apply these constraints to every block produced in Scope mode.

#### Metrics

| Metric                                                          | Target        | Red Flag — must split |
| :-------------------------------------------------------------- | :------------ | :-------------------- |
| **Task count**                                                  | 3–5           | >7                    |
| **Repos touched**                                               | 1–2           | >2                    |
| **Dependency depth** (longest sequential chain in the task DAG) | 2–3           | >4                    |
| **Spec length**                                                 | 100–200 lines | >300                  |

#### Cross-Repo Changes

If a change touches >2 repos, it is a **cross-cutting change** and must be decomposed into sequential deploy phases. Each phase = one block.

- **Phase = one deployable, verifiable step** (e.g., "add dual-listen to consumers," "switch publishers to new names," "remove old bindings")
- Each phase has its own acceptance criteria and can be validated before the next phase starts
- The **parent change** lives in `pipeline.md` as a phase or epic — blocks are its children
- Example: a rename across 8 services becomes 3–4 blocks (prepare → migrate consumers → migrate publishers → cleanup), not 1 block with 17 tasks

#### Self-Check (Mandatory Before Finalizing)

Before producing a block plan, answer these questions. If any answer is "no," split the block.

- [ ] Can one engineer agent complete this in a single session (~2–4 hours)?
- [ ] Does the task count stay at 7 or below?
- [ ] Does the block touch 2 repos or fewer? If not, is it decomposed into deploy phases?
- [ ] Is the longest sequential dependency chain 4 or fewer?
- [ ] Can the block be validated independently — without waiting for another block to ship first?

## Mode 3: Evaluate — "Should we do X? Is X the right approach?"

Assess a proposal, compare alternatives, or review an existing design.

- [ ] State the options explicitly (including "do nothing")
- [ ] For each option: cost, complexity, risk, reversibility, operability, time-to-value
- [ ] Identify second-order effects (what breaks, what gets harder, what gets easier)
- [ ] Make a recommendation with reasoning — not a neutral summary
- [ ] Produce output as a **Trade-off Analysis** (format below)

## Mode 4: Audit — "What's wrong with the current state?"

Review part or all of the architecture for problems.

- [ ] Check for: unnecessary coupling, single points of failure, cost waste, security gaps, operational blind spots, complexity that doesn't earn its keep
- [ ] Cross-reference architecture docs against actual implementation
- [ ] Cross-reference technical debt registry for known issues
- [ ] Review prior decisions in architecture logs — any now invalid given changed context?
- [ ] Prioritize findings by blast radius (what hurts most if it fails)
- [ ] Produce output as an **Architecture Review** (format below)

---

## Output Formats

Keep them short. If a section doesn't apply, omit it.

### Design Brief

```/dev/null/design-brief-template.md#L1-17
## Design: {Title}

### Problem
What's broken or missing. One paragraph max.

### Proposal
How it works. Data flows, component interactions, key decisions.

### Failure Modes
What breaks. How the system degrades. How you detect it.

### Affected Components
| Component | Change | Risk |
|:--|:--|:--|

### Infrastructure & Ops Impact
Cost, resources, new dependencies, operational burden.

### Open Questions
Decisions the user must make before engineering starts.
```

### Scope Breakdown

```/dev/null/scope-breakdown-template.md#L1-13
## Scope: {Title}

### Goal
One sentence.

### Increments
1. **{Name}** — {what changes}. Acceptance: {how you know it works}.
2. ...

### Dependencies
What must happen first. What blocks what.

### User Decisions Required
- {Decision}: {options}
```

### Trade-off Analysis

```/dev/null/tradeoff-template.md#L1-17
## Evaluation: {Title}

### Context
Why this decision matters now.

### Options
| | Option A | Option B | Do Nothing |
|:--|:--|:--|:--|
| Cost | | | |
| Complexity | | | |
| Risk | | | |
| Operability | | | |
| Reversibility | | | |
| Time to value | | | |

### Second-Order Effects
What each option makes harder or easier downstream.

### Recommendation
{Pick one}. {Why in 2-3 sentences}.
```

### Architecture Review

```/dev/null/review-template.md#L1-8
## Review: {Title}

### Findings
| # | Finding | Severity | Blast Radius | Recommendation |
|:--|:--|:--|:--|:--|

### Priority Actions
Ordered by impact. Max 5.
```

---

## Project-Specific Extensions

Each project's Architect agent file adds project-specific checks to each mode. Common additions:

- **Design mode:** "Specify {project data stores, message brokers, schemas} affected"
- **Scope mode:** "Estimate {project-specific validation method, e.g., smoke test cost}"
- **Evaluate mode:** "Include {project-specific option, e.g., 'Remove' column}"
- **Audit mode:** "Cross-reference {project-specific architecture doc path}"

These additions belong in the project's `meta/agents/architect.md`, not in this skill.

---

**Last Updated:** 2026-02-28

---

## Changelog

| Date       | Change                                                                                                               |
| :--------- | :------------------------------------------------------------------------------------------------------------------- |
| 2026-02-28 | Added Block Sizing Guardrails to Scope mode — metrics, cross-repo decomposition rule, mandatory self-check           |
| 2026-02-25 | Initial creation — extracted from 42Bros, Batcave, and TRON Architect agents into shared skill with extension points |
