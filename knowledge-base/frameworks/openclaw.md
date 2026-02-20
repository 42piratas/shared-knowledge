# OpenClaw — Knowledge Base

**Last Updated:** 2026-02-20
**Category:** Frameworks
**Applies to:** Any project using OpenClaw as an AI agent gateway

---

## What Is OpenClaw

OpenClaw is a self-hosted, open-source personal AI assistant gateway. It bridges messaging apps (Telegram, WhatsApp, Discord, Slack, Signal, iMessage, etc.) to AI agents via a single Gateway process. It is agent-native: built for tool use, sessions, memory, multi-agent routing, cron jobs, browser control, and more.

- **GitHub:** https://github.com/openclaw/openclaw
- **Docs:** https://docs.openclaw.ai
- **License:** MIT

---

## Provider Model List Is Not Optional

If you route OpenClaw through a custom LLM provider (e.g., a budget gateway or proxy), you **must** declare every model that could be used as primary or fallback in the provider's `models` array inside `openclaw.json`.

If a model is NOT in the list, OpenClaw silently assigns minimal defaults:

- `input: ["text"]` (no image support)
- `reasoning: false`
- `contextWindow: 125000`
- **Tool schemas are NOT sent in API requests**

There is no warning. The model simply doesn't receive tools.

### Diagnosis

```bash
docker exec {container} node dist/index.js models list
# BAD:  litellm/{model}    text    125k    no    no    default
# GOOD: litellm/{model}    text+image 200k  no    yes   default
```

### Required Fields Per Model

```json
{
  "id": "{model-alias}",
  "name": "{Human-Readable Name}",
  "reasoning": true,
  "input": ["text", "image"],
  "contextWindow": 200000,
  "maxTokens": 8192
}
```

Reference: https://docs.openclaw.ai/providers/litellm

### Key Insight

If the entrypoint dynamically selects the primary model at boot (e.g., by querying an external API), the provider model list must include **all models that could ever be selected** — not just the one that was active when the config was first written.

---

## OpenClaw Uses the `developer` Role for Reasoning Models

When a model is configured with `"reasoning": true` in the provider model list, OpenClaw sends the system prompt with `role: "developer"` instead of `role: "system"`. This follows the newer OpenAI API convention introduced for reasoning models (O1, etc.).

**Impact:** Any LLM proxy between OpenClaw and the provider that validates message roles against a fixed set like `(system, user, assistant, tool)` will reject the request with a 422.

**Fix:** Accept any string for the message role. The downstream LLM library (e.g., litellm) handles per-provider role translation automatically.

**When does this happen?** Only when `reasoning: true` is set in the provider model config. If you set `reasoning: false`, OpenClaw uses the standard `system` role.

---

## Tool-Calling Messages Have Null Content

In OpenAI-compatible tool-calling conversations, certain messages have `content: null`:

- Assistant messages with `tool_calls` (the model is calling a tool, not producing text)
- Some tool result messages

If your proxy:

1. Always includes `"content": null` in forwarded requests → some providers reject it
2. Calls `len(msg.content)` for token estimation → `TypeError: object of type 'NoneType' has no len()`

**Fix:** Only include the `content` key when the value is not None. Guard any `len()` calls.

---

## Docker Volumes Override Image Filesystem

OpenClaw uses persistent Docker volumes for workspace, state, and cron data. If you `COPY` files into the workspace directory in your Dockerfile, **the volume wins after first creation**. Updated files baked into the image are invisible to the running container.

**Fix:** Copy IaC-managed files to a staging directory in the Dockerfile (e.g., `/opt/openclaw/workspace-init/`), then sync from staging to the volume in the entrypoint on every boot.

This ensures:

- IaC changes always reach the container
- User-created files in the volume are preserved (don't overwrite data directories)
- Skills, scripts, and agent instructions stay up-to-date

---

## Docker Volume Ownership Mismatch

OpenClaw runs as a non-root user (`openclaw`), but persistent volume files may be owned by a different UID from a previous build. This causes permission errors on workspace sync.

**Fix:** Run the entrypoint as root, `chown` the workspace volume, then drop to the `openclaw` user via `gosu`:

```bash
#!/bin/sh
set -e
chown -R openclaw:openclaw /home/openclaw/.openclaw
# ... sync workspace files ...
exec gosu openclaw "$@"
```

---

## Cron Store Is Ephemeral

OpenClaw's cron engine reads from an internal store (e.g., `/home/openclaw/.openclaw/cron/jobs.json`). At runtime, OpenClaw modifies this file:

- Adds `staggerMs` fields
- Sets `enabled: false` after repeated errors
- Increments `scheduleErrorCount`

If your IaC defines cron jobs, **overwrite the cron store on every boot** — not just when it's empty. Otherwise, runtime error state persists across restarts and jobs stay disabled.

**Known issue:** The `0 */3 * * *` step syntax causes a `TypeError` in some OpenClaw versions. Explicit hour lists (`0 0,3,6,9,12,15,18,21 * * *`) may work as a workaround. The `0 6 * * *` syntax works fine.

---

## Stacked Issues in Proxy Configurations

When routing OpenClaw through a custom proxy, tool execution requires **multiple things to work simultaneously**. Issues can stack — fixing one reveals the next:

1. **Model capabilities** → OpenClaw must know the model supports tools
2. **Role validation** → Proxy must accept `developer` role (triggered by `reasoning: true`)
3. **Null content handling** → Proxy must handle `content: null` in tool messages

**Rule of thumb:** If two different model families (e.g., Gemini and Claude) exhibit the same broken behavior through your proxy, the problem is almost certainly in the proxy infrastructure, not in the models.

**Debugging approach:** When a proxy returns 422, capture the raw request body BEFORE validation. A few bytes of log output reveals exactly which field fails — saves hours of guessing.

---

## Skill Gating

OpenClaw silently excludes skills whose required environment variables are not set. There is no error message — the skill simply doesn't appear in `skills list`.

**Diagnosis:**

```bash
docker exec {container} node dist/index.js skills list
# Look for ✗ missing vs ✓ ready
```

**Fix:** Ensure all `requires.env` variables listed in the skill's `SKILL.md` are injected via the `skills.entries.{name}.env` section in `openclaw.json`.

---

## SMTP Is Blocked on Most Cloud Providers

DigitalOcean, AWS Lightsail, GCP, and others block ports 25/587/465 by default for new accounts. OpenClaw skills that need to send email **must use HTTP-based APIs** (Brevo, SendGrid, Postmark, etc.) over HTTPS port 443.

Do not attempt SMTP-based email sending from cloud-hosted OpenClaw instances.

---

## Context Size Limits

Oversized workspace files (`AGENTS.md`, `MEMORY.md`, skill definitions) can trigger "context too long" errors. Keep injected files under ~50KB combined. Use `.openclawignore` to exclude large files from auto-loading.

---

## References

- OpenClaw Docs: https://docs.openclaw.ai
- LiteLLM Provider Config: https://docs.openclaw.ai/providers/litellm
- Tools Documentation: https://docs.openclaw.ai/tools
- Skills Documentation: https://docs.openclaw.ai/skills
- Cron Documentation: https://docs.openclaw.ai/cron
- OpenClaw GitHub: https://github.com/openclaw/openclaw
