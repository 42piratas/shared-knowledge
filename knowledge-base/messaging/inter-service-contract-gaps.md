# Lesson: Inter-Service Data Contract Gaps

**Category:** Messaging / Integration
**Applies to:** Any system where multiple services independently produce and consume shared data payloads

---

## Problem

Two services are built to communicate via a shared data structure (e.g., a feed entry, a queue message, a Valkey key). Each service is implemented independently. The producer writes fields using its own naming conventions. The consumer reads fields using its own naming conventions. No shared schema document exists. No contract test exists.

Result: all fields on the consumer side are silently `null` or missing. No error is raised. The system appears to work — data flows, no exceptions — but the consumer is reading nothing.

**This failure mode is silent by default.** It only surfaces during end-to-end validation, not unit tests or CI.

---

## Example

**42Bros — Playroom `feed:simulation` (2026-03-05):**

Mario wrote to `feed:simulation`:
```json
{
  "sim_run_id": "...",
  "timestamp": "...",
  "mode": "upstream",
  "condition": "RSI_OVERSOLD_TREND",
  "asset": "BTC-USDT",
  "decision": "REJECTED",
  "reason": "Trigger failed: yoshi:rsi lte 35 (current: 58.2)",
  "simulated": true
}
```

Daisy API read from `feed:simulation` expecting:
```json
{
  "source": "...",
  "event": "...",
  "status": "...",
  "detail": "..."
}
```

None of the Daisy fields existed in the Mario payload. All resolved to `null`. No exception raised anywhere. Discovered only during E2E validation.

Root cause: the ADR defined the intent ("simulated events go to feed:simulation") but specified no field-level schema. Mario and Daisy were written in separate blocks by separate sessions with no shared contract document.

---

## Prevention

1. **Define the payload schema before implementation, not after.** The ADR or block spec must include every field: name, type, source (which service writes it), and consumer mapping (which field the consumer reads it as).

2. **Explicit field mapping table in block specs** for any producer→consumer relationship:

| Producer field | Consumer field | Type | Notes |
|:--|:--|:--|:--|
| `decision` | `status` | `string` | `WOULD_EXECUTE` / `EXECUTED` / `REJECTED` / `INFO` |
| `reason` | `detail` | `string` | Human-readable rejection or execution note |

3. **Contract defined once, referenced by both blocks.** If producer and consumer are in different blocks, the producing block defines the schema; the consuming block references it explicitly.

4. **Acceptance criteria must include field-level assertions**, not just "data appears in the UI." E.g.: "FeedEntry.status equals `REJECTED`, not null."

---

## Detection

If you suspect a contract gap:

1. Read the raw payload directly from the source (Valkey, RabbitMQ, log output).
2. Read the consumer's field mapping code.
3. Check every field the consumer reads against every field the producer writes.
4. Any mismatch → gap confirmed.

For Valkey feeds in 42Bros:
```bash
# Read raw feed entries
docker exec <service> redis-cli -h <host> -p <port> --tls LRANGE feed:simulation 0 9
```

---

## Recovery

1. Identify all mismatched fields between producer and consumer.
2. Decide canonical field names (prefer producer names if producer is already deployed; prefer consumer names if consumer is already deployed; if both — decide with the user before touching either).
3. Update whichever side has fewer consumers first.
4. Do not mix fixes across both sides in the same PR — one change at a time, validate between.

---

## Related

- 42Bros block-12-06 issue triage (2026-03-05): `meta/logs/architecture/log-260305-1339-phase13-scoping.md`
- ADR-260301 §D3: `feed:simulation` intent defined without field-level schema

---

**Last Updated:** 2026-03-05
