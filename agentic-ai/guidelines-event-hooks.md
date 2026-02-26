# Guidelines: AI Agent Hooks/Events

**Category:** Prompt Engineering
**Last Updated:** 2026-02-25

---

## Overview

Events are **structured signals that drive agent behavior** — they represent things that happened, things that changed, or conditions that were met. In agentic AI systems, events serve the same role they serve in traditional software architecture: they decouple producers from consumers, enable reactive behavior, and make complex workflows observable.

This guide covers event concepts, design principles, implementation patterns, and anti-patterns for AI agent systems — from simple prompt-driven assistants to multi-agent orchestrations. It is project-agnostic.

---

## 1. What Is an Event (in an AI Agent Context)?

An event is a **discrete, structured notification that something occurred** which may require the agent to act, adapt, or record.

| Aspect       | Description                                                                       |
| :----------- | :-------------------------------------------------------------------------------- |
| **Trigger**  | Something happened: user input, tool result, external signal, timer, state change |
| **Payload**  | Structured data describing what happened (who, what, when, context)               |
| **Consumer** | The agent, sub-agent, or orchestrator that reacts to the event                    |
| **Outcome**  | Action taken, state updated, event forwarded, or event ignored                    |

### Events vs. Instructions vs. Tool Results

| Concept         | Direction                         | Nature                                               |
| :-------------- | :-------------------------------- | :--------------------------------------------------- |
| **Instruction** | Human → Agent                     | "Do this task" (imperative)                          |
| **Event**       | System → Agent (or Agent → Agent) | "This happened" (declarative)                        |
| **Tool result** | Tool → Agent                      | "Here's the output of what you asked for" (response) |

Events are **declarative** — they describe facts. The agent decides what to do about them. Instructions are **imperative** — they prescribe behavior. Tool results are **responses** to agent-initiated actions.

---

## 2. Why Events Matter for AI Agents

### 2.1. Reactive Behavior

Without events, agents are purely request-response: the user asks, the agent answers. Events enable agents to **react to changes** — a test failure, a deployment completion, a file modification, an upstream service publishing new data.

### 2.2. Decoupling

Events break hard dependencies between components. An agent that publishes "schema migration completed" doesn't need to know which downstream agents (or humans) care. Consumers subscribe to events they need; producers don't know or care who's listening.

### 2.3. Observability

Events create a natural audit trail. When every significant action and state change emits an event, you can reconstruct what happened, when, and why — invaluable for debugging agent behavior and understanding decision chains.

### 2.4. Composability

Events are the glue for multi-agent and multi-step workflows. Agent A completes a task and emits an event; Agent B picks it up and continues. No tight coupling, no direct calls — just events flowing through the system.

---

## 3. Event Taxonomy

### 3.1. By Origin

| Type                | Source                   | Examples                                                       |
| :------------------ | :----------------------- | :------------------------------------------------------------- |
| **User events**     | Human interaction        | New message, file attachment, approval, rejection              |
| **System events**   | Platform/infrastructure  | Session start, context window warning, timeout, rate limit     |
| **Tool events**     | Tool execution results   | Command output, API response, file read result, error          |
| **Agent events**    | Agent's own actions      | Task completed, decision made, handoff initiated, state saved  |
| **External events** | Outside the agent system | Webhook, cron trigger, CI/CD pipeline result, monitoring alert |

### 3.2. By Purpose

| Type                     | Purpose                                  | Agent Response                                        |
| :----------------------- | :--------------------------------------- | :---------------------------------------------------- |
| **Trigger events**       | Signal that work should begin            | Start a task, activate a skill                        |
| **State-change events**  | Report that something changed            | Update internal state, re-evaluate decisions          |
| **Completion events**    | Signal that work finished                | Move to next step, summarize results, clean up        |
| **Error events**         | Signal that something failed             | Retry, degrade gracefully, escalate to user           |
| **Informational events** | Provide context without requiring action | Log, acknowledge, optionally use for future decisions |

### 3.3. By Urgency

| Level        | Meaning                                         | Agent Behavior                         |
| :----------- | :---------------------------------------------- | :------------------------------------- |
| **Critical** | Requires immediate action; blocks progress      | Stop current work, handle immediately  |
| **Warning**  | Important but not blocking; may degrade quality | Acknowledge, factor into next decision |
| **Info**     | Contextual; no action required                  | Log, optionally display to user        |

---

## 4. Design Principles

### 4.1. Events Are Facts, Not Commands

An event says **"X happened."** It does not say "do Y." The consumer decides what to do.

**Good:** `{ "event": "tests_failed", "suite": "unit", "failures": 3 }`
**Bad:** `{ "event": "rerun_tests_and_fix_failures" }` ← This is an instruction disguised as an event.

### 4.2. Events Are Immutable

Once emitted, an event cannot be changed. If the situation changes, emit a **new** event — don't update the old one. This preserves the audit trail and prevents race conditions.

### 4.3. Events Carry Sufficient Context

The consumer should be able to act on the event **without querying the producer** for more information. Include:

- **What** happened (event type/category)
- **When** it happened (timestamp, always UTC)
- **Who/what** caused it (source identifier)
- **Where** it applies (scope — file, service, environment)
- **Key data** needed to act (IDs, values, paths)

### 4.4. Events Have a Schema

Define a consistent structure. Ad-hoc event formats lead to parsing errors and missed events.

```
## Standard Event Schema

| Field | Type | Required | Description |
|:--|:--|:--|:--|
| `event_type` | string (enum) | Yes | Category of the event |
| `source` | string | Yes | Who/what emitted it |
| `timestamp` | ISO 8601 UTC | Yes | When it happened |
| `severity` | critical / warning / info | Yes | Urgency level |
| `summary` | string | Yes | Human-readable one-liner |
| `payload` | object | No | Structured data specific to this event type |
| `correlation_id` | string | No | Links related events across a workflow |
```

### 4.5. Events Are Typed

Use an explicit, finite set of event types. An `event_type` enum prevents typos, enables routing, and makes the system enumerable.

```
## Example Event Types (software project context)

- `session.started`
- `session.ended`
- `task.assigned`
- `task.completed`
- `task.failed`
- `code.lint_failed`
- `code.tests_passed`
- `code.tests_failed`
- `deploy.started`
- `deploy.succeeded`
- `deploy.failed`
- `review.requested`
- `review.approved`
- `review.changes_requested`
- `alert.infra`
- `alert.security`
- `user.approval_required`
- `user.input_required`
```

### 4.6. Minimize Event Fanout

Not every micro-action needs an event. Emit events at **meaningful boundaries** — task completion, state changes, errors, decisions. Over-eventing drowns the signal in noise.

---

## 5. Implementation Patterns

### 5.1. Event-Triggered, State-Evaluated

This is one of the most powerful patterns for AI agent systems. Any event triggers the agent to **re-evaluate current state** — not just the event payload. The event is a nudge; the state is the truth.

**Why this works:**

- Events can arrive out of order or with delays
- State captures the full picture; events capture moments
- The agent catches valid conditions even when multiple events align with a delay

**Pattern:**

```
On ANY relevant event:
  1. Read current state (from state store, files, or context)
  2. Evaluate conditions against current state (not event payload)
  3. If conditions met → act
  4. If conditions not met → log, wait for next event
```

**Example (multi-signal trading system):**

```
On event (technical_trigger OR regime_change OR structure_update):
  1. Read current indicators from state store
  2. Check: Is technical trigger active? Is regime favorable? Is structure healthy?
  3. ALL conditions met → execute trade
  4. Not all met → log which conditions are missing, wait
```

**Example (CI/CD agent):**

```
On event (push OR test_result OR approval):
  1. Read pipeline state
  2. Check: Are all tests passing? Is approval granted? Is deploy window open?
  3. ALL conditions met → deploy
  4. Not all met → report status, wait
```

### 5.2. Event Routing (Classification → Skill)

Use events to select which skill or sub-agent handles the work:

```
## Event Router

| Event Type | Routes To |
|:--|:--|
| `code.tests_failed` | debugging-skill |
| `review.requested` | code-review-skill |
| `deploy.failed` | infra-troubleshooting-skill |
| `user.input_required` | human-in-the-loop handler |
| `alert.security` | security-audit-skill |
```

This is the agentic equivalent of a message broker's topic-based routing.

### 5.3. Event Sourcing for Agent Memory

Instead of storing "current state" directly, store the **sequence of events** that produced it. Replay events to reconstruct state.

**Benefits:**

- Full audit trail (every decision traceable)
- Time-travel debugging ("what was the state at 3 PM?")
- Enables "what-if" analysis (replay with modified events)

**When to use:** Complex multi-step workflows, compliance-sensitive environments, agents that need to explain their reasoning chain.

**When to avoid:** Simple single-turn assistants, high-volume low-value events (use state snapshots instead).

### 5.4. Event-Driven Session Handover

When one agent session ends and another begins, events bridge the gap:

```
## Session End Event

event_type: session.ended
source: architect-agent
summary: "Design session completed. Mario v0.1 design approved."
payload:
  completed:
    - "Mario connector abstraction designed"
    - "ADR written: adr-260222-connector-abstraction.md"
  next_action: "Engineer to implement Mario v0.1 per block-03-01-mario.md"
  blockers: none
  read_first: "meta/agents/engineer.md"
```

This replaces unstructured handover notes with a parseable signal the next agent can consume programmatically.

### 5.5. Webhook-to-Event Bridge (External Integrations)

For agents that react to external systems (GitHub, CI, monitoring):

```
External webhook (GitHub push)
    ↓
Adapter: Normalize to standard event schema
    ↓
{ event_type: "code.pushed", source: "github", payload: { repo, branch, commit_sha } }
    ↓
Agent event handler
```

The adapter is critical — it translates vendor-specific payloads into your standard event schema so the agent logic doesn't couple to any specific external API.

---

## 6. Events in Prompt Engineering

### 6.1. Instructing Agents to Emit Events

In the agent's system prompt or skill definition, explicitly define **when and how** to emit events:

```
## Event Emission Rules

After completing any of the following, emit a structured event:
- Task completion → event_type: task.completed
- Error encountered → event_type: task.failed
- User decision needed → event_type: user.approval_required
- State change that affects other agents → event_type: state.changed

Format: Include event_type, timestamp, summary, and relevant payload.
```

### 6.2. Instructing Agents to Consume Events

Tell the agent how to interpret incoming events:

```
## Event Handling

When you receive an event notification:
1. Parse the event_type and severity.
2. CRITICAL events → stop current work, handle immediately.
3. WARNING events → acknowledge, incorporate into current task.
4. INFO events → log, continue current work.
5. Unknown event types → log and ask the user.
```

### 6.3. Event Streams in Multi-Turn Conversations

In chat-based agents, "events" are often messages injected into the conversation. Structure them clearly so the agent can distinguish events from user messages:

```
[SYSTEM EVENT] test_suite: unit | status: FAILED | failures: 3 | timestamp: 2026-02-25T14:00:00Z
Details: test_order_validation (AssertionError), test_price_rounding (TypeError), test_null_input (ValueError)
```

Use consistent markers (`[SYSTEM EVENT]`, `[ALERT]`, `[NOTIFICATION]`) so the agent can parse them reliably.

---

## 7. Common Pitfalls

| Pitfall                       | Symptom                                                 | Fix                                                      |
| :---------------------------- | :------------------------------------------------------ | :------------------------------------------------------- |
| **Events as commands**        | Events contain instructions ("fix this"), not facts     | Rewrite as facts; let the consumer decide action         |
| **Missing context**           | Agent asks follow-up questions after every event        | Include sufficient payload data                          |
| **No schema**                 | Events have inconsistent structure; parsing breaks      | Define and enforce a standard schema                     |
| **Event storms**              | Agent is overwhelmed by high-frequency low-value events | Batch, debounce, or filter by severity                   |
| **Lost events**               | Events emitted but never consumed; silent failures      | Add acknowledgment/logging; monitor for unhandled events |
| **Tight coupling via events** | Producer changes event format, consumer breaks          | Version event schemas; validate at boundaries            |
| **Acting on stale events**    | Agent acts on an event whose conditions no longer hold  | Use event-triggered, state-evaluated pattern (§5.1)      |
| **No correlation**            | Related events across a workflow can't be linked        | Add `correlation_id` to connect related events           |

---

## 8. Events Across Agent Architectures

### 8.1. Single Agent (Prompt-Driven)

Events are messages or context injections within a conversation. The agent's system prompt defines how to interpret them.

```
User message → Agent processes → Tool call → Tool result (event) → Agent continues
```

### 8.2. Workflow (Orchestrated Multi-Step)

Events connect stages in a predefined pipeline. The orchestrator routes events between steps.

```
Step 1 (Lint) → completion_event → Step 2 (Test) → completion_event → Step 3 (Deploy)
                                         ↓ failure_event
                                   Step 2b (Fix & Retry)
```

### 8.3. Multi-Agent (Dynamic Delegation)

Events trigger handoffs between specialized agents. No central orchestrator — agents emit and subscribe to events.

```
Agent A (Analyst): emits → { event: "design_approved", payload: { spec_path } }
Agent B (Engineer): subscribes → picks up event → implements spec
Agent B: emits → { event: "implementation_complete", payload: { pr_url } }
Agent C (Reviewer): subscribes → picks up event → reviews PR
```

### 8.4. Human-in-the-Loop

Events that require human judgment emit a `user.approval_required` or `user.input_required` event, then **pause execution** until a response event arrives:

```
Agent: emits → { event: "user.approval_required", summary: "Deploy to production?" }
    ↓ (agent pauses, waits)
Human: responds → { event: "user.approved", correlation_id: "deploy-42" }
    ↓
Agent: resumes → proceeds with deployment
```

---

## 9. Designing Event Contracts

When multiple agents or systems communicate via events, treat the event schema as an **API contract**.

### 9.1. Define the Contract

```
## Event Contract: task.completed

Version: 1.0
Producer: Any agent completing a task
Consumers: Orchestrator, logging system, downstream agents

| Field | Type | Required | Description |
|:--|:--|:--|:--|
| event_type | "task.completed" | Yes | Constant |
| source | string | Yes | Agent name |
| timestamp | ISO 8601 | Yes | Completion time |
| task_id | string | Yes | Unique task identifier |
| summary | string | Yes | What was completed |
| artifacts | string[] | No | Paths to produced files |
| duration_ms | int | No | How long the task took |
| next_steps | string[] | No | Suggested follow-up actions |
```

### 9.2. Version the Contract

When the schema changes:

- **Additive changes** (new optional fields): No version bump needed.
- **Breaking changes** (removed fields, type changes): Bump version, support both old and new for a transition period.
- **Never silently change** event semantics — if `task.completed` used to mean "code written" and now means "code written and tested," that's a breaking change.

---

## 10. Quick-Start Checklist

When adding events to an agent system:

- [ ] Have I defined a standard event schema?
- [ ] Have I enumerated the event types my system uses?
- [ ] Does each event carry enough context to act on without extra queries?
- [ ] Are events facts (not disguised commands)?
- [ ] Is there a clear consumer for each event type?
- [ ] Do I have error/failure events, not just success events?
- [ ] Is there a correlation mechanism for multi-step workflows?
- [ ] Have I tested: What happens if an event is missed? Duplicated? Delayed?
- [ ] Am I using event-triggered, state-evaluated pattern where conditions involve multiple signals?

---

## Cross-References

- [guidelines-skills.md](guidelines-skills.md) — Skills that events trigger
- [guidelines-sub-agents.md](guidelines-sub-agents.md) — Multi-agent systems where events are the communication layer
- [guidelines-prompt-engineering.md](guidelines-prompt-engineering.md) — Advanced prompting techniques for event-handling instructions

---

## External Resources

- [Anthropic: Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Workflow and agent patterns
- [OpenAI: Agents SDK — Handoffs](https://openai.github.io/openai-agents-python/handoffs/) — Event-like handoff mechanisms
- [Martin Fowler: Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) — Foundational pattern
- [Martin Fowler: Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html) — Broader context

---

## Changelog

| Date       | Change           |
| :--------- | :--------------- |
| 2026-02-25 | Initial creation |
