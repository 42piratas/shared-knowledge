# Docker

Infrastructure patterns and troubleshooting for Docker and Docker Compose.

---

## Docker Compose v1 vs v2 ContainerConfig Error

**Problem:**
Deployment fails with cryptic error when running `docker-compose up`.

```
KeyError: 'ContainerConfig'
```

**Root Cause:**
Legacy `docker-compose` (v1.29.2, Python-based) is incompatible with newer Docker image formats. The v1 Python implementation cannot handle certain metadata fields in images built with newer Docker versions.

**Solution:**
Always use `docker compose` (v2, Go-based plugin) instead of `docker-compose`:

```bash
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
- Note: Legacy v1 is Python-based, v2 is Go-based plugin

**Tags:** `docker` `docker-compose` `deployment`

---

## Stale Container Blocking Deployment

**Problem:**
New deployment fails because old container with corrupted name is blocking.

```
Conflict. The container name "/3813f7a3c5a8_servicename" is already in use
```

**Root Cause:**
A previous container with a corrupted or mismatched name persists after a failed deployment, preventing the new container from starting with the expected name.

**Solution:**
Add cleanup step before starting new container:

```bash
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
- Note: Happens after interrupted deployments or OOM kills

**Tags:** `docker` `deployment` `cleanup`

---

## Private Git Dependency Requires PAT in Docker Build

**Problem:**
Docker build fails when installing private GitHub repository as dependency.

```
ERROR: Could not find a version that satisfies the requirement package @ git+https://github.com/org/private-repo.git
```

**Root Cause:**
pip cannot authenticate to private GitHub repositories during Docker build without credentials.

**Solution:**
Pass GitHub PAT as build argument and configure git to use it:

```dockerfile
# In Dockerfile
ARG GH_PAT
RUN git config --global url."https://${GH_PAT}@github.com/".insteadOf "https://github.com/" \
    && poetry install --no-root --only main \
    && git config --global --unset url."https://${GH_PAT}@github.com/".insteadOf
```

```yaml
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
- Note: PAT needs `repo` scope for private repos

**Tags:** `docker` `github` `private-repository` `authentication`
