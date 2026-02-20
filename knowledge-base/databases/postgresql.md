# Knowledge Base: PostgreSQL Configuration

**Category:** `databases/`
**Topic:** `postgresql.md`
**Last Updated:** 2026-02-20

---

## 1. Managed Service Connections (DigitalOcean / AWS / GCP)

### 1.1. Idle Connection Timeouts

Managed PostgreSQL services (DigitalOcean, AWS RDS) typically enforce strict idle connection timeouts (e.g., 5-10 minutes). If a pooled connection sits idle for longer than this limit, the server closes it.

**Symptom:**
- `OperationalError: server closed the connection unexpectedly`
- Applications work fine initially, then fail after periods of inactivity
- Daemon threads or listeners crash silently if error handling is poor

**Solution: `pool_pre_ping=True`**

When using SQLAlchemy, ALWAYS enable `pool_pre_ping=True` for long-lived applications (web servers, consumers, ETL jobs). This flag causes the engine to emit a lightweight `SELECT 1` before checking out a connection from the pool. If the connection is stale, it's discarded and a fresh one is created transparently.

**Implementation (b42_common pattern):**

```python
from sqlalchemy import create_engine

engine = create_engine(
    url,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True  # CRITICAL for managed DBs
)
```

**Anti-Pattern (Do NOT do this):**

```python
# Creates a bare engine without pre-ping -> stale connections will crash app
engine = create_engine(url)
```

### 1.2. SSL Mode

Managed databases usually require SSL. Ensure connection URLs include `?sslmode=require`.

```python
url = f"postgresql://{user}:{password}@{host}:{port}/{dbname}?sslmode=require"
```

## 2. Connection Pooling Best Practices

- **Web Servers (FastAPI/Flask):** Use a global engine instance created at startup. Do NOT create a new engine per request.
- **Worker Threads (Celery/Daemon):**
  - Use `pool_pre_ping=True` to handle long idle periods between tasks.
  - Wrap database operations in retry loops for transient `OperationalError`.
  - Ensure sessions are scoped correctly (created and closed within the task unit).

## 3. Schema Management (Alembic)

- Always use Alembic for schema migrations.
- Store migrations in the service repository (`alembic/versions/`).
- Use `autogenerate` for initial drafts, but review manually for accuracy.
- Test `upgrade` and `downgrade` locally before applying to production.

## 4. Troubleshooting

**"OperationalError: server closed the connection unexpectedly"**
- **Cause:** Idle connection timeout or network blip.
- **Fix:** Enable `pool_pre_ping=True` and implement application-level retries.

**"IntegrityError: duplicate key value violates unique constraint"**
- **Cause:** Race condition or idempotency failure.
- **Fix:** Use `ON CONFLICT DO UPDATE` (upsert) for idempotent writes, or catch and ignore for at-most-once delivery.

**"DataError: value too long for type"**
- **Cause:** String truncation or numeric overflow.
- **Fix:** Check column definitions vs input data. Use `text` instead of `varchar` if length is unbounded. Ensure `NUMERIC` precision matches financial requirements (e.g., `NUMERIC(20,8)` for crypto prices).
