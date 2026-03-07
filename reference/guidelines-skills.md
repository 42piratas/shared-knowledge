# Reference: Designing Agent Skills

Actionable principles for creating and evaluating reusable agent skills. Consult when designing new skills or refining existing ones.

---

## What Is a Skill?

A **bounded instruction set** for a specific class of work. A skill orchestrates tools with domain-specific instructions.

| Concept           | What it is                                       | Granularity    |
| :---------------- | :----------------------------------------------- | :------------- |
| **System prompt** | Agent's global identity and behavioral rules     | One per agent  |
| **Skill**         | Task-specific instruction module the agent loads | Many per agent |
| **Tool**          | A single callable function (API, shell, file op) | Many per skill |

---

## Design Principles

1. **Single responsibility.** One skill, one job. If the description needs "and" → split it.
2. **Self-contained.** The skill must contain everything needed to execute without relying on implicit knowledge from other skills. Assume amnesia.
3. **Explicit inputs and outputs.** Define what the skill expects and what it produces — no ambiguity.
4. **Constraints over freedom.** Specify what NOT to do, quality gates, and stop conditions. Unbounded skills produce inconsistent results.
5. **Composability.** Skills should be stackable into workflows. Each skill is a step; the workflow is orchestration.

---

## Skill Template

```/dev/null/skill-template.md#L1-19
# Skill: {Name}

## When to Use
{One sentence — trigger condition.}

## Instructions
{Step-by-step procedure. Numbered for sequential, bullets for unordered.}

## Constraints
- {What the agent must NOT do}
- {Quality requirements}
- {Scope boundaries}

## Error Handling
{What to do when things go wrong. When to retry vs. ask the user.}
```

---

## Evaluating Skill Quality — CLEAR

| Criterion      | Question                                                                  |
| :------------- | :------------------------------------------------------------------------ |
| **C**omplete   | Contains everything needed to perform the task without guessing?          |
| **L**imited    | Scope narrow enough to be testable and predictable?                       |
| **E**xplicit   | Inputs, outputs, constraints, and error handling spelled out?             |
| **A**ctionable | Agent can execute without asking clarifying questions in the common case? |
| **R**epeatable | Produces consistent results across different inputs?                      |

---

## Common Pitfalls

| Pitfall                 | Symptom                                                | Fix                                              |
| :---------------------- | :----------------------------------------------------- | :----------------------------------------------- |
| Over-scoped skill       | Agent gets confused, skips steps, mixed-quality output | Split into smaller skills                        |
| Implicit assumptions    | Agent makes wrong choices                              | Add explicit constraints and examples            |
| No error handling       | Agent loops forever or silently produces garbage       | Add stop conditions and fallback behavior        |
| Tool overload           | Agent has 20+ tools and picks the wrong one            | Scope tools per skill; only expose what's needed |
| Stale instructions      | Skill references outdated paths or conventions         | Review periodically; version skills              |
| No output specification | Agent produces output in unpredictable formats         | Define exact output structure                    |

---

## Advanced Patterns

**Chaining:** One skill's output feeds the next. Keep inter-skill contracts explicit — define what each skill produces that the next one consumes.

**Conditional branching:** A skill that branches based on input classification. Use IF/THEN in the instructions, not separate skills for each branch.

**Guardrails:** Wrap skills with pre/post checks (tests passing? no secrets in output? user approved?).

**Skill with state:** For long-running sessions, skills can read/write a shared state file. Define the state format explicitly.

---

## Quick-Start Checklist

When creating a new skill:

- [ ] Can I describe what it does in one sentence?
- [ ] Are inputs, outputs, and constraints listed explicitly?
- [ ] Are there constraints on what the agent must NOT do?
- [ ] Is there error handling for the top 3 failure modes?
- [ ] Have I tested it with at least one real input?
- [ ] Is it stored in the right location (shared vs. project-specific)?
- [ ] Does it pass the CLEAR framework?

---

## Cross-References

- `shared-knowledge/reference/guidelines-prompt-engineering.md` — Prompting techniques applicable within skills
- `shared-knowledge/reference/guidelines-sub-agents.md` — Delegating skills to specialized sub-agents

---

**Last Updated:** 2026-03-07
