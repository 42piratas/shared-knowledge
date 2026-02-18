# OpenClaw

**Category:** infra
**Last Updated:** 2026-02-18
**Entries:** 9

---

## Overview

Lessons learned from deploying and configuring the OpenClaw agent platform, focusing on Docker deployment patterns, resource management, configuration, and plugin systems.

---

## Quick Reference

| Task                | Command/Pattern                                                   |
| ------------------- | ----------------------------------------------------------------- |
| Check logs          | `docker compose logs openclaw --tail 100`                         |
| Verify Node version | `docker exec openclaw node --version`                             |
| Check config        | `docker exec openclaw cat /home/openclaw/.openclaw/openclaw.json` |
| Monitor memory      | `docker stats openclaw`                                           |
| Plugin status       | `node scripts/run-node.mjs plugins status`                        |

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
        cpus: "1.0"
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

| Droplet Size | Available RAM | OpenClaw Allocation | Verdict                |
| :----------- | :------------ | :------------------ | :--------------------- |
| 1GB droplet  | ~800MB        | 512MB (limit)       | ❌ Too small, will OOM |
| 2GB droplet  | ~1.6GB        | 1GB-1.5GB (limit)   | ✅ Recommended minimum |
| 4GB droplet  | ~3.2GB        | 2GB-3GB (limit)     | ✅ Comfortable margin  |

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

### Entry 8: LiteLLM Streaming Response Incompatible with Proxy Cost Tracking {#entry-8}

**Problem:**
A FastAPI proxy wrapping `litellm.acompletion()` returns HTTP 503 for every request when the upstream client sends `"stream": true`. Error in logs:

```
'CustomStreamWrapper' object is not subscriptable
```

The service works perfectly with `"stream": false` (direct curl tests pass), but fails for all real traffic.

**Root Cause:**
When `stream=True`, LiteLLM returns a `CustomStreamWrapper` generator object instead of a dict. Any code that then accesses `response['usage']`, `response['choices']`, or `response['id']` raises `TypeError: 'CustomStreamWrapper' object is not subscriptable`. The proxy crashes before returning a response.

This is a common pattern: a proxy needs the complete response to calculate cost, deduct budget, and log usage — all of which require `response['usage']`. This is fundamentally incompatible with streaming.

**Solution (Simple — Recommended):**
Force `stream=False` internally regardless of client preference:

```python
async def execute_single_model(..., request):
    response = await litellm.acompletion(
        model=litellm_model,
        messages=messages,
        stream=False,  # Force non-streaming for cost tracking
        # ... other params
    )
    usage = response['usage']  # Now safe — response is a dict
    cost = calculate_cost(...)
    await deduct_budget(cost)
    return response
```

**Trade-off:** Clients that rely on streaming (e.g., showing typing indicators) will receive responses only after the full generation completes. For most agent-to-agent proxies, this is acceptable.

**Solution (Advanced — Proper Streaming):**
If streaming to the client is required, collect chunks and extract usage from the final chunk:

```python
chunks = []
usage = None
async for chunk in await litellm.acompletion(..., stream=True):
    chunks.append(chunk)
    if chunk.choices[0].finish_reason == "stop":
        usage = chunk.usage  # Final chunk contains usage on some providers
# Reconstruct full response or stream chunks to client while tracking cost
```

Note: Not all providers include usage in the final streaming chunk. Test with each provider.

**Prevention:**

- When building a proxy that needs `response['usage']`, always force `stream=False` unless streaming passthrough is explicitly required
- Test with `stream=True` explicitly — do not assume because `stream=False` works that `stream=True` will too
- Log the type of the response object when debugging: `logger.debug(f"Response type: {type(response)}")`
- Add a check: `if hasattr(response, '__aiter__'): raise ValueError("Got streaming response, expected dict")`

**Context:**

- LiteLLM versions affected: all (this is intended behavior, not a bug)
- Upstream clients affected: OpenClaw (always sends `stream: true`)
- First documented: 2026-02-18
- Source: `black-box/logs/log-260218-1700-llm-blockers-partial-fix.md`

**Tags:** `litellm` `streaming` `fastapi` `proxy` `budget-tracking` `openclaw`

---

### Entry 9: Gemini API Free Tier Hard Cap — 20 Requests/Day on gemini-2.5-flash {#entry-9}

**Problem:**
A production agent stops responding after a small number of test requests. BudgetGateway logs show:

```
HTTP 429 RESOURCE_EXHAUSTED
"Quota exceeded for metric: generativelanguage.googleapis.com/generate_content_free_tier_requests,
limit: 20, model: gemini-2.5-flash. Please retry in 58s."
```

Every subsequent request fails, even after retrying. The agent appears completely broken.

**Root Cause:**
`gemini-2.5-flash` on the Google AI free tier is hard-capped at **20 requests per day per project**. Debugging sessions alone can exhaust this quota. Once exhausted, all requests fail with 429 until midnight UTC when the quota resets.

Free tier limits by model (as of 2026-02):

- `gemini-2.5-flash`: 20 req/day ⚠️ Very low
- `gemini-1.5-flash`: 1,500 req/day ✅ Much higher
- `gemini-1.5-flash-8b`: 1,500 req/day ✅ Much higher

**Solution:**

Option A — Use a model with a higher free quota:

```python
MODEL_MAPPING = {
    "gemini-flash": "gemini/gemini-1.5-flash",    # 1,500 req/day free
    "gemini-flash-8b": "gemini/gemini-1.5-flash-8b",  # 1,500 req/day free
}
```

Option B — Upgrade to Gemini API paid tier:

- Go to https://aistudio.google.com/ → Billing
- Paid tier removes the daily cap (rate-limited by RPM/TPM instead)
- Recommended for any production workload

Option C — Wait for quota reset:

- Free tier resets at midnight UTC
- Acceptable only for development, not production

**Symptoms vs. Other Issues:**
| Symptom | Likely Cause |
|:--------|:-------------|
| 429 with "quota exceeded" | Free tier exhausted |
| 429 with "rate limit" | Too many requests per minute |
| 401 / invalid API key | Wrong or expired key |
| 503 all models | Check both primary and fallback |

**Prevention:**

- Never use `gemini-2.5-flash` free tier in production (20 req/day is exhausted in minutes of testing)
- Use `gemini-1.5-flash` or `gemini-1.5-flash-8b` for development (1,500 req/day)
- Set up paid billing before any real usage
- Add quota monitoring: alert when daily usage exceeds 80% of limit
- Ensure the fallback model (e.g., Claude) has a valid API key — otherwise quota exhaustion = complete outage

**Context:**

- API: Google AI (Gemini) via LiteLLM `gemini/` prefix
- Model: `gemini/gemini-2.5-flash` on free tier
- First documented: 2026-02-18
- Source: `black-box/logs/log-260218-1700-llm-blockers-partial-fix.md`

**Tags:** `gemini` `litellm` `quota` `rate-limiting` `free-tier` `production-readiness`

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

| Date       | Change                                                                     | Source                                        |
| ---------- | -------------------------------------------------------------------------- | --------------------------------------------- |
| 2026-02-18 | Initial creation with 7 entries                                            | Multiple TMP sources consolidated             |
| 2026-02-18 | Added Entry 8 (LiteLLM streaming bug) and Entry 9 (Gemini free tier quota) | `log-260218-1700-llm-blockers-partial-fix.md` |
