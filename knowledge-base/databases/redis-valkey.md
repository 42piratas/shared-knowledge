# Redis / Valkey

**Category:** databases
**Last Updated:** 2026-02-16
**Entries:** 1

---

## Overview

Lessons learned working with Redis and Valkey (Redis-compatible in-memory data store). Covers common errors, data type handling, and operational patterns. Consult this file when troubleshooting Redis/Valkey connectivity, command errors, or data structure issues.

---

## Quick Reference

| Task | Command/Pattern |
|------|-----------------|
| Check key type | `TYPE {key}` |
| Get string value | `GET {key}` |
| Get all hash fields | `HGETALL {key}` |
| Pattern match keys | `KEYS {pattern}` (avoid in production; use `SCAN`) |
| Check memory usage | `INFO memory` |
| List all key types matching pattern | `for key in $(redis-cli KEYS "pattern:*"); do echo "$key: $(redis-cli TYPE $key)"; done` |

---

## Entries

### Entry 1: WRONGTYPE Error When Pattern Matching Returns Mixed Key Types {#entry-1}

**Problem:**

When using `KEYS` with a glob pattern (e.g., `yoshi:*:*:*:latest`), the result set may contain keys of different Redis data types (HASH, STRING, LIST, etc.). Calling a type-specific command (e.g., `HGETALL`) on all matched keys causes a `WRONGTYPE` error and crashes the application.

```
redis.exceptions.ResponseError: WRONGTYPE Operation against a key holding the wrong kind of value
```

**Root Cause:**

Redis/Valkey enforces strict type safety per key. Each key has exactly one type (string, hash, list, set, sorted set, stream). Commands are type-specific — `HGETALL` only works on HASH keys, `GET` only on STRING keys, etc. When a glob pattern matches keys of mixed types, iterating with a single command type will fail on keys that don't match.

This commonly happens when:
- A naming convention evolves over time (e.g., individual timeframe keys are HASH, but summary keys are STRING/JSON)
- Different parts of the application write different data structures under overlapping key patterns
- Migration from one storage format to another leaves legacy keys

**Solution:**

Always check key type before executing type-specific commands:

```python
# Python (redis-py / valkey)
matched_keys = client.keys("myapp:*:*:latest")

for key in matched_keys:
    key_str = key.decode() if isinstance(key, bytes) else key
    key_type = client.type(key_str)

    if key_type == 'string':
        data = client.get(key_str)
        # Parse JSON if needed: json.loads(data)
    elif key_type == 'hash':
        data = client.hgetall(key_str)
    elif key_type == 'list':
        data = client.lrange(key_str, 0, -1)
    else:
        # Skip or log unknown types
        continue
```

**Prevention:**

1. **Use distinct key patterns for different data types.** Don't let HASH keys and STRING keys share the same glob-matchable namespace. For example:
   - `myapp:item:1h:latest` (HASH) — individual records
   - `myapp:item:summary:latest` (STRING) — aggregated JSON
   - These both match `myapp:item:*:latest` — dangerous
   - Better: `myapp:hash:item:1h:latest` vs `myapp:json:item:summary:latest`

2. **If mixed types are unavoidable**, always call `TYPE` before any type-specific command. The `TYPE` command is O(1) and adds negligible overhead.

3. **Document the data type** in your key pattern documentation (e.g., in system-map or README) so future developers know what to expect.

4. **Prefer `SCAN` over `KEYS` in production** — `KEYS` blocks the server on large datasets. `SCAN` is cursor-based and non-blocking.

**Context:**
- Versions affected: All Redis versions, all Valkey versions
- OS: all
- First documented: 2026-02-16
- Source: 42bros Daisy API — `/api/yoshi/status` endpoint crashed when Yoshi started writing both HASH (per-timeframe) and STRING (JSON summary) keys matching the same glob pattern

**Tags:** `redis` `valkey` `wrongtype` `pattern-matching` `mixed-types` `type-checking`

---

## Cross-References

_None yet._

---

## External Resources

- [Redis TYPE command](https://redis.io/commands/type/)
- [Redis Data Types](https://redis.io/docs/data-types/)
- [Valkey Documentation](https://valkey.io/docs/)
- [SCAN vs KEYS](https://redis.io/commands/scan/)

---

## Changelog

| Date | Change | Source |
|------|--------|--------|
| 2026-02-16 | Initial creation with WRONGTYPE entry | 42bros iter-01-01 session |
