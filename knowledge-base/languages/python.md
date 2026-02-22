# Python

Lessons learned for Python development patterns and tooling.

---

## Ruff Linter Errors Block CI/CD Pipeline

**Problem:**
CI/CD fails with multiple linting errors after adding new code.

```
F401 'module.function' imported but unused
F841 local variable 'x' is assigned to but never used
E701 multiple statements on one line
```

**Root Cause:**
Rapid development introduces unused imports, variables, and style violations that aren't caught until CI runs. Integration tests particularly prone to having unused fixtures.

**Solution:**

1. Run linter locally before pushing
2. Fix unused imports by removing or using `# noqa: F401`
3. Prefix intentionally unused variables with underscore: `_unused_var`
4. Split multi-statement lines

```python
# For necessary but "unused" imports (like pandas_ta accessor registration)
import pandas_ta  # noqa: F401

# For intentionally unused variables
_mock_channel, _mock_properties = mock_callback_args
```

**Prevention:**

- Configure pre-commit hooks for ruff
- Run `ruff check .` before every commit
- Configure IDE to show linting errors inline
- Add linting step early in CI pipeline

**Context:**

- Tool/Version: ruff 0.2+
- Note: Replace flake8/pylint with ruff for speed

**Tags:** `python` `linting` `ruff` `ci-cd`

---

## `datetime.utcnow()` Deprecated — Use `datetime.now(timezone.utc)`

**Problem:**
`datetime.utcnow()` is deprecated since Python 3.12 and scheduled for removal. It produces a **naive** datetime (no timezone info), which causes subtle bugs when mixed with timezone-aware datetimes or when running on servers not set to UTC.

```python
# DEPRECATED — produces naive datetime, causes DeprecationWarning in Python 3.12+
from datetime import datetime
ts = datetime.utcnow()
```

**Solution:**

```python
# CORRECT — produces timezone-aware UTC datetime
from datetime import datetime, timezone
ts = datetime.now(timezone.utc)

# Python 3.11+ shorthand (also correct)
from datetime import UTC
ts = datetime.now(UTC)
```

**Gotchas:**

- APScheduler's `next_run_time` parameter should also use timezone-aware datetimes (`datetime.now(UTC)`) to avoid scheduling drift on non-UTC servers
- `datetime.utcnow()` and `datetime.now(timezone.utc)` produce the same _time value_ but different types — naive vs aware — which causes `TypeError` when comparing them

**Tags:** `python` `datetime` `timezone` `deprecation`

---

## Incremental Calculation for Financial Indicators Performance Optimization

**Problem:**
Financial indicator calculations (EMA, RSI, SMA, MACD) recalculating entire dataframes (~300+ candles) on every cycle, causing ~99% computational waste when only 1 new candle is added per cycle.

```python
# Before: Recalculates all 300 candles every cycle
def calculate_indicators(df, config):
    for indicator in config["indicators"]:
        df[f"{indicator}_20"] = calculate_ema(df["close"], 20)  # Processes all 300 rows
```

**Root Cause:**
Traditional indicator libraries (pandas-ta, ta-lib) operate on full datasets without state persistence. Each cycle processes historical data that was already calculated in previous cycles.

**Solution:**
Implement incremental calculation with cached intermediate state:

1. **Cache Management**: Store intermediate values (previous EMA, RSI gain/loss averages) in Redis/Valkey with TTL
2. **Incremental Formulas**: Use mathematical properties to update indicators with only new data
3. **Fallback Strategy**: Auto-fallback to full calculation on cache miss or errors

```python
# After: Only processes new candle using cached state
class IncrementalIndicatorCalculator:
    def calculate_ema_incremental(self, new_price, cached_ema, multiplier):
        return (new_price * multiplier) + (cached_ema * (1 - multiplier))

    def calculate_rsi_incremental(self, new_price, prev_price, cached_avg_gain, cached_avg_loss, period):
        gain = max(0, new_price - prev_price)
        loss = max(0, prev_price - new_price)
        # Wilder's smoothing (RMA)
        new_avg_gain = (cached_avg_gain * (period - 1) + gain) / period
        new_avg_loss = (cached_avg_loss * (period - 1) + loss) / period
        rs = new_avg_gain / new_avg_loss if new_avg_loss != 0 else 0
        return 100 - (100 / (1 + rs))
```

**Prevention:**

- Design indicator systems with state persistence from start
- Use cache-first approach for any rolling calculations
- Implement cache invalidation strategies for data gaps
- Maintain Decimal precision throughout for financial calculations

**Context:**

- Performance: ~15,000 operations → ~10 operations per cycle (99% reduction)
- Memory: Minimal increase (~1KB per asset/timeframe for cache keys)
- Accuracy: 100% identical outputs maintained with proper Decimal math
- Tools: Redis/Valkey for cache, pandas for data handling

**Tags:** `python` `performance` `financial-calculations` `caching` `incremental-algorithms` `redis`

---

## Decimal Objects Don't Have `.round()` — Use Builtin `round()`

**Problem:**
Code using `x.round(n)` on `Decimal` values crashes with `AttributeError: 'Decimal' object has no attribute 'round'`.

```
AttributeError: 'Decimal' object has no attribute 'round'
```

**Root Cause:**
Python's `Decimal` class implements `__round__` (the dunder method called by the builtin `round()` function), but does **not** expose a `.round()` instance method. This differs from pandas `Series`/`DataFrame` which have `.round()`, and from numpy arrays which also have `.round()`. The APIs are easily conflated.

Note: `pandas Series.round()` internally calls `round()` on each element for object-dtype columns, which is why `df[col].round(n)` works on a Series of Decimals — but calling `.round()` on an individual `Decimal` does not.

**Solution:**
Use the Python builtin `round(x, n)` instead of `x.round(n)`:

```python
# WRONG — Decimal has no .round() method
df[col] = df[col].apply(lambda x: x.round(4) if x is not None else None)

# CORRECT — builtin round() calls Decimal.__round__
df[col] = df[col].apply(lambda x: round(x, 4) if x is not None else None)
```

**Prevention:**

- Always validate planned code against the actual types in the codebase before implementing
- When reviewing code from other agents or iteration plans, treat proposed code as pseudocode — verify API compatibility
- Remember the distinction: `round(decimal_val, n)` ✅ vs `decimal_val.round(n)` ❌

**Context:**

- Versions affected: Python 3.x (all)
- OS: all
- First documented: 2026-02-10

**Tags:** `python` `decimal` `rounding` `pandas` `api-mismatch`

---
