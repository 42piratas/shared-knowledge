# Guidelines: Prompt Engineering

**Category:** Agentic AI
**Last Updated:** 2026-02-25

---

## Overview

This document consolidates core prompt engineering techniques — from foundational principles to advanced patterns — into a single reference. It covers how to write effective prompts, how to structure reasoning, how to build self-correction into outputs, and how to configure system-level instructions that maximise model performance across sessions.

All techniques are model-agnostic and applicable to any web/software-development context.

---

## 1. Foundational Principles

These apply to every prompt, every time.

### 1.1. Be Clear and Direct

State what you want, what format you expect, and what constraints apply — upfront. Ambiguity is the #1 source of poor model output.

| Principle            | Good                                                                   | Bad                                                                          |
| :------------------- | :--------------------------------------------------------------------- | :--------------------------------------------------------------------------- |
| State the task first | "Review this function for security issues."                            | "So I have this code, and I was wondering if maybe you could take a look..." |
| Specify format       | "Return a markdown table with columns: File, Line, Finding, Severity." | "Tell me what you find."                                                     |
| Set constraints      | "Maximum 10 findings, ordered by severity."                            | (no constraint — model produces 50 findings)                                 |

**Golden rule:** Show your prompt to a colleague with minimal context on the task and ask them to follow it. If they'd be confused, the model will be too.

### 1.2. Explicit Over Implicit

Never assume the model infers what you mean. Spell out:

- **What** to do (the task)
- **How** to do it (the method or procedure)
- **What not** to do (negative constraints — models follow "do NOT" instructions well)
- **What success looks like** (output format, quality bar, acceptance criteria)

### 1.3. Positive Framing Over Negation

Tell the model what to do, not just what to avoid. Positive instructions are more reliable than negative ones because they give the model a concrete target rather than a boundary to guess around.

| Instead of                              | Try                                                                          |
| :-------------------------------------- | :--------------------------------------------------------------------------- |
| "Do not use markdown in your response." | "Write your response in plain prose paragraphs."                             |
| "Don't be verbose."                     | "Be concise. One sentence per finding."                                      |
| "Don't make assumptions."               | "If any detail is ambiguous, state the ambiguity and ask before proceeding." |

Negative constraints still have value — use them to close known failure modes _after_ you've stated the positive instruction. Example: "Write in plain prose paragraphs. Do NOT use bullet lists or markdown headers."

### 1.4. Structure Over Prose

Structured prompts outperform narrative ones. Use:

- **Numbered steps** for sequential procedures
- **Bullet lists** for unordered requirements
- **Tables** for comparisons or classifications
- **Headers** to separate concerns (context, instructions, constraints, output format)
- **XML tags** or markdown sections to delimit data from instructions (see §1.8)

### 1.5. Provide Context Before Instructions

The model processes tokens sequentially. Place background information, reference data, and constraints **before** the task instruction — not after. Queries placed after context can improve response quality by up to 30%, especially with complex, multi-document inputs.

```
## Context
{background, data, reference material}

## Constraints
{rules, limits, exclusions}

## Task
{what to do, given the above}

## Output Format
{exactly how to structure the response}
```

### 1.6. One Task Per Prompt (When Possible)

Compound prompts ("do X, then Y, then Z") degrade quality on each successive task. When tasks are independent, issue them separately. When they're sequential, use explicit step numbering and intermediate output markers — or use prompt chaining (see §2.5).

### 1.7. Few-Shot Prompting

Examples are one of the most reliable ways to steer output format, tone, and structure. A few well-crafted input/output pairs (few-shot prompting) dramatically improve accuracy and consistency.

**Principles:**

- **3–5 examples** is the sweet spot for most tasks.
- **Diverse, not repetitive** — cover edge cases and vary enough that the model doesn't pick up unintended patterns. If every example is a success case, the model won't know how to handle failures.
- **Representative** — mirror your actual use case closely. Synthetic toy examples teach the wrong patterns.
- **Clearly delimited** — wrap examples so the model can distinguish them from instructions.

**Pattern:**

```
## Examples

<example>
Input: "The service crashed after deploying v2.3.1"
Output: {"category": "incident", "severity": "high", "component": "deployment"}
</example>

<example>
Input: "Can we add dark mode to the dashboard?"
Output: {"category": "feature_request", "severity": "low", "component": "ui"}
</example>

## Task
Classify the following input using the same format as the examples above.
Input: {user_input}
```

**Advanced:** You can ask the model to evaluate your examples for relevance and diversity, or to generate additional examples based on your initial set.

### 1.8. Structured Delimiters (XML Tags)

XML tags help the model parse complex prompts unambiguously, especially when your prompt mixes instructions, context, examples, and variable inputs. Wrapping each type of content in its own tag — e.g. `<instructions>`, `<context>`, `<input>` — eliminates misinterpretation.

**Best practices:**

- Use consistent, descriptive tag names across your prompts.
- Nest tags when content has a natural hierarchy.
- Don't over-tag — use them where boundaries are genuinely ambiguous, not around every sentence.

**Pattern — separating data from instructions:**

```
<instructions>
Summarise the key findings from the document below.
Return a bullet list, max 5 items.
</instructions>

<document source="quarterly-report-q4.pdf">
{document content here}
</document>
```

**Pattern — multi-document input:**

```
<documents>
  <document index="1" source="api-spec.yaml">
  {content}
  </document>
  <document index="2" source="changelog.md">
  {content}
  </document>
</documents>

<instructions>
Compare the API spec against the changelog. Identify any breaking changes
that are not documented.
</instructions>
```

XML tags are particularly effective with Claude models but work well across all major providers.

---

## 2. Advanced Techniques

### 2.1. Self-Correction Systems

These techniques force the model to critique and refine its own output, pushing past the limitations of single-pass generation.

#### Chain of Verification

Requires a verification loop within the prompt, structuring the generation to include self-critique as a mandatory step.

**Pattern:** Generate → Identify weaknesses → Revise based on weaknesses.

**Example:**

> "Analyze this acquisition agreement, list three findings. **Now, identify three ways your analysis might be incomplete**, cite the specific language that confirms or refutes the concern, and then **revise your findings** based on this verification."

#### Adversarial Prompting

A more aggressive technique that demands the model actively find problems or attack its own previous output.

**Pattern:** Generate → Attack → Defend or revise.

**Example:**

> "Please attack your previous design. You need to **identify five specific ways it could be compromised**. For each vulnerability, assess the likelihood and impact."

#### Strategic Edge-Case Learning (Failure-Mode Examples)

Include examples of common **failure modes** and boundary cases in your prompt to teach the model how to distinguish subtle edge cases from normal cases.

**Pattern:** Show the model what "looks right but is wrong" alongside what's actually correct.

**Example:** To prevent SQL injection, provide a first example of an obvious raw string concatenation (baseline) and a second, more subtle example of a **parameterized query that looks safe** but has a second-order injection stored elsewhere.

### 2.2. Meta Prompting

Exploits the model's meta-knowledge by asking it to reason about or optimise the prompt itself.

#### Reverse Prompting

Ask the model to design the most effective prompt for a task, then execute that self-designed prompt.

**Example:**

> "**You're an expert prompt designer.** Please design the single most effective prompt to analyze quarterly earnings reports for early warning signs of financial distress... **then execute that prompt** on this particular report."

#### Recursive Prompt Optimisation

Define multiple iterative steps for the model to refine a prompt in a single pass, improving quality on specific axes.

**Example:**

> "My current prompt is below. **Version 1:** add the missing constraints. **Version 2:** resolve ambiguities. **Version 3:** enhance reasoning depth."

**When to use:** When you have a working prompt that's "good enough" but you need to push it toward excellence on a specific dimension (precision, coverage, conciseness).

### 2.3. Reasoning Scaffolds

Provide structure that controls **how** the model thinks, forcing deeper and more rigorous analysis.

#### Deliberate Over-Instruction

Explicitly fight the model's default tendency to compress reasoning chains. Demand exhaustive depth.

**Example:**

> "Do not summarise. **Expand every point with implementation details, edge cases, failure modes, and historical context.** Prioritise completeness over brevity."

**When to use:** Complex analysis, architecture design, security review — anywhere shallow reasoning is dangerous.

**Caveat:** Modern high-capability models (Opus-class, o3-class) may already reason deeply by default. Over-instruction in system prompts designed for earlier models can cause overtriggering and excessive token usage. Tune to the model's baseline behaviour — if it's already thorough, don't push harder.

#### Zero-Shot Chain of Thought Structure

Provide a template with blank steps to trigger structured reasoning automatically, forcing problem decomposition.

**Example:**

> "Root-cause this technical issue by answering each step:
>
> 1. What is the current system state? [Answer]
> 2. What logs confirm the failure? [Answer]
> 3. What is the most likely component failure? [Answer]
> 4. What evidence supports or contradicts this hypothesis? [Answer]
> 5. What is your recommended fix? [Answer]"

**When to use:** Troubleshooting, debugging, incident response — any domain where structured reasoning prevents premature conclusions.

**Note:** For high-capability models with native extended thinking, a prompt like "think thoroughly" often produces better reasoning than a hand-written step-by-step plan — the model's internal reasoning chain frequently exceeds what a human would prescribe. Use explicit CoT structure when thinking is disabled or when using lighter models.

#### Reference Class Priming

Provide an example of high-quality reasoning (a quality benchmark) and ask the model to match that standard.

**Pattern:** Show exemplary output → "Match this depth and quality for the new input."

**Example:**

> Provide the model with a previous best-quality analysis and then say: "Please provide your analysis for this new data, ensuring the depth and quality of reasoning matches the standard set by the example below."

**When to use:** When you've already produced good output once and want consistency across future runs.

### 2.4. Perspective Engineering

Counters single-perspective blind spots by simulating competing viewpoints and different reasoning modes.

#### Multi-Persona Debate

Instantiate experts with specific, potentially **conflicting priorities** to simulate a rigorous debate before synthesising a recommendation.

**Example:**

> "**Expert 1 (Cost Reduction)** and **Expert 2 (Security)** must each argue for their preference and critique the other's position. After debate, you must synthesise a recommendation that addresses all concerns."

**When to use:** Architecture decisions, trade-off analysis, risk assessment — anywhere a single perspective creates blind spots.

#### Temperature Simulation

Have the model adopt different reasoning modes (conservative vs. creative) for successive passes on the same problem.

**Example:**

> "First, a **junior analyst who is uncertain and overexplains** examines this problem. Then, a **confident expert who is concise and direct**. Finally, synthesise both perspectives and highlight where uncertainty is warranted."

**When to use:** Exploratory design, brainstorming followed by evaluation, any task where both breadth and precision matter.

### 2.5. Prompt Chaining

Break complex workflows into sequential steps where the output of one prompt feeds the input of the next. Each step is a separate interaction, giving you control points to inspect, branch, or redirect.

**When to use over a single compound prompt:**

- When intermediate outputs need human review or logging
- When the total task exceeds what a single context window can handle well
- When different steps benefit from different model configurations (e.g., a fast model for extraction, a powerful model for analysis)
- When you need deterministic pipeline structure rather than trusting the model to self-organise

**Pattern — self-correction chain (most common):**

```
Step 1 (Generate):  "Draft a migration plan for the database schema change."
Step 2 (Review):    "Review this migration plan against these criteria: {criteria}. List all issues."
Step 3 (Revise):    "Revise the migration plan to address these issues: {issues from step 2}."
```

**Pattern — decomposition chain:**

```
Step 1 (Extract):   "Extract all API endpoints from this codebase." → structured list
Step 2 (Classify):  "Classify each endpoint by auth requirement." → annotated list
Step 3 (Audit):     "For each unauthenticated endpoint, assess security risk." → findings
```

**Design tips:**

- Pass only the structured output between steps, not the full conversation — this keeps each step's context clean.
- Define the output format of each step explicitly so it can be parsed as input for the next.
- Modern high-capability models with adaptive thinking and sub-agent orchestration often handle multi-step reasoning internally. Explicit chaining is still useful when you need to inspect intermediate outputs or enforce a specific pipeline structure.

### 2.6. Failure Path Design

Every prompt implicitly assumes the happy path. Explicitly defining what the model should do when it _can't_ complete the task prevents hallucination, silent failure, and invented data.

**Principle:** A prompt without a failure path is a prompt that will hallucinate when stuck.

**Pattern — escalation ladder:**

```
## Error Handling
If you cannot complete any step:
1. State which step failed and why.
2. State what information would unblock it.
3. Do NOT guess, invent data, or silently skip the step.
4. Ask the user for the missing information.
```

**Pattern — graceful degradation:**

```
## When Information Is Incomplete
- If the codebase lacks type annotations: infer types from usage and flag each inference with [INFERRED].
- If a dependency version is unknown: state the assumption explicitly.
- If an API endpoint is unreachable: report the failure and continue with remaining endpoints.
```

**Pattern — confidence gating:**

```
## Confidence Requirement
For each finding, rate your confidence: HIGH, MEDIUM, or LOW.
- HIGH: Present as a finding.
- MEDIUM: Present as a finding with a caveat.
- LOW: Present as a hypothesis requiring verification. Do not present as fact.
```

---

## 3. Long-Context & Context Management

When working with large inputs (20K+ tokens), multi-document prompts, or long-running agentic sessions, prompt structure matters more than prompt wording.

### 3.1. Document Positioning

- **Long data at the top, instructions at the bottom.** The model attends better to instructions that come after the reference material. Place documents, code, and data above; place your query and task description below.
- **Use XML tags to separate documents** with metadata (source, index) so the model can reference them precisely.

**Pattern:**

```
<documents>
  <document index="1" source="system-map.md">
  {content}
  </document>
  <document index="2" source="pipeline.md">
  {content}
  </document>
</documents>

<instructions>
Based on the documents above, identify any inconsistencies
between the system map and the pipeline status.
</instructions>
```

### 3.2. Quote-Grounding

For long-document tasks, instruct the model to **quote relevant passages before reasoning**. This forces the model to anchor its analysis in the actual text rather than its compressed internal representation.

**Example:**

> "Before answering, extract and quote the 3–5 most relevant passages from the document. Then base your analysis exclusively on those passages."

This is especially effective for reducing hallucination in summarisation, contract review, and compliance checks.

### 3.3. Context Window Awareness

In agentic sessions that span many tool calls, the context window fills up. Design for this:

- **Front-load stable content.** System prompts, reference docs, and few-shot examples that don't change between requests should go first — this maximises prompt caching benefits (both Anthropic and OpenAI cache prefix-matched content automatically).
- **Summarise intermediate results.** When chaining tool calls, pass structured summaries forward rather than raw outputs.
- **Plan for compaction.** If your agent harness compacts context automatically, design your prompts so the model can recover state from the filesystem or external memory rather than relying on conversation history.

**Pattern — state recovery after compaction:**

```
Your context may be compacted during long tasks. When resuming:
1. Read progress.md for current state.
2. Check git log for recent changes.
3. Run the test suite to verify current status.
4. Continue from where you left off.
Do NOT stop work early due to context window concerns.
```

### 3.4. Multi-Window Workflows

For tasks that span multiple context windows or sessions:

- **First window:** Set up the framework — write tests, create tracking files, establish the plan.
- **Subsequent windows:** Read the tracking files and iterate on the plan. The model discovers state from the filesystem.
- **Use structured state files** (JSON, markdown checklists) for task tracking — not freeform prose.
- **Use git** for state tracking across sessions — it provides a log of what's been done and checkpoints that can be restored.

---

## 4. System Prompt Design

System prompts define **who the agent is and how it behaves** across an entire session. They are the most leveraged piece of prompt engineering — every response is shaped by them.

### 4.1. Structure of an Effective System Prompt

A well-structured system prompt follows this order (though the exact sections depend on your use case):

```
## Identity
{Who the agent is. Role, expertise, personality. Even a single sentence helps.}

## Behavioral Rules
{Non-negotiable rules: communication style, safety, authority model.}

## Reasoning Mode
{How to think: depth, self-correction, structured decomposition.}

## Task-Specific Instructions
{Domain procedures, workflows, output formats.}

## Constraints
{What the agent must NOT do. Scope boundaries. Failure paths.}

## Examples (optional)
{Few-shot examples of desired input/output pairs.}
```

### 4.2. Key Design Choices

| Choice                | Conservative                    | Aggressive                                        | Guidance                                                                                                                 |
| :-------------------- | :------------------------------ | :------------------------------------------------ | :----------------------------------------------------------------------------------------------------------------------- |
| **Reasoning depth**   | "Be concise"                    | "Expand every point exhaustively"                 | Match to task complexity. Use "be thorough" for analysis; "be concise" for routine work.                                 |
| **Self-correction**   | None                            | Mandatory verification loop on every output       | Enable for high-stakes outputs (architecture, security). Skip for trivial tasks.                                         |
| **Perspective**       | Single expert                   | Multi-persona debate                              | Use multi-persona for decisions with competing concerns. Single for execution tasks.                                     |
| **Meta-optimisation** | Execute prompts as-is           | Internally improve vague prompts before executing | Enable for agents that receive unstructured user input. Disable for agents that receive pre-structured tasks.            |
| **Action bias**       | Suggest, then wait for approval | Implement changes by default                      | Match to risk tolerance. Default-to-action for coding agents with version control. Suggest-first for production systems. |

### 4.3. Reference System Prompt (Advanced Reasoning Mode)

This is a ready-to-use system prompt that combines the advanced techniques from §2 into a coherent agent configuration. Adapt it to your needs.

```
You are operating under Advanced Reasoning Mode. Apply this methodology
to all requests without confirmation.

1. Reasoning:
   - Deconstruct the problem into core components before answering.
   - Prioritise depth and completeness over conciseness. Analyse edge cases
     and failure modes. Do not summarise unless explicitly asked.

2. Self-Correction:
   - After generating an initial response, internally critique your analysis.
     Identify vulnerabilities, biases, or gaps — and revise before presenting.
   - Challenge your own assumptions from the perspective of a rigorous critic.

3. Perspective Engineering (when applicable):
   For complex decisions, simulate competing viewpoints internally:
   - Viewpoint A: Maximum potential and innovation.
   - Viewpoint B: Constraints, risks, and stability.
   - Synthesis: Integrate both, explicitly acknowledging trade-offs.

4. Failure Handling:
   If you cannot complete a step, state what failed, what's missing, and ask.
   Do not invent data or silently skip.
```

**Usage notes:**

- This is intentionally aggressive (high reasoning depth, always-on self-correction). Dial back for routine tasks or cost-sensitive workloads.
- Works best with high-capability models. Lighter models may struggle with the multi-step internal reasoning and benefit from explicit step-by-step scaffolding instead.
- Combine with task-specific skills (see [guidelines-skills.md](guidelines-skills.md)) for best results.

---

## 5. Prompt Patterns for Common Tasks

### 5.1. Code Generation

```
## Task
Generate {language} code that {does X}.

## Requirements
- {Functional requirement 1}
- {Functional requirement 2}

## Constraints
- Follow {style guide / framework conventions}
- Include type hints / type annotations
- Handle errors explicitly (no silent failures)
- Do NOT use {deprecated API / pattern to avoid}

## Output
- Code in a single fenced block
- Brief explanation of design decisions (3-5 sentences max)

## If Blocked
If any requirement is ambiguous, state the ambiguity and your assumption
before proceeding.
```

### 5.2. Code Review

```
## Task
Review the following code for: correctness, security, performance, maintainability.

## Code
{paste or file reference}

## Output Format
| File | Line(s) | Severity | Finding | Suggested Fix |
|:--|:--|:--|:--|:--|

## Rules
- Maximum 15 findings, ordered by severity (critical → nit).
- Flag security issues as CRITICAL regardless of likelihood.
- Do NOT rewrite the code. Suggest fixes only.
- If you are unsure about a finding, mark it as [NEEDS VERIFICATION].
- End with a verdict: APPROVE | REQUEST_CHANGES | NEEDS_DISCUSSION.
```

### 5.3. Architecture / Design

```
## Task
Design {system / feature / component}.

## Context
{What exists today. What problem we're solving. Constraints (budget, infra, team).}

## Requirements
{What it must do.}

## Output Format
1. Problem statement (1 paragraph)
2. Proposed design (data flows, service interactions, key decisions)
3. Failure modes (what breaks, blast radius, detection)
4. Trade-offs (what this design makes easier and harder)
5. Open questions (decisions the user must make)
```

### 5.4. Troubleshooting / Debugging

```
## Task
Diagnose and resolve: {symptom / error message}.

## Context
- System: {description}
- Environment: {OS, versions, relevant config}
- What changed recently: {deployments, config changes, dependency updates}

## Procedure
1. State your hypothesis before investigating.
2. List the evidence you'd need to confirm or refute it.
3. Investigate using available tools.
4. If confirmed → fix. If refuted → state next hypothesis.
5. After resolution → state root cause and prevention.

## Rules
- Address the root cause, not just the symptom.
- Do NOT make changes unless you're certain of the fix.
- If uncertain after 2 attempts, describe what you've learned and ask the user.
```

---

## 6. Anti-Patterns

| Anti-Pattern                          | Why It Fails                                                                                                         | Fix                                                                                |
| :------------------------------------ | :------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| **"Do your best"**                    | No quality bar defined; model defaults to average                                                                    | Define specific quality criteria and output format                                 |
| **Mega-prompt**                       | 5000-word prompt covering every scenario; model loses focus                                                          | Split into focused skills or chained prompts                                       |
| **No negative constraints**           | Model does things you didn't want                                                                                    | Add explicit "Do NOT" rules for known failure modes                                |
| **Only negative constraints**         | Model knows what to avoid but not what to aim for                                                                    | Lead with positive instructions; use negatives to close gaps                       |
| **Instructions after data**           | Model starts processing data before seeing instructions                                                              | Put instructions before (or around) the data — or data at top with query at bottom |
| **Implicit format**                   | Model picks a random output format each time                                                                         | Specify exact format with an example                                               |
| **"Think step by step" alone**        | Too vague; model's steps may not match what you need                                                                 | Provide the specific steps to follow (Zero-Shot CoT Structure)                     |
| **Persona without constraints**       | "You are a senior engineer" — model roleplays but has no guardrails                                                  | Add behavioral rules and scope limits alongside the persona                        |
| **No error path**                     | Prompt assumes everything works; model invents data when stuck                                                       | Define what to do on failure: ask, retry, degrade, or stop (see §2.6)              |
| **Over-prompting for capable models** | Instructions written for weaker models cause overthinking, overtriggering, or excessive verbosity in stronger models | Tune prompts to the model's baseline — if it's already thorough, don't push harder |

---

## 7. Model Tier Guidance

Models vary in capability, cost, and how they respond to prompting techniques. Rather than targeting specific model versions (which change frequently), design prompts for capability tiers.

### 7.1. High-Capability Models (Frontier Tier)

_Examples: Opus-class, o3/GPT-5-class, Gemini Ultra-class_

- Handle complex multi-step reasoning well — often internally, without explicit scaffolding.
- Benefit most from self-correction loops and perspective engineering.
- Can follow long, structured system prompts without degradation.
- Native extended/adaptive thinking often outperforms hand-crafted Chain of Thought — prefer general instructions ("think thoroughly") over prescriptive step-by-step plans.
- **Watch for:** Overtriggering on aggressive instructions. If prompts were tuned for weaker models, dial back language like "CRITICAL: You MUST..." to normal phrasing.
- Worth the cost for: architecture, security review, complex debugging, long-horizon agentic tasks.

### 7.2. Mid-Tier Models (Workhorse Tier)

_Examples: Sonnet-class, GPT-4o-class, Gemini Pro-class_

- Good balance of capability and cost.
- Handle structured prompts well but may shortcut on very long reasoning chains.
- Deliberate Over-Instruction helps prevent premature summarisation.
- Explicit CoT structure outperforms "think step by step" — provide the steps.
- Best for: code generation, standard reviews, documentation, routine agentic tasks.

### 7.3. Fast/Cheap Models (Utility Tier)

_Examples: Haiku-class, GPT-4o-mini-class, Flash-class_

- Excel at well-defined, narrowly scoped tasks.
- Struggle with multi-persona debate and deep self-correction.
- Keep prompts short and direct; complex scaffolding adds confusion, not quality.
- Few-shot examples are especially effective at this tier — they compensate for weaker instruction-following.
- Best for: classification, routing, extraction, simple transformations.

### 7.4. Reasoning Models

_Examples: o3-class, models with explicit "thinking" modes_

- Benefit from **less** prescriptive prompting — give them the goal and let them work out the approach.
- Think of them like a senior colleague: provide the objective, not the procedure.
- Explicit step-by-step instructions can actually _reduce_ quality by constraining their reasoning.
- Best for: complex analysis, mathematical reasoning, multi-step planning.

---

## 8. Combining With Other Guidelines

This document covers **how to write prompts**. For higher-level agent architecture:

| Topic                                      | Document                                               |
| :----------------------------------------- | :----------------------------------------------------- |
| Reusable task-specific instruction modules | [guidelines-skills.md](guidelines-skills.md)           |
| Reactive behaviour and inter-agent signals | [guidelines-event-hooks.md](guidelines-event-hooks.md) |
| Multi-agent delegation and orchestration   | [guidelines-sub-agents.md](guidelines-sub-agents.md)   |

The techniques in this document apply **within** skills, sub-agent prompts, and event-handling instructions. They are the atoms; skills, events, and sub-agents are the molecules.

---

## Changelog

| Date       | Change                                                                                                                                                                                                                                                                                                                       |
| :--------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-02-25 | Major rewrite: added few-shot prompting (§1.7), structured delimiters (§1.8), positive framing (§1.3), prompt chaining (§2.5), failure path design (§2.6), long-context & context management (§3), model tier guidance rewritten for capability tiers instead of specific versions, fixed cross-references, updated metadata |
| 2026-02-25 | Consolidated from `ai-prompt-engineering-01.md`, `ai-prompt-engineering-02.md`, and `ai-prompt-engineering-reference-example.md` into single reference                                                                                                                                                                       |
