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

## RabbitMQ Event Format — YOSHI YOSHI_REPORT

Mario's `_on_event` handler expects these fields for YOSHI events:

```python
{
    "event": "YOSHI_REPORT",   # NOT "ANALYSIS_COMPLETED" (renamed in v0.10.8), NOT "event_type"
    "pair": "BTC-USDT",        # NOT "asset"
    "timeframe": "1h",
    "exchange": "kucoin",
    # ...other fields optional
}
```

Identified via `engine._identify_source()`: returns "YOSHI" only if `event["event"] == "YOSHI_REPORT"`.

Publishing via RabbitMQ HTTP API (for testing/smoke tests):
```bash
curl -u {user}:{pass} -H 'Content-Type: application/json' -X POST \
  http://{host}:15672/api/exchanges/%2F/42B_YOSHI/publish \
  -d '{"properties":{},"routing_key":"","payload":"{\"event\":\"YOSHI_REPORT\",\"pair\":\"BTC-USDT\",\"timeframe\":\"1h\",\"exchange\":\"kucoin\"}","payload_encoding":"string"}'
```

**Note:** `ANALYSIS_COMPLETED` was the pre-v0.10.8 literal — removed in block-14-03 (2026-03-07). Use `YOSHI_REPORT` for all new code.

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

---

## RabbitMQClient — Dedicated Thread Connection Drops Silently After Idle

**Problem:**
A `RabbitMQClient` created in a background daemon thread (e.g., a polling loop) may have its connection reset by the broker after a period of inactivity. `publish()` then raises `MessagingError: Stream connection lost: ConnectionResetError(104, 'Connection reset by peer')`. `ensure_connected()` does NOT detect stale connections — it only checks that the connection/channel objects exist, not whether the underlying socket is live.

```
ERROR | b42_common.messaging | Failed to publish message to exchange {EXCHANGE}:
  Stream connection lost: ConnectionResetError(104, 'Connection reset by peer')
```

**Root Cause:**
pika `BlockingConnection` TCP sockets can be silently reset by the broker or NAT after idle. `RabbitMQClient.ensure_connected()` guards against `None` but not against a stale socket. A daemon thread created at service startup may hold a connection for minutes or hours between publishes.

**Solution:**
Wrap `publish()` calls in long-lived threads with a reconnect-and-retry pattern:

```python
try:
    client.publish(exchange, event)
except Exception as pub_err:
    if client is self._own_rabbitmq_client:
        logger.warning(f"Dedicated connection lost ({pub_err}), reconnecting...")
        try:
            self._own_rabbitmq_client.connect()
            self._own_rabbitmq_client.declare_exchange(exchange)
            self._own_rabbitmq_client.publish(exchange, event)
            logger.info("Reconnected and published successfully")
        except Exception as reconnect_err:
            logger.warning(f"Reconnect failed ({reconnect_err}), falling back to shared client")
            self.shared_rabbitmq_client.publish(exchange, event)
    else:
        raise
```

**Prevention:**
- Always wrap `publish()` in daemon threads with reconnect + fallback logic
- Alternatively, configure RabbitMQ heartbeat on connection parameters to keep sockets alive and detect drops earlier
- Do NOT share a `pika.BlockingConnection` across threads — each thread needs its own connection

**Context:**
- pika 1.x, Python 3.12, RabbitMQ 3.x
- Manifests as intermittent missing notifications: Valkey write succeeds, RabbitMQ publish fails silently
- Idle threshold depends on broker/NAT config; observed at ~20s in some environments

**Source:** 42bros order-monitor fix, 2026-03-06
