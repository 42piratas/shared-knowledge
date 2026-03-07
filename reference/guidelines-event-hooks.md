# Reference: Event-Driven Agent Patterns

Principles for using events as communication signals in agent systems. Consult when designing event-based workflows or inter-agent communication.

---

## Core Concept

Events are **structured signals representing things that happened**. They decouple producers from consumers, enable reactive behavior, and make workflows observable. Events say "X happened" — the consumer decides what to do.

---

## Design Principles

1. **Events are facts, not commands.** `tests_failed` is an event. `rerun_tests` is an instruction disguised as an event.
2. **Events are immutable.** Once emitted, never changed. If the situation changes, emit a new event.
3. **Events carry sufficient context.** The consumer must be able to act without querying the producer. Include: what happened, when (UTC), source, scope, and key data.
4. **Events have a schema.** Ad-hoc formats lead to parsing errors. Define a consistent structure.
5. **Events are typed.** Use an explicit, finite enum of event types. Prevents typos, enables routing.
6. **Minimize fanout.** Emit at meaningful boundaries — state changes, errors, decisions. Over-eventing drowns signal in noise.

---

## Standard Event Schema

| Field            | Type                      | Required | Description                                 |
| :--------------- | :------------------------ | :------- | :------------------------------------------ |
| `event_type`     | string (enum)             | Yes      | Category of the event                       |
| `source`         | string                    | Yes      | Who/what emitted it                         |
| `timestamp`      | ISO 8601 UTC              | Yes      | When it happened                            |
| `severity`       | critical / warning / info | Yes      | Urgency level                               |
| `summary`        | string                    | Yes      | Human-readable one-liner                    |
| `payload`        | object                    | No       | Structured data specific to this event type |
| `correlation_id` | string                    | No       | Links related events across a workflow      |

---

## Key Pattern: Event-Triggered, State-Evaluated

Don't act on events blindly. When an event arrives:

1. **Acknowledge** the event (it happened)
2. **Evaluate current state** (is the condition still true?)
3. **Act on the state**, not the event alone

This prevents acting on stale events whose conditions no longer hold.

---

## Common Pitfalls

| Pitfall                   | Symptom                                              | Fix                                                  |
| :------------------------ | :--------------------------------------------------- | :--------------------------------------------------- |
| Events as commands        | Events contain instructions, not facts               | Rewrite as facts; let consumer decide action         |
| Missing context           | Consumer asks follow-up questions after every event  | Include sufficient payload data                      |
| No schema                 | Inconsistent structure; parsing breaks               | Define and enforce a standard schema                 |
| Event storms              | Agent overwhelmed by high-frequency low-value events | Batch, debounce, or filter by severity               |
| Lost events               | Events emitted but never consumed                    | Add acknowledgment/logging; monitor unhandled events |
| Tight coupling via events | Producer changes format, consumer breaks             | Version schemas; validate at boundaries              |
| Acting on stale events    | Agent acts on outdated conditions                    | Use event-triggered, state-evaluated pattern         |
| No correlation            | Related events across a workflow can't be linked     | Add `correlation_id`                                 |

---

## Cross-References

- `shared-knowledge/reference/guidelines-sub-agents.md` — Multi-agent systems where events are the communication layer
- `shared-knowledge/reference/guidelines-skills.md` — Skills that events trigger

---

**Last Updated:** 2026-03-07
