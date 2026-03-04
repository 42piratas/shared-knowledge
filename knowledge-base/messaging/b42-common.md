# b42-common Knowledge Base

Knowledge about the `b42_common` shared Python library used across 42Bros services.

---

## FeedEvent Constructor — Required Fields

**Problem:**
Instantiating `FeedEvent` with `service` and `message` kwargs causes a `ValidationError` at runtime — these are not valid fields.

```python
# WRONG — causes: ValidationError: source: Field required, detail: Field required
FeedEvent(
    timestamp_utc=now_iso,
    service="mario",          # ❌ not a field
    category=FeedCategory.INFRA_WARNING,
    level="WARNING",
    message=msg,              # ❌ not a field
)
```

**Correct Fields:**
```python
FeedEvent(
    timestamp_utc=now_iso,
    source="MARIO",           # ✅ required, uppercase service name
    detail=msg,               # ✅ required, the message text
    category=FeedCategory.INFRA_WARNING,
    level="WARNING",          # optional, default "INFO"
    # Optional: asset, indicator, indicator_value, threshold_value, timeframe, metadata
)
```

**Full signature:**
```
FeedEvent(*, source: str, timestamp_utc: str, detail: str, category: FeedCategory,
          asset=None, indicator=None, indicator_value=None, threshold_value=None,
          timeframe=None, level: str = "INFO", metadata=None)
```

**Source:** block-03-11 smoke test, 2026-03-04

---

## RabbitMQ Event Format — YOSHI ANALYSIS_COMPLETED

Mario's `_on_event` handler expects these fields for YOSHI events:

```python
{
    "event": "ANALYSIS_COMPLETED",   # NOT "event_type"
    "pair": "BTC-USDT",              # NOT "asset"
    "timeframe": "1h",
    "exchange": "kucoin",
    # ...other fields optional
}
```

Identified via `engine._identify_source()`: returns "YOSHI" only if `event["event"] == "ANALYSIS_COMPLETED"`.

Publishing via RabbitMQ HTTP API (for testing/smoke tests):
```bash
curl -u {user}:{pass} -H 'Content-Type: application/json' -X POST \
  http://{host}:15672/api/exchanges/%2F/42B_YOSHI/publish \
  -d '{"properties":{},"routing_key":"","payload":"{\"event\":\"ANALYSIS_COMPLETED\",\"pair\":\"BTC-USDT\",\"timeframe\":\"1h\",\"exchange\":\"kucoin\"}","payload_encoding":"string"}'
```

All `42B_*` exchanges are **fanout** type on the `/` (default) vhost.

**Source:** block-03-11 smoke test, 2026-03-04

---

## RabbitMQClient Constructor

```python
RabbitMQClient(
    host=os.environ["RABBITMQ_HOST"],   # Use env var, NOT "rabbitmq" hostname
    user=os.environ["RABBITMQ_USER"],   # "username" is WRONG
    password=os.environ["RABBITMQ_PASS"],
    port=int(os.environ["RABBITMQ_PORT"]),
)
```

Note: `username=` is not a valid kwarg — must be `user=`.

**Source:** block-03-11 smoke test, 2026-03-04
