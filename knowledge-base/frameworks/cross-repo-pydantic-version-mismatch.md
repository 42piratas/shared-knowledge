# Cross-Repo: Pydantic Literal Validation Fails When Dependency Not Updated

## Problem

When updating a Pydantic model's `Literal` field in a shared library (`b42_common`), code in a consumer repo that uses the new literal value will fail with:

```
pydantic_core._pydantic_core.ValidationError: 1 validation error for ModelName
field_name
  Input should be 'old_value', ... [type=literal_error, input_value='new_value', ...]
```

This happens when:
1. The consumer's test environment (or production) still has the **old version** of the shared library installed.
2. The consumer code creates model instances with the **new literal value** before the library is updated.

## Root Cause

Poetry pins exact versions in `poetry.lock`. If `b42-common` is pinned to `tag = "v0.10.1"` but the new literal is in `v0.10.2`, the installed package rejects the new value.

In `42Bros`: this manifests during local testing after the consumer branch is created but before `42bros-common` is merged, tagged, and the consumer's `pyproject.toml` is updated.

## Impact in 42Bros (Block 03-12)

`MarioCircuitEvent.trigger` changed from `"daily_loss"` to `"daily_exposure"`. Mario's `_trip_circuit_breaker()` created the event with `trigger="daily_exposure"` while the installed `b42_common` still had `v0.10.1`. The `ValidationError` propagated through the outer `except Exception` block, causing `_check_circuit_breaker()` to return `False` even though `self.circuit_tripped` was already `True`.

## Fixes Applied

**Code hardening (Mario):**
- Wrapped `MarioCircuitEvent(...)` creation inside the existing try/except block in `_trip_circuit_breaker()` so validation failures are caught locally and don't propagate.
- Changed `_check_circuit_breaker()` to `return self.circuit_tripped` instead of literal `True`, so circuit state is correct even if event publish fails.

**Process fix (Cross-Repo Ordering):**
- Follow `skill-branching-strategy.md` §Cross-Repo Ordering: merge and **tag** `42bros-common` first, then update the consumer's `pyproject.toml` and run `poetry update b42-common`, then run tests.

## Prevention

1. **Always run tests AFTER updating the dependency pin**, not before.
2. **Wrap Pydantic model construction in try/except** for publish paths — a model validation failure should never prevent critical state changes.
3. **Test the pattern:** state mutations (Valkey writes, in-memory flag changes) must happen BEFORE any potentially-failing publish calls.

## Affected Files (B03-12)
- `42bros-mario/src/core/engine.py` `_trip_circuit_breaker()`
- `42bros-common/src/b42_common/models/events.py` `MarioCircuitEvent.trigger`

**Date:** 2026-03-04
