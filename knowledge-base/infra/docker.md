# Docker

**Category:** infra
**Last Updated:** 2026-02-18
**Entries:** 6

---

## Overview

Infrastructure patterns and troubleshooting for Docker and Docker Compose — focusing on volume behavior, deployment patterns, configuration management, and build issues.

---

## Entries

### Entry 1: Docker Compose v1 vs v2 ContainerConfig Error {#entry-1}

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

### Entry 2: Stale Container Blocking Deployment {#entry-2}

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

### Entry 3: Private Git Dependency Requires PAT in Docker Build {#entry-3}

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

---

### Entry 4: Named Volumes Mask Files Baked Into Images at the Mount Path {#entry-4}

**Problem:**
A configuration template file copied into the Docker image during build is missing at runtime. The entrypoint script reports the file as not found, even though the Dockerfile's `COPY` instruction is confirmed correct.

```
[entrypoint] WARNING: Template file not found at /home/app/.config/config.json.template
```

Listing the directory inside the running container shows files from a previous deployment but NOT the template.

**Root Cause:**
Docker named volumes completely overlay the image filesystem at the mount point. Any files placed there during `docker build` (via `COPY` or `RUN`) are invisible once the volume is mounted. On first run, Docker populates the volume from the image, but on subsequent runs the volume's existing contents are used — meaning new files added in a rebuilt image are never seen.

```yaml
volumes:
  - app_state:/home/app/.config # This overlays everything at that path
```

**Solution:**
Move any files that must survive a volume mount to a path outside the mount point:

```dockerfile
# WRONG — hidden by volume mount at /home/app/.config/
COPY config.json.template /home/app/.config/config.json.template

# CORRECT — outside the volume mount
COPY config.json.template /opt/app/config.json.template
```

The entrypoint reads from `/opt/app/` (image layer) and writes rendered output to `/home/app/.config/` (volume).

**Prevention:**

- Before placing any file via `COPY` in a Dockerfile, check docker-compose.yml for volume mounts at or above that path
- Document all volume mount points in a comment at the top of the Dockerfile
- Rule of thumb: **build-time assets go in `/opt/` or `/etc/`; runtime state goes in volume-mounted paths**
- When debugging "file not found" inside a container: `docker inspect <container> --format '{{json .Mounts}}'`

**Context:**

- Versions affected: Docker 24+, Docker Compose v2
- OS: all
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `docker` `volumes` `named-volumes` `mount-overlay` `dockerfile`

---

### Entry 5: Entrypoint Config Generation Overwrites Runtime Changes {#entry-5}

**Problem:**
After manually fixing a configuration inside a running container, restarting the container reverts the fix. Changes made at runtime are lost on every restart.

**Root Cause:**
The container's entrypoint script uses `envsubst` (or similar template rendering) to generate the config file from a template on every startup:

```bash
envsubst < /opt/app/config.template > /home/app/.config/config.json
```

This overwrites the entire config file on every start, including any changes made at runtime by the application itself, CLI tools, or manual edits.

**Solution:**
Ensure the template file contains ALL required configuration — the template must be the complete source of truth. Alternatively:

- Generate config only if it doesn't exist: `if [ ! -f config.json ]; then envsubst ...; fi`
- Use the application's native env var substitution if it supports one (avoids external `envsubst` and allows the app to manage its own config)
- Run any auto-fix/doctor commands AFTER config generation in the entrypoint

**Prevention:**

- When using config generation at container startup, the template MUST be the complete source of truth for all settings
- Never rely on runtime config modifications persisting across container restarts if the entrypoint regenerates the config
- If a tool needs to write to its own config, evaluate whether you can use its native variable substitution instead of external `envsubst`

**Context:**

- Versions affected: Docker (any), GNU gettext `envsubst`
- OS: all
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `docker` `entrypoint` `envsubst` `config-generation` `config-persistence`

---

### Entry 6: Read-Only Config Mounts Prevent Tool Auto-Fix Features {#entry-6}

**Problem:**
A tool's auto-fix or plugin management command fails because the config file is mounted as read-only (`:ro`). The tool tries to modify its config but gets a permission/lock error:

```
Error: EBUSY: resource busy or locked, rename 'config.json.tmp' -> 'config.json'
```

**Root Cause:**
Many modern tools expect to manage their own configuration files for features like auto-enablement, plugin management, and state persistence. Read-only volume mounts prevent these capabilities.

**Solution:**
Either remove the read-only restriction to allow tool self-management, or use environment variables exclusively for tools that support full env-var configuration:

```yaml
# Instead of read-only mount:
volumes:
  - ./config.json:/app/config.json:ro

# Allow tool to manage config:
volumes:
  - ./config.json:/app/config.json

# Or use env vars only:
environment:
  - TOOL_CONFIG_SETTING=value
```

**Prevention:**

- Research whether tools expect to modify their own config files before using `:ro` mounts
- Check if tools have auto-fix or plugin management features that require write access
- Consider trade-offs between immutable config (security) vs tool autonomy (functionality)
- Use environment variables for tools that fully support env-var configuration

**Context:**

- Versions affected: Docker Compose (any)
- OS: all
- First documented: 2026-02-07
- Source: `260207-2205-lessons-learned.md`

**Tags:** `docker` `volumes` `read-only` `configuration` `tool-autonomy`

---

## Cross-References

- [infra/openclaw.md](infra/openclaw.md) — OpenClaw-specific Docker deployment patterns (entries 6-7)
- [infra/github-actions.md](infra/github-actions.md) — CI/CD Docker image building

---

## Changelog

| Date       | Change                                                            | Source                                       |
| ---------- | ----------------------------------------------------------------- | -------------------------------------------- |
| 2026-02-05 | Initial creation with 3 entries                                   | `260203-1500-retroactive-lessons-learned.md` |
| 2026-02-18 | Added entries #4-6 (volumes, entrypoint config, read-only mounts) | Multiple TMP sources                         |
