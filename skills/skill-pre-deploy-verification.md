# Skill: Pre-Deploy Verification

Mandatory verification checklist before pushing code to a deploy-tracked repo. Prevents the most common first-deploy failures observed across sessions.

---

## When to Use

**Before every `git push` to a branch that will trigger CI/CD on a deploy-tracked repo.** No exceptions. Run through this checklist after all code changes are complete and before creating the PR.

This skill complements the DONE gate in `skill-change-tracking.md` — that gate verifies *after* deploy; this skill prevents deploy failures *before* they happen.

---

## Prerequisites

Before running this checklist, the engineer MUST have read `playbook-infra.md` (or the project's equivalent infrastructure playbook) for the current session. Many pre-deploy failures are caused by not knowing the deploy pipeline mechanics. If you haven't read it → read it now before proceeding.

---

## Instructions

### 1. Lint Check (Local)

Run the project's linter locally before pushing. Do not rely on CI to catch lint errors.

```/dev/null/lint-check.sh#L1-2
# Python services (42Bros pattern)
ruff check .
```

Common failures: line length, unused imports, import ordering, missing blank lines between import groups.

### 2. Import Path Verification

Verify all import paths will resolve correctly **inside Docker**, not just locally.

- [ ] If the service has `src/__init__.py` (package mode) → all internal imports MUST be **relative** (`from .settings import settings`, not `from settings import settings`)
- [ ] If the service does NOT have `src/__init__.py` → absolute imports are correct (`from settings import settings`)
- [ ] No imports reference local filesystem paths or dev-only modules

**Why this fails:** Local Python resolves imports differently than Docker's `WORKDIR` + `CMD` combination. Absolute imports work locally but crash in Docker when `src/` is a package.

### 3. Dependency Verification

- [ ] Every `import` statement in the codebase has a corresponding dependency in `pyproject.toml` (or is a stdlib module)
- [ ] If a new dependency was added → `poetry lock` has been run and `poetry.lock` is committed
- [ ] If using `b42_common` → the git tag pin in `pyproject.toml` matches the version that contains the features you're using

**Why this fails:** Code works locally because the package exists in the dev venv, but the Docker build only installs what `pyproject.toml` declares.

### 4. Environment Variable & Secrets Verification

- [ ] Every env var the code reads at startup (via `BaseSettings`, `os.getenv`, or config loaders) is present in **one** of:
  - The deploy workflow's `.env` generation step (fetched from Vault or GH secrets)
  - The `docker-compose.prod.yml` environment section
  - A config file that is SCP'd or volume-mounted
- [ ] If a new Vault secret was added → the Vault path exists and the deploy workflow reads from the correct path
- [ ] If a new GitHub Actions secret was added → it exists in the repo's settings and is referenced in `deploy.yml`

**Why this fails:** Deploy workflows write `.env` from scratch using only the secrets they know about. A new env var in code without a corresponding deploy workflow update = crash-loop on deploy.

### 5. Payload & Contract Verification

When the change involves inter-service communication (RabbitMQ messages, REST API calls, Valkey keys):

- [ ] Publisher payload schema matches what consumers expect (field names, types, nesting structure)
- [ ] If a field was renamed → all consumers handle both old and new names (dual-key pattern) OR all consumers are deployed first
- [ ] If a new RabbitMQ exchange/queue was added → the exchange exists on the broker (created by the publishing service or IaC)

**Why this fails:** Publisher sends `{funding_rate: 0.01}` flat; consumer expects `{BTC: {funding_rate: 0.01}}` nested. Works in unit tests with mocked data, fails in production.

### 6. Deploy Workflow Verification

- [ ] `deploy.yml` references the correct Vault secret path for this service
- [ ] Docker image name in `deploy.yml` matches `docker-compose.prod.yml`
- [ ] If the service uses Alembic → the deploy script runs migrations before starting the container (or the entrypoint handles it)
- [ ] `docker-compose.prod.yml` has memory limits set

---

## Constraints

- This checklist is a **gate**, not a suggestion. Do not push until all applicable items pass.
- Items that don't apply to the current change can be skipped (e.g., skip §5 if no inter-service communication changed).
- Do NOT run `ruff check` on repos you didn't change — only on the repo being pushed.
- This skill does NOT replace the DONE gate (`skill-change-tracking.md` §Final). This runs *before* push; the DONE gate runs *after* deploy.

---

## Error Handling

- **Lint fails:** Fix all errors before pushing. Do not push with `# noqa` unless the suppression is justified and documented.
- **Import path uncertain:** Check the Dockerfile's `WORKDIR` and the presence of `__init__.py` in the source directory. When in doubt, match the pattern used by existing services in the same project.
- **Missing env var in deploy workflow:** Add it to `deploy.yml` now — do not defer. If the value comes from Vault, verify the Vault path exists. If the value is a new GH secret, add it to the repo before pushing.
- **Payload schema mismatch:** If the change is a rename, implement the dual-key backward-compatible pattern (consumers accept both old and new names). Deploy consumers before publisher.

---

**Last Updated:** 2026-03-08
