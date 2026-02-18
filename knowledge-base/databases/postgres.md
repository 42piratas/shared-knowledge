# PostgreSQL

**Category:** databases
**Last Updated:** 2026-02-18
**Entries:** 1

---

## Overview

Lessons learned for PostgreSQL database management, SQLAlchemy ORM integration, and migration strategies.

---

## Entries

### Entry 1: SQLAlchemy Reserved Word Conflict with 'metadata' Column {#entry-1}

**Problem:**
A database migration creates a table with a column named `metadata`. When trying to map this table using SQLAlchemy's Declarative system, the application crashes with:

```
sqlalchemy.exc.InvalidRequestError: Attribute name 'metadata' is reserved when using the Declarative API.
```

**Root Cause:**
`metadata` is a reserved attribute name in SQLAlchemy's `DeclarativeBase`. It stores the `MetaData` collection of table definitions. Attempting to define a column field named `metadata` on a model class overwrites this internal attribute, causing the ORM initialization to fail.

**Solution:**
Rename the column in the model and database. Common alternatives:
- `meta`
- `item_metadata`
- `_metadata` (if strictly internal)
- `properties`
- `attributes`

Example Fix:

```python
# ❌ Bad
class MyModel(Base):
    metadata = Column(JSONB)  # CONFLICT!

# ✅ Good
class MyModel(Base):
    resource_metadata = Column(JSONB)
```

**Prevention:**
- Avoid naming columns `metadata` in SQLAlchemy models.
- If the database column *must* be named `metadata` (legacy schema), use a different attribute name on the model:
  ```python
  resource_metadata = Column("metadata", JSONB)
  ```

**Context:**
- Versions affected: SQLAlchemy 1.4+
- First documented: 2026-02-18
- Source: `260218-0130-fix-regime-persistence.md` (Session Log)

**Tags:** `postgres` `sqlalchemy` `orm` `naming-convention`

---

## Changelog

| Date       | Change                          | Source      |
| ---------- | ------------------------------- | ----------- |
| 2026-02-18 | Initial creation with 1 entry   | Session Log |
