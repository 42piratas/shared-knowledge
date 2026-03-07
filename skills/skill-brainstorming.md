# Skill: Brainstorming — Collaborative Ideation

Structured collaborative dialogue to turn vague ideas into clear, actionable designs. User-triggered, never automatic.

---

## When to Use

User explicitly activates this skill. Typical triggers:

- "Let's brainstorm..." / "I want to think through..."
- Product direction, workflow design, feature ideation, process improvement
- Any creative/strategic exploration before committing to implementation

---

## Principles

1. **One question at a time.** Never overwhelm with multiple questions in a single message. If a topic needs more exploration, break it into sequential questions.
2. **Multiple choice when possible.** Easier to answer, faster to converge. Open-ended when the space is genuinely unknown.
3. **YAGNI ruthlessly.** Strip unnecessary complexity from every proposal. The simplest version that solves the problem wins.
4. **Explore before converging.** Always surface 2–3 approaches with trade-offs before settling. Premature convergence kills good ideas.
5. **Incremental validation.** Present ideas in pieces. Get confirmation before building on top. Don't deliver a monolith and ask "looks good?"
6. **Challenge, don't comply.** The agent's job is to stress-test ideas, not agree with everything. Push back when something doesn't hold up.
7. **Capture the output.** Brainstorming that isn't recorded is brainstorming that didn't happen. The session must produce a tangible artifact.

---

## Hard Gates

⛔ **No implementation during brainstorming.** Do not write code, create infrastructure, scaffold projects, or execute changes. Brainstorming produces decisions and designs — not artifacts. Implementation is a separate step, owned by the appropriate agent.

⛔ **No design without user approval.** The agent proposes, the user decides. Every design section requires explicit user confirmation before it becomes the basis for further work.

---

## Procedure

### 1. Understand Context

- [ ] Read any documents the user references or that are relevant to the topic
- [ ] If resuming a prior brainstorm → read the previous output document first
- [ ] Summarize your understanding of the current state in 2–3 sentences
- [ ] Ask the user to confirm or correct before proceeding

### 2. Explore the Idea

- [ ] Ask clarifying questions — **one per message**
- [ ] Focus on: purpose, constraints, success criteria, who's affected, what changes
- [ ] Prefer multiple choice when the option space is bounded
- [ ] Continue until the problem is well-defined — don't rush to solutions

### 3. Propose Approaches

- [ ] Present 2–3 distinct approaches with trade-offs
- [ ] Lead with your recommended option and explain why
- [ ] For each approach, cover:
  - What it solves
  - What it doesn't solve or makes harder
  - Effort / complexity estimate (relative, not absolute)
- [ ] Let the user pick, combine, or reject all and redirect

### 4. Develop the Design

- [ ] Once an approach is chosen, flesh it out in sections
- [ ] Scale each section to its complexity — a sentence if straightforward, a paragraph if nuanced
- [ ] Present each section and confirm before moving to the next
- [ ] Sections to cover (skip any that don't apply):
  - **Goal** — what this achieves, in one sentence
  - **Scope** — what's in, what's explicitly out
  - **Components / workflow** — the moving parts and how they connect
  - **Data** — what's stored, what flows, what changes
  - **Edge cases / failure modes** — what can go wrong and how it's handled
  - **Dependencies** — what this requires to exist or change
  - **Open questions** — anything unresolved that needs future decision

### 5. Record the Output

- [ ] Present the final design as a single, copy-pasteable document using the output template below
- [ ] Format: concise, scannable, no filler — tables and bullets over prose
- [ ] Include a `## Decisions` section capturing every choice made during the brainstorm with brief rationale
- [ ] Include an `## Open Questions` section for anything deferred
- [ ] **Stop and wait for user instructions** — the user decides where to save, what to do next, and whether to hand off to another agent

---

## Output Template

```/dev/null/brainstorm-output-template.md#L1-30
# Brainstorm: {Topic}

**Date:** {YYYY-MM-DD}
**Participants:** User, {Agent name}

## Goal
{One sentence.}

## Scope
- **In:** {what's covered}
- **Out:** {what's explicitly excluded}

## Design
{Sections as needed — components, workflow, data, etc.}

## Decisions

| # | Decision | Rationale |
|:--|:--|:--|
| 1 | ... | ... |

## Open Questions

| # | Question | Context |
|:--|:--|:--|
| 1 | ... | ... |
```

---

## Anti-Patterns

| Anti-Pattern                          | Why It's Bad                                               | Instead                                                       |
| :------------------------------------ | :--------------------------------------------------------- | :------------------------------------------------------------ |
| Jumping to solutions before exploring | Anchors on the first idea, kills alternatives              | Finish step 2 before step 3                                   |
| Asking 5 questions at once            | Overwhelms, gets shallow answers                           | One question per message                                      |
| Agreeing with everything              | Produces fragile designs that fall apart in implementation | Challenge assumptions, surface trade-offs                     |
| Skipping the record step              | Design evaporates — next session starts from zero          | Always present the final document for the user to persist     |
| Brainstorming indefinitely            | Diminishing returns after convergence                      | Once the user approves the design, stop. Capture and move on. |

---

## Project-Specific Extensions

Each project may extend this skill with:

1. **Domain-specific sections** — additional design sections relevant to the project (e.g., cost analysis, data contracts, UI specs)

These extensions belong in the project's agent files or project-specific skills — not in this shared skill.

---

**Last Updated:** 2026-03-07

---

## Changelog

| Date       | Change           |
| :--------- | :--------------- |
| 2026-03-07 | Initial creation |
