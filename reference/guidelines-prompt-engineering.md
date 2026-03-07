# Reference: Prompt Engineering

Actionable principles for writing effective agent prompts. Consult when creating or modifying agent docs, skills, or system prompts.

---

## Foundational Principles

1. **Be clear and direct.** State the task, expected format, and constraints upfront. Ambiguity is the #1 source of poor output.
2. **Explicit over implicit.** Spell out: what to do, how to do it, what NOT to do, and what success looks like.
3. **Positive framing first, then negative constraints.** Lead with what to do ("Write in plain prose paragraphs"), then close known failure modes ("Do NOT use bullet lists").
4. **Structure over prose.** Numbered steps for sequential work, bullets for unordered, tables for comparisons, headers to separate concerns.
5. **Context before instructions.** Place background, data, and constraints before the task. Queries after context improve quality significantly.
6. **One task per prompt when possible.** Compound prompts degrade quality on each successive task. If sequential, use explicit step numbering.

---

## Few-Shot Prompting

When the output format or reasoning pattern is non-obvious, provide 1–3 examples of input → output pairs. Effective when:

- The task has a specific format the model wouldn't default to
- Classification or extraction with domain-specific categories
- The quality bar is hard to describe but easy to demonstrate

**Rules:** Examples must be representative and correct. Vary them to prevent the model from pattern-matching to a single case. Label examples clearly.

---

## Failure Path Design

Every prompt implicitly assumes the happy path. Explicitly define what to do when stuck.

**A prompt without a failure path is a prompt that will hallucinate.**

| Pattern                  | When to use                                                                                                                          |
| :----------------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| **Escalation ladder**    | When the agent might lack information: "State which step failed and why → state what would unblock it → do NOT guess → ask the user" |
| **Graceful degradation** | When partial results are acceptable: "If type annotations missing → infer from usage and flag each with [INFERRED]"                  |
| **Confidence gating**    | When certainty varies: "Rate confidence HIGH/MEDIUM/LOW. LOW = hypothesis requiring verification, not fact."                         |

---

## Context Management

For large inputs (20K+ tokens) or long-running sessions:

- **Long data at the top, instructions at the bottom.** The model attends better to instructions after reference material.
- **Use delimiters** (XML tags, markdown headers) to separate documents with metadata so the model can reference them precisely.
- **Quote-grounding:** For long-document tasks, instruct the model to quote relevant passages before reasoning. Reduces hallucination.
- **Front-load stable content.** System prompts and reference docs that don't change go first — maximizes prompt caching.
- **Summarize intermediate results.** Pass structured summaries forward between tool calls, not raw outputs.
- **Plan for compaction.** Design so the model can recover state from filesystem/git rather than relying on conversation history.

---

## System Prompt Design

| Section              | Purpose                                                    |
| :------------------- | :--------------------------------------------------------- |
| **Identity**         | Who the agent is, what it does, what it does NOT do        |
| **Behavioral rules** | Communication style, authority model, security constraints |
| **Procedures**       | Step-by-step protocols for common workflows                |
| **Constraints**      | Hard limits, quality gates, stop conditions                |
| **Output formats**   | Templates for recurring outputs                            |
| **Error handling**   | What to do when stuck, uncertain, or out of scope          |

**Key choices:**

- Persona depth: enough to guide behavior, not so much it becomes roleplay
- Constraint density: high for junior/utility models, lower for frontier models
- Tool instructions: explicit "when to use tool X" reduces misuse

---

## Anti-Patterns

| Anti-Pattern                  | Why it fails                                                | Fix                                                         |
| :---------------------------- | :---------------------------------------------------------- | :---------------------------------------------------------- |
| "Do your best"                | No quality bar; model defaults to average                   | Define specific criteria and output format                  |
| Mega-prompt                   | 5000 words covering every scenario; model loses focus       | Split into focused skills or chained prompts                |
| No negative constraints       | Model does things you didn't want                           | Add "Do NOT" rules for known failure modes                  |
| Only negative constraints     | Model knows what to avoid but not what to aim for           | Lead with positive instructions                             |
| Instructions after data       | Model processes data before seeing instructions             | Data at top, query at bottom — or wrap data in delimiters   |
| Implicit format               | Model picks random format each time                         | Specify exact format with an example                        |
| "Think step by step" alone    | Too vague; model's steps may not match yours                | Provide the specific steps to follow                        |
| Persona without constraints   | Model roleplays but has no guardrails                       | Add behavioral rules alongside the persona                  |
| No error path                 | Model invents data when stuck                               | Define failure behavior: ask, retry, degrade, or stop       |
| Over-prompting capable models | Aggressive instructions cause overthinking in strong models | Tune to the model's baseline; dial back if already thorough |

---

## Model Tier Guidance

| Tier                                       | Characteristics                                                | Prompting approach                                                                                                                                 |
| :----------------------------------------- | :------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frontier** (Opus-class, o3-class)        | Complex reasoning, long system prompts, self-correction        | Less prescriptive — give goals, not procedures. Native thinking outperforms hand-crafted CoT. Watch for overtriggering on aggressive instructions. |
| **Workhorse** (Sonnet-class, GPT-4o-class) | Good balance of capability and cost. May shortcut long chains. | Explicit CoT structure helps. Deliberate over-instruction prevents premature summarization.                                                        |
| **Utility** (Haiku-class, Flash-class)     | Narrowly scoped tasks. Struggles with deep reasoning.          | Keep prompts short and direct. Few-shot examples compensate for weaker instruction-following.                                                      |
| **Reasoning** (o3-class thinking modes)    | Explicit reasoning traces. Benefits from less constraint.      | Provide the objective, not the procedure. Step-by-step instructions can reduce quality.                                                            |

---

## Cross-References

- `shared-knowledge/reference/guidelines-skills.md` — Applying these techniques within skills
- `shared-knowledge/reference/guidelines-sub-agents.md` — Prompting for orchestrators and sub-agents

---

**Last Updated:** 2026-03-07
