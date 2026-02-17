# GitHub Actions

**Category:** infra
**Last Updated:** 2026-02-18
**Entries:** 7

---

## Overview

Lessons learned for GitHub Actions CI/CD workflows — focusing on secrets management, pipeline ordering, environment variable handling, and debugging strategies.

---

## Entries

### Entry 1: Secrets Typo Causes Silent Deployment Failure {#entry-1}

**Problem:**
GitHub Actions workflow fails silently or with cryptic Vault connection errors.

```
connection refused: localhost:8200
```

**Root Cause:**
A typo in GitHub repository secrets (e.g., `VAULT_ADDRVAULT_ADDR` instead of `VAULT_ADDR`) causes the environment variable to be undefined. The workflow defaults to localhost, failing to reach the actual Vault server.

**Solution:**

1. Audit all repository secrets for typos
2. Delete incorrect secrets
3. Re-add with correct names
4. Add validation step in workflow

```yaml
- name: Validate secrets
  run: |
    if [ -z "${{ secrets.VAULT_ADDR }}" ]; then
      echo "ERROR: VAULT_ADDR secret is missing"
      exit 1
    fi
```

**Prevention:**

- Use consistent naming conventions across repos
- Add secret validation step at start of workflows
- Document required secrets in repository README
- Periodically audit secrets across all repos

**Context:**

- Versions affected: GitHub Actions (all)
- OS: N/A (cloud CI/CD)
- First documented: 2026-02-03
- Source: `260203-1500-retroactive-lessons-learned.md`

**Tags:** `github-actions` `secrets` `debugging`

---

### Entry 2: `workflow_run` with Multiple Workflows Uses OR Logic, Not AND {#entry-2}

**Problem:**
A CI/CD pipeline split into separate workflows (CI, Docker image build, CD) consistently deploys before the Docker image build completes. CD finishes while the build is still running, deploying the old image.

The CD trigger was configured as:

```yaml
on:
  workflow_run:
    workflows: ["CI - Continuous Integration", "Build OpenClaw Docker Image"]
    types:
      - completed
```

**Root Cause:**
GitHub Actions `workflow_run` with multiple workflows listed fires when **ANY** of the listed workflows completes — it uses OR logic, not AND. There is no native way to wait for multiple workflows to all complete before triggering a downstream workflow.

Sequence: Push triggers CI and Build in parallel → CI completes in ~1 minute → CD triggers immediately → CD deploys old image → Build completes ~5 minutes later (but CD already ran).

**Solution:**
Consolidate into a single workflow file with explicit job dependencies using `needs`:

```yaml
jobs:
  ci:
    # validation steps

  check-changes:
    needs: ci
    # detect if specific files changed

  build-image:
    needs: check-changes
    if: needs.check-changes.outputs.files_changed == 'true'
    # build Docker image

  deploy:
    needs: [ci, build-image]
    if: |
      always() &&
      needs.ci.result == 'success' &&
      (needs.build-image.result == 'success' || needs.build-image.result == 'skipped')
    # deploy to production
```

The `deploy` job's `if` condition requires CI to succeed AND the build to either succeed or be skipped (when no files changed). The `if: always()` is required when a `needs` job might be skipped; without it, the dependent job is also skipped by default.

**Prevention:**

- Never use separate workflows when one workflow's output (artifact, Docker image, package) is consumed by another — use a single workflow with dependent jobs instead
- When you must use `workflow_run`, understand that it fires on ANY completion, not ALL completions
- Document the pipeline execution order in comments at the top of the workflow file

**Context:**

- Versions affected: GitHub Actions (all)
- OS: N/A (cloud CI/CD)
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `github-actions` `ci-cd` `workflow_run` `race-condition` `pipeline-ordering`

---

### Entry 3: Verify Secrets Are Populated, Not Just Present {#entry-3}

**Problem:**
After a successful deployment, the application returns HTTP 401 authentication errors from AI providers. Inspecting the `.env` file on the server shows keys are present but empty:

```
GEMINI_API_KEY=
ANTHROPIC_API_KEY=
```

**Root Cause:**
GitHub Actions returns an empty string for secrets that are not set AND for secrets set to an empty string. The `-z` check catches truly unset secrets, but the validation list didn't include all required secrets (some were considered "optional"). The deployment "succeeds" — containers start and health checks pass — but the application fails when actually trying to use the empty credentials.

**Solution:**
Add ALL required secrets to the validation step. Additionally, verify `.env` values are non-empty on the server after deployment:

```yaml
- name: Validate Required Secrets
  run: |
    MISSING=()
    # Infrastructure secrets
    [ -z "${{ secrets.VAULT_ADDR }}" ] && MISSING+=("VAULT_ADDR")
    # Application secrets — include ALL, not just "infrastructure" ones
    [ -z "${{ secrets.GEMINI_API_KEY }}" ] && MISSING+=("GEMINI_API_KEY")
    [ -z "${{ secrets.ANTHROPIC_API_KEY }}" ] && MISSING+=("ANTHROPIC_API_KEY")
    if [ ${#MISSING[@]} -gt 0 ]; then
      echo "❌ Missing secrets: ${MISSING[*]}"
      exit 1
    fi
```

Post-deployment server-side check for empty values:

```bash
if grep -q "^GEMINI_API_KEY=$" .env; then
  echo "⚠️ GEMINI_API_KEY is set but EMPTY"
fi
```

**Prevention:**

- Add ALL secrets required for the application to function to the validation step — not just infrastructure secrets
- When onboarding a new service that requires API keys, explicitly add them to the workflow validation
- Consider a post-deployment health check that tests external service connectivity
- Check for empty VALUES, not just missing keys

**Context:**

- Versions affected: GitHub Actions (all)
- OS: N/A (cloud CI/CD)
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`, `260208-2207-lessons-learned-phase6-deployment.md`

**Tags:** `github-actions` `secrets` `env-vars` `deployment-validation`

---

### Entry 4: CI/CD Environment Variable Gap — New Services Missing from .env Generation {#entry-4}

**Problem:**
Dashboard deployment fails because environment variables required by a new service are missing from the server's `.env` file, even though GitHub Secrets exist. Docker containers fail to start or deployment rolls back with no clear error about missing variables.

**Root Cause:**
The CI/CD workflow's "Create .env file" step only includes variables from the original services. When a new service is added (e.g., a dashboard requiring `NEXTAUTH_SECRET`, `DASHBOARD_AUTH_USER`), its variables are not added to the `.env` heredoc. Similarly, conditional image tags (e.g., OpenClaw image tag) may reference a newly-built image that doesn't exist when no files changed.

**Solution:**

1. Add all new service variables to the `.env` creation step
2. Use conditional image tag logic based on whether the image was actually built:

```yaml
- name: Create .env file
  run: |
    # Legacy variables
    echo "LITELLM_MASTER_KEY=${{ secrets.LITELLM_MASTER_KEY }}" > .env
    # New service variables — ADD WHEN ADDING NEW SERVICES
    echo "DASHBOARD_AUTH_USER=${{ secrets.DASHBOARD_AUTH_USER }}" >> .env
    echo "NEXTAUTH_SECRET=${{ secrets.NEXTAUTH_SECRET }}" >> .env
    # Conditional image tags
    if [ "${{ needs.check-changes.outputs.image_changed }}" = "true" ]; then
      echo "IMAGE_TAG=main-$(git rev-parse --short HEAD)" >> .env
    else
      echo "IMAGE_TAG=latest" >> .env
    fi
```

**Prevention:**

- When adding a new service, always update the CI/CD `.env` creation step AND the secrets validation step
- Use a checklist: new service → add GitHub Secrets → add to validation → add to `.env` generation → add to docker-compose
- Use conditional logic for image tags based on actual build status — never assume an image was built
- Validate image existence before deployment: `docker manifest inspect $IMAGE`

**Context:**

- Versions affected: GitHub Actions (all)
- OS: N/A (cloud CI/CD)
- First documented: 2026-02-08
- Source: `260208-2207-lessons-learned-phase6-deployment.md`

**Tags:** `github-actions` `env-vars` `ci-cd` `deployment` `docker-images`

---

### Entry 5: Always Read Actual CI/CD Error Logs Before Diagnosing {#entry-5}

**Problem:**
Multiple hours spent fixing real but secondary issues (TailwindCSS v4, Prisma v7, dockerode bundling) while the primary failure — missing source files — was visible in the CI/CD logs the entire time. The error `Module not found: Can't resolve '@/lib/data'` was interpreted as a webpack configuration problem rather than a missing file problem.

**Root Cause:**
Confirmation bias. Because the local build had different errors initially, those were assumed to be the same errors in CI/CD. The CI/CD logs were not carefully read until the user explicitly provided them.

**Solution:**
When CI/CD fails, read the EXACT error message from CI/CD logs FIRST:

```bash
gh run view <run-id> --log-failed
```

Then match the error to the correct category:

- `Module not found` → Check if the file exists in git (`git ls-files`)
- `Cannot resolve` → Check path aliases AND file existence
- `Build failed` → Read the specific compilation error, not just the exit code

**Prevention:**

- First response to any CI/CD failure: read the actual log output, not the summary annotation
- If local build succeeds but CI/CD fails: the difference is almost always about what's in git vs what's on disk — check `git status`, `git ls-files`, and `git check-ignore` before changing any configuration
- Never assume a CI/CD error matches a local error without verifying

**Context:**

- Versions affected: GitHub Actions (all), any CI/CD platform
- OS: N/A
- First documented: 2026-02-09
- Source: `260209-2118-lessons-learned-gitignore-docker-build.md`

**Tags:** `github-actions` `ci-cd` `debugging` `root-cause-analysis`

---

### Entry 6: Template Files with Variable Placeholders Cannot Be Validated as Their Target Format {#entry-6}

**Problem:**
A CI pipeline step validated JSON configuration files using `jsonlint`. After renaming a config to `.json.template` (processed by `envsubst` at runtime), CI validation fails because `${VARIABLE_NAME}` placeholders are not valid JSON:

```json
{
  "botToken": "${TELEGRAM_BOT_TOKEN}",
  "apiKey": "${LITELLM_MASTER_KEY}"
}
```

**Root Cause:**
Template files are, by definition, not valid instances of their target format. They contain placeholder syntax that will be replaced at rendering time. Running a format-specific linter on a template will always fail.

**Solution:**
Remove the template from format-specific validation in CI. If explicit template validation is needed, render it with dummy values first:

```bash
export TELEGRAM_BOT_TOKEN=dummy LITELLM_MASTER_KEY=dummy
envsubst < template.json | jsonlint -q
```

**Prevention:**

- Never add template files to format-specific linting steps without a rendering step first
- Name template files distinctly (`.template`, `.j2`, `.tpl`) so they stand out in CI configuration
- This applies to any template system: Jinja2, Go templates, Helm charts, envsubst, confd, etc.

**Context:**

- Versions affected: jsonlint (npm), GNU gettext `envsubst`, any CI linting
- OS: all
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `github-actions` `ci` `templates` `jsonlint` `validation`

---

### Entry 7: Validate Lint and Tests Locally Before Pushing to CI/CD {#entry-7}

**Problem:**
Multiple consecutive CI/CD failures (7+ failed runs) caused by lint errors (unused imports, duplicate dict keys) and test failures (changed return types breaking assertions). Each fix attempt was pushed to GitHub, consuming CI credits and adding ~3-5 minutes per cycle. Total waste: ~30 minutes of CI time and unnecessary git history noise.

**Root Cause:**
Code was pushed without local validation. The development environment lacked a local Python virtualenv with `ruff` and `pytest` installed, so lint/test checks were only happening in CI. Each error was discovered only after GitHub Actions ran, creating a slow feedback loop.

**Solution:**
Before pushing any change to a repo with CI gates:

1. **Lint locally:** `poetry run ruff check .` (or install ruff globally via `pipx install ruff`)
2. **Run tests locally:** `poetry run pytest`
3. **If no local env exists**, at minimum do a syntax + import check:
   ```bash
   python3 -c "import ast; ast.parse(open('path/to/changed_file.py').read())"
   ```
4. **For return type changes**, grep callers and tests before pushing:
   ```bash
   grep -r "function_name" tests/
   ```

**Prevention:**

- Set up a local virtualenv per service: `cd service && poetry install`
- If local env is impractical, use `pipx install ruff` for global linting
- Never change a public function's return type without searching for all callers and updating tests in the same commit
- Treat each CI/CD push as expensive — batch fixes into a single validated commit rather than incremental guesses
- When fixing CI failures: read the FULL error output, fix ALL issues found, verify locally, push once

**Context:**

- Versions affected: Any CI/CD with lint + test gates
- OS: all
- First documented: 2026-02-17
- Source: Session `260217-1842-nabbit-phase13-confidence-transitions`

**Tags:** `github-actions` `ci-cd` `local-validation` `ruff` `pytest` `workflow-efficiency`

---

## Cross-References

- [infra/docker.md](infra/docker.md) — Docker build patterns and entrypoint config issues
- [infra/openclaw.md](infra/openclaw.md) — OpenClaw CI/CD image building
- [tooling/git.md](tooling/git.md) — .gitignore patterns causing CI/CD build failures

---

## External Resources

- [GitHub Actions Documentation — workflow_run](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run)
- [GitHub Actions Documentation — Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

---

## Changelog

| Date       | Change                                                                     | Source                                       |
| ---------- | -------------------------------------------------------------------------- | -------------------------------------------- |
| 2026-02-05 | Initial creation with 1 entry                                              | `260203-1500-retroactive-lessons-learned.md` |
| 2026-02-18 | Added entries #2-6 (workflow_run, secrets, env vars, debugging, templates) | Multiple TMP sources                         |
| 2026-02-17 | Added entry #7 (validate locally before pushing)                           | Session 260217-1842                          |
