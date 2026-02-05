# Telegram API

Lessons learned working with the Telegram Bot API.

---

## MarkdownV2 Special Character Escaping

**Problem:**
Telegram API rejects messages with 400 Bad Request.

```
Can't parse entities: can't find end of italic entity at byte offset 30
```

**Root Cause:**
Telegram MarkdownV2 interprets special characters as formatting markers. Underscores (`_`), asterisks (`*`), and other characters in data values (e.g., `BTC_USDT`, `EXTREME_FEAR`) break the parser.

**Solution:**
Escape all MarkdownV2 special characters in dynamic content:

```python
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
- Other relevant context: MarkdownV2 is stricter than original Markdown

**Tags:** `telegram` `api` `markdown` `escaping`

---
