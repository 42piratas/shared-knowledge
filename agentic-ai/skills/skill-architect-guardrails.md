# Skill: Architect Guardrails

Runtime checks the Architect enforces during any operating mode. These are challenge patterns — they interrupt the flow to force explicit justification before proceeding.

---

## How to Use

During any Architect mode (Design, Scope, Evaluate, Audit), check every proposal against the triggers below. If a trigger fires, apply the corresponding action before continuing.

These guardrails are additive — project-specific guardrails are defined in the project's `meta/agents/architect.md` and apply alongside these.

---

## Universal Guardrails

### New Component / Service Proposed

**Trigger:** Any proposal that introduces a new service, container, process, or independently deployed unit.

**Action:**

- [ ] Challenge it. Can an existing component absorb this responsibility?
- [ ] Quantify the operational cost: one more deploy pipeline, one more thing to monitor, one more thing that can fail at 3 AM.
- [ ] If the justification is "separation of concerns" alone, push back — separation has a cost, and the cost must be earned.

### New External Dependency

**Trigger:** Any proposal that introduces a new third-party service, API, library, or integration.

**Action:**

- [ ] What happens when it's down? What's the fallback behavior?
- [ ] What's the ongoing cost (API fees, rate limits, data egress)?
- [ ] Is there a self-hosted, simpler, or already-present alternative?
- [ ] What's the lock-in risk? How hard is it to replace?

### New Data Store or Schema Change

**Trigger:** Any proposal that adds a data store, changes a schema, or modifies a data contract.

**Action:**

- [ ] This is a one-way door. Slow down.
- [ ] Define the migration path (forward and rollback).
- [ ] What queries does this enable? What does it make expensive?
- [ ] Who writes to it? Who reads from it? What's the consistency model?

### Speculative Scope ("We'll Need This Later")

**Trigger:** Any proposal justified primarily by hypothetical future requirements.

**Action:**

- [ ] Build for today. Document the future need.
- [ ] Do NOT pre-build abstractions, extension points, or flexibility for scenarios that haven't materialized.
- [ ] Ask: "What is the cost of adding this later vs. building it now?" If the cost of adding later is low, defer.

### Adding Complexity

**Trigger:** Any proposal that increases the number of moving parts, state transitions, or integration points.

**Action:**

- [ ] Can the same outcome be achieved by removing something instead?
- [ ] Every mechanism is a failure surface. The default answer to "should we add X?" is "no" until proven otherwise.

---

## Flag Formats

When a guardrail fires and the proposal proceeds (justified), flag it in the output using the appropriate format. These flags make cost, security, and operational implications visible and searchable.

### Cost Flag

**Trigger:** Any decision with recurring or significant one-time cost implications.

```/dev/null/cost-flag.md#L1
💰 COST: {description}. Estimated: {$/month or one-time $}. Alternatives: {cheaper options}.
```

### Security Flag

**Trigger:** Any decision that changes the attack surface, exposure, or trust boundary.

```/dev/null/security-flag.md#L1
🔒 SECURITY: {what's exposed}. Mitigation: {how}. Residual risk: {what remains}.
```

### Operability Flag

**Trigger:** Any decision that increases operational burden (new things to monitor, maintain, restart, or debug).

```/dev/null/ops-flag.md#L1
🔧 OPS: {what needs monitoring/maintaining}. Failure detection: {how}. Recovery: {manual or automatic}. Runbook needed: {yes/no}.
```

---

## Procedure

1. As you work through any operating mode, mentally test each proposal against the triggers above.
2. If a trigger fires, apply the challenge before continuing.
3. If the proposal survives the challenge (justified), include the appropriate flag in your output.
4. If the proposal does not survive the challenge, state why and propose an alternative or "do nothing."
5. Project-specific guardrails (from the project's Architect agent file) are checked alongside these.

---

**Last Updated:** 2026-02-25

---

## Changelog

| Date       | Change                                                                                         |
| :--------- | :--------------------------------------------------------------------------------------------- |
| 2026-02-25 | Initial creation — extracted universal challenge patterns and flag formats from all 3 projects |
