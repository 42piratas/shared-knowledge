# Shared Knowledge Base: Project Meta-Instructions Additions

**Purpose:** This document provides standardized content to be added to any project's AI meta-instructions. It details the protocol for integrating with the centralized `shared-knowledge` GitHub repository, ensuring that AI agents can leverage shared learnings and report new ones without direct filesystem access.

---

## Add to Section: Session Start Protocol

### Notifications Check (via GitHub API)

At the start of each session, scan the `shared-knowledge` repository for relevant notifications. Access is performed via the GitHub API.

1.  **Procedure:**
    1.  Use `curl` to query the GitHub API for the contents of the `notifications` directory.
        ```bash
        curl -H "Accept: application/vnd.github.v3+json" https://api.github.com/repos/42piratas/shared-knowledge/contents/notifications
        ```
        _(Note: Unauthenticated GitHub API requests are rate-limited. If issues arise, inform the user.)_
    2.  Parse the JSON response to identify files and their `download_url`.
    3.  Filter for notifications created within the last 7 days.

2.  **Triage:**
    - **Same-day notifications:** Always surface — may indicate outages or blockers.
    - **Last 7 days:** Review for relevance (e.g., service outages, infrastructure changes, known issues).

3.  **Display if relevant:**
    ```
    📢 ACTIVE NOTIFICATIONS
    {date} - {title}
    {summary}
    ---
    ```

---

## Add to Section: Troubleshooting Protocol (or create new section)

### Shared Knowledge Base Integration (via GitHub API)

Before attempting to solve any infrastructure, tooling, or language-specific issue, you **MUST** consult the `shared-knowledge` repository first.

1.  **Procedure:**
    1.  Identify the relevant category (e.g., `infra`, `tooling`) and topic file (e.g., `vercel.md`).
    2.  Use `curl` to query the GitHub API for the contents of the relevant directory to find the file's `download_url`.
        ```bash
        # Example: Listing files in the 'infra' directory
        curl -H "Accept: application/vnd.github.v3+json" https://api.github.com/repos/42piratas/shared-knowledge/contents/infra
        ```
    3.  Parse the JSON response to find the target file and its `download_url`.
    4.  Use the `fetch` tool with the `download_url` to read the document.

2.  **Application:**
    - If a solution for the problem exists, apply it directly.
    - If no solution is found, proceed with standard troubleshooting. If you find a solution, you must document it at the end of the session.

3.  **Mandatory Check:** Never skip this step for issues related to the following topics:

| Category      | Topics                                                             |
| :------------ | :----------------------------------------------------------------- |
| `infra/`      | digitalocean, terraform, vercel, tailscale, docker, github-actions |
| `tooling/`    | poetry, git, npm-pnpm, zed-editor                                  |
| `languages/`  | python, typescript, rust                                           |
| `frameworks/` | nextjs, fastapi, nicegui                                           |
| `databases/`  | postgresql, redis-valkey, sqlite                                   |
| `messaging/`  | rabbitmq                                                           |
| `secrets/`    | hashicorp-vault                                                    |

---

## Add to Section: Session End Protocol

### Lessons Learned Documentation (MANDATORY)

At session end, review for generalizable lessons.

1.  **Identify Lessons:** A lesson is any non-project-specific, non-obvious solution to a problem with infrastructure, tooling, languages, etc., that took significant time to resolve.

2.  **If lessons exist, create a file:**
    - **Location:** `{project-folder}/lessons/YYMMDD-HHMM-lessons-learned.md`
    - **Template:** Fetch the template from its canonical source.
      ```bash
      # Command to fetch the template
      fetch https://raw.githubusercontent.com/42piratas/shared-knowledge/main/meta/sk-template-lessons-learned.md
      ```
    - **Content:** Include Problem, Root Cause, Solution, Prevention, and Context.
    - **Sanitize:** Remove all project-specific details, credentials, and internal URLs.

3.  **Notify User:**
    > "Session generated {N} lessons learned. File created at `lessons/YYMMDD-HHMM-lessons-learned.md`. Please move to the `shared-knowledge` repository for processing."

---

## Add to Change Checklist

Add these items to the session change checklist:

- [ ] Session involved troubleshooting? → Checked `shared-knowledge` base **via GitHub API** first?
- [ ] New lessons learned? → Documented in `lessons/YYMMDD-HHMM-lessons-learned.md`?
- [ ] Notified user to move lessons file to `shared-knowledge` repo?

---

**Last Updated:** 2026-02-05
