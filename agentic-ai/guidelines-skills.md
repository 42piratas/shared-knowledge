# Guidelines: AI Agent Skills

**Category:** Prompt Engineering
**Last Updated:** 2026-02-25

---

## Overview

Skills are reusable, self-contained capability modules that teach an AI agent **how to perform a specific type of task**. Rather than stuffing every instruction into a single monolithic system prompt, skills decompose agent behavior into composable units — each with its own instructions, tools, and constraints.

This guide covers the concept, design principles, implementation patterns, and pitfalls of skills across platforms (Anthropic Claude, OpenAI Codex/Agents SDK, IDE assistants, custom agent systems). It is project-agnostic.

---

## 1. What Is a Skill?

A skill is a **bounded instruction set** that an agent can invoke (or be configured with) to handle a specific class of work. Think of it as a "micro-prompt with tools attached."

| Aspect         | Description                                                                                       |
| :------------- | :------------------------------------------------------------------------------------------------ |
| **Scope**      | One well-defined capability (e.g., "write unit tests," "review SQL migrations," "deploy via SSH") |
| **Activation** | Triggered by user request, routing logic, or agent self-selection                                 |
| **Content**    | Instructions (prompt text), tool definitions, constraints, and optionally examples                |
| **Isolation**  | A skill should be understandable and testable on its own                                          |

### Skills vs. System Prompts vs. Tools

| Concept           | What it is                                                | Granularity    |
| :---------------- | :-------------------------------------------------------- | :------------- |
| **System prompt** | The agent's global identity and behavioral rules          | One per agent  |
| **Skill**         | A task-specific instruction module the agent can activate | Many per agent |
| **Tool**          | A single callable function (API, shell, file op)          | Many per skill |

A skill **orchestrates tools** with domain-specific instructions. A tool is a primitive; a skill is a strategy.

---

## 2. Design Principles

### 2.1. Single Responsibility

Each skill does **one thing well**. If a skill description requires the word "and" to explain what it does, it's probably two skills.

**Good:** "Write pytest unit tests for Python functions."
**Bad:** "Write tests and deploy the service and update documentation."

### 2.2. Self-Contained Instructions

A skill's prompt must contain everything the agent needs to perform the task **without relying on implicit knowledge from other skills**. Assume the agent has amnesia about everything except:

- The global system prompt (identity, behavioral rules)
- The skill's own instructions
- The current conversation context

### 2.3. Explicit Inputs and Outputs

Define what the skill expects and what it produces:

```
## Skill: Generate API Client

### Inputs
- OpenAPI spec (URL or file path)
- Target language (TypeScript | Python)
- Output directory

### Outputs
- Generated client code in target directory
- Type definitions file
- README with usage examples

### Tools Required
- file_read, file_write, fetch (for remote specs)
```

### 2.4. Constraints Over Freedom

Constrain the agent's behavior within the skill. Unbounded skills produce inconsistent results.

- Specify **what NOT to do** (negative constraints are powerful)
- Set **quality gates** (e.g., "All generated functions must have type hints")
- Define **stop conditions** (e.g., "Stop after 3 failed attempts and ask the user")

### 2.5. Composability

Skills should be stackable. A "Deploy Service" workflow might chain: `lint-code` → `run-tests` → `build-docker` → `push-registry` → `deploy-ssh`. Each is a separate skill; the workflow is orchestration.

---

## 3. Anatomy of a Well-Written Skill

### Minimal Template

```
# Skill: {Name}

## Purpose
{One sentence — what this skill does and when to use it.}

## Instructions
{Step-by-step procedure. Be explicit. Use numbered steps for sequential work, bullets for parallel/unordered.}

## Constraints
- {What the agent must NOT do}
- {Quality requirements}
- {Scope boundaries}

## Tools
- {tool_name}: {when/how to use it}

## Output Format
{What the final output looks like — structure, format, location.}

## Error Handling
{What to do when things go wrong. When to retry vs. ask the user.}
```

### Extended Example: Code Review Skill

```
# Skill: Python Code Review

## Purpose
Review Python code changes for correctness, style, security, and maintainability.

## Instructions
1. Read the files or diff provided by the user.
2. Check for: type safety, error handling, naming conventions, unnecessary complexity, security issues (SQL injection, credential exposure, path traversal).
3. For each finding, state: file, line(s), severity (critical/warning/nit), and a concrete fix.
4. Summarize with a table: | File | Finding | Severity | Suggested Fix |
5. End with an overall assessment: APPROVE, REQUEST_CHANGES, or NEEDS_DISCUSSION.

## Constraints
- Do NOT rewrite the code unless asked. Suggest fixes, don't impose them.
- Do NOT comment on formatting if an autoformatter (black, ruff) is configured.
- Flag security issues as CRITICAL regardless of context.
- Maximum 15 findings per review. Prioritize by severity.

## Tools
- read_file: Read source files
- grep: Search for patterns across codebase
- diagnostics: Check for compiler/linter errors

## Output Format
Markdown table of findings + summary verdict.

## Error Handling
- If a file path is invalid, list it as "skipped" and continue.
- If the diff is too large (>2000 lines), ask the user to scope it down.
```

---

## 4. Implementation Patterns

### 4.1. Platform-Specific Approaches

#### Anthropic Claude (API / Claude Code)

Claude's "Agent Skills" (as of early 2026) are markdown instruction files that the model reads from the project or system prompt. In IDE integrations (e.g., Zed, VS Code), skills are typically:

- **Markdown files** in the project (e.g., `meta/agents/engineer.md`)
- **Attached as context** at session start via file references
- **Prompt-cached** for efficiency on repeated use

Key pattern: The system prompt defines the agent's identity and behavioral rules. Skill files are loaded as user-attached context and contain the task-specific procedures.

#### OpenAI Codex / Agents SDK

OpenAI formalizes skills as **composable prompt + tool bundles** that can be:

- Defined in an `AGENTS.md` or config file at the project root
- Activated by the agent or the orchestrator based on task classification
- Combined with shell access, file tools, and web search

In the Agents SDK (Python), a "skill" maps to an `Agent` object with its own `instructions` and `tools`:

```
# Conceptual — not runnable code
review_agent = Agent(
    name="Code Reviewer",
    instructions="You review Python code for...",
    tools=[read_file, grep, diagnostics],
)
```

#### IDE Assistants (Cursor, Windsurf, Zed)

Most IDE-integrated AI assistants support skills via:

- **Rules files** (`.cursorrules`, `.windsurfrules`, project-level config)
- **Attached context files** (markdown documents loaded into every session)
- **Slash commands** mapped to skill-like behaviors

The common thread: **skills are prompt text + tool access, delivered as files.**

### 4.2. Skill Selection Strategies

| Strategy                 | How it works                                                   | Best for                                            |
| :----------------------- | :------------------------------------------------------------- | :-------------------------------------------------- |
| **User-directed**        | User explicitly invokes a skill ("Review this code")           | Simple setups, high-trust tasks                     |
| **Router-based**         | A lightweight classifier (LLM or rule-based) selects the skill | Multi-skill agents with distinct task types         |
| **Agent self-selection** | The agent reads available skills and decides which to apply    | Flexible agents, open-ended tasks                   |
| **Always-on**            | Skill is baked into the system prompt permanently              | Core behaviors (e.g., "always follow code style X") |

### 4.3. Skill Libraries (Reusable Across Projects)

Store generic skills in a shared location (like a `shared-knowledge/` repo) and project-specific skills alongside the project. Layer them:

```
shared-knowledge/agentic-ai/skills/          ← Cross-project
    skill-python-review.md
    skill-docker-debug.md
    skill-git-conventional-commits.md

project/meta/agents/                          ← Project-specific
    engineer.md                               (loads shared + local skills)
    skill-deploy-to-digitalocean.md
```

---

## 5. Evaluating Skill Quality

### 5.1. The CLEAR Framework

| Criterion      | Question to ask                                                                    |
| :------------- | :--------------------------------------------------------------------------------- |
| **C**omplete   | Does the skill contain everything needed to perform the task without guessing?     |
| **L**imited    | Is the scope narrow enough to be testable and predictable?                         |
| **E**xplicit   | Are inputs, outputs, constraints, and error handling spelled out?                  |
| **A**ctionable | Can the agent execute this without asking clarifying questions in the common case? |
| **R**epeatable | Does the skill produce consistent results across different inputs?                 |

### 5.2. Testing Skills

Treat skills like code — test them:

1. **Smoke test:** Give the agent a simple input. Does it follow the procedure?
2. **Edge case test:** Give ambiguous, incomplete, or adversarial input. Does the error handling work?
3. **Regression test:** After modifying a skill, re-run previous inputs. Did quality hold?
4. **Cross-model test:** If you use multiple models (e.g., Opus for hard tasks, Haiku for easy ones), verify the skill works on each.

---

## 6. Common Pitfalls

| Pitfall                     | Symptom                                                            | Fix                                              |
| :-------------------------- | :----------------------------------------------------------------- | :----------------------------------------------- |
| **Over-scoped skill**       | Agent gets confused, skips steps, or produces mixed-quality output | Split into smaller skills                        |
| **Implicit assumptions**    | Agent makes wrong choices because the skill doesn't specify        | Add explicit constraints and examples            |
| **No error handling**       | Agent loops forever or silently produces garbage on failure        | Add stop conditions and fallback behavior        |
| **Tool overload**           | Agent has 20+ tools and picks the wrong one                        | Scope tools per skill; only expose what's needed |
| **Stale instructions**      | Skill references outdated APIs, paths, or conventions              | Version skills; review periodically              |
| **No output specification** | Agent produces output in unpredictable formats                     | Define exact output structure                    |

---

## 7. Advanced Patterns

### 7.1. Skill Chaining

One skill's output feeds the next:

```
[Parse Requirements] → [Design Schema] → [Generate Migrations] → [Write Tests]
```

Each step is a separate skill. The orchestrator (or agent) passes context between them. Keep inter-skill contracts explicit: define what each skill produces that the next one consumes.

### 7.2. Conditional Skills

A skill that branches based on input:

```
## Skill: Handle Database Change

1. Read the proposed schema change.
2. IF change is additive (new column, new table) → apply standard migration pattern.
3. IF change is destructive (drop column, rename) → STOP. Flag as high-risk.
   Require user confirmation before proceeding.
4. IF change modifies constraints → generate both up and down migrations.
```

### 7.3. Skills with Memory/State

For long-running sessions, skills can read and write to a shared state file:

```
## Skill: Session-Aware Debugging

1. Read `meta/tmp/debug-state.md` for prior findings this session.
2. Perform current debugging step.
3. Append findings to `meta/tmp/debug-state.md`.
4. On session end, summarize all findings.
```

### 7.4. Skill Guardrails

Wrap skills with pre/post checks:

```
## Pre-check (before any deployment skill)
- [ ] All tests passing?
- [ ] No uncommitted changes?
- [ ] User explicitly approved?

## Post-check (after any code generation skill)
- [ ] Generated code passes linter?
- [ ] No secrets in output?
- [ ] Output matches declared format?
```

---

## 8. Quick-Start Checklist

When creating a new skill:

- [ ] Can I describe what it does in one sentence?
- [ ] Are inputs, outputs, and tools listed explicitly?
- [ ] Are there constraints on what the agent must NOT do?
- [ ] Is there error handling for the top 3 failure modes?
- [ ] Have I tested it with at least one real input?
- [ ] Is it stored in the right location (shared vs. project-specific)?
- [ ] Does it work with the model(s) I'm using?

---

## Cross-References

- [guidelines-event-hooks.md](guidelines-event-hooks.md) — Event-driven patterns for triggering skills
- [guidelines-sub-agents.md](guidelines-sub-agents.md) — Delegating skills to specialized sub-agents
- [guidelines-prompt-engineering.md](guidelines-prompt-engineering.md) — Advanced prompting techniques applicable within skills

---

## External Resources

- [Anthropic: Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) — Foundational patterns
- [OpenAI: Agents SDK Documentation](https://openai.github.io/openai-agents-python/) — SDK-level skill implementation
- [OpenAI: Codex Skills](https://developers.openai.com/codex/skills) — IDE-level skill configuration
- [Anthropic: Agent Skills](https://docs.anthropic.com/en/docs/build-with-claude/agent-skills/overview) — Claude-native skills

---

## Changelog

| Date       | Change           |
| :--------- | :--------------- |
| 2026-02-25 | Initial creation |
