# Guidelines: AI Sub-Agents

**Category:** Prompt Engineering
**Last Updated:** 2026-02-25

---

## Overview

Sub-agents are **specialized AI agents that operate under the coordination of a parent agent or orchestrator** to handle distinct portions of a complex task. Instead of one monolithic agent doing everything, work is decomposed and delegated — each sub-agent brings focused instructions, scoped tools, and domain-specific constraints to its piece of the problem.

This guide covers the concept, architecture patterns, delegation strategies, and practical implementation of sub-agent systems across platforms (Anthropic Claude, OpenAI Agents SDK, IDE assistants, custom orchestrations). It is project-agnostic.

---

## 1. What Is a Sub-Agent?

A sub-agent is an **AI agent instance with a narrower scope than the parent**, created or invoked to handle a specific category of work. The parent retains overall coordination; sub-agents own execution within their domain.

| Aspect        | Description                                                                              |
| :------------ | :--------------------------------------------------------------------------------------- |
| **Identity**  | Has its own system prompt, instructions, and persona                                     |
| **Scope**     | Handles one domain or task type (e.g., "code review," "infrastructure," "data analysis") |
| **Tools**     | Has access only to the tools it needs — not the full tool set                            |
| **Lifecycle** | Created/invoked by a parent agent or orchestrator; returns results upstream              |
| **Authority** | Operates within boundaries set by the parent; escalates when out of scope                |

### Sub-Agents vs. Skills vs. Tools

| Concept       | What it is                    | Has its own prompt?                          | Makes decisions?                              |
| :------------ | :---------------------------- | :------------------------------------------- | :-------------------------------------------- |
| **Tool**      | A single callable function    | No                                           | No — it executes and returns                  |
| **Skill**     | A reusable instruction module | Partial — instructions within parent context | Parent decides; skill guides execution        |
| **Sub-agent** | A fully scoped agent instance | Yes — its own system prompt and identity     | Yes — makes autonomous decisions within scope |

**Key distinction:** A skill is a recipe the parent follows. A sub-agent is a delegate that thinks for itself within defined boundaries.

---

## 2. When to Use Sub-Agents

### 2.1. Use Sub-Agents When

- **Tasks require different expertise.** A code review requires different reasoning than infrastructure debugging. Specialized prompts outperform generalist ones.
- **Context windows are a constraint.** Each sub-agent starts with a focused context rather than inheriting the full (potentially bloated) conversation history.
- **Tool isolation matters.** A security-review agent shouldn't have access to deployment tools. Sub-agents enable least-privilege tool access.
- **Parallel work is possible.** Independent subtasks can run concurrently as separate sub-agents.
- **Failure isolation is important.** If a sub-agent fails or loops, it doesn't corrupt the parent's state.

### 2.2. Do NOT Use Sub-Agents When

- **A single prompt can handle it.** Don't add architectural complexity for simple tasks. A well-written skill within the main agent is often enough.
- **The task is inherently sequential and tightly coupled.** If every step depends on the full output of the previous step, chaining through a single agent (with skills) is simpler.
- **You can't define clear boundaries.** If you can't articulate where one sub-agent's responsibility ends and another's begins, you'll get gaps and overlaps.
- **Cost or latency is critical.** Each sub-agent invocation is a separate API call (or conversation). This adds cost and latency. Optimize only when the quality gain justifies it.

### 2.3. Decision Framework

```
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

## 3. Architecture Patterns

### 3.1. Orchestrator-Workers

A central orchestrator agent analyzes the task, decomposes it, delegates to worker sub-agents, and synthesizes their results.

```
                    ┌─────────────┐
                    │ Orchestrator │
                    └──────┬──────┘
                           │ decomposes task
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Worker A │ │ Worker B │ │ Worker C │
        │ (Review) │ │ (Test)   │ │ (Deploy) │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │             │             │
             └─────────────┴─────────────┘
                           │ results
                    ┌──────┴──────┐
                    │ Orchestrator │
                    │ (synthesize) │
                    └─────────────┘
```

**When to use:** The number and nature of subtasks can't be predicted in advance (e.g., "fix all failing tests" — the orchestrator must first discover what's failing, then delegate fixes).

**Characteristics:**

- Orchestrator decides how many workers to spawn and what each does
- Workers are independent — they don't communicate with each other
- Orchestrator owns the final synthesis and quality check

### 3.2. Pipeline (Sequential Handoff)

Each sub-agent handles one stage, then hands off to the next. Like an assembly line.

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Analyst  │ ──▶│ Designer │ ──▶│ Engineer │ ──▶│ Reviewer │
│ (scope)  │    │ (design) │    │ (build)  │    │ (review) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

**When to use:** Each stage has a well-defined input/output contract, and stages require genuinely different agent configurations (different system prompts, tools, constraints).

**Characteristics:**

- Each agent transforms its input and produces output for the next
- Failure at any stage halts the pipeline (or triggers a retry/escalation)
- Clear interfaces between stages enable independent testing

### 3.3. Router (Classification → Delegation)

A lightweight router agent classifies the incoming task and delegates to the appropriate specialist sub-agent.

```
                    ┌────────┐
         input ──▶  │ Router │
                    └───┬────┘
                        │ classifies
              ┌─────────┼─────────┐
              ▼         ▼         ▼
        ┌──────────┐ ┌────────┐ ┌──────────┐
        │ Billing  │ │ Tech   │ │ General  │
        │ Agent    │ │ Support│ │ FAQ      │
        └──────────┘ └────────┘ └──────────┘
```

**When to use:** The task type determines which specialist should handle it, and specialists have meaningfully different configurations.

**Characteristics:**

- Router is simple and fast (can be a smaller/cheaper model)
- Specialists are deep and focused
- Misrouting is the primary failure mode — invest in router accuracy

### 3.4. Evaluator-Optimizer (Iterative Refinement)

One sub-agent generates output; another evaluates it. They loop until quality is met.

```
        ┌───────────┐         ┌───────────┐
        │ Generator │ ──────▶ │ Evaluator │
        │ (produce) │ ◀────── │ (critique) │
        └───────────┘         └───────────┘
              │                      │
              │    (iterate until     │
              │     quality met)      │
              │                      │
              └──── final output ────┘
```

**When to use:** Quality criteria are clear and articulable, and iterative refinement demonstrably improves output (e.g., writing, code generation, translation).

**Characteristics:**

- Generator and evaluator have different system prompts (one creates, one critiques)
- Must have a maximum iteration limit to prevent infinite loops
- Evaluator should provide actionable feedback, not just pass/fail

### 3.5. Parallel Sectioning

The task is split into independent sections; sub-agents process them simultaneously.

```
                    ┌─────────────┐
                    │ Coordinator │
                    └──────┬──────┘
                           │ splits task
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Section A│ │ Section B│ │ Section C│
        │ (files   │ │ (files   │ │ (files   │
        │  1-10)   │ │  11-20)  │ │  21-30)  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             └─────────────┴─────────────┘
                           │ merge
                    ┌──────┴──────┐
                    │ Coordinator │
                    └─────────────┘
```

**When to use:** Work is naturally partitionable (processing multiple files, analyzing different data sources, checking different aspects of a system).

**Characteristics:**

- All sub-agents use the same prompt/configuration but different inputs
- Coordination cost is low (just split and merge)
- Quality depends on the merge step handling conflicts/overlaps

---

## 4. Design Principles

### 4.1. Clear Boundaries

Every sub-agent must have a **crisp definition of its scope**. If you can't write one sentence describing exactly what a sub-agent does and doesn't do, the boundary is wrong.

**Good:** "This agent reviews Python code for security vulnerabilities. It does not fix them — it reports findings."

**Bad:** "This agent handles code stuff."

### 4.2. Minimal Authority (Least Privilege)

Each sub-agent should have access to **only the tools it needs** for its specific task. A review agent doesn't need write access. A test runner doesn't need deployment credentials.

| Sub-Agent        | Tools it SHOULD have                                | Tools it should NOT have               |
| :--------------- | :-------------------------------------------------- | :------------------------------------- |
| Code Reviewer    | `read_file`, `grep`, `diagnostics`                  | `edit_file`, `terminal`, `deploy`      |
| Test Runner      | `terminal` (scoped to test commands), `read_file`   | `edit_file`, `deploy`, `delete`        |
| Deployment Agent | `terminal` (scoped to deploy commands), `read_file` | `edit_file` (it deploys, not modifies) |

### 4.3. Explicit Contracts

Define what each sub-agent **receives** (input contract) and what it **produces** (output contract):

```
## Sub-Agent: Code Reviewer

### Input Contract
- File paths or diff to review
- Coding standards document (reference)
- Scope: what to focus on (security, performance, style, all)

### Output Contract
- Markdown table: | File | Line | Severity | Finding | Suggested Fix |
- Summary verdict: APPROVE | REQUEST_CHANGES | NEEDS_DISCUSSION
- Maximum 15 findings, ordered by severity
```

When the output of Agent A is the input of Agent B, these contracts must align — treat this as an API boundary.

### 4.4. Fail Gracefully

Sub-agents will fail. Design for it:

- **Timeout:** Set maximum execution time. If exceeded, return partial results + "timed out" status.
- **Max iterations:** For evaluator-optimizer loops, cap iterations (typically 2-4).
- **Error propagation:** Sub-agent errors should surface to the parent with enough context to decide: retry, skip, or escalate.
- **Fallback:** If a specialist sub-agent fails, can the parent handle the task (at lower quality) itself?

### 4.5. Preserve Context Across Handoffs

When one sub-agent hands off to another, critical context can be lost. Explicitly pass:

- **What was done** (summary of completed work)
- **What to do next** (specific instructions, not vague pointers)
- **What to watch out for** (known issues, edge cases encountered)
- **Artifacts produced** (file paths, state changes, decisions made)

**Bad handoff:** "I finished the design. Over to you."
**Good handoff:**

```
## Handoff: Design → Engineering

### Completed
- Database schema designed: 3 tables (orders, line_items, audit_log)
- ADR written: meta/logs/architecture/adr-260225-order-schema.md
- Schema SQL: migrations/001_create_orders.sql

### Next Action
Implement the Order service per the schema above. Start with the write path (create order → validate → persist).

### Watch Out For
- The `amount` column uses NUMERIC(20,8) — use Decimal in Python, never float.
- Foreign key on `line_items.order_id` has ON DELETE CASCADE — test this explicitly.

### Files to Read First
- meta/blocks/block-03-01-orders.md (task breakdown)
- 42bros-common/docs/guidelines-code.md §6 (financial math rules)
```

### 4.6. Observability

Every sub-agent should produce **structured logs** of what it did, what it decided, and why. Without this, debugging multi-agent workflows is nearly impossible.

Minimum log entry per sub-agent invocation:

- Agent name and role
- Input received (summary, not full payload)
- Actions taken (tools called, decisions made)
- Output produced (summary)
- Duration
- Errors or warnings encountered

---

## 5. Implementation Patterns

### 5.1. Anthropic Claude (Prompt-Based Sub-Agents)

In Claude's ecosystem, sub-agents are implemented via **separate system prompts loaded as context files**. Since Claude doesn't natively support spawning sub-processes, the patterns are:

#### Agent Definitions as Markdown Files

Each agent is a markdown file with role, instructions, session procedures, and tool access:

```
project/
├── meta/agents/
│   ├── architect.md     ← System design, scoping, decision records
│   ├── engineer.md      ← Implementation, testing, deployment
│   ├── reviewer-code.md ← Code review against standards
│   └── analyst.md       ← Domain-specific evaluation
```

The **user selects** which agent to activate per session. The agent file functions as both the system prompt and the skill library.

#### Session Handover (Human-Mediated Sub-Agent Switching)

Since Claude sessions are single-agent, "sub-agent delegation" happens via structured handover files:

```
Session 1: Architect agent designs the system
    → Writes handover to session-handover.md
    → "Next: Engineer agent to implement per block-03-01.md"

Session 2: User activates Engineer agent
    → Engineer reads handover, picks up where Architect left off
```

The handover file is the event contract between agents. Structure it well (see §4.5).

#### Single-Session Multi-Persona

For simpler cases, one agent can adopt multiple personas sequentially within a single session:

```
## Instructions

When the user says "review mode," switch to Code Reviewer behavior:
- Read code, produce findings table, give verdict.
- Do NOT modify code.

When the user says "fix mode," switch to Engineer behavior:
- Read findings, implement fixes, run tests.
- Do NOT produce review findings.
```

This is lighter than separate agents but loses the isolation benefits.

### 5.2. OpenAI Agents SDK (Programmatic Sub-Agents)

OpenAI's Agents SDK provides native support for sub-agents via **handoffs** and **agents-as-tools**:

#### Handoffs

An agent can delegate to another agent mid-conversation:

```
# Conceptual — not runnable without SDK setup
from agents import Agent, handoff

billing_agent = Agent(
    name="Billing Agent",
    instructions="You handle billing inquiries...",
    tools=[lookup_invoice, process_refund],
)

tech_agent = Agent(
    name="Tech Support Agent",
    instructions="You handle technical issues...",
    tools=[check_status, restart_service],
)

triage_agent = Agent(
    name="Triage Agent",
    instructions="Classify the user's request and hand off to the appropriate specialist.",
    handoffs=[billing_agent, tech_agent],
)
```

When the triage agent decides the user has a billing question, it invokes `transfer_to_billing_agent`, and the billing agent takes over the conversation.

#### Agents as Tools

A parent agent can invoke a sub-agent as a tool — the sub-agent processes a focused subtask and returns the result to the parent (rather than taking over the conversation):

```
# Conceptual
code_analyzer = Agent(
    name="Code Analyzer",
    instructions="Analyze code for complexity and security issues. Return structured findings.",
    tools=[read_file, grep],
)

lead_engineer = Agent(
    name="Lead Engineer",
    instructions="You coordinate code changes. Use the code analyzer for review, then implement fixes.",
    tools=[code_analyzer.as_tool(), edit_file, terminal],
)
```

### 5.3. IDE-Based Multi-Agent Patterns

IDE assistants (Cursor, Windsurf, Zed) typically support multi-agent behavior via:

- **Agent definition files** (e.g., `AGENTS.md`, `.cursor/agents/`) that define specialized personas
- **Slash commands** that switch the active agent context
- **Rules files** that scope behavior per directory or file type
- **MCP servers** that provide tool access to specific agents

Pattern: Define agents in config, let the user (or a router rule) select which agent is active for a given task.

### 5.4. Custom Orchestration (DIY)

For systems that manage their own agent calls (via API):

```
# Conceptual orchestration loop
def orchestrate(task):
    # 1. Decompose
    subtasks = call_agent(decomposer_prompt, task)

    # 2. Delegate
    results = []
    for subtask in subtasks:
        agent_config = select_agent(subtask.type)
        result = call_agent(agent_config.prompt, subtask, tools=agent_config.tools)
        results.append(result)

    # 3. Synthesize
    final = call_agent(synthesizer_prompt, results)
    return final
```

This gives maximum control but requires you to manage:

- Agent selection logic
- Context passing between agents
- Error handling and retries
- Cost and latency optimization

---

## 6. Communication Between Sub-Agents

Sub-agents need to exchange information. The patterns, ranked from simplest to most complex:

### 6.1. Shared File/State (Simplest)

Sub-agents read from and write to shared files or a state store. No direct communication.

```
Agent A writes → shared-state.md → Agent B reads
```

**Pros:** Simple, auditable, works across sessions.
**Cons:** No real-time coordination; race conditions if agents run in parallel.

### 6.2. Parent-Mediated (Most Common)

Sub-agents communicate only through the parent/orchestrator. The parent passes relevant information between them.

```
Agent A → result → Parent → (extracts relevant parts) → Agent B
```

**Pros:** Parent controls information flow; prevents info overload.
**Cons:** Parent becomes a bottleneck; must understand both agents' domains well enough to mediate.

### 6.3. Event-Based (Most Scalable)

Sub-agents emit and consume structured events. An event bus (or simple file/queue) routes them.

```
Agent A emits → { event: "analysis.complete", findings: [...] }
Event bus routes to → Agent B (subscribed to "analysis.complete")
Agent B processes findings
```

**Pros:** Fully decoupled; scales to many agents.
**Cons:** Requires event infrastructure; harder to debug; eventual consistency.

See [guidelines-event-hooks.md](guidelines-event-hooks.md) for event design details.

### 6.4. Direct Handoff (OpenAI Agents SDK)

One agent transfers the conversation directly to another. The receiving agent gets the full (or filtered) conversation history.

**Pros:** Seamless user experience; receiving agent has full context.
**Cons:** Context can be overwhelming; harder to scope; platform-specific.

---

## 7. Prompt Engineering for Sub-Agents

### 7.1. The Parent/Orchestrator Prompt

The orchestrator needs to know:

```
## Orchestrator Instructions

You coordinate work across specialized agents. Your responsibilities:

1. Analyze the user's request to determine which agent(s) are needed.
2. Decompose complex tasks into subtasks with clear boundaries.
3. Delegate each subtask to the appropriate agent.
4. Review results from sub-agents for completeness and correctness.
5. Synthesize final output from sub-agent results.
6. Escalate to the user when sub-agents disagree or fail.

### Available Agents
| Agent | Handles | Tools |
|:--|:--|:--|
| Code Reviewer | Security, style, correctness review | read_file, grep |
| Engineer | Implementation, testing | edit_file, terminal |
| Infra Agent | Deployment, server config | terminal (ssh), terraform |

### Delegation Rules
- Never delegate to an agent outside its stated scope.
- Always include: what to do, what files to look at, what constraints apply.
- If unsure which agent to use, ask the user.
```

### 7.2. The Sub-Agent Prompt

Each sub-agent needs:

```
## Agent: {Name}

### Identity
You are {role}. You specialize in {domain}. You do NOT {out-of-scope activities}.

### Behavioral Rules
{Global rules inherited from the project — style, safety, communication norms.}

### Task Handling
When you receive a task:
1. Confirm you understand the scope. If anything is ambiguous, ask.
2. Execute using your tools, following your domain-specific procedures.
3. Produce output in the specified format.
4. Flag anything outside your scope — do not attempt it.

### Tools Available
{Only the tools this agent should use.}

### Output Format
{Exactly what the parent or next agent expects.}
```

### 7.3. Scoping Context for Sub-Agents

**Problem:** If you pass the entire conversation history to a sub-agent, it wastes context tokens and may confuse the agent with irrelevant information.

**Solution: Input filters.** Before delegating, the parent extracts only the relevant context:

| Strategy                | When to use                                                      |
| :---------------------- | :--------------------------------------------------------------- |
| **Summary handoff**     | Pass a structured summary, not raw history                       |
| **Relevant files only** | Only include files the sub-agent needs to read                   |
| **Filtered history**    | Remove tool calls and intermediate reasoning from the history    |
| **Clean slate**         | Sub-agent starts fresh with just the task description and inputs |

**Rule of thumb:** The less irrelevant context a sub-agent receives, the better it performs. Err on the side of giving too little context (the agent can ask for more) rather than too much (the agent can't un-see noise).

---

## 8. Common Pitfalls

| Pitfall                      | Symptom                                                                    | Fix                                                                                |
| :--------------------------- | :------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- |
| **Over-decomposition**       | Simple task split across 5 agents; high latency, no quality gain           | Use the decision framework (§2.3). Start with one agent + skills.                  |
| **Blurry boundaries**        | Two sub-agents both try to handle the same task, or neither does           | Sharpen scope definitions. Explicit "you do / you don't" lists.                    |
| **Context loss at handoffs** | Downstream agent asks questions already answered upstream                  | Structured handoffs with explicit context (§4.5)                                   |
| **Orchestrator bottleneck**  | Parent agent can't meaningfully evaluate sub-agent output                  | Keep orchestrator lightweight; trust sub-agents within their domain                |
| **Sub-agent sprawl**         | 15 sub-agents, most rarely used; maintenance burden grows                  | Consolidate related functions. A sub-agent must earn its existence.                |
| **Cascading failures**       | One sub-agent fails → parent retries endlessly → costs spiral              | Set retry limits (typically 1-2). Fail fast. Escalate to user.                     |
| **Lost observability**       | Multi-agent workflow fails, no one knows where or why                      | Require structured logs from every sub-agent (§4.6)                                |
| **Inconsistent behavior**    | Sub-agents contradict each other (Agent A says X, Agent B says not-X)      | Give sub-agents the same foundational rules; resolve conflicts at the orchestrator |
| **Privilege creep**          | Sub-agent gradually gets more tools "for convenience"                      | Audit tool access periodically. Least privilege is non-negotiable.                 |
| **Gold-plating delegation**  | Every task gets decomposed into sub-agents because "it's the architecture" | Only delegate when it demonstrably improves quality or enables parallelism         |

---

## 9. Cost and Performance Considerations

### 9.1. Token Economics

Every sub-agent invocation has a cost:

| Component                         | Tokens consumed                                |
| :-------------------------------- | :--------------------------------------------- |
| Sub-agent system prompt           | Paid on every invocation (cache when possible) |
| Input context passed to sub-agent | Proportional to context size                   |
| Sub-agent reasoning and output    | Output tokens (most expensive per token)       |
| Result passed back to parent      | Input tokens for the parent                    |

**Optimization strategies:**

- **Prompt caching:** Cache static system prompts and shared context
- **Minimal context:** Only pass what the sub-agent needs (§7.3)
- **Model selection:** Use smaller/cheaper models for simple sub-agents (routing, classification) and larger models for complex ones (architecture, code generation)
- **Parallel execution:** Run independent sub-agents concurrently to reduce wall-clock time (not token cost)

### 9.2. Latency Budget

```
Total latency = orchestrator_time + max(sub_agent_times) + synthesis_time
                                    ↑ if parallel
Total latency = orchestrator_time + sum(sub_agent_times) + synthesis_time
                                    ↑ if sequential
```

For user-facing applications, budget your latency and work backward:

- If you have 10 seconds, and orchestration + synthesis takes 4, each sub-agent gets ~6 seconds (parallel) or ~2 seconds each (3 sequential agents).

### 9.3. Model Tiering

Not every sub-agent needs the most capable model:

| Sub-Agent Role        | Model Tier                | Rationale                             |
| :-------------------- | :------------------------ | :------------------------------------ |
| Router / classifier   | Fast, cheap (Haiku-class) | Simple classification, speed matters  |
| Code reviewer         | Mid-tier (Sonnet-class)   | Needs reasoning but not maximum depth |
| Architecture designer | Top-tier (Opus-class)     | Complex multi-factor reasoning        |
| Log summarizer        | Fast, cheap               | Extractive task, low complexity       |
| Security auditor      | Top-tier                  | High stakes, needs thorough analysis  |

---

## 10. Testing Multi-Agent Systems

### 10.1. Unit Test Each Agent

Test each sub-agent in isolation with representative inputs:

- Does it follow its instructions?
- Does it stay within scope?
- Does it produce output matching its contract?
- Does it handle errors gracefully?

### 10.2. Integration Test the Workflow

Test the full orchestration with end-to-end scenarios:

- Does the orchestrator route correctly?
- Do handoffs preserve necessary context?
- Does the system produce correct final output?
- What happens when one sub-agent fails?

### 10.3. Adversarial Testing

- What if the user asks something that falls between two sub-agents' scopes?
- What if a sub-agent produces output that contradicts another's?
- What if the router misclassifies?
- What if a sub-agent tries to use a tool outside its scope?

### 10.4. Performance Testing

- Measure end-to-end latency across the full workflow.
- Measure token consumption per sub-agent.
- Identify the most expensive sub-agent and evaluate whether it can use a cheaper model.

---

## 11. Quick-Start Checklist

When designing a multi-agent system:

- [ ] Have I confirmed that a single agent with skills can't handle this adequately?
- [ ] Can I draw clear boundaries between each sub-agent's responsibilities?
- [ ] Does each sub-agent have a one-sentence purpose statement?
- [ ] Have I defined input and output contracts for each sub-agent?
- [ ] Is each sub-agent scoped to minimum necessary tools (least privilege)?
- [ ] Is there a clear escalation path when a sub-agent fails or is out of scope?
- [ ] Are handoffs between agents structured and explicit (not "over to you")?
- [ ] Does every sub-agent produce structured logs of its actions?
- [ ] Have I considered the cost and latency implications?
- [ ] Have I tested each sub-agent in isolation AND in the full workflow?
- [ ] Is there a maximum iteration/retry limit to prevent runaway costs?

---

## 12. Reference Architecture: Project Agent System

A concrete example of how sub-agents can be structured for a software project:

```
┌──────────────────────────────────────────────────────────┐
│                    User (human)                          │
│              Selects agent per session                    │
└──────────────────────┬───────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
  ┌───────────┐  ┌───────────┐  ┌───────────┐
  │ Architect │  │ Engineer  │  │ Reviewer  │
  │           │  │           │  │           │
  │ • Design  │  │ • Build   │  │ • Review  │
  │ • Scope   │  │ • Test    │  │ • Audit   │
  │ • Decide  │  │ • Deploy  │  │ • Grade   │
  │ • Record  │  │ • Document│  │ • Flag    │
  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
        │              │              │
        └──────────────┴──────────────┘
                       │
              Shared artifacts:
              • session-handover.md
              • block plans
              • architecture decision records
              • pipeline.md
```

**Key properties:**

- Each agent has its own markdown file defining role, instructions, and procedures
- Agents communicate via structured handover files, not direct conversation
- Shared documents (system-map, pipeline, block plans) serve as the common state
- The human selects which agent to activate; agents don't self-delegate
- Each agent reads the same foundational context (system overview, principles) but applies its own lens

---

## Cross-References

- [guidelines-skills.md](guidelines-skills.md) — Skills as the building blocks within each sub-agent
- [guidelines-event-hooks.md](guidelines-event-hooks.md) — Event-driven communication between sub-agents
- [guidelines-prompt-engineering.md](guidelines-prompt-engineering.md) — Advanced prompting techniques for agent instructions

---

## External Resources

- [Anthropic: Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Orchestrator-workers, evaluator-optimizer, and other patterns
- [OpenAI: Agents SDK — Handoffs](https://openai.github.io/openai-agents-python/handoffs/) — Native sub-agent delegation
- [OpenAI: Agents SDK — Multi-Agent](https://openai.github.io/openai-agents-python/multi_agent/) — Orchestrating multiple agents
- [OpenAI: Codex Multi-Agents](https://developers.openai.com/codex/multi-agent) — IDE-level multi-agent patterns

---

## Changelog

| Date       | Change           |
| :--------- | :--------------- |
| 2026-02-25 | Initial creation |
