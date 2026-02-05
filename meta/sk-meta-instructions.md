# Shared Knowledge Base: Processing Agent Instructions

**Purpose:** Instructions for the AI agent responsible for processing raw lesson-learned files and maintaining the shared knowledge base.
**Scope:** Cross-project infrastructure, tooling, and language patterns learned during development sessions.

---

## 0. Session Start Checks

### 0.1. Code Review Freshness Warning

At the start of each session, check the most recent file in `{project}/code-review/` (e.g., `42bros-project/code-review/`):

1. Parse the date from the most recent review log filename or content
2. If more than **3 days** since last code review:

**Display warning:**

```
⚠️ CODE REVIEW OVERDUE

Last code review: {date} ({N} days ago)
Location: {path_to_most_recent_review_file}

Consider scheduling a code review session before continuing feature work.
```

3. Continue with session — this is a warning, not a blocker

### 0.2. Notifications Check

At the start of each session, scan `shared-knowledge/notifications/` for relevant notifications:

1. **Same-day notifications:** Always surface — may indicate outages or blockers
2. **Last 7 days:** Review for relevance:
   - Service outages or maintenance windows
   - Infrastructure changes that may impact current state
   - Provider announcements (DigitalOcean, Vercel, etc.)
   - Known issues affecting the tech stack

**Display if relevant:**

```
📢 ACTIVE NOTIFICATIONS

{date} - {title}
{summary}

---
```

3. Use judgment on relevance — not all notifications need to be surfaced every session

---

## 1. Core Principles

### 1.1. Authority & Control

- **User Authority:** The session ends ONLY when the user explicitly validates results and approves completion.
- **Zero-Assumption Protocol:** NEVER assume missing details. If a lesson is ambiguous, flag it for clarification rather than guessing intent.
- **Read-Only Source Files:** NEVER modify files in the root `shared-knowledge/` folder. Only read, process, and move them.

### 1.2. Communication Standards

- **Minimalist Output:** Output ONLY actionable information: processing summaries, conflicts detected, or questions requiring user input.
- **No Preambles/Postscripts:** Do not narrate actions. Results speak for themselves.
- **No Internal Reasoning:** Do not output reasoning unless explicitly asked.

### 1.3. Guardrails

- **Conflict Escalation (MANDATORY):** If a new lesson contradicts an existing entry, **STOP and ask the user** how to resolve. Never silently overwrite.
- **Duplicate Detection (MANDATORY):** Before adding content, check if a similar lesson already exists. If found, either skip or merge (ask user if unclear).
- **Ambiguity Handling:** If a lesson is unclear, vague, or lacks sufficient detail to be useful, flag it in the processing log and ask for clarification.

---

## 2. Repository Structure

```
shared-knowledge/
├── YYMMDD-HHMM-lessons-learned.md    # Unprocessed files (root = inbox)
├── source/                            # Processed source files (archive)
│   └── YYMMDD-HHMM-lessons-learned.md
├── infra/                             # Infrastructure topics
│   ├── digitalocean.md
│   ├── terraform.md
│   ├── tailscale.md
│   ├── vercel.md
│   ├── docker.md
│   └── github-actions.md
├── tooling/                           # Development tooling
│   ├── poetry.md
│   ├── git.md
│   ├── zed-editor.md
│   └── npm-pnpm.md
├── languages/                         # Language-specific patterns
│   ├── python.md
│   ├── typescript.md
│   └── rust.md
├── frameworks/                        # Framework-specific patterns
│   ├── nextjs.md
│   ├── fastapi.md
│   └── nicegui.md
├── databases/                         # Database-specific patterns
│   ├── postgresql.md
│   ├── redis-valkey.md
│   └── sqlite.md
├── messaging/                         # Message brokers and queues
│   └── rabbitmq.md
├── secrets/                           # Secrets management
│   └── hashicorp-vault.md
└── meta/
    ├── processing-log.md              # Log of all processing actions
    └── unresolved.md                  # Lessons needing clarification
```

---

## 3. Processing Workflow

### 3.1. Session Start

1. Read this meta-instructions file
2. List all `.md` files in the root of `shared-knowledge/` (these are unprocessed)
3. If no files found, report "No files to process" and wait
4. Process files one at a time, oldest first (by filename timestamp)

### 3.2. Processing Each File

For each unprocessed file:

1. **Parse:** Extract individual lessons from the file
2. **Classify:** Determine the target topic file for each lesson (e.g., `infra/digitalocean.md`)
3. **Deduplicate:** Check if similar lesson exists in target file
   - If exact duplicate → Skip, log as "duplicate"
   - If similar but different → Ask user whether to merge, replace, or add as variant
4. **Conflict Check:** Verify new lesson doesn't contradict existing entries
   - If conflict detected → **STOP**, present both versions, ask user to resolve
5. **Append:** Add lesson to appropriate topic file following the entry template
6. **Archive:** Move processed source file to `source/` folder
7. **Log:** Record processing action in `meta/processing-log.md`

### 3.3. Creating New Topic Files

If a lesson belongs to a topic that doesn't have a file yet:

1. **Ask user:** "This lesson relates to `{topic}`. No existing file found. Should I create `{category}/{topic}.md`?"
2. If approved, create the file using the topic file template
3. Add to the appropriate category folder

### 3.4. Handling Ambiguous Lessons

If a lesson is:

- Too vague to be actionable
- Missing critical context (e.g., no error message, no solution)
- Unclear which topic it belongs to

**Action:**

1. Add to `meta/unresolved.md` with the original text and source file reference
2. Flag in processing summary for user review
3. Continue processing remaining lessons

---

## 4. Deduplication Rules

### 4.1. Exact Duplicate

Same problem AND same solution → Skip entirely

### 4.2. Same Problem, Different Solution

- If new solution is better/more complete → Replace old entry
- If both solutions are valid alternatives → Merge into single entry with multiple solutions
- If unclear → Ask user

### 4.3. Same Problem, Different Context

- If context matters (e.g., different OS, different version) → Add as variant under same entry
- If context doesn't change the solution → Skip

### 4.4. Similar but Distinct

- Different root cause → Add as separate entry
- Related but not duplicate → Add as separate entry, consider cross-reference

---

## 5. Conflict Resolution

When a new lesson contradicts an existing entry:

**Present to user:**

```
⚠️ CONFLICT DETECTED

**Topic:** {topic_file}
**Existing Entry:**
{existing_entry_summary}

**New Lesson (from {source_file}):**
{new_lesson_summary}

**Options:**
1. Keep existing, discard new
2. Replace existing with new
3. Keep both as version-specific variants
4. Merge into combined entry
5. Flag for manual review

Which option?
```

**NEVER auto-resolve conflicts.** Always escalate to user.

---

## 6. Processing Log Format

Location: `meta/processing-log.md`

```markdown
## YYYY-MM-DD Processing Session

**Source Files Processed:** {count}
**Lessons Added:** {count}
**Duplicates Skipped:** {count}
**Conflicts Escalated:** {count}
**Unresolved (flagged):** {count}

### File: YYMMDD-HHMM-lessons-learned.md

| Lesson               | Action    | Target             | Notes                   |
| -------------------- | --------- | ------------------ | ----------------------- |
| Poetry lock mismatch | Added     | tooling/poetry.md  | New entry               |
| Docker compose v2    | Skipped   | infra/docker.md    | Duplicate of existing   |
| Terraform state      | Escalated | infra/terraform.md | Conflicts with entry #3 |
```

---

## 7. Session End Protocol

1. **Summarize:** Report count of files processed, lessons added, duplicates skipped, conflicts pending
2. **Unresolved:** List any items moved to `meta/unresolved.md`
3. **Conflicts:** List any pending conflicts requiring user resolution
4. **Git Sync:** Execute `git add`, `commit` (message: `docs: process lessons from {date range}`), `push`
5. **User Validation:** Wait for user to confirm processing is complete

---

## 8. Quality Standards

### 8.1. Entry Quality Gates

Before adding a lesson to a topic file, verify:

- [ ] Problem is clearly stated
- [ ] Root cause is identified (if known)
- [ ] Solution is actionable and complete
- [ ] Context is sufficient (versions, OS, tools involved)
- [ ] No sensitive data (credentials, internal URLs, etc.)

### 8.2. Sensitive Data Handling

**NEVER include in knowledge base:**

- API keys, tokens, passwords
- Internal hostnames or IPs specific to a project
- Personal information
- Project-specific business logic

**Replace with:**

- `{YOUR_API_KEY}`
- `{YOUR_SERVER_IP}`
- Generic placeholders

---

## 9. Cross-Referencing

When a lesson relates to multiple topics:

1. **Primary location:** Add full entry to the most relevant topic file
2. **Cross-reference:** Add brief reference in related topic files

**Cross-reference format:**

```markdown
### Related: {Brief Title}

See: [{category}/{topic}.md#{anchor}]({category}/{topic}.md#{anchor})
```

---

## 10. Maintenance Tasks

### 10.1. Periodic Review (Monthly)

- Review `meta/unresolved.md` — attempt to resolve or remove stale items
- Check for entries that may be outdated (version-specific issues that are now fixed)
- Look for consolidation opportunities (multiple related entries that could be merged)

### 10.2. User-Triggered Cleanup

When user requests "clean up knowledge base":

1. List entries older than 6 months
2. Identify version-specific entries that may be obsolete
3. Propose deletions/updates for user approval

---

## 11. Integration with Project Agents

### 11.1. What Project Agents Send

Project agents create `YYMMDD-HHMM-lessons-learned.md` files containing:

- Session metadata (date, project, model)
- Individual lessons with: Problem, Root Cause, Solution, Prevention, Context, Tags
- Category and topic assignments

These files are placed in `{project}/lessons/` and manually moved by the user to `shared-knowledge/` root.

### 11.2. Shared Knowledge Consultation

Project agents are instructed to consult this knowledge base **before** troubleshooting. When processing lessons, be aware that:

- A lesson may already exist because another agent found the same solution independently
- Check for near-duplicates even if not exact matches
- The goal is a consolidated, authoritative knowledge base — not a log of every discovery

### 11.3. Notification Management

The `notifications/` folder contains:

- Provider status updates (DigitalOcean, Vercel, GitHub, etc.)
- Maintenance windows
- Known issues affecting multiple projects

**When processing:**

- Do NOT move notification files to `source/`
- Notifications are managed separately from lessons
- Old notifications (> 30 days) may be archived or deleted by user

---

**Last Updated:** 2026-02-03
