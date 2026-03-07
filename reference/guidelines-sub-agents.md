# Reference: Multi-Agent Orchestration

Principles for designing systems where multiple specialized agents collaborate. Consult when designing orchestrators, adding new agents, or refining handoff protocols.

---

## When to Use Sub-Agents

**Use sub-agents when:**

- Tasks require different expertise (review ≠ debugging ≠ deployment)
- Context windows are a constraint — focused context per agent beats bloated shared context
- Tool isolation matters — least-privilege tool access per agent
- Parallel work is possible — independent subtasks run concurrently
- Failure isolation is important — one agent failing doesn't corrupt the parent's state

**Do NOT use sub-agents when:**

- A single agent with skills can handle it — don't add architecture for simple tasks
- The task is inherently sequential and tightly coupled — use skill chaining instead
- You can't define clear boundaries — blurry scope creates gaps and overlaps
- Cost/latency is critical — each sub-agent is a separate invocation

### Decision Framework

```/dev/null/decision-framework.md#L1-10
Is the task complex enough that a single agent struggles with quality?
  └─ No → Use a single agent with skills. Stop here.
  └─ Yes ↓

Can you decompose it into independent subtasks with clear boundaries?
  └─ No → Use prompt chaining (sequential skills in one agent). Stop here.
  └─ Yes ↓

Do subtasks require different tools, context, or expertise?
  └─ No → Use parallelized prompts (same agent, different inputs). Stop here.
  └─ Yes → Use sub-agents.
```

---

## Architecture Patterns

| Pattern                  | How it works                                                             | When to use                                         |
| :----------------------- | :----------------------------------------------------------------------- | :-------------------------------------------------- |
| **Orchestrator-Workers** | Central agent decomposes task, delegates to workers, synthesizes results | Unpredictable number/nature of subtasks             |
| **Pipeline**             | Sequential handoff: Agent A → Agent B → Agent C                          | Each step transforms output for the next            |
| **Router**               | Classifier routes to the right specialist agent                          | Distinct task types with clear routing criteria     |
| **Evaluator-Optimizer**  | Generator produces, evaluator critiques, loop until quality met          | Quality is measurable and iteration improves output |
| **Parallel Sectioning**  | Independent subtasks run concurrently, results merged                    | Subtasks have no dependencies on each other         |

---

## Design Principles

1. **Clear boundaries.** Every sub-agent needs a one-sentence scope definition. If you can't write it, the boundary is wrong.

2. **Minimal authority (least privilege).** Each sub-agent gets only the tools it needs. A reviewer doesn't need write access. A test runner doesn't need deploy credentials.

3. **Explicit contracts.** Define what each sub-agent receives (input) and produces (output). When Agent A's output feeds Agent B, treat it as an API boundary — contracts must align.

4. **Fail gracefully.** Set timeouts, max iterations (2–4 for loops), and retry limits (1–2). Sub-agent errors surface to the parent with enough context to decide: retry, skip, or escalate.

5. **Preserve context across handoffs.** Explicitly pass: what was done, what to do next, what to watch out for, and artifacts produced. "I finished the design. Over to you." is a failure.

6. **Observability.** Every sub-agent produces structured logs: agent name, input summary, actions taken, output summary, duration, errors. Without this, debugging multi-agent workflows is impossible.

---

## Handoff Format

```/dev/null/handoff-example.md#L1-14
## Handoff: {Source Agent} → {Target Agent}

### Completed
- {What was done — specific, with file references}

### Next Action
{What the target agent should do — specific task, not vague pointer}

### Watch Out For
- {Known issues, edge cases, constraints encountered}

### Files to Read First
- {Paths to relevant documents}
```

---

## Model Tiering

Not every sub-agent needs the most capable model:

| Role                  | Model Tier  | Rationale                            |
| :-------------------- | :---------- | :----------------------------------- |
| Router / classifier   | Fast, cheap | Simple classification, speed matters |
| Code reviewer         | Mid-tier    | Needs reasoning, not maximum depth   |
| Architecture designer | Top-tier    | Complex multi-factor reasoning       |
| Log summarizer        | Fast, cheap | Extractive task, low complexity      |
| Security auditor      | Top-tier    | High stakes, thorough analysis       |

---

## Common Pitfalls

| Pitfall                  | Symptom                                                          | Fix                                                        |
| :----------------------- | :--------------------------------------------------------------- | :--------------------------------------------------------- |
| Over-decomposition       | Simple task split across 5 agents; high latency, no quality gain | Start with one agent + skills. Use the decision framework. |
| Blurry boundaries        | Two agents both handle the same task, or neither does            | Explicit "you do / you don't" lists per agent              |
| Context loss at handoffs | Downstream agent asks questions already answered upstream        | Structured handoffs with explicit context                  |
| Orchestrator bottleneck  | Parent can't meaningfully evaluate sub-agent output              | Keep orchestrator lightweight; trust sub-agents            |
| Sub-agent sprawl         | 15 agents, most rarely used; maintenance burden grows            | A sub-agent must earn its existence. Consolidate.          |
| Cascading failures       | One agent fails → parent retries endlessly → costs spiral        | Set retry limits (1–2). Fail fast. Escalate to user.       |
| Lost observability       | Multi-agent workflow fails, no one knows where or why            | Require structured logs from every sub-agent               |
| Privilege creep          | Sub-agent gradually gets more tools "for convenience"            | Audit tool access periodically. Least privilege always.    |

---

## Quick-Start Checklist

- [ ] Confirmed a single agent with skills can't handle this?
- [ ] Clear boundaries between each sub-agent's responsibilities?
- [ ] One-sentence purpose statement per sub-agent?
- [ ] Input and output contracts defined?
- [ ] Minimum necessary tools per sub-agent (least privilege)?
- [ ] Escalation path when a sub-agent fails or is out of scope?
- [ ] Handoffs are structured and explicit?
- [ ] Every sub-agent produces structured logs?
- [ ] Maximum iteration/retry limits set?

---

## Cross-References

- `shared-knowledge/reference/guidelines-skills.md` — Skills as building blocks within each agent
- `shared-knowledge/reference/guidelines-event-hooks.md` — Event-driven communication between agents

---

**Last Updated:** 2026-03-07
