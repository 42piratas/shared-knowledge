# OpenClaw — Knowledge Base

**Last Updated:** 2026-02-22
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

## Skills Do NOT Inject Tools Into the LLM's Tool Schema

OpenClaw skills (`<workspace>/skills/<name>/`) provide **guidance text** to the LLM, not callable tool definitions. When a skill is loaded and eligible:

1. OpenClaw injects a **compact XML entry** into the system prompt: `<skill name="..." description="..." location="..." />`
2. That's it — name, description, and filesystem location. Nothing else from the skill directory reaches the LLM.

**What the LLM does NOT see:**

- The `SKILL.md` markdown body (instructions below the frontmatter) — not injected
- Any `skill.json` file — **OpenClaw completely ignores this file** (zero references in compiled source as of commit `3100b77`)
- Tool schemas defined in `skill.json` — never added to the model's function-calling API

**How skills are meant to work:**

Skills teach the LLM to use **built-in tools** (like `exec`, `bash`, `read`, `write`) for specific tasks. The teaching happens via workspace instruction files (`AGENTS.md`, `TOOLS.md`) that explicitly tell the LLM what commands to run. The skill's XML entry in the prompt provides context (name + description), but the actual "how to invoke" instructions must be in workspace files.

**Example — correct pattern:**

```
# In AGENTS.md or TOOLS.md:
To send emails, use the bash tool:
echo "body" | /home/openclaw/.openclaw/workspace/scripts/send-email.sh "[PREFIX] Subject"
```

**Example — broken pattern (tools never reach the LLM):**

```
# In skill.json (OpenClaw ignores this entirely):
{ "tools": [{ "name": "set_status", "input_schema": { ... } }] }
```

### Diagnosis

```bash
# Confirm what the skill object contains (no instructions, no tools):
docker exec {container} node -e "
const { r: load } = require('./dist/skills-{hash}.js');
const entries = load('{workspace_path}');
const skill = entries.find(e => e.skill.name === '{name}');
console.log(Object.keys(skill.skill));
console.log('instructions:', !!skill.skill.instructions);
"
# Expected: ['name','description','filePath','baseDir','source','disableModelInvocation']
# instructions: false
```

---

## command-dispatch Only Works With Built-In Tools

The `command-dispatch: tool` frontmatter in `SKILL.md` creates a fast path that bypasses the LLM for slash commands. However, the `command-tool` field **must reference a built-in OpenClaw tool or a plugin-registered tool** — not a custom tool defined in `skill.json`.

When a user types `/<skill> <args>`, OpenClaw:

1. Resolves the skill command from the slash command name ✅
2. Reads `dispatch.kind` and `dispatch.toolName` from the skill's frontmatter ✅
3. Searches for the tool in `createOpenClawTools()` — which returns **only** built-in tools (browser, canvas, nodes, cron, message, tts, gateway, agents_list, sessions_list, sessions_history, sessions_send, sessions_spawn, subagents, session_status, web_search, web_fetch) plus plugin-registered tools
4. If the tool is not found: returns `"❌ Tool not available: {toolName}"`

**This means `command-tool: my_custom_tool` will always fail** if `my_custom_tool` is defined only in `skill.json`. OpenClaw never reads `skill.json`, and custom skill tools are not in the built-in tool pool.

### What command-dispatch IS useful for

Dispatching to built-in tools or plugin-registered tools. For example, a plugin that registers a `prose` tool can use `command-dispatch: tool` + `command-tool: prose` successfully.

### Source code evidence (commit 3100b77)

```
dist/reply-{hash}.js line ~61113:
const tool = createOpenClawTools({...}).find(t => t.name === dispatch.toolName);
if (!tool) return { kind: "reply", reply: { text: "❌ Tool not available: " + dispatch.toolName } };
```

---

## Sending Email via HTTP API (Cloud Deployments)

Cloud providers (DigitalOcean, AWS, GCP) block outbound SMTP ports (25, 587, 465) by default. Email skills that use SMTP will silently fail — the agent may report success but no email is delivered.

**Solution:** Use an HTTP-based email API (Brevo, SendGrid, Postmark, Mailgun, Amazon SES) over HTTPS port 443, which is never blocked.

### Implementation Pattern

```
Agent receives email request
  → Reads SKILL.md for email skill (learns the bash invocation)
  → Uses `exec` tool to run send-email.sh via bash
  → send-email.sh calls email API (e.g., https://api.brevo.com/v3/smtp/email)
  → API delivers the email
  → Script returns messageId to agent
  → Agent confirms delivery to user
```

### Key Components

1. **Bash script** (`scripts/send-email.sh`) — reads body from stdin, takes subject as argument, reads API key and addresses from env vars, POSTs to API, returns messageId or error.

2. **Skill definition** (`skills/email/SKILL.md`) — uses OpenClaw's gating system (`requires.env`) to declare required env vars. If any are missing, the skill is silently excluded — preventing the agent from attempting email when unconfigured.

3. **Environment injection** — env vars scoped to the skill via `skills.entries.{name}.env` in `openclaw.json`, not leaked into the global shell:

```json
{
  "skills": {
    "entries": {
      "email": {
        "enabled": true,
        "env": {
          "BREVO_API_KEY": "${BREVO_API_KEY}",
          "EMAIL_FROM": "${EMAIL_FROM}",
          "EMAIL_FROM_NAME": "{Your Agent Name}",
          "EMAIL_TO": "${EMAIL_TO}"
        }
      }
    }
  }
}
```

4. **Explicit AGENTS.md instructions** — without these, the agent invents non-existent tool calls like `message(action='send', channel='email')`:

```markdown
To send emails, you MUST use the bash tool to execute the send-email.sh script:
echo "body" | /path/to/scripts/send-email.sh "[PREFIX] Subject"

Do NOT use message(action='send', channel='email') — that tool does not exist.
```

### Brevo Quick Reference

| Item         | Value                                      |
| :----------- | :----------------------------------------- |
| Endpoint     | `POST https://api.brevo.com/v3/smtp/email` |
| Auth header  | `api-key: {YOUR_API_KEY}`                  |
| Content-Type | `application/json`                         |
| Success      | HTTP 201, `{"messageId": "{id}"}`          |
| Get API key  | https://app.brevo.com/settings/keys/api    |
| Free tier    | 300 emails/day                             |

**Note:** Sender email must be verified in your email provider's sender list.

### Debugging Email Issues

```bash
# Verify API key is in the container
docker exec {container} printenv BREVO_API_KEY

# Verify skill is loaded
docker exec {container} node dist/index.js skills list 2>&1 | grep email

# Check logs for email activity
docker logs {container} 2>&1 | tail -50 | grep -i "email\|brevo\|messageId"

# Test script directly
docker exec -it {container} bash -c \
  'echo "Test body" | /path/to/scripts/send-email.sh "Test Subject"'
```

---

## Messaging Noise Control

### The Problem

When streaming is enabled, OpenClaw can leak raw internal tool-call syntax into the chat channel:

```
<ctrl42>call:read(path='data/opportunities.json'){"opportunities": [...]}
```

This is the internal wire format bleeding through before the tool result is available. Additionally, agents may dump raw JSON, debugging info, or tool execution logs into responses.

### Root Causes

1. **Streaming mode** — partial text chunks sent in real time; tool-call tokens appear before interception
2. **Model behavior** — some models (especially Gemini with high thinking levels) leak reasoning tokens
3. **No formatting guardrails** — agent treats tool outputs as conversational content

### Fix: Three-Layered Approach

#### Layer 1: Disable Streaming

In `openclaw.json`, set `streamMode: "off"` for the channel:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "streamMode": "off"
    }
  }
}
```

The Gateway waits for the full response (including all tool calls and results) before sending anything. Trade-off: longer "typing..." indicator, but complete clean output.

#### Layer 2: Explicit Formatting Rules in AGENTS.md

```markdown
### Response Formatting Rules (CRITICAL)

**NEVER include in your responses to users:**

- Raw tool call syntax like `<ctrl42>call:...`
- JSON data dumps from file reads
- Internal debugging information
- Tool execution logs

**ALWAYS:**

- Summarize information in natural language
- Wait for tool execution to complete before responding
- Only share relevant results, not raw data
```

Include concrete bad vs. good examples — LLMs respond well to explicit contrast.

#### Layer 3: Channel-Specific Markdown Formatting

OpenClaw converts agent output into channel-appropriate formats:

- **Telegram:** HTML tags (`<b>`, `<i>`, `<code>`, `<pre>`, `<a href>`)
- **Slack:** mrkdwn tokens
- **Signal:** plain text + style ranges
- **WhatsApp/Discord:** plain text or native formatting

Tables can be rendered as code blocks or bullet lists per channel:

```json
{
  "channels": {
    "telegram": {
      "markdown": {
        "tables": "code"
      }
    }
  }
}
```

### Noise Sources Reference

| Noise Type                   | Cause                        | Fix                                  |
| :--------------------------- | :--------------------------- | :----------------------------------- |
| `<ctrl42>call:...` fragments | Streaming + tool-call tokens | `streamMode: "off"`                  |
| Gemini reasoning/metadata    | High thinking level          | Lower thinking level or switch model |
| Raw JSON dumps               | Agent dumping tool output    | AGENTS.md formatting rules           |
| Usage footers                | Per-response token stats     | `/usage off` or config               |
| Orphaned tool_result blocks  | Context compaction edge case | Start new session (`/new`)           |

### Model Choice Affects Noise

Anthropic models (Claude) have better prompt-injection resistance and are less likely to leak internal tokens. Gemini models with high thinking levels are known to leak reasoning tokens. If noise persists despite `streamMode: "off"`:

- Use `/reasoning low` or `/reasoning off`
- Switch to Claude for the primary model
- Set `agents.defaults.subagents.thinking` to `"low"`

---

## Key Architecture Concepts

### Gateway

The Gateway is the single control plane. It manages:

- Sessions (per-sender, per-group, per-agent isolation)
- Channel connections (WhatsApp, Telegram, Discord, etc.)
- Tool routing (browser, exec, cron, message, nodes)
- WebSocket control protocol
- Control UI (web dashboard)

Config lives at `~/.openclaw/openclaw.json` (or template-injected for Docker).

### Agent Workspace

The workspace (`~/.openclaw/workspace/`) is the agent's persistent filesystem:

```
workspace/
├── AGENTS.md          # Agent behavior instructions
├── SOUL.md            # Personality definition
├── TOOLS.md           # Tool usage guidance
├── skills/            # Custom skills (SKILL.md per skill)
├── scripts/           # Custom scripts (send-email.sh, etc.)
├── data/              # Runtime data files
├── hooks/             # Event-driven automation
└── memory/            # Session memory snapshots
```

Injected prompt files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`) are loaded into the system prompt on every agent turn.

### Tools vs. Skills

- **Tools** are built-in, typed, first-class capabilities: `exec`, `browser`, `message`, `cron`, `web_search`, `canvas`, `nodes`, etc.
- **Skills** are prompt-based extensions. A `SKILL.md` file teaches the agent how to combine tools to accomplish a task (e.g., "use exec to run send-email.sh"). Skills are discovered, gated, and injected into the system prompt automatically.
- **Hooks** are event-driven TypeScript handlers that run inside the Gateway when events fire (e.g., session reset, gateway startup, message received).

### Tool Profiles

OpenClaw supports tool profiles to restrict what tools an agent can use:

| Profile     | Tools Included                                     |
| :---------- | :------------------------------------------------- |
| `minimal`   | `session_status` only                              |
| `coding`    | File system, runtime, sessions, memory, image      |
| `messaging` | Message tool, session listing/history/send, status |
| `full`      | Everything (default)                               |

---

## Skills System (Extended)

### Skill Locations & Precedence

1. **Workspace skills** (`<workspace>/skills/`) — highest precedence, per-agent
2. **Managed skills** (`~/.openclaw/skills/`) — shared across agents
3. **Bundled skills** (shipped with OpenClaw) — lowest precedence

Same-name conflicts: workspace wins → managed → bundled.

### Gating Details

Skills declare requirements in their `SKILL.md` frontmatter. OpenClaw checks these at load time:

- `requires.bins` — binaries that must be on PATH
- `requires.env` — env vars that must exist (or be provided in config)
- `requires.config` — config paths that must be truthy
- `os` — platform restrictions (`darwin`, `linux`, `win32`)

If a gate fails, the skill is marked `✗ missing` and excluded from the prompt. **If your env vars aren't set, the agent won't even know the skill exists.**

### ClawHub

ClawHub (https://clawhub.com) is the public skill registry:

```bash
clawhub install <skill-slug>
clawhub update --all
```

### Token Impact

Each eligible skill adds ~97 characters + name + description to the system prompt. Keep skill counts reasonable — a dozen skills is fine; fifty may squeeze context.

---

## Hooks & Automation

### Event-Driven Hooks

Hooks run inside the Gateway when events fire:

| Event              | When                                |
| :----------------- | :---------------------------------- |
| `command:new`      | User issues `/new`                  |
| `command:reset`    | User issues `/reset`                |
| `agent:bootstrap`  | Before workspace files are injected |
| `gateway:startup`  | After channels start                |
| `message:received` | Inbound message from any channel    |
| `message:sent`     | Outbound message sent               |

Bundled hooks:

- **session-memory** — saves session context to `memory/` on `/new`
- **bootstrap-extra-files** — injects additional bootstrap files
- **command-logger** — audit trail to `logs/commands.log`
- **boot-md** — runs `BOOT.md` on gateway startup

### Cron Jobs

OpenClaw has native cron support. Jobs can be defined in config or created by the agent at runtime:

```json
{
  "cron": {
    "enabled": true
  }
}
```

The agent can use the `cron` tool to `add`, `update`, `remove`, `list`, and `run` jobs.

**Known issue:** One-shot (`at`) jobs auto-delete after success. Use `--keep-after-run` to retain them.

**Known issue:** Step-syntax cron expressions (e.g., `0 */3 * * *`) trigger a stagger computation bug that effectively disables the job. **Fix:** Add `"staggerMs": 0` to the job definition to bypass the computation.

**Note:** The cron store (`cron/jobs.json`) is ephemeral state. If using IaC, ensure your entrypoint overwrites this file on boot to reflect config changes and clear error states.

### Heartbeats

Heartbeats are periodic agent wake-ups. Configure in `openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "telegram"
      }
    }
  }
}
```

Create a `HEARTBEAT.md` in the workspace to define what the agent checks on each heartbeat.

### Webhooks & Gmail Pub/Sub

OpenClaw can receive external webhooks (Gmail Pub/Sub, Sentry, GitHub, etc.) and route them to agent sessions. Gmail integration uses `gog gmail watch serve` → Pub/Sub push → OpenClaw hook endpoint.

---

## OpenClaw Operations (Generic)

### Version Pinning

OpenClaw evolves rapidly (11,888+ commits, 673+ contributors). Always pin to a specific version in production:

```yaml
openclaw:
  image: ghcr.io/{your-org}/{your-repo}/openclaw:${OPENCLAW_IMAGE_TAG:-v2026.2.15}
```

**Release channels:**

| Channel  | Tag Pattern           | Stability                         |
| :------- | :-------------------- | :-------------------------------- |
| `stable` | `vYYYY.M.D`           | High (recommended for production) |
| `beta`   | `vYYYY.M.D-beta.N`    | Medium                            |
| `dev`    | Moving head of `main` | Low                               |

### Upgrade Procedure

1. Read release notes for breaking changes
2. Backup workspace (data, skills, config)
3. Update image tag in docker-compose / IaC
4. Pull new image, stop old container, start new
5. Verify: Telegram responds, UI accessible, memory within range, no error logs, skills functional
6. If issues: revert image tag, restart

### Health Monitoring

**Gateway health:**

```bash
curl http://localhost:18789/
# Expected: 200 OK
```

**Resource baselines:**

| Metric                 | Baseline    | Alert Threshold |
| :--------------------- | :---------- | :-------------- |
| Memory                 | 200–400MB   | >3GB sustained  |
| Memory (with browser)  | 700MB–1.4GB | >3GB sustained  |
| CPU                    | 5–15% idle  | >80% sustained  |
| CPU (during LLM calls) | 20–40%      | >80% sustained  |
| Disk growth            | 10–50MB/day | Monitor weekly  |

**Monitoring commands:**

```bash
docker stats {container} --no-stream
docker logs {container} --since 1h 2>&1 | grep -i error
docker logs {container} --since 1h 2>&1 | grep -i cron
docker exec {container} du -sh /home/openclaw/.openclaw/workspace
```

### Common Troubleshooting

#### Container won't start

- Check logs: `docker logs {container} --tail 100`
- Common causes: invalid env var, port conflict (18789), memory limit exceeded, workspace permissions
- Fix permissions: `docker exec {container} chown -R openclaw:openclaw /home/openclaw/.openclaw`

#### High memory

- Disable browser tool if enabled
- Reduce concurrent LLM calls
- Clear old session history
- Restart to clear memory leaks: `docker compose restart {service}`

#### Cron jobs not running

- Check: `docker logs {container} --since 2h | grep -i cron`
- Verify workspace cron config: `docker exec {container} cat /home/openclaw/.openclaw/workspace/cron.json`
- Gateway restart resets cron state — may need to re-sync from IaC

#### Skills not working

- Check skill status: `docker exec {container} node dist/index.js skills list`
- Verify skill directory exists and has correct `SKILL.md`
- Check sandbox tool allowlist if in sandbox mode

---

## Additional Pitfalls

### Agent Invents Non-Existent Tools

Without explicit instructions, the agent may try plausible-sounding but non-existent tool calls like `message(action='send', channel='email')`. Always provide explicit do/don't instructions in AGENTS.md for critical workflows.

### Cron Reminders May Target Wrong Channel

Known issue: cron reminders created by the agent may not inherit the originating channel. Workaround: explicitly specify the target channel in the reminder text.

### Context Compaction Can Break Sessions

After compaction, sessions can sometimes become permanently broken with "orphaned tool_result" errors. Workaround: start a new session with `/new`.

### Mac Sleep Kills the Bot

If running OpenClaw on a Mac, use `caffeinate -s` or install as a launchd service (`openclaw service install`). For headless Mac minis: `sudo pmset -a sleep 0 displaysleep 0`.

---

## References

- OpenClaw Docs: https://docs.openclaw.ai
- LiteLLM Provider Config: https://docs.openclaw.ai/providers/litellm
- Tools Documentation: https://docs.openclaw.ai/tools
- Skills Documentation: https://docs.openclaw.ai/skills
- Cron Documentation: https://docs.openclaw.ai/cron
- OpenClaw GitHub: https://github.com/openclaw/openclaw
- ClawHub (Community Skills): https://clawhub.com
- Brevo API Docs: https://developers.brevo.com/reference/sendtransacemail
