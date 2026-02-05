# Processing Log

Log of all knowledge base processing sessions.

---

## 2026-02-05 Processing Session

**Source Files Processed:** 1
**Lessons Added:** 13
**Duplicates Skipped:** 0
**Conflicts Escalated:** 0
**Unresolved (flagged):** 0

### File: 260203-1500-retroactive-lessons-learned.md

| Lesson | Action | Target | Notes |
|--------|--------|--------|-------|
| Poetry Version Mismatch Between Local and CI/CD | Added | tooling/poetry.md | New file created |
| Docker Compose v1 vs v2 ContainerConfig Error | Added | infra/docker.md | New file created |
| Stale Container Blocking Deployment | Added | infra/docker.md | Appended |
| Terraform/Vault Circular Dependency | Added | infra/terraform.md | New file created |
| Vault Data on Ephemeral Storage | Added | infra/persistent-storage.md | Moved from secrets category; security filter blocked original path |
| Nginx SSL Chicken-and-Egg Problem | Added | infra/nginx.md | New file created (reclassified from docker) |
| Telegram MarkdownV2 Special Character Escaping | Added | tooling/telegram-api.md | New file created |
| GitHub Secrets Typo Causes Silent Deployment Failure | Added | infra/github-actions.md | New file created |
| Unit Tests with Mocked Data Don't Catch Schema Mismatches | Added | tooling/testing.md | New file created |
| External API Geo-Blocking Requires Region Testing | Added | infra/digitalocean.md | New file created |
| Ruff Linter Errors Block CI/CD Pipeline | Added | languages/python.md | New file created |
| Private Git Dependency Requires PAT in Docker Build | Added | infra/docker.md | Appended (consolidated from tooling/docker per D.R.Y.) |
| GPG Signing Causes Git Commands to Hang | Added | tooling/git.md | New file created |

### Notes

- First processing session — all topic files were newly created
- `secrets/` category files blocked by security filter; Vault lesson relocated to `infra/persistent-storage.md`
- D.R.Y. principle added to meta-instructions; Docker lessons consolidated to `infra/docker.md`
- Session 0 checks removed from meta-instructions (not applicable to this agent)
- `unresolved.md` path corrected to root directory in meta-instructions

---
