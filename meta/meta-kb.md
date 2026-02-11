# Shared Knowledge Base — Agent Instructions

**Purpose:** Instructions for any AI agent working on any project that has `shared-knowledge` as a second root directory in the Zed workspace. This file governs how agents consult the knowledge base during troubleshooting and how they write new lessons directly into the KB at session end.

**Scope:** Cross-project infrastructure, tooling, language patterns, and framework-specific lessons learned during development sessions.

---

## 1. Core Principles

### 1.1. Authority & Control

- **User Authority:** Only the user decides when a session is complete.
- **Zero-Assumption Protocol:** NEVER assume missing details. If something is ambiguous, ask — don't guess.
- **D.R.Y. (Don't Repeat Yourself):** Each lesson has ONE authoritative location in the KB. Never duplicate content across files. If a lesson could fit multiple categories, choose the single best fit based on market/community conventions and add a cross-reference if needed.

### 1.2. Guardrails

- **Conflict Escalation (MANDATORY):** If a new lesson contradicts an existing entry, **STOP and ask the user** how to resolve. Never silently overwrite.
- **Duplicate Detection (MANDATORY):** Before adding content, read the target topic file and check if a similar lesson already exists. If found, either skip or merge — ask the user if unclear.
- **Sensitive Data (MANDATORY):** NEVER write API keys, tokens, passwords, internal hostnames/IPs, or project-specific business logic into the KB. Replace with generic placeholders like `{YOUR_API_KEY}`, `{YOUR_SERVER_IP}`.

---

## 2. Repository Structure

```
shared-knowledge/
├── knowledge-base/                    # Topic files organized by category
│   ├── infra/                         # Infrastructure (digitalocean, terraform, docker, etc.)
│   ├── tooling/                       # Dev tooling (poetry, git, monitoring, etc.)
│   ├── languages/                     # Language-specific patterns (python, typescript, etc.)
│   ├── frameworks/                    # Framework-specific patterns (nextjs, fastapi, etc.)
│   ├── databases/                     # Database-specific patterns (supabase, postgresql, etc.)
│   ├── messaging/                     # Message brokers and queues
│   └── secrets/                       # Secrets management
├── templates/
│   └── sk-template-topic-file.md      # Template for new topic files
├── notifications/                     # Provider status updates and known issues
└── meta/
    └── meta-kb.md                     # This file
```

---

## 3. Session Start — Consulting the KB

### 3.1. Notifications Check

At the start of each session:

1. List the contents of `shared-knowledge/notifications/` (excluding the `archive/` subfolder)
2. **Archive old notifications:** Move any notification file whose event date is older than 3 days to `shared-knowledge/notifications/archive/`
3. Read any remaining notifications (these are all ≤ 3 days old)
4. Same-day notifications should always be surfaced — they may indicate outages or blockers

If relevant notifications exist, display:

```
📢 ACTIVE NOTIFICATIONS
{date} — {title}
{summary}
---
```

### 3.2. Troubleshooting — KB Lookup (MANDATORY)

Before attempting to solve any infrastructure, tooling, or language-specific issue, you **MUST** check the knowledge base first.

**Procedure:**

1. Identify the relevant category and topic (e.g., `infra/docker`, `tooling/poetry`)
2. Read `shared-knowledge/knowledge-base/{category}/{topic}.md`
3. If a solution exists, apply it directly
4. If no solution exists, proceed with standard troubleshooting — and if you find a solution, document it at session end (see Section 4)

**Never skip this step** for issues related to any topic that has an existing file in the KB.

---

## 4. Session End — Writing Lessons to the KB

At session end, review the session for generalizable lessons learned.

### 4.1. What Qualifies as a Lesson

A lesson is any **non-project-specific, non-obvious** solution to a problem with infrastructure, tooling, languages, frameworks, databases, etc., that required meaningful effort to resolve.

**NOT a lesson:**

- Project-specific business logic
- Trivial fixes (typos, missing imports)
- Well-documented behavior easily found in official docs

### 4.2. Writing Directly to the KB

For each lesson identified:

1. **Determine the target file:** `shared-knowledge/knowledge-base/{category}/{topic}.md`
2. **Read the target file** to check for duplicates and conflicts (see Section 5)
3. **If the file exists:** Append the new entry following the existing format in that file. Increment the entry number. Update the `Entries` count and `Last Updated` date in the file header. Add a row to the `Changelog` table at the bottom.
4. **If the file does NOT exist:** Ask the user for approval, then create it using the template at `shared-knowledge/templates/sk-template-topic-file.md`. Place it in the appropriate category folder.
5. **If the category folder doesn't exist:** Ask the user before creating a new category.

### 4.3. Entry Format

Each entry added to a topic file must include:

- **Brief Descriptive Title**
- **Problem:** Clear description, including exact error messages when applicable
- **Root Cause:** Why it happened
- **Solution:** Actionable, complete fix with commands/code
- **Prevention:** How to avoid it in the future
- **Context:** Versions, OS, relevant conditions
- **Tags:** For searchability

Refer to `shared-knowledge/templates/sk-template-topic-file.md` for the exact format.

### 4.4. Quality Gates

Before writing a lesson to the KB, verify:

- [ ] Problem is clearly stated
- [ ] Root cause is identified (if known)
- [ ] Solution is actionable and complete
- [ ] Context is sufficient (versions, OS, tools involved)
- [ ] No sensitive data (credentials, internal URLs, etc.)
- [ ] Not a duplicate of an existing entry
- [ ] Does not contradict an existing entry (if it does → escalate to user)

### 4.5. Git Sync (MANDATORY)

After writing lessons to the KB, execute:

```bash
cd shared-knowledge && git add . && git commit -m "kb: add lessons from {project-name} session {YYYY-MM-DD}" && git push
```

This step is required. The KB must stay in sync.

---

## 5. Deduplication & Conflict Rules

### 5.1. Before Writing Any Entry

Always read the target topic file first. Then:

| Situation                                     | Action                                           |
| --------------------------------------------- | ------------------------------------------------ |
| Same problem AND same solution                | **Skip** — it's a duplicate                      |
| Same problem, better/more complete solution   | **Replace** the old entry                        |
| Same problem, different valid solution        | **Merge** into one entry with multiple solutions |
| Same problem, different context (OS, version) | **Add as variant** under the same entry          |
| Similar but distinct root cause               | **Add as separate entry**                        |
| New lesson **contradicts** existing entry     | **STOP — ask the user**                          |

When in doubt, ask the user.

### 5.2. Conflict Resolution

When a conflict is detected, present:

```
⚠️ CONFLICT DETECTED

Topic: {topic_file}
Existing Entry: {summary}
New Lesson: {summary}

Options:
1. Keep existing, discard new
2. Replace existing with new
3. Keep both as version-specific variants
4. Merge into combined entry

Which option?
```

**NEVER auto-resolve conflicts.**

---

## 6. Cross-Referencing

When a lesson relates to multiple topics:

1. **Primary location:** Full entry in the single most relevant topic file
2. **Cross-reference in related files:**

```markdown
### Related: {Brief Title}

See: [{category}/{topic}.md#{anchor}]({category}/{topic}.md#{anchor})
```

---

## 7. Notification Management

The `notifications/` folder contains provider status updates, maintenance windows, and known issues affecting multiple projects.

- At session start, agents archive notifications older than 3 days to `notifications/archive/` (Section 3.1)
- Do not delete notifications — only move to archive
- Only consult them at session start (Section 3.1)

---

## 8. Summary Checklist

Add to your session-end checklist:

- [ ] Session involved troubleshooting? → Consulted `shared-knowledge` KB first?
- [ ] New generalizable lessons? → Written directly to `knowledge-base/{category}/{topic}.md`?
- [ ] KB changes committed and pushed?

---

**Last Updated:** 2026-02-18
