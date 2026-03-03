# 3Commas API

**Category:** frameworks
**Last Updated:** 2026-03-03
**Entries:** 2

---

## Overview

Lessons learned integrating with the 3Commas Simple Trading API (`/ver1/trades`). Covers authentication, order creation, take-profit configuration, and API error handling.

---

## Quick Reference

| Task | Notes |
|:-----|:------|
| Create market order | `POST /ver1/trades` with `instant: true` — do NOT include `take_profit` |
| Create limit order | `POST /ver1/trades` with `instant: false` — `take_profit` supported |
| Auth | HMAC-SHA256: sign `{path}?{sorted_query}` with secret; send as `Signature` header |
| Pair format | Internal `BTC-USDT` → 3Commas `USDT_BTC` (split on `-`, reverse, join with `_`) |

---

## Entries

### Entry 1: `take_profit.steps` Rejected for Market Orders (instant=true) {#entry-1}

**Problem:**

Creating a Smart Trade with `instant: true` (market order) and a `take_profit.steps` block returns `record_invalid`:

```json
{
  "error": "record_invalid",
  "error_description": "Invalid parameters",
  "error_attributes": {
    "take_profit": ["is invalid"],
    "take_profit.steps": ["is invalid"],
    "take_profit.steps.0.price_type": ["can't be blank", "is not included in the list"]
  }
}
```

No combination of `price_type` values fixes this for market orders — the `steps` field is rejected entirely when `instant: true`.

**Root Cause:**

3Commas Smart Trade v2 API does not support take-profit steps for instant (market) orders. The take-profit mechanism requires a defined entry price to calculate exit levels, which market orders don't have at creation time.

**Solution:**

Skip `take_profit` entirely for `instant: true` orders:

```python
payload = {
    "account_id": account_id,
    "pair": pair,
    "instant": True,
    "units": {"value": str(quantity)},
}
# Do NOT add take_profit for market orders.
# The position can be managed via the 3Commas UI after creation.

if not payload.get("instant", False):
    # Limit orders only
    payload["take_profit"] = {
        "enabled": True,
        "price_type": "value",
        "steps": [
            {
                "order_type": "limit",
                "price": {"type": "value", "value": tp_value},
                "volume": 100,
            }
        ],
    }
```

**Context:**

- Confirmed broken: `instant: true` + `take_profit.steps` with any `price_type` value
- Limit order TP format (`instant: false`): deployed but not end-to-end confirmed against API (unit-tested only — see V10 in pipeline.md)
- First documented: 2026-03-03
- Source: 42bros Mario block-03-07-C T3 validation

**Tags:** `3commas` `smart-trade` `take-profit` `market-order` `instant` `record_invalid`

---

### Entry 2: `mario:bootstrap:approved` Must Be JSON-Encoded String {#entry-2}

**Problem:**

Setting `mario:bootstrap:approved` to the bare string `all` in Valkey causes Mario to reject the approval:

```
malformed value: 'all' — no approval granted
```

**Root Cause:**

Mario parses the Valkey value with `json.loads()`. The expected value is the JSON string `"all"` (with quotes), which when stored in Valkey appears as `'"all"'`. A bare `all` without JSON encoding is not valid JSON and fails parsing.

**Solution:**

```python
import json
conn.set("mario:bootstrap:approved", json.dumps("all"))
# Stored value: '"all"'  ← includes the JSON quotes
```

Or from redis-cli:
```bash
SET mario:bootstrap:approved '"all"'
```

**Context:**

- First documented: 2026-03-03
- Source: 42bros Mario block-03-07-C T2 bootstrap approval

**Tags:** `3commas` `mario` `bootstrap` `valkey` `json-encoding`

---

## Changelog

| Date       | Change                                                            | Source                  |
| ---------- | ----------------------------------------------------------------- | ----------------------- |
| 2026-03-03 | Initial creation with entries #1 (TP market orders) and #2 (bootstrap approval encoding) | block-03-07-C session |
