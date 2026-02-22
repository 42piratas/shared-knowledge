# Discord Bot Integration

**Category:** messaging
**Last Updated:** 2026-02-22
**Entries:** 4

---

## Overview

Reference guide for integrating a Discord bot into an AI agent or automation platform. Covers bot creation, permissions, slash commands, rich embeds, channel architecture, and implementation approaches.

---

## Quick Reference

| Task | Resource |
|------|----------|
| Create bot | https://discord.com/developers/applications |
| Discord.js docs | https://discord.js.org/ |
| Discord.py docs | https://discordpy.readthedocs.io/ |
| Permission calculator | https://discordapi.com/permissions.html |
| Developer docs | https://discord.com/developers/docs/intro |

---

## Entries

### Entry 1: Bot Creation & Permission Setup {#entry-1}

**Problem:**
Setting up a Discord bot with correct permissions and intents for an AI agent that needs to read/send messages, use slash commands, and react to messages.

**Solution:**

#### Bot Creation Steps

1. Go to https://discord.com/developers/applications
2. Create new application
3. Navigate to "Bot" section
4. Create bot, copy token (store securely — never commit to code)
5. Enable required intents:
   - **Message Content Intent** (required to read message text)
   - **Server Members Intent** (if you need member info)
   - **Presence Intent** (optional, for online/offline status)

#### Required Bot Permissions

| Permission | Purpose |
|:-----------|:--------|
| Read Messages / View Channels | See channel content |
| Send Messages | Post to channels |
| Send Messages in Threads | Reply in threads |
| Embed Links | Rich embed messages |
| Attach Files | Upload files |
| Read Message History | Access past messages |
| Add Reactions | React to messages |
| Use Slash Commands | Register and respond to `/` commands |

#### Invite URL Format

```
https://discord.com/api/oauth2/authorize?client_id={YOUR_CLIENT_ID}&permissions={PERMISSION_INTEGER}&scope=bot%20applications.commands
```

Calculate the permission integer from your selected permissions using the Discord permission calculator or the developer portal.

**Prevention:**
- Always use the principle of least privilege — only request permissions you need
- Store `DISCORD_BOT_TOKEN` in a secrets manager (GitHub Secrets, Vault, env vars)
- Never commit bot tokens to source control

**Context:**
- Discord API version: v10+
- Note: Message Content Intent requires verification for bots in 100+ servers

**Tags:** `discord` `bot` `permissions` `intents` `setup`

---

### Entry 2: Channel Architecture for Agent Notifications {#entry-2}

**Problem:**
Designing a channel layout that separates different types of agent notifications and interactions without cross-channel noise.

**Solution:**

#### Recommended Channel Structure

**Notification channels (bot posts, humans read):**

| Channel | Purpose |
|:--------|:--------|
| `#agent-alerts` | High-priority notifications (errors, budget warnings, status changes) |
| `#agent-reports` | Periodic reports, summaries, opportunity reports |
| `#agent-parking-lot` | Lower-priority items for weekly review |
| `#agent-logs` | Detailed debug logs (optional, can be noisy) |

**Interaction channels (humans command, bot responds):**

| Channel | Purpose |
|:--------|:--------|
| DMs with bot | Private commands and queries |
| `#agent-commands` | Public command channel (team visibility) |
| Threads | Per-topic discussions spawned from reports |

#### Configuration Schema

```json
{
  "discord": {
    "enabled": true,
    "botToken": "${DISCORD_BOT_TOKEN}",
    "channels": {
      "alerts": "{CHANNEL_ID}",
      "reports": "{CHANNEL_ID}",
      "parkingLot": "{CHANNEL_ID}",
      "logs": "{CHANNEL_ID}"
    },
    "dmPolicy": "allowlist",
    "allowedUsers": ["{USER_ID_1}", "{USER_ID_2}"]
  }
}
```

**Prevention:**
- Use allowlists for DM interaction to prevent unauthorized access
- Separate noisy channels (logs) from actionable ones (alerts)
- Use threads for long discussions to keep channels clean

**Context:**
- Channel IDs are numeric strings (e.g., `"1234567890123456789"`)
- Enable Developer Mode in Discord settings to copy IDs easily

**Tags:** `discord` `channels` `architecture` `notifications`

---

### Entry 3: Slash Commands for Agent Interaction {#entry-3}

**Problem:**
Designing a slash command structure for controlling an AI agent via Discord.

**Solution:**

#### Recommended Command Structure

| Command | Description | Permissions |
|:--------|:------------|:------------|
| `/agent status` | Show agent status and health | All users |
| `/agent help` | Show available commands | All users |
| `/agent approve <id>` | Approve a queued item | Owner only |
| `/agent reject <id> [reason]` | Reject with reason | Owner only |
| `/agent hibernate` | Put agent in low-power mode | Owner only |
| `/agent wake` | Resume from hibernation | Owner only |
| `/agent config <key> <value>` | Modify runtime configuration | Owner only |

#### Implementation Notes

- Use Discord's built-in slash command system (not message prefix parsing)
- Register commands via `PUT /applications/{app_id}/commands`
- Use command permissions (v2) for role-based access control
- Support both slash commands and natural language in DMs for flexibility

#### Security

- Authenticate users by Discord user ID (allowlist)
- Use role-based permissions for multi-user setups
- Log all commands for audit trail
- Rate-limit commands to prevent abuse

**Prevention:**
- Always validate user permissions server-side, not just in the UI
- Use ephemeral responses for sensitive data (only visible to the command invoker)

**Context:**
- Slash commands must be registered with Discord before use
- Global commands take up to 1 hour to propagate; guild commands are instant

**Tags:** `discord` `slash-commands` `interaction` `security`

---

### Entry 4: Rich Embeds for Agent Reports {#entry-4}

**Problem:**
Formatting structured agent output (reports, status updates, alerts) as readable Discord messages.

**Solution:**

#### Opportunity / Report Embed

Discord embeds support structured layouts with fields, colors, and footers:

```json
{
  "embeds": [{
    "title": "🎯 New Opportunity: {title}",
    "color": 5025616,
    "fields": [
      { "name": "Confidence", "value": "82%", "inline": true },
      { "name": "Source", "value": "Product Hunt", "inline": true },
      { "name": "Model", "value": "Proven", "inline": true },
      { "name": "📊 Scores", "value": "• Cost: 85/100\n• Revenue: 75/100\n• Time to Revenue: 90/100" },
      { "name": "💡 Analysis", "value": "Strong demand signal, low competition." },
      { "name": "🔗 Source", "value": "[View](https://example.com)" }
    ],
    "footer": { "text": "React with 👍 to approve or 👎 to reject" }
  }]
}
```

#### Status Update Format

```
✅ Agent Status: Operational
Last Heartbeat: 2 minutes ago
Items Found Today: 3
Token Usage: 12,450 / 100,000 daily limit
Cost Today: $0.42 / $10.00 budget
```

#### Error Alert Format

```
🚨 ALERT: Budget threshold exceeded
Current spend: $8.50 / $10.00 daily limit
Recommendation: Review token usage or increase budget
```

#### Reaction-Based Workflows

Use reactions for lightweight approval flows:
- Bot posts report with instructions to react
- Bot watches for 👍 (approve) or 👎 (reject) reactions
- Only reactions from allowlisted users are processed
- Bot confirms the action in a reply or thread

**Prevention:**
- Keep embeds under 6000 total characters (Discord limit)
- Max 25 fields per embed, max 10 embeds per message
- Use color coding: green (0x4CB05E) for success, red (0xE74C3C) for errors, yellow (0xF1C40F) for warnings

**Context:**
- Embed limits: https://discord.com/developers/docs/resources/message#embed-object-embed-limits
- Discord.js embed builder simplifies construction

**Tags:** `discord` `embeds` `formatting` `reactions` `workflow`

---

## Implementation Approaches

### Option A: Discord.js (Node.js)

**Best for:** Projects already running Node.js (e.g., OpenClaw-based agents)

- Same runtime, no additional language dependency
- Modern async/await API
- Direct integration with Node.js agent platforms

### Option B: Discord.py (Python)

**Best for:** Python-based agents or backends

- Mature, well-documented library
- Rich ecosystem of extensions (discord-py-interactions, etc.)
- Easy integration with Python ML/AI tooling

### Option C: Platform Plugin

**Best for:** Agent platforms with native Discord support

- Check if your agent platform (OpenClaw, LangChain, etc.) has a Discord channel plugin
- Avoids custom code for basic messaging
- May limit advanced features (embeds, reactions, threads)

---

## Multi-Platform Coexistence

Discord should be an **addition**, not a replacement for existing channels (Telegram, email, etc.):

- Both channels remain active simultaneously
- User chooses preferred channel for notifications
- Provides redundancy if one platform is down
- Different audiences: personal (Telegram/email) vs team (Discord)

**Conflict handling:** If the same action can be triggered from multiple channels (e.g., approve via Telegram AND Discord), implement idempotent actions — approving an already-approved item is a no-op, not an error.

---

## Cross-References

- [infra/digitalocean.md](../infra/digitalocean.md) — Hosting considerations for bot processes
- [frameworks/openclaw.md](../frameworks/openclaw.md) — OpenClaw channel integration

---

## External Resources

- [Discord Developer Portal](https://discord.com/developers/docs/intro)
- [Discord.js Guide](https://discordjs.guide/)
- [Discord.py Documentation](https://discordpy.readthedocs.io/)
- [Discord Permissions Calculator](https://discordapi.com/permissions.html)

---

## Changelog

| Date | Change | Source |
|------|--------|--------|
| 2026-02-22 | Initial creation — consolidated from project-specific planning doc | `alfred-01/docs/discord-integration.md` |
