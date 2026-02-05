# Shared Knowledge Base: Project Meta-Instructions Additions

**Purpose:** Content to be added to each project's AI meta-instructions to integrate with the shared knowledge base system.

---

## Add to Section: Session Start Protocol

### Code Review Freshness Check

At the start of each session, check the most recent file in `{project}/code-review/`:

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

### Notifications Check

At the start of each session, scan `shared-knowledge/notifications/` for relevant notifications:

1. **Same-day notifications:** Always surface — may indicate outages or blockers
2. **Last 7 days:** Review for relevance:
   - Service outages or maintenance windows
   - Infrastructure changes that may impact current state
   - Provider announcements (DigitalOcean, Vercel, GitHub, etc.)
   - Known issues affecting the tech stack

**Display if relevant:**

```
📢 ACTIVE NOTIFICATIONS

{date} - {title}
{summary}

---
```

3. Use judgment on relevance — not all notifications need to be surfaced every session

Add the following step after reading project-specific documents:

```markdown
### Shared Knowledge Base Consultation

Before troubleshooting any infrastructure, tooling, or language-specific issue:

1. **Consult the shared knowledge base** at `~/Spaceship/shared-knowledge/`
2. **Check relevant topic files** based on the technology involved:
   - Infrastructure issues → `infra/{tool}.md` (digitalocean, terraform, vercel, tailscale, docker, github-actions)
   - Tooling issues → `tooling/{tool}.md` (poetry, git, npm-pnpm, zed-editor)
   - Language issues → `languages/{lang}.md` (python, typescript, rust)
   - Framework issues → `frameworks/{framework}.md` (nextjs, fastapi, nicegui)
   - Database issues → `databases/{db}.md` (postgresql, redis-valkey, sqlite)
   - Messaging issues → `messaging/{broker}.md` (rabbitmq)
   - Secrets issues → `secrets/{tool}.md` (hashicorp-vault)
3. **If a matching entry exists**, apply the documented solution before attempting alternative approaches
4. **If no match exists**, proceed with standard troubleshooting
```

---

## Add to Section: Session End Protocol

Add the following as a new checklist item:

```markdown
### Lessons Learned Documentation (MANDATORY)

At session end, review the session for generalizable lessons:

1. **Identify lessons:** Any issue that was:
   - Related to infrastructure, tooling, languages, frameworks, databases, messaging, or secrets management
   - NOT project-specific business logic
   - Took significant time to diagnose or resolve
   - Had a non-obvious root cause
   - Could benefit other projects using the same technology

2. **If lessons exist, create file:**
   - Location: `{project-folder}/lessons/YYMMDD-HHMM-lessons-learned.md`
   - Template: `~/Spaceship/shared-knowledge/meta/sk-template-lessons-learned.md`
   - Include: Problem, Root Cause, Solution, Prevention, Context
   - Sanitize: Remove project-specific details, credentials, internal URLs

3. **Checklist item:**
   - [ ] Lessons learned documented? → If session involved troubleshooting infra/tooling/language issues, document in `lessons/` folder

4. **Notify user:**
   > "Session generated {N} lessons learned. File created at `lessons/YYMMDD-HHMM-lessons-learned.md`. Please move to `~/Spaceship/shared-knowledge/` for processing."
```

---

## Add to Section: Troubleshooting Protocol (New Section or Append)

```markdown
### Shared Knowledge Base Integration

**Before attempting to solve infrastructure/tooling/language issues:**

1. **Check shared knowledge base first**
   - This repository contains lessons learned across all projects
   - Solutions here are verified to work
   - Saves time by avoiding re-discovery of known solutions

2. **If shared knowledge has a solution:**
   - Apply it directly
   - If it doesn't work, note the discrepancy for later documentation

3. **If shared knowledge doesn't have a solution:**
   - Proceed with standard troubleshooting
   - Document the solution at session end if successful

4. **Never skip this step** for issues involving:
   - DigitalOcean (droplets, managed databases, spaces, networking)
   - Terraform (state, providers, provisioners)
   - Docker (compose, networking, volumes, builds)
   - Poetry (lock files, dependencies, versions)
   - GitHub Actions (workflows, secrets, runners)
   - Vercel (deployments, environment variables, domains)
   - Tailscale (networking, ACLs, exit nodes)
   - Database connections (SSL, connection strings, migrations)
   - Any error message that looks generic/environmental
```

---

## Add to Change Checklist

Add this item to the session change checklist:

```markdown
### Knowledge Sharing

- [ ] Session involved troubleshooting? → Check shared-knowledge base first
- [ ] New lessons learned? → Document in `lessons/YYMMDD-HHMM-lessons-learned.md`
- [ ] Notify user to move lessons file to shared-knowledge repo
```

---

## Full Example: Minimal Integration Block

For projects with simpler meta-instructions, add this consolidated block:

```markdown
## Shared Knowledge Base Protocol

### On Session Start

1. **Code Review Check:** If `{project}/code-review/` exists, check last review date. Warn if > 3 days old.
2. **Notifications Check:** Scan `shared-knowledge/notifications/` for same-day or recent relevant items (outages, maintenance, known issues).
3. **Knowledge Base Consultation:** Before troubleshooting infra/tooling issues, consult `~/Spaceship/shared-knowledge/{category}/{topic}.md`

### On Session End

- If troubleshooting occurred, create `lessons/YYMMDD-HHMM-lessons-learned.md` with:
  - Problem, Root Cause, Solution, Prevention, Context
  - No project-specific details or credentials
- Notify user: "Lessons file created. Please move to shared-knowledge repo."

### Categories

| Category    | Topics                                                             |
| ----------- | ------------------------------------------------------------------ |
| infra/      | digitalocean, terraform, vercel, tailscale, docker, github-actions |
| tooling/    | poetry, git, npm-pnpm, zed-editor                                  |
| languages/  | python, typescript, rust                                           |
| frameworks/ | nextjs, fastapi, nicegui                                           |
| databases/  | postgresql, redis-valkey, sqlite                                   |
| messaging/  | rabbitmq                                                           |
| secrets/    | hashicorp-vault                                                    |
```

---

## Notes for Implementation

1. **Folder creation:** Each project needs a `lessons/` folder created at the appropriate location
2. **Template location:** The lessons-learned template is stored at `~/Spaceship/shared-knowledge/meta/sk-template-lessons-learned.md` — agents should reference this centralized template
3. **User workflow:** User must manually move files from `{project}/lessons/` to `~/Spaceship/shared-knowledge/` root
4. **Processing:** A separate agent session processes files from shared-knowledge root into topic files

---

**Last Updated:** 2026-02-03
