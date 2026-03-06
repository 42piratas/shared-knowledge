# Lesson: Telegram Bot — MarkdownV2 & Template Design

**Category:** Messaging / Telegram
**Applies to:** Any Python service sending Telegram messages via `python-telegram-bot` with `parse_mode="MarkdownV2"`

---

## 1. MarkdownV2 Reserved Characters

All of the following must be escaped with `\` in message text:
```
_ * [ ] ( ) ~ ` > # + - = | { } . !
```

**The trap:** Parentheses `(` and `)` are commonly forgotten because they look "safe". If your template has:
```python
"UPDATE ({indicator_value})"
```
Telegram returns HTTP 400: `Can't parse entities: character '(' is reserved`.

**Fix:** Escape all literal parens in the template string itself:
```python
"UPDATE \\({indicator_value}\\)"  # Python string
```
In YAML config:
```yaml
template: "UPDATE \\({indicator_value}\\)"
```
Note: in YAML, `\\(` writes the literal two characters `\(` into the string — which is what MarkdownV2 needs.

**Rule:** Run `escape_markdown()` on **data values**, but also manually escape all literal reserved characters in the template string itself. They are different concerns.

---

## 2. Injecting a Universal Prefix Across All Templates

If you need a tag (e.g. `[TEST]`) to appear at the **beginning** of every Telegram message regardless of source:

- **Do NOT inject into `detail` or `message` fields** — these are source-specific and not always in the first line of every template.
- **Inject into a field that appears first in ALL templates** — `asset` is a reliable candidate if all your templates start with it (e.g. `ℹ️ \`{asset}\` ...`).
- **Inject at the producer**, not at the notification router (Toad/sender). The router doesn't know which field each template uses as its opener.

**Pattern:**
```python
# In the service that creates the notification event:
asset = f"[TEST] {asset}" if is_simulated else asset
```

**Corollary:** Design all message templates to start with `{asset}` or another shared field. This makes universal prefixing possible and keeps templates consistent.

---

## 3. Template Design — Source Field Placement

Place `{asset}` as the first content field in every template header line. This enables:
- Consistent visual identity (asset is always first)
- Universal prefix injection (see §2)
- Easier human scanning in channel history

**Before (fragile):**
```
ℹ️ {condition} UPDATE ({indicator_value}) ℹ️
...
Asset: `{asset}`
```

**After (robust):**
```
ℹ️ `{asset}` {condition} UPDATE \({indicator_value}\) ℹ️
...
```

---

**Last Updated:** 2026-03-06
