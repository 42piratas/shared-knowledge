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
