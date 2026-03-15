# GitHub Actions

**Category:** infra
**Last Updated:** 2026-03-15
**Entries:** 12

**Last Updated:** 2026-02-20

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

### Entry 8: VPC-Restricted Vault Breaks Runner-Side Auth — Move to Server-Side {#entry-8}

**Problem:**
All CI/CD deploy jobs fail with a timeout after Vault's firewall is restricted to VPC-only traffic:

```
Error writing data to auth/approle/login: Put "http://{VAULT_HOST}:8282/v1/auth/approle/login": dial tcp {IP}:8282: i/o timeout
```

**Root Cause:**
The deploy workflow authenticated with Vault from the GitHub Actions runner (internet-facing). After a correct security hardening commit restricted Vault's port to the VPC CIDR (`10.x.x.x/16`), the runner can no longer reach Vault. This affects all services simultaneously.

Timeline signature: last successful deploy succeeds, then all subsequent deploys fail after `terraform apply` propagates the firewall rule.

**Solution:**
Move all Vault authentication and secret fetching inside the SSH deploy step. The droplet is inside the VPC and can reach Vault directly. The `appleboy/ssh-action` `envs:` parameter injects secrets into the remote shell session without exposing them in logs.

**Before (broken — runner-side):**

```yaml
- name: Install Vault CLI
  run: |
    wget https://releases.hashicorp.com/vault/1.9.0/vault_1.9.0_linux_amd64.zip
    unzip vault_1.9.0_linux_amd64.zip
    sudo mv vault /usr/local/bin/

- name: Authenticate with Vault
  env:
    VAULT_ADDR: ${{ secrets.VAULT_ADDR }}   # public URL — now blocked
    VAULT_ROLE_ID: ${{ secrets.VAULT_ROLE_ID }}
    VAULT_SECRET_ID: ${{ secrets.VAULT_SECRET_ID }}
  run: |
    TOKEN=$(vault write -field=token auth/approle/login ...)
    echo "VAULT_TOKEN=$TOKEN" >> $GITHUB_ENV

- name: Fetch secrets from Vault
  env:
    VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
    VAULT_TOKEN: ${{ env.VAULT_TOKEN }}
  run: |
    MY_SECRET=$(vault kv get -field=my_secret secret/myservice)
    echo "MY_SECRET=$MY_SECRET" >> $GITHUB_ENV

- name: SSH and deploy
  uses: appleboy/ssh-action@master
  with:
    ...
    script: |
      echo "MY_SECRET=${{ env.MY_SECRET }}" > .env
```

**After (fixed — server-side):**

```yaml
# Delete: Install Vault CLI, Authenticate with Vault, Fetch secrets from Vault steps

- name: SSH and deploy
  uses: appleboy/ssh-action@master
  with:
    host: ${{ secrets.DROPLET_HOSTNAME }}
    username: ${{ secrets.DROPLET_USER }}
    key: ${{ secrets.DEPLOY_KEY_PRIVATE }}
    envs: VAULT_ROLE_ID,VAULT_SECRET_ID,GH_PAT,GITHUB_ACTOR,IMAGE_NAME
    script: |
      set -e

      # Install Vault CLI on server (idempotent)
      if ! command -v vault &> /dev/null; then
        apt-get update -qq && apt-get install -y -qq unzip  # unzip may not be installed
        wget -q https://releases.hashicorp.com/vault/1.9.0/vault_1.9.0_linux_amd64.zip
        unzip -q vault_1.9.0_linux_amd64.zip
        mv vault /usr/local/bin/vault
        rm vault_1.9.0_linux_amd64.zip
      fi

      # Authenticate from inside the VPC (hardcode private IP — do NOT use public URL)
      export VAULT_ADDR="http://{VPC_PRIVATE_IP}:8282"
      VAULT_TOKEN=$(vault write -field=token auth/approle/login \
        role_id="$VAULT_ROLE_ID" \
        secret_id="$VAULT_SECRET_ID")
      [ -z "$VAULT_TOKEN" ] && echo "ERROR: Vault auth failed" && exit 1
      export VAULT_TOKEN

      # Fetch secrets server-side
      MY_SECRET=$(vault kv get -field=my_secret secret/myservice)

      # Write .env (flush-left heredoc — indentation causes leading spaces in .env)
      cat > .env <<EOF
      MY_SECRET=$MY_SECRET
      EOF

      # Pull image and restart
      echo "$GH_PAT" | docker login ghcr.io -u "$GITHUB_ACTOR" --password-stdin
      docker pull ghcr.io/$IMAGE_NAME:latest
      docker compose down --remove-orphans 2>/dev/null || true
      docker compose up -d
```

**Key implementation details:**

- `envs:` in `appleboy/ssh-action` lists variable NAMES (not values). They are injected as environment variables into the remote shell, so `$VAULT_ROLE_ID` is available server-side without ever appearing in workflow logs.
- `VAULT_ADDR` must be the **VPC-internal private IP** of the Vault server (e.g., `http://10.x.x.x:8282`), hardcoded in the script. Do NOT use `${{ secrets.VAULT_ADDR }}` — that secret still holds the public URL which is now firewalled.
- `GITHUB_ACTOR` is an automatic GitHub Actions variable — include in `envs:` for `docker login -u "$GITHUB_ACTOR"`.
- `IMAGE_NAME` is set in the workflow-level `env:` block — include in `envs:` for `docker pull`.
- The SCP step (copying `docker-compose.yml`) is unaffected — it uses SSH port 22, which is not VPC-restricted.
- **Heredoc indentation warning:** Content inside `cat > .env <<EOF` must be flush-left. Indented heredoc content produces leading whitespace in `.env` values, breaking env var parsing. Use `<<-EOF` for tab-tolerant heredocs or keep content flush-left.

**Diagnosis checklist when all deploys suddenly fail at Vault auth:**

1. Check recent IAC/Terraform commits for firewall changes (`git log --oneline 42bros-iac/`)
2. Confirm the error is `i/o timeout` (network blocked) not `403` (wrong credentials)
3. Confirm the droplet can reach Vault internally: `ssh {droplet} "curl -s http://{VPC_IP}:8282/v1/sys/health"`
4. Check if Terraform was recently applied (`terraform apply`) after a firewall rule change

**Prevention:**

- When restricting Vault/internal-service firewall rules to VPC-only, audit all CI/CD workflows immediately — any that call Vault from the runner will break on next deploy.
- Document the VPC-internal Vault address in the infra playbook so it's always available without secrets.
- Never use `${{ secrets.VAULT_ADDR }}` inside SSH scripts — hardcode the private IP.

**Context:**

- Versions: GitHub Actions (all), `appleboy/ssh-action` (all), HashiCorp Vault 1.9.0, DigitalOcean VPC
- First documented: 2026-02-20
- Source: Session `260220-1930-toadette-fix-daisy-tests-vault-ci-plan`

**Tags:** `github-actions` `vault` `vpc` `firewall` `ci-cd` `secrets` `appleboy-ssh-action` `server-side-auth`

---

### Entry 9: CI/CD Pipeline Rewrite Drops Existing Credentials — Silent Failure via Bare Exception Handlers {#entry-9}

**Problem:**
After rewriting a deployment workflow for an architecture migration (e.g., monolith → microservices), the new API container returns empty data for all endpoints. No errors in logs. Database contains valid records. All upstream services are alive. Multiple sessions (6+ commits across 3 agent sessions) fail to identify the root cause, instead patching code-level symptoms (SSL config, SQL queries, missing packages, timestamp mappings).

```
HTTP 200 — {"triggers": [], "total": 0}
```

**Root Cause:**
The new CI/CD workflow was written from scratch and only fetched a subset of credentials from the secret manager. The **database credentials** that the old workflow fetched were completely omitted. The API code had a silent fallback chain:

1. Try PostgreSQL → `os.getenv("POSTGRES_URL")` returns `None` → exception
2. Bare `except Exception:` → fall back to in-memory cache
3. Cache is empty (drained by an archival service every 60s by design)
4. Return empty results with HTTP 200

The bare exception handler masked the missing configuration completely.

**Solution:**

1. Diff the old and new workflows side by side — identify all credential-fetching steps that were dropped
2. Add the missing credential fetching to the new workflow:
   ```yaml
   PG_HOST=$(vault kv get -field=postgres_host secret/common)
   PG_PORT=$(vault kv get -field=postgres_port secret/common)
   # ... all database credentials
   ```
3. Add the constructed connection string to the deployment `.env` and `docker-compose` environment
4. Add logging to all exception handlers so configuration failures are visible

**Prevention:**

1. **Migration checklist — environment variable audit:** When rewriting a deployment pipeline, **diff the old and new workflows side by side** before the first deploy:
   - [ ] All secret/credential fetching steps carried over
   - [ ] All environment variables present in the new container config
   - [ ] Database connection strings constructed and passed through

2. **Never use bare exception handlers without logging:**

   ```python
   # ❌ Hides configuration errors
   except Exception:
       return fallback_result

   # ✅ Failure is visible
   except Exception as e:
       logger.error(f"Primary data source failed: {e}", exc_info=True)
       return fallback_result
   ```

3. **Validate required environment variables at startup, not at request time:**

   ```python
   REQUIRED_ENV = ["POSTGRES_URL", "VALKEY_HOST", "JWT_SECRET_KEY"]
   missing = [v for v in REQUIRED_ENV if not os.getenv(v)]
   if missing:
       raise RuntimeError(f"Missing required env vars: {missing}")
   ```

4. **First-deploy smoke test** after any pipeline rewrite:
   ```bash
   docker exec <api-container> env | grep -E '(POSTGRES|DATABASE|VALKEY|REDIS)'
   curl -s <api-url>/health | jq  # Should show all dependencies connected
   ```

**Context:**

- Versions affected: GitHub Actions (all), any CI/CD platform
- OS: N/A (cloud CI/CD)
- First documented: 2026-02-10
- Source: `260210-1321-lessons-learned.md`

**Tags:** `github-actions` `ci-cd` `credentials` `silent-failure` `exception-handling` `migration`

---

### Entry 10: Deploy Scripts Corrupt containerd overlayfs — Two Root Causes {#entry-10}

**Problem:**
`docker pull` fails during CI deploy with one of two errors:

Error A (concurrent race):

```
failed to extract layer ... to overlayfs as "extract-NNN-XXXX ...": failed call to UtimesNanoAt for .../snapshots/NNN/fs/etc: no such file or directory
```

Error B (shared layer destruction):

```
failed to Lchown ".../snapshots/NNN/fs/usr/local/lib/python3.12/..." for UID 0, GID 0: lchown ...: no such file or directory
```

Both are fatal and non-recoverable without a full containerd state reset.

**Root Cause 1 — Concurrent deploys:**
Multiple services deploy to the same physical server. Their CI workflows run concurrently, executing `docker pull` in parallel. The containerd overlayfs snapshotter maintains two data structures that must stay in sync:

1. `/var/lib/containerd/io.containerd.metadata.v1.bolt/meta.db` — BoltDB recording snapshot IDs, parent chains, and content blob references
2. `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/` — actual filesystem directories on disk

Concurrent pulls race on these structures, leaving `meta.db` referencing snapshot parent IDs whose directories no longer exist on disk. A partial wipe recovery (snapshots only, not `meta.db`) makes things worse — `meta.db` retains stale references, so the very next pull fails immediately looking for a parent snapshot that was wiped.

**Root Cause 2 — `docker image prune -af` destroys shared base layers:**
Deploy scripts included `docker image prune -af` before pulling the new image. The `-a` flag removes **all unused images**, not just dangling ones. On a multi-service host, services share base image layers (e.g., `python:3.12-slim`). When service A deploys, it stops its container (making its image "unused"), then prunes — destroying base layers that service B's running image also depends on. The next deploy for service B tries to extract layers on top of a parent snapshot whose filesystem was pruned away.

Even with serialized deploys (concurrency groups), this still happens: service A's deploy prunes shared layers, then service B's deploy (queued next) fails because parent snapshots are gone.

**Solution:**

Fix 1 — Serialize deploys with `flock` on the server (primary mechanism):

Wrap the entire pull+restart block in `flock -w 300 /var/lock/deploy.lock`. This is a POSIX file lock — only one SSH session can hold it at a time. Others wait up to 300 seconds.

```bash
echo "$GH_PAT" | docker login ghcr.io -u "$ACTOR" --password-stdin
flock -w 300 /var/lock/deploy.lock bash -c '
  set -e
  echo ">>> Lock acquired — deploying SERVICE..."
  cd /root/42bros-service
  docker compose down --remove-orphans 2>/dev/null || true
  docker rm -f service 2>/dev/null || true
  docker rmi ghcr.io/org/service:latest 2>/dev/null || true
  docker container prune -f 2>/dev/null || true
  docker pull ghcr.io/org/service:latest
  docker compose up -d
  docker image prune -f 2>/dev/null || true
  echo ">>> SERVICE Deployed via GHCR!"
'
```

**Why `flock` instead of GitHub Actions concurrency groups?** GH Actions `concurrency` groups are **repo-scoped**, not global. The same group name in two different repos creates two independent groups — deploys from different repos still run concurrently. `flock` enforces serialization at the server level regardless of which repo triggered the deploy.

GH Actions `concurrency` groups (`deploy-shroom-x`, `deploy-shroom-a`) remain as a secondary guard within each repo, but `flock` is the primary cross-repo mechanism.

Fix 2 — Remove `docker image prune -af` from all deploy scripts. The correct cleanup pattern:

```bash
# Pre-pull: remove ONLY the service's own tagged image
docker rmi ghcr.io/org/service:latest 2>/dev/null || true

# Post-deploy: remove only dangling (untagged) images — safe
docker image prune -f 2>/dev/null || true
```

Never use `-a` in a deploy script. Disk reclamation for unused images is handled by a scheduled weekly `docker system prune` cron that runs when no deploys are active.

**Prevention:**

- Every deploy script MUST wrap the pull+restart block in `flock -w 300 /var/lock/deploy.lock`.
- Never use `docker image prune -a` or `docker image prune -af` in deploy scripts. Only `docker image prune -f` (dangling only).
- Never remove the `flock` wrapper. Without it, any simultaneous push to two repos on the same server will reproduce corruption.
- GH Actions `concurrency` groups are a secondary guard only — do not rely on them for cross-repo serialization.

**Context:**

- Versions affected: GitHub Actions (all), containerd with overlayfs snapshotter
- Manifests on: Ubuntu 20.04+, kernel 5.15, containerd 1.6+
- Requires: `flock` from `util-linux` (standard on Ubuntu/Debian)
- First documented: 2026-02-28
- Source: Engineer session 260228

**Tags:** `github-actions` `ci-cd` `docker` `containerd` `overlayfs` `concurrency` `flock` `deploy` `image-prune`

---

## Cross-References

- [infra/docker.md](infra/docker.md) — Docker build patterns and entrypoint config issues
- [infra/openclaw.md](infra/openclaw.md) — OpenClaw CI/CD image building
- [tooling/git.md](tooling/git.md) — .gitignore patterns causing CI/CD build failures

---

## New Env Vars in Deploy Script Are Not Automatically Picked Up

**Problem:**
A service is deployed with a new feature that reads an env var. The env var is set in Vault and manually verified to exist. But after deploy, the container doesn't have the env var — the feature silently fails.

**Root Cause:**
The deploy script (`.github/workflows/deploy.yml`) explicitly fetches secrets from Vault and writes them to `.env`. It does not dynamically discover all secrets — only the ones listed in the script are fetched. A new Vault secret is invisible to the container until someone adds it to the deploy script.

**Solution:**
Whenever you add a new env var to a service's runtime config, update the deploy script in the same PR:

1. Add a Vault fetch line: `NEW_VAR=$(vault kv get -field=new_key secret/service)`
2. Add it to the `.env` write block: `NEW_ENV_VAR=$NEW_VAR`

**Checklist when adding a new env var:**
- [ ] New Vault secret written (`vault kv patch secret/{service} key=value`)
- [ ] Deploy script updated to fetch + inject the new var
- [ ] `.env.example` updated (if repo has one)
- [ ] README env var table updated

**Prevention:**
Treat the deploy script as the canonical env var manifest for a service. Any env var the code reads must appear in the deploy script's Vault fetch + `.env` write block. Code review should cross-check these.

**Context:**
- Discovered when `TELEGRAM_MARIO_CHAT_ID` was added to Toad config but not to `deploy.yml`
- The first deploy was silent — Toad started fine but silently dropped all MARIO-routed notifications
- Required a second fix PR after the feature was already deployed

**Tags:** `github-actions` `deploy` `env-vars` `vault` `docker`

**Source:** log-260301-0230-block-03-04-mario-notifications.md

---

### Entry 12: Deploy Key Comments Must Match Service Identity — Mislabeled Keys Cause Accidental Deletion {#entry-12}

**Problem:**
After a security audit removed a key identified as stale (`daisy-deploy@42bros.xyz` from `authorized_keys` on a production server), Lens deploys immediately started failing with SSH handshake errors:

```
ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain
```

The key was not stale — it was the active Lens deploy key, mislabeled with a comment from a different service.

**Root Cause:**
The Lens deploy key was generated with the comment `daisy-deploy@42bros.xyz` instead of `lens-deploy@42bros.xyz`. When a security reviewer audited `authorized_keys` and identified the `daisy-deploy` key as stale (the actual Daisy service had been decommissioned from that server), they removed it — unknowingly deleting the Lens deploy key.

The key comment in `authorized_keys` is the only human-readable identifier. There is no technical enforcement linking a key comment to its actual purpose. Vault also stored the key with the misleading comment.

**Solution:**

1. Restore the key from Vault backup and rename the comment:
   ```bash
   # Get the key from Vault
   vault kv get -field=lens_deploy_key_public secret/lens
   # Add to authorized_keys with correct comment
   echo "ssh-ed25519 AAAA... lens-deploy@42bros.xyz" >> /root/.ssh/authorized_keys
   ```
2. Update the comment in Vault to match the service identity
3. Re-run failed deploys

**Prevention:**

- **Naming convention:** All deploy key comments MUST follow `{service}-deploy@{domain}` format. Enforce during key generation: `ssh-keygen -t ed25519 -C "{service}-deploy@42bros.xyz"`
- **Before deleting ANY key from `authorized_keys`:** Cross-reference against Vault (`vault kv get -field={service}_deploy_key_public secret/{service}`) to verify the key's actual identity — do NOT rely on the comment alone
- **Key audit procedure:** Dump all keys, match each against Vault entries by public key value (not comment), then identify genuinely orphaned keys
- **When rotating keys:** Update the Vault entry simultaneously to keep comment and actual usage in sync

**Context:**

- Versions affected: OpenSSH (all), any system using SSH `authorized_keys`
- OS: Linux (Ubuntu)
- First documented: 2026-03-15
- Source: Block 31-01/31-02 security review — deploy key incident

**Tags:** `ssh` `deploy-keys` `authorized_keys` `security-audit` `key-management` `naming-conventions`

---

## External Resources

- [GitHub Actions Documentation — workflow_run](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run)
- [GitHub Actions Documentation — Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

---

## Changelog

| Date       | Change                                                                        | Source                                       |
| ---------- | ----------------------------------------------------------------------------- | -------------------------------------------- |
| 2026-02-05 | Initial creation with 1 entry                                                 | `260203-1500-retroactive-lessons-learned.md` |
| 2026-02-18 | Added entries #2-6 (workflow_run, secrets, env vars, debugging, templates)    | Multiple TMP sources                         |
| 2026-02-17 | Added entry #7 (validate locally before pushing)                              | Session 260217-1842                          |
| 2026-02-20 | Added entry #8 (server-side Vault auth for VPC-restricted environments)       | Session 260220-1930                          |
| 2026-02-20 | Entry #8: Added unzip dependency note (discovered during iter 1.6)            | Session 260220-2030                          |
| 2026-02-22 | Added entry #9 (pipeline rewrite drops credentials, bare exception hiding)    | `260210-1321-lessons-learned.md`             |
| 2026-02-28 | Added entry #10 (concurrent deploys corrupt containerd overlayfs)             | Engineer session 260228                      |
| 2026-02-28 | Entry #10: Added root cause 2 (docker image prune -af destroys shared layers) | Engineer session 260228                      |
| 2026-02-28 | Entry #10: flock replaces GH Actions concurrency groups as primary mechanism  | Engineer session 260228                      |
| 2026-03-01 | Added entry #11 (new env vars in deploy script not auto-picked up)            | Engineer session 260301                      |
| 2026-03-15 | Added entry #12 (mislabeled deploy key comment causes accidental deletion)    | Block 31-02 security review                 |
