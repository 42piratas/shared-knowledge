# Skill: Change Tracking

Mandatory procedure for any engineering session that involves code, infrastructure, or documentation changes. Creates a structured checklist at the start of work and updates it in real-time as changes are made.

---

## When to Use

At the start of **any task involving changes** — code, config, infrastructure, documentation, or UI. No exceptions. If you're modifying files, you're tracking changes.

---

## Procedure

### 1. Create the Log File

At the start of the task, create a session log file in the project's engineering log directory.

**Naming convention:** `log-YYMMDD-HHMM-{short-description}.md`

Each project defines where engineering logs live (typically `meta/logs/engineering/`).

### 2. Initialize the Checklist

Copy the template below into the log file. Remove sections that don't apply to the project (e.g., remove Infrastructure for projects without IaC). Add project-specific sections as defined in the project's `meta/agents/engineer.md` or `meta/agents/skills/`.

### 3. Update in Real-Time

Check off items **as you complete them**, not at session end. This is a living document — if the session is interrupted, the checklist shows exactly where things stand.

### 4. Verify at Session End

Before closing the session, walk through every unchecked item. Either complete it, mark it N/A with a reason, or flag it as a remaining action for the user.

---

## Checklist Template

```/dev/null/change-checklist-template.md#L1-40
## Change Checklist

### Infrastructure
- [ ] Change requires IaC? → If YES, implement in IaC FIRST
- [ ] IaC validated? (plan reviewed by user)
- [ ] IaC applied? (by user)
- [ ] No manual server/cloud changes made outside IaC?

### Code
- [ ] Changes are minimal and focused on the task?
- [ ] No unrelated refactors mixed in?
- [ ] Error handling covers failure cases?
  - ✅ Pass: No bare `except: pass`, no silent swallowing; all catch blocks log or re-raise
  - ❌ Fail: Any exception silently ignored, OR error path produces no log output

### Testing
- [ ] Unit/integration tests added for logic changes?
  - ✅ Pass: Every new function/method with conditional logic has at least one test
  - ❌ Fail: Logic-bearing code committed without tests, OR "will add tests later"
- [ ] Tests passing?
- [ ] Test-resistant code? → Manual verification protocol documented

### Documentation
- [ ] Architecture docs updated? (if architecture changed)
- [ ] Pipeline/roadmap updated? (if scope or status changed)
- [ ] Relevant block plan updated?
- [ ] Service/component docs updated?
- [ ] **Doc-gate (if renaming contracts, schemas, keys, exchanges, or data models):**
  - ✅ Pass: Every doc in the routing table that references the renamed artifact has been updated in this session
  - ❌ Fail: Any doc still contains old name, OR update deferred to a later task

### Quality
- [ ] Code follows project coding guidelines?
- [ ] Security best practices followed? (secrets, input validation)

### Pre-Deploy Verification
- [ ] **⛔ GATE — before pushing to any deploy-tracked repo:** Run `shared-knowledge/skills/skill-pre-deploy-verification.md` checklist. Do not push until all applicable items pass.

### Final — DONE Gate (see `principles-base.md` §11)

**Push ≠ Done.** A task is only DONE when every applicable step below passes, in order:

1. **Committed and pushed?** *(after pre-deploy verification above)*
   - [ ] All changes committed and pushed to the correct repo(s)
2. **CI/CD green?** *(if repo has CI/CD)*
   - [ ] Pipeline passed — check and confirm. If failed → fix before proceeding.
3. **Server validated?** *(if change deploys to a server)*
   - [ ] Service running, no errors, change-specific checks pass (follow project deploy validation procedure if one exists)
4. **User approved?**
   - [ ] Present completion summary with what was verified and what needs manual testing
   - [ ] **USER ACTION REQUIRED:** list items the user must test/validate
   - [ ] Wait for explicit user confirmation — do NOT mark the task done until received

❌ If any step is blocked or cannot be verified → flag it as **USER ACTION REQUIRED** and leave the task **in-progress**.

### Knowledge Sharing
- [ ] Session involved troubleshooting? → Consulted shared-knowledge KB first?
- [ ] New lessons learned? → Written to shared-knowledge KB?
- [ ] KB changes committed and pushed separately?
```

---

## Documentation Update Routing

When a change occurs, update the relevant document **immediately** — not at session end. Each project defines its own routing table mapping change types to documents. Common pattern:

| Change Type               | Update Immediately          |
| :------------------------ | :-------------------------- |
| Architecture change       | Architecture/system-map doc |
| New technical debt        | Pipeline (debt section)     |
| Feature completed/planned | Pipeline + block plan       |
| Infrastructure change     | Infrastructure playbook     |
| Code pattern/standard     | Code guidelines             |
| Service implementation    | Service README              |

The project's `meta/agents/engineer.md` or `meta/agents/skills/` contains the project-specific routing table with exact file paths. This table is a reference for the pattern — always defer to the project-specific version.

---

## Project-Specific Extensions

Each project extends this template by:

1. **Adding sections** relevant to the project (e.g., "Data & UI" for projects with dashboards, "Agent Prompts" for agentic frameworks).
2. **Adding items** within existing sections (e.g., specific schema or contract checks under Code).
3. **Defining the routing table** with exact document paths.

These extensions belong in the project's `meta/agents/skills/` or inline in the Engineer agent file — not in this shared skill.

---

**Last Updated:** 2026-03-08

---

## Changelog

| Date       | Change                                                                                                                                    |
| :--------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-03-08 | Added Pre-Deploy Verification gate before push — references `skill-pre-deploy-verification.md`                                            |
| 2026-02-28 | Replaced `### Final` with staged DONE gate — enforces CI/CD → server validation → user approval sequence (refs `principles-base.md` §11)  |
| 2026-02-25 | Initial creation — extracted change tracking structure from 42Bros, Batcave, and TRON engineer agents into skill                           |
