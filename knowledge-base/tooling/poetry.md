# Poetry

Lessons learned for Python dependency management with Poetry.

---

## Poetry Version Mismatch Between Local and CI/CD

**Problem:**
CI/CD pipeline fails with lock file errors despite working locally.

```
pyproject.toml changed significantly since poetry.lock was last generated. Run poetry lock to fix the lock file.
```

**Root Cause:**
`poetry.lock` was generated with Poetry 2.x locally, but GitHub Actions installed Poetry 1.8.x via `pipx install poetry`. Poetry 1.x cannot parse lock files from Poetry 2.x. Additionally, `package-mode = false` in `pyproject.toml` requires Poetry 2.0+.

**Solution:**
1. Pin Poetry version in workflow: `pipx install poetry==2.3.0`
2. Update Dockerfile to install matching version: `curl -sSL https://install.python-poetry.org | python3 - --version 2.3.0`
3. Regenerate lock file locally with the pinned version

```
# In .github/workflows/deploy.yml
- name: Install Poetry
  run: pipx install poetry==2.3.0

# In Dockerfile
ARG POETRY_VERSION=2.3.0
RUN curl -sSL https://install.python-poetry.org | python3 - --version ${POETRY_VERSION}
```

**Prevention:**
- Always pin Poetry version explicitly in CI/CD and Dockerfiles
- Use `[tool.poetry]` format (not PEP 621 `[project]`) for maximum compatibility
- Document required Poetry version in README

**Context:**
- Tool/Version: Poetry 1.8.x vs 2.3.0
- OS: Linux (GitHub Actions runners)
- Other relevant context: pipx default installs latest 1.x branch

**Tags:** `poetry` `ci-cd` `github-actions` `docker`

**Source:** 260203-1500-retroactive-lessons-learned.md

---

## `--no-root` Breaks CI for Library Packages

**Problem:**
CI passes lint but pytest fails with `ModuleNotFoundError: No module named 'X'` when testing a library package.

```
E   ModuleNotFoundError: No module named 'b42_common'
ERROR tests/test_messaging.py
ERROR tests/test_models.py
```

**Root Cause:**
`poetry install --no-root` skips installing the package itself — it only installs dependencies. For **application** repos (those with `package-mode = false`), this is correct — there is no installable root package. For **library** repos (those with `packages = [...]` in pyproject.toml), `--no-root` means the library code is not on the Python path, so tests cannot import it.

**Solution:**
Remove `--no-root` from the `poetry install` step in the library's CI workflow:

```yaml
# ❌ Wrong for libraries
- name: Install dependencies
  run: poetry install --sync --no-interaction --no-root

# ✅ Correct for libraries
- name: Install dependencies
  run: poetry install --sync --no-interaction
```

**How to distinguish app vs library:**
- **App** (`package-mode = false` or no `packages = [...]`): use `--no-root`
- **Library** (`packages = [{include = "X", from = "src"}]`): omit `--no-root`

**Prevention:**
When writing CI for a new repo, check `pyproject.toml` for `packages = [...]`. If present, the repo is a library — do not use `--no-root`.

**Context:**
- Discovered when adding CI to `42bros-common` (shared library) — previously had zero CI
- All other 42Bros services are apps and correctly use `--no-root`

**Tags:** `poetry` `ci-cd` `github-actions` `library` `testing`

**Source:** log-260301-0000-branching-ci-prep.md

---

## Stale Cached Dependency Despite Correct pyproject.toml Tag

**Problem:**
A service's venv has an old version of a git-tagged dependency even though `pyproject.toml` pins the correct tag. Tests fail with `ImportError` for symbols that exist in the correct version.

```
ImportError: cannot import name 'EXCHANGE_NOTIFICATIONS' from 'b42_common'
# pyproject.toml correctly pins: tag = "v0.9.1"
# but installed version is 0.7.0 / 0.8.0
```

**Root Cause:**
`poetry install` resolves the lock file, not `pyproject.toml` directly. If `poetry.lock` was generated when the tag pointed to an older commit, or if the lock was not regenerated after the tag was updated, the installed version will be stale. `poetry install --sync` faithfully reproduces the lock file — it does not re-resolve from `pyproject.toml`.

**Solution:**
Run `poetry update <package-name>` to force re-resolution and update the lock file:

```bash
poetry update b42-common
# Re-resolves the tag, fetches latest commit for that tag, updates poetry.lock
```

Then re-run tests to confirm the correct version is installed.

**Prevention:**
- After pushing a new git tag on a shared library, always run `poetry update <lib>` in each consumer repo and commit the updated `poetry.lock`
- CI will use whatever is in `poetry.lock` — if the lock is stale, CI will reproduce the stale version

**Context:**
- Dependency: `b42-common` (private git repo, tag-pinned)
- Stale version: 0.7.0/0.8.0 | Correct version: 0.9.1
- Discovered during: 42bros-mario block-03-04 implementation

**Tags:** `poetry` `dependency` `git-tag` `lock-file` `b42-common`

**Source:** log-260301-0230-block-03-04-mario-notifications.md

---

## `poetry lock --no-update` Does Not Re-Resolve Git-Tagged Dependencies

**Problem:**
After bumping a git-tagged dependency's tag in `pyproject.toml` (e.g. `tag = "v0.10.7"` → `tag = "v0.10.8"`), running `poetry lock --no-update` fails in CI with:

```
pyproject.toml changed significantly since poetry.lock was last generated. Run poetry lock to fix the lock file.
```

All consumer repos fail CI simultaneously — they cannot install the new tag version.

**Root Cause:**
`--no-update` tells Poetry to keep all currently-resolved package versions and only update the lock file metadata (format version, hashes). It does NOT re-resolve the dependency graph. When a git-sourced dependency changes its tag reference, `--no-update` cannot satisfy the new pin because it would require re-resolving. Poetry correctly rejects this.

**Solution:**
Run a full `poetry lock` (without `--no-update`) after bumping a git-tagged dependency:

```bash
# ❌ Wrong — does not re-resolve git tags
poetry lock --no-update

# ✅ Correct — re-resolves the full dependency graph
poetry lock
```

Commit the updated `poetry.lock` alongside the `pyproject.toml` change.

**Prevention:**
- When bumping a `{git = "...", tag = "vX.Y.Z"}` dependency, always use full `poetry lock`, never `--no-update`
- `--no-update` is only safe for lock file format migrations (Poetry version bumps) where no dependency versions change

**Context:**
- Poetry 2.3.0, git-sourced private library (`b42-common`)
- Affects all consumer repos simultaneously when a shared library tag is bumped
- Discovered during 42Bros block-14-03 Phase C pin bump (2026-03-07)

**Tags:** `poetry` `dependency` `git-tag` `lock-file` `no-update`

**Source:** log-260307-1130-block-14-03-common-class-renames.md
