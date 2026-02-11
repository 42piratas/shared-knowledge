# Git

**Category:** tooling
**Last Updated:** 2026-02-18
**Entries:** 2

---

## Overview

Lessons learned for Git version control — focusing on configuration issues, signing, and .gitignore patterns.

---

## Entries

### Entry 1: GPG Signing Causes Git Commands to Hang {#entry-1}

**Problem:**
`git tag` or `git commit` hangs indefinitely with no output.

**Root Cause:**
GPG signing is enabled globally, and the GPG agent is waiting for a passphrase prompt that never displays in non-interactive environments (CI, AI assistants, scripts).

**Solution:**
Disable GPG signing for the affected operations:

```bash
# Check current settings
git config --global tag.gpgsign
git config --global commit.gpgsign

# Disable globally
git config --global tag.gpgsign false
git config --global commit.gpgsign false

# Or per-command
git tag -a v1.0.0 --no-sign -m "Release v1.0.0"
```

**Prevention:**

- Disable GPG signing in automated/AI environments
- Use repository-local config for personal signing preferences
- Document signing requirements in project contributing guide

**Context:**

- Versions affected: git 2.x
- OS: macOS, Linux
- First documented: 2026-02-03
- Source: `260203-1500-retroactive-lessons-learned.md`

**Tags:** `git` `gpg` `signing` `automation`

---

### Entry 2: Global .gitignore Patterns Silently Exclude Application Source Files {#entry-2}

**Problem:**
Every CI/CD Docker build of a Next.js dashboard fails with `Module not found: Can't resolve '@/lib/utils'` and `Can't resolve '@/lib/data'`. The build works perfectly on the developer's local machine. Five consecutive CI/CD runs fail, repeatedly misdiagnosed as webpack, TailwindCSS, or Prisma issues.

**Root Cause:**
A global `.gitignore` pattern `lib/` (a Python template convention) matched `dashboard/src/lib/`, preventing 5 critical TypeScript source files from being committed to git. Local builds worked because the files existed on disk — `npm run build` reads from the filesystem, not from git. But CI/CD checks out only what git tracks, so the files were absent in every CI/CD build.

A second pattern, `.dockerignore` in the root `.gitignore`, prevented tracking a dashboard-specific `.dockerignore` that was needed to override the root `.dockerignore` (which also contained `lib/`).

**Solution:**

1. Add gitignore exceptions:

```gitignore
# After existing 'lib/'
!dashboard/src/lib/

# After existing '.dockerignore'
!dashboard/.dockerignore
```

2. Force-add the excluded files:

```bash
git add -f dashboard/src/lib/
```

3. Create a dashboard-specific `.dockerignore` that does NOT contain `lib/`, overriding the root `.dockerignore` for dashboard Docker builds.

**Prevention:**

- When adding a new language/framework to a project, **audit the `.gitignore` for conflicting patterns**. Python's `lib/` conflicts with JavaScript/TypeScript `src/lib/` conventions. Use directory-scoped patterns like `/lib/` (root-only) instead of bare `lib/`.
- When CI/CD fails but local build succeeds, ALWAYS check what git actually tracks first:

```bash
git ls-files <directory>/     # List tracked files
git check-ignore -v <file>    # Check if a specific file is ignored
```

- When creating a Dockerfile, verify all COPY source paths exist in git
- Use directory-scoped gitignore patterns (`/lib/` not `lib/`) for language-specific exclusions
- Multi-context Docker projects should have per-context `.dockerignore` files

**Context:**

- Versions affected: Git 2.x, Docker Buildx, GitHub Actions
- OS: all
- First documented: 2026-02-09
- Source: `260209-2118-lessons-learned-gitignore-docker-build.md`

**Tags:** `git` `gitignore` `docker` `ci-cd` `build-failures`

---

## Cross-References

- [infra/github-actions.md](infra/github-actions.md) — CI/CD debugging and error reading strategies
- [infra/docker.md](infra/docker.md) — Docker build patterns

---

## External Resources

- [Git Documentation — gitignore](https://git-scm.com/docs/gitignore)
- [GitHub — gitignore templates](https://github.com/github/gitignore)

---

## Changelog

| Date       | Change                                    | Source                                                  |
| ---------- | ----------------------------------------- | ------------------------------------------------------- |
| 2026-02-05 | Initial creation with 1 entry             | `260203-1500-retroactive-lessons-learned.md`            |
| 2026-02-18 | Added entry #2 (.gitignore CI/CD pattern) | `260209-2118-lessons-learned-gitignore-docker-build.md` |
