# Lessons Learned: Retroactive Extraction from 42Bros Project

**Date:** 2026-02-03
**Project:** 42bros
**Agent/Model:** Claude Opus 4
**Session Focus:** Retroactive extraction of generalizable lessons from all project session logs (Nov 2025 - Feb 2026)

---

## Lessons Learned

### Lesson: Poetry Version Mismatch Between Local and CI/CD

**Category:** tooling

**Topic:** poetry

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

---

### Lesson: Docker Compose v1 vs v2 ContainerConfig Error

**Category:** infra

**Topic:** docker

**Problem:**
Deployment fails with cryptic error when running `docker-compose up`.

```
KeyError: 'ContainerConfig'
```

**Root Cause:**
Legacy `docker-compose` (v1.29.2, Python-based) is incompatible with newer Docker image formats. The v1 Python implementation cannot handle certain metadata fields in images built with newer Docker versions.

**Solution:**
Always use `docker compose` (v2, Go-based plugin) instead of `docker-compose`:

```
# ❌ WRONG (legacy v1)
docker-compose up -d

# ✅ CORRECT (v2 plugin)
docker compose up -d
```

**Prevention:**
- Update all deployment scripts to use `docker compose` (with space)
- Remove legacy `docker-compose` binary from servers
- Add version check in deploy scripts: `docker compose version`

**Context:**
- Tool/Version: docker-compose 1.29.2 vs Docker Compose v2.x
- OS: Ubuntu (DigitalOcean droplets)
- Other relevant context: Legacy v1 is Python-based, v2 is Go-based plugin

**Tags:** `docker` `docker-compose` `deployment`

---

### Lesson: Stale Container Blocking Deployment

**Category:** infra

**Topic:** docker

**Problem:**
New deployment fails because old container with corrupted name is blocking.

```
Conflict. The container name "/3813f7a3c5a8_servicename" is already in use
```

**Root Cause:**
A previous container with a corrupted or mismatched name persists after a failed deployment, preventing the new container from starting with the expected name.

**Solution:**
Add cleanup step before starting new container:

```
docker compose down --remove-orphans 2>/dev/null || true
docker rm -f servicename 2>/dev/null || true
docker container prune -f
docker compose up -d
```

**Prevention:**
- Always include cleanup step in deploy scripts
- Use `--remove-orphans` flag with `docker compose down`
- Implement container health checks to detect stuck containers

**Context:**
- Tool/Version: Docker 20.x+
- OS: All
- Other relevant context: Happens after interrupted deployments or OOM kills

**Tags:** `docker` `deployment` `cleanup`

---

### Lesson: Terraform/Vault Circular Dependency (DR Vulnerability)

**Category:** infra

**Topic:** terraform

**Problem:**
Terraform cannot plan/apply because it depends on Vault for secrets, but Vault runs on infrastructure managed by Terraform. If Vault is destroyed, Terraform is deadlocked.

```
context deadline exceeded
Error reading from Vault: connection refused
```

**Root Cause:**
Terraform Vault provider reads secrets during the planning phase. If Vault is unavailable (destroyed, unreachable), Terraform cannot even generate a plan to recreate it.

**Solution:**
1. Comment out Vault provider and all `data.vault_kv_secret_v2` blocks
2. Add defaults to Vault variables: `default = ""`
3. Store bootstrap secrets (DO token) outside Vault for DR
4. Run `terraform apply` to recreate infrastructure
5. Restore Vault from backup
6. Uncomment Vault integration and reapply

```
# In variables.tf during DR
variable "vault_role_id" {
  default = ""  # Allow empty during DR
}
```

**Prevention:**
- Store critical bootstrap secrets (cloud provider tokens) in environment variables OR separate secret manager
- Maintain offline backup of DO API token
- Document DR procedure in playbook
- Consider blue-green infrastructure for migrations

**Context:**
- Tool/Version: Terraform 1.x, Vault 1.x
- OS: All
- Other relevant context: Discovered during region migration from nyc3 to ams3

**Tags:** `terraform` `vault` `disaster-recovery` `secrets`

---

### Lesson: Vault Data on Ephemeral Storage

**Category:** secrets

**Topic:** hashicorp-vault

**Problem:**
Vault data lost when droplet is destroyed or resized via Terraform.

**Root Cause:**
Vault was configured to store data on the droplet's local disk, which is ephemeral. Running `terraform apply` to resize or replace the droplet permanently deletes all secrets.

**Solution:**
1. Create persistent block storage volume
2. Mount volume before Vault starts
3. Configure Vault to use mounted path
4. Add `prevent_destroy = true` lifecycle rule

```
resource "digitalocean_volume" "vault_data" {
  name   = "vault-data-prod"
  region = var.do_region
  size   = 10

  lifecycle {
    prevent_destroy = true
  }
}
```

**Prevention:**
- Never run stateful services on ephemeral storage
- Always use persistent volumes for databases and secret stores
- Test backup/restore before any infrastructure changes
- Use lifecycle rules to prevent accidental destruction

**Context:**
- Tool/Version: HashiCorp Vault 1.x, DigitalOcean Volumes
- OS: All
- Other relevant context: Cost is minimal (~$1/month for 10GB)

**Tags:** `vault` `digitalocean` `persistence` `storage`

---

### Lesson: Nginx SSL Chicken-and-Egg Problem

**Category:** infra

**Topic:** docker

**Problem:**
Nginx fails to start because SSL certificate paths don't exist, but Certbot can't run because Nginx isn't serving port 80 for the ACME challenge.

```
cannot load certificate "/etc/letsencrypt/live/domain.xyz/fullchain.pem": No such file or directory
```

**Root Cause:**
Nginx configuration references SSL certificate paths before Certbot has obtained them. Certbot requires HTTP access (port 80) to complete the challenge, which requires Nginx to be running.

**Solution:**
Two-stage deployment:
1. Deploy HTTP-only config first (no SSL blocks)
2. Run Certbot to obtain certificates
3. Deploy HTTPS config with SSL paths

```
# Stage 1: Deploy HTTP config
cp nginx-http-only.conf /etc/nginx/sites-available/
nginx -t && systemctl reload nginx

# Stage 2: Get certificates
certbot --nginx -d domain.xyz -d www.domain.xyz --non-interactive --agree-tos

# Stage 3: Deploy full HTTPS config
cp nginx-https.conf /etc/nginx/sites-available/
nginx -t && systemctl reload nginx
```

**Prevention:**
- Always use two-stage SSL deployment
- Keep HTTP-only configs as templates for bootstrap
- Automate the sequence in IaC provisioners

**Context:**
- Tool/Version: Nginx 1.x, Certbot 1.x
- OS: Ubuntu
- Other relevant context: Common during server migrations

**Tags:** `nginx` `ssl` `certbot` `letsencrypt`

---

### Lesson: Telegram MarkdownV2 Special Character Escaping

**Category:** tooling

**Topic:** telegram-api

**Problem:**
Telegram API rejects messages with 400 Bad Request.

```
Can't parse entities: can't find end of italic entity at byte offset 30
```

**Root Cause:**
Telegram MarkdownV2 interprets special characters as formatting markers. Underscores (`_`), asterisks (`*`), and other characters in data values (e.g., `BTC_USDT`, `EXTREME_FEAR`) break the parser.

**Solution:**
Escape all MarkdownV2 special characters in dynamic content:

```
def escape_markdown(text: str) -> str:
    """Escape Telegram MarkdownV2 special characters."""
    special_chars = ['_', '*', '[', ']', '(', ')', '~', '`', '>', '#', '+', '-', '=', '|', '{', '}', '.', '!']
    for char in special_chars:
        text = text.replace(char, f'\\{char}')
    return text

# Usage
message = f"Asset: {escape_markdown(asset_name)}"
```

**Prevention:**
- Always escape user/dynamic content in Telegram messages
- Use HTML parse mode as alternative (simpler escaping)
- Test with data containing special characters

**Context:**
- Tool/Version: Telegram Bot API
- OS: All
- Other relevant context: MarkdownV2 is stricter than original Markdown

**Tags:** `telegram` `api` `markdown` `escaping`

---

### Lesson: GitHub Secrets Typo Causes Silent Deployment Failure

**Category:** infra

**Topic:** github-actions

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

```
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
- Tool/Version: GitHub Actions
- OS: All
- Other relevant context: Discovered after weeks of silent failure

**Tags:** `github-actions` `secrets` `vault` `debugging`

---

### Lesson: Unit Tests with Mocked Data Don't Catch Schema Mismatches

**Category:** tooling

**Topic:** testing

**Problem:**
Unit tests pass but production fails with data parsing errors.

```
unsupported operand type(s) for /: 'dict' and 'float'
```

**Root Cause:**
Unit tests mocked data with flat structure `{"score": 60.0}` but production service publishes nested structure `{"score": {"value": 60.0, ...}}`. The mocked data didn't match the actual producer output.

**Solution:**
1. Create integration tests that validate against real producer output
2. Use production data samples as test fixtures
3. Add contract tests between services

```
# Use actual production data structure in tests
PRODUCTION_PEACH_REPORT = {
    "score": {
        "value": 60.15,
        "regime": "NEUTRAL_ACCUMULATION",
        "components": {...}
    },
    "microstructure": {...}
}

def test_parse_production_structure():
    result = parse_peach_report(PRODUCTION_PEACH_REPORT)
    assert result.score == 60.15
```

**Prevention:**
- Always capture real production data for test fixtures
- Implement integration tests between services
- Use contract testing (Pact, etc.) for service boundaries
- Review test data against actual API responses

**Context:**
- Tool/Version: pytest
- OS: All
- Other relevant context: Particularly important for microservices

**Tags:** `testing` `integration-tests` `schema` `mocking`

---

### Lesson: External API Geo-Blocking Requires Region Testing

**Category:** infra

**Topic:** digitalocean

**Problem:**
Service works locally but fails in production with 451 error.

```
ExchangeNotAvailable: binance GET 451 - Service unavailable from a restricted location
```

**Root Cause:**
External APIs (Binance, others) may geo-block certain regions. DigitalOcean NYC3 region is blocked by Binance due to US regulations.

**Solution:**
1. Test API connectivity from target region BEFORE deployment
2. Migrate to unblocked region (e.g., Amsterdam `ams3`)
3. Document region requirements in playbook

```
# Test from target region
ssh user@server "curl -I https://api.binance.com/api/v3/ping"
# Expected: HTTP/1.1 200 OK
# Blocked: HTTP/1.1 451 Unavailable For Legal Reasons
```

**Prevention:**
- Research API geo-restrictions before choosing cloud region
- Test API access as part of infrastructure provisioning
- Maintain list of region requirements per external service
- Have migration procedure ready for region changes

**Context:**
- Tool/Version: Binance API, DigitalOcean
- OS: All
- Other relevant context: US-based regions commonly blocked by crypto exchanges

**Tags:** `digitalocean` `api` `geo-blocking` `binance`

---

### Lesson: Ruff Linter Errors Block CI/CD Pipeline

**Category:** tooling

**Topic:** python

**Problem:**
CI/CD fails with multiple linting errors after adding new code.

```
F401 'module.function' imported but unused
F841 local variable 'x' is assigned to but never used
E701 multiple statements on one line
```

**Root Cause:**
Rapid development introduces unused imports, variables, and style violations that aren't caught until CI runs. Integration tests particularly prone to having unused fixtures.

**Solution:**
1. Run linter locally before pushing
2. Fix unused imports by removing or using `# noqa: F401`
3. Prefix intentionally unused variables with underscore: `_unused_var`
4. Split multi-statement lines

```
# For necessary but "unused" imports (like pandas_ta accessor registration)
import pandas_ta  # noqa: F401

# For intentionally unused variables
_mock_channel, _mock_properties = mock_callback_args
```

**Prevention:**
- Configure pre-commit hooks for ruff
- Run `ruff check .` before every commit
- Configure IDE to show linting errors inline
- Add linting step early in CI pipeline

**Context:**
- Tool/Version: ruff 0.2+
- OS: All
- Other relevant context: Replace flake8/pylint with ruff for speed

**Tags:** `python` `linting` `ruff` `ci-cd`

---

### Lesson: Private Git Dependency Requires PAT in Docker Build

**Category:** tooling

**Topic:** docker

**Problem:**
Docker build fails when installing private GitHub repository as dependency.

```
ERROR: Could not find a version that satisfies the requirement package @ git+https://github.com/org/private-repo.git
```

**Root Cause:**
pip cannot authenticate to private GitHub repositories during Docker build without credentials.

**Solution:**
Pass GitHub PAT as build argument and configure git to use it:

```
# In Dockerfile
ARG GH_PAT
RUN git config --global url."https://${GH_PAT}@github.com/".insteadOf "https://github.com/" \
    && poetry install --no-root --only main \
    && git config --global --unset url."https://${GH_PAT}@github.com/".insteadOf

# In docker-compose.yml
services:
  myservice:
    build:
      context: .
      args:
        - GH_PAT=${GH_PAT}
```

**Prevention:**
- Store `GH_PAT` in GitHub repository secrets
- Always unset git credential config after install
- Consider publishing private packages to private registry
- Document PAT requirements in README

**Context:**
- Tool/Version: Docker, pip, Poetry
- OS: All
- Other relevant context: PAT needs `repo` scope for private repos

**Tags:** `docker` `github` `private-repository` `authentication`

---

### Lesson: GPG Signing Causes Git Commands to Hang

**Category:** tooling

**Topic:** git

**Problem:**
`git tag` or `git commit` hangs indefinitely with no output.

**Root Cause:**
GPG signing is enabled globally, and the GPG agent is waiting for a passphrase prompt that never displays in non-interactive environments (CI, AI assistants, scripts).

**Solution:**
Disable GPG signing for the affected operations:

```
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
- Tool/Version: git 2.x
- OS: macOS, Linux
- Other relevant context: Common when using AI coding assistants

**Tags:** `git` `gpg` `signing` `automation`

---

## Session Notes

This retroactive extraction covers lessons from approximately 100+ session logs spanning November 2025 to February 2026. The lessons are focused on infrastructure, tooling, and development patterns that apply broadly beyond the 42Bros project.

Key themes identified:
1. **Version pinning is critical** - Poetry, Docker Compose, Python versions must be explicit
2. **CI/CD requires validation** - Don't assume workflows succeed; add explicit checks
3. **Stateful services need persistent storage** - Never use ephemeral disk for databases/secrets
4. **Integration tests catch what unit tests miss** - Mock data doesn't reflect production schemas
5. **External dependencies have hidden requirements** - Geo-blocking, rate limits, auth methods

---

## Checklist Before Submission

- [x] All lessons are generalizable (not project-specific)
- [x] No sensitive data (API keys, internal URLs, passwords)
- [x] Error messages are included where applicable
- [x] Solutions are complete and actionable
- [x] Categories and topics are correctly assigned
- [x] File saved as `YYMMDD-HHMM-lessons-learned.md`
