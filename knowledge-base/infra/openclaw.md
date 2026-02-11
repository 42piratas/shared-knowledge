# OpenClaw

**Category:** infra
**Last Updated:** 2026-02-18
**Entries:** 7

---

## Overview

Lessons learned from deploying and configuring the OpenClaw agent platform, focusing on Docker deployment patterns, resource management, configuration, and plugin systems.

---

## Quick Reference

| Task | Command/Pattern |
|------|-----------------|
| Check logs | `docker compose logs openclaw --tail 100` |
| Verify Node version | `docker exec openclaw node --version` |
| Check config | `docker exec openclaw cat /home/openclaw/.openclaw/openclaw.json` |
| Monitor memory | `docker stats openclaw` |
| Plugin status | `node scripts/run-node.mjs plugins status` |

---

## Entries

### Entry 1: Runtime npm Install Causes Resource Exhaustion {#entry-1}

**Problem:**
Server becomes completely unresponsive with 100% CPU usage immediately after OpenClaw container deployment. SSH times out, DigitalOcean Console unresponsive, requires power cycle. Server enters crash-restart loop with `restart: unless-stopped` policy.

**Root Cause:**
OpenClaw was installed at container runtime via `npm install -g openclaw@latest` in entrypoint script. OpenClaw has native modules requiring compilation which monopolizes CPU on low-spec droplets (1-2 vCPU). Combined with auto-restart policy, this creates an infinite crash loop. OpenClaw is NOT meant to be installed via runtime npm install — the official deployment builds a custom Docker image with dependencies pre-compiled.

**Solution:**
Build OpenClaw Docker image in CI/CD with dependencies pre-installed:

```dockerfile
FROM node:22-bookworm
RUN npm install -g pnpm
WORKDIR /opt/openclaw
RUN git clone https://github.com/openclaw/openclaw.git .
RUN pnpm install --frozen-lockfile
RUN pnpm build
COPY openclaw.json.template /opt/openclaw/
COPY entrypoint.sh /opt/openclaw/
RUN useradd -m -u 1000 openclaw && chown -R openclaw:openclaw /opt/openclaw
USER openclaw
WORKDIR /home/openclaw
ENTRYPOINT ["/opt/openclaw/entrypoint.sh"]
CMD ["node", "/opt/openclaw/dist/index.js", "gateway"]
```

Reference the pre-built image in docker-compose.yml instead of running npm install at runtime. Always add resource limits:

```yaml
openclaw:
  image: ghcr.io/{org}/{repo}/openclaw:latest
  deploy:
    resources:
      limits:
        memory: 1536M
        cpus: '1.0'
      reservations:
        memory: 768M
```

**Prevention:**
- NEVER run `npm install`, `pip install`, or `apt-get install` in production container entrypoints
- Always build Docker images in CI/CD with dependencies pre-compiled
- Add CPU/memory limits to all Docker services
- Use `restart: "no"` during testing to prevent crash loops
- Test deployments incrementally (one service at a time)

**Context:**
- Versions affected: OpenClaw (any version with native modules)
- OS: Linux (DigitalOcean 2GB droplet, 1 vCPU)
- First documented: 2026-02-07
- Source: `260207-2000-lessons-learned-openclaw-deployment.md`

**Tags:** `openclaw` `docker` `resource-exhaustion` `npm` `deployment`

---

### Entry 2: OpenClaw Memory Requirements {#entry-2}

**Problem:**
Need to understand OpenClaw's actual memory requirements to properly size infrastructure and avoid out-of-memory failures.

**Root Cause:**
OpenClaw's official Fly.io guide states: "512MB is too small. 1GB may work but can OOM under load. 2GB is recommended."

**Solution:**

| Droplet Size | Available RAM | OpenClaw Allocation | Verdict |
|:-------------|:--------------|:--------------------|:--------|
| 1GB droplet  | ~800MB        | 512MB (limit)       | ❌ Too small, will OOM |
| 2GB droplet  | ~1.6GB        | 1GB-1.5GB (limit)   | ✅ Recommended minimum |
| 4GB droplet  | ~3.2GB        | 2GB-3GB (limit)     | ✅ Comfortable margin |

Additional memory considerations:
- OpenClaw builds UI assets on first start (temporary spike)
- Multiple agents or plugins increase memory usage
- Browser automation (if enabled) requires additional memory
- Leave 20% free memory for system and other services

**Prevention:**
- Size infrastructure based on documented requirements (2GB minimum)
- Add memory limits and reservations to prevent one service consuming all resources
- Monitor memory usage: `docker stats`

**Context:**
- Versions affected: OpenClaw (all versions)
- OS: all
- First documented: 2026-02-07
- Source: `260207-2000-lessons-learned-openclaw-deployment.md`

**Tags:** `openclaw` `memory` `infrastructure-sizing`

---

### Entry 3: Node.js 22+ Required {#entry-3}

**Problem:**
OpenClaw fails to start with cryptic error messages about missing Node.js features or module resolution failures.

**Root Cause:**
OpenClaw requires Node.js 22 or higher. Using older versions (18, 20) causes compatibility issues.

**Solution:**

```dockerfile
# Always use Node.js 22
FROM node:22-bookworm
# Not node:20, not node:18, not node:latest
```

```bash
node --version  # Should output v22.x.x or higher
```

**Prevention:**
- Always check official documentation for minimum version requirements
- Use specific version tags in Dockerfiles (e.g., `node:22-bookworm`)
- Test with minimum required version, not just what you have locally

**Context:**
- Versions affected: Node.js < 22
- OS: all
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `openclaw` `nodejs` `version-requirements`

---

### Entry 4: Two-Layer Channel Enablement System {#entry-4}

**Problem:**
OpenClaw Telegram integration shows "Telegram configured, not enabled yet" despite `channels.telegram.enabled: true` with a valid bot token. Running `openclaw doctor --fix` does not resolve the issue. Bot is completely non-functional.

**Root Cause:**
OpenClaw uses a two-layer enablement system:
1. **Channel configuration** (`channels.telegram.enabled: true`) — defines transport settings
2. **Plugin runtime** (`plugins.entries.telegram.enabled: true`) — controls whether the plugin is loaded and started

Both must be `true`. The `doctor` command can write `plugins.entries.<name>.enabled: false` as a safe default, which persists and causes confusion.

**Solution:**
Ensure BOTH layers are enabled in the configuration:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "dmPolicy": "pairing"
    }
  },
  "plugins": {
    "entries": {
      "telegram": {
        "enabled": true
      }
    }
  }
}
```

After adding both, `openclaw doctor --fix` should confirm: `Telegram: ok` and `Plugins: Loaded: 2`.

**Prevention:**
- When configuring any OpenClaw channel, always set BOTH `channels.<name>.enabled` AND `plugins.entries.<name>.enabled` to `true`
- When debugging channel issues, inspect the full config including the `plugins` section
- Check plugin count in `openclaw doctor` output (`Loaded: N, Disabled: M`)

**Context:**
- Versions affected: OpenClaw 2026.2.6+
- OS: all (Docker containers)
- First documented: 2026-02-07
- Source: `260207-2205-lessons-learned.md`, `260208-0035-lessons-learned.md`

**Tags:** `openclaw` `telegram` `plugins` `two-layer-enablement`

---

### Entry 5: Config Validation — Strict Enum Values {#entry-5}

**Problem:**
OpenClaw fails to start with validation errors about invalid configuration values, even though config appears correct.

**Root Cause:**
OpenClaw has strict validation for configuration values. Common mistakes:
1. Gateway bind must be `"lan"`, not `"0.0.0.0"` or `"all"`
2. Model API type must be `"openai-completions"`, not `"openai"` or `"openai-chat"`

**Solution:**
Use exact enum values from documentation:

```json
{
  "gateway": {
    "mode": "http",
    "port": 18789,
    "bind": "lan"
  },
  "models": {
    "providers": {
      "litellm": {
        "api": "openai-completions",
        "baseUrl": "http://litellm:4000/v1"
      }
    }
  }
}
```

**Prevention:**
- Consult OpenClaw documentation for exact enum values
- Test configuration changes in isolation
- Check OpenClaw logs for specific validation errors
- Keep configuration simple initially, add complexity incrementally

**Context:**
- Versions affected: OpenClaw (all versions)
- OS: all
- First documented: 2026-02-07
- Source: `260207-2205-lessons-learned.md`

**Tags:** `openclaw` `configuration` `validation`

---

### Entry 6: Docker Volume Mounting Hides Config Templates {#entry-6}

**Problem:**
OpenClaw container cannot find configuration file — template is missing at runtime even though it's in the Dockerfile.

```
[entrypoint] WARNING: Template file not found at /home/openclaw/.openclaw/openclaw.json.template
```

**Root Cause:**
Docker volume mount at `/home/openclaw/.openclaw/` masks files placed there during `docker build`. The volume's contents (from a previous lifecycle) take precedence over the image filesystem.

**Solution:**
Place templates outside volume mount paths:

```dockerfile
# WRONG — hidden by volume mount
COPY openclaw.json.template /home/openclaw/.openclaw/openclaw.json.template

# CORRECT — outside the volume mount
COPY openclaw.json.template /opt/openclaw/openclaw.json.template
```

The entrypoint reads from `/opt/openclaw/` (image layer) and writes rendered output to `/home/openclaw/.openclaw/` (volume).

**Prevention:**
- Before placing any file via `COPY` in a Dockerfile, check docker-compose.yml for volume mounts at or above that path
- Rule of thumb: build-time assets go in `/opt/` or `/etc/`; runtime state goes in volume-mounted paths

**Context:**
- Versions affected: Docker 24+, Docker Compose v2
- OS: Linux
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `openclaw` `docker` `volumes` `configuration`

See also: [infra/docker.md#entry-4](infra/docker.md#entry-4) for the generalized Docker volume masking lesson.

---

### Entry 7: Entrypoint Config Generation Overwrites Runtime Changes {#entry-7}

**Problem:**
After manually fixing the OpenClaw config inside a running container, restarting the container reverts the fix. Plugin goes back to "configured, not enabled yet" on every restart.

**Root Cause:**
The entrypoint script uses `envsubst` to regenerate `openclaw.json` from a template on every startup, overwriting any runtime changes (including those made by `openclaw doctor --fix`).

**Solution:**
Ensure the template file contains ALL required configuration, including settings like `plugins.entries.telegram.enabled: true`. The template must be the complete source of truth.

Alternatively, consider:
- Running `doctor` as part of the entrypoint AFTER config generation
- Using OpenClaw's native `${VAR}` substitution instead of external `envsubst` (allows doctor to make persistent changes)
- Generating config only if it doesn't exist: `if [ ! -f config.json ]; then envsubst ...; fi`

**Prevention:**
- When using config generation at container startup, the template MUST be the complete source of truth
- Never rely on runtime config modifications persisting if the entrypoint regenerates the config
- OpenClaw natively supports `${VAR_NAME}` substitution — evaluate using this instead of external `envsubst`

**Context:**
- Versions affected: Docker, GNU gettext `envsubst`
- OS: Linux
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `openclaw` `docker` `entrypoint` `envsubst` `config-persistence`

See also: [infra/docker.md#entry-5](infra/docker.md#entry-5) for the generalized Docker entrypoint config lesson.

---

## Cross-References

- [infra/docker.md](infra/docker.md) — Docker volume masking and entrypoint config patterns
- [infra/github-actions.md](infra/github-actions.md) — CI/CD image building and secrets validation
- [infra/nginx.md](infra/nginx.md) — Subpath proxying failures for web UIs

---

## External Resources

- [OpenClaw Official Docs](https://docs.openclaw.ai/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Fly.io Deployment Guide (memory requirements)](https://fly.io/docs/)

---

## Changelog

| Date | Change | Source |
|------|--------|--------|
| 2026-02-18 | Initial creation with 7 entries | Multiple TMP sources consolidated |
