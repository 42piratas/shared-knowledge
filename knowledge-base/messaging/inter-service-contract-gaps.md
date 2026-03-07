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

---

## Example 2: Simulate Scenario Key Format Mismatch

**42Bros — Playroom simulate scenario (2026-03-06):**

Daisy sent scenario keys as `"source:indicator"` (e.g. `"nabbit:regime": "BULL"`, `"yoshi:rsi": 25`). Mario's `_apply_scenario_overrides` only matched bare keys (`"regime"`, `"structure_score"`, `"macro_blackout"`). The `"nabbit:regime"` key fell through to the Yoshi indicator path, was silently applied as a Yoshi indicator, and the regime state was never set. Trigger evaluation always read `None` or stale Valkey → FAIL.

No error was raised. The simulation returned `FAIL` with message `"Trigger failed: nabbit:regime eq (value: BULL)"` — showing the scenario value WAS evaluated, but against the wrong state.

**Root cause:** Two separate blocks designed the simulate API payload (Daisy → Mario) without agreeing on whether scenario keys would be prefixed or bare. The spec said "scenario dict" without specifying key format.

**Fix:** Strip `source:` prefix in the consumer (`_apply_scenario_overrides`) before matching.

**Prevention:** Scenario/override key format must be explicit in the block spec. Either: (a) producer strips prefix before sending, or (b) consumer handles both formats.

---

## Related

- 42Bros block-12-06 issue triage (2026-03-05): `meta/logs/architecture/log-260305-1339-phase13-scoping.md`
- ADR-260301 §D3: `feed:simulation` intent defined without field-level schema
- 42Bros block-12-07 simulate fixes (2026-03-06): `meta/logs/engineering/log-260306-1527-block-12-07-playroom-v02-fixes.md`

---

---

## Example 3: Raw Dict Reads Missed During Rolling Field Rename

**42Bros — Toadette listeners.py (2026-03-07):**

During a rolling rename (`trigger_source` → `event_source`, `gates` → `conditions` in `MarioEvaluationEvent`), Phase B added a bidirectional Pydantic `model_validator` that populated both old and new field names in serialized JSON. This correctly protected all Pydantic `model_validate()` call sites.

However, `toadette/src/listeners.py` also read the RabbitMQ message body as a **raw dict** in `_persist_mario_decision()`, accessing `data["trigger_source"]` and `data.get("gates", {})` directly — bypassing Pydantic entirely. Phase C removed the bidirectional validator (narrowing to canonical names only), and the raw dict read broke at runtime:

```
ERROR | Invalid MARIO_EVALUATION payload: missing key 'trigger_source'
```

This was not caught by tests because test fixtures used Pydantic model instances, not raw dicts.

**Root Cause:** A rolling rename migration checked all Pydantic `model_validate()` call sites but missed a parallel raw-dict read path in the same service. The two paths were in different methods with no obvious link.

**Fix:** Use `.get()` with fallback for both field names at any raw dict read site:

```python
# ✅ Defensive raw dict read — accepts both old and new field names
trigger_source = data.get("event_source") or data.get("trigger_source")
gates = data.get("conditions") or data.get("gates") or {}
```

**Prevention:**
- Before marking a rolling rename complete, grep for raw dict reads (`data["old_name"]`, `data.get("old_name")`) in every consumer, not just Pydantic `model_validate()` sites
- Add a Phase B migration checklist item: "grep all repos for old field name as dict key literal"
- Contract tests should use raw dict payloads (not Pydantic instances) where raw-dict code paths exist

**Source:** log-260307-1130-block-14-03-common-class-renames.md

---

**Last Updated:** 2026-03-07
