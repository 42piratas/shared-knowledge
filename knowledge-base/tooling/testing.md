# Testing

Lessons learned for software testing patterns and practices.

---

## Async Test Functions Require Async Mocking After Sync-to-Async Migration

**Problem:**
Tests calling an async function return a coroutine object instead of the expected result. Assertions like `assert result is None` pass incorrectly (because the coroutine object is truthy), or fail with confusing comparison errors.

**Root Cause:**
A function was migrated from synchronous (e.g., `requests.get`) to asynchronous (e.g., `httpx.AsyncClient`), but the tests were not updated. They call the async method without `await` and mock the old synchronous library instead of the new async one.

```python
# BROKEN — calls async function without await, mocks wrong library
@patch("src.clients.my_client.requests.get")
def test_fetch(mock_get):
    result = client.fetch_data()  # Returns coroutine, not actual result
    assert result is None  # May pass incorrectly — coroutine is truthy!
```

**Solution:**

1. Mark test functions with `@pytest.mark.asyncio` and `async def`
2. Use `AsyncMock` for the HTTP client context manager
3. Patch the new async library at the correct module path
4. `await` the async method call

```python
# CORRECT — async test with proper mocking
import pytest
from unittest.mock import AsyncMock, patch, MagicMock

@pytest.mark.asyncio
@patch("src.clients.my_client.httpx.AsyncClient")
async def test_fetch(mock_client_class):
    mock_response = MagicMock()
    mock_response.json.return_value = {"value": 42}
    mock_response.status_code = 200

    mock_client = AsyncMock()
    mock_client.__aenter__.return_value = mock_client
    mock_client.get.return_value = mock_response
    mock_client_class.return_value = mock_client

    result = await client.fetch_data()
    assert result == {"value": 42}
```

**Prevention:**

- When changing a function from sync to async, always search for existing tests and callers that need updating: `grep -r "function_name" tests/`
- Run `pytest` locally before pushing — coroutine-related failures are immediately visible
- Use `pytest-asyncio` and configure it in `pyproject.toml`: `asyncio_mode = "auto"` or mark tests explicitly

**Context:**

- Tool/Version: pytest, pytest-asyncio, httpx
- Key detail: `AsyncMock` is in `unittest.mock` since Python 3.8 — no extra dependency needed
- First documented: 2026-02-11
- Source: `260211-1604-lessons-learned.md`

**Tags:** `testing` `async` `mocking` `httpx` `pytest-asyncio` `migration`

---

## Unit Tests with Mocked Data Don't Catch Schema Mismatches

**Problem:**
Unit tests pass but production fails with data parsing errors.

```
unsupported operand type(s) for /: 'dict' and 'float'
```

**Root Cause:**
Unit tests mocked data with flat structure `{"score": 60.0}` but production service publishes nested structure `{"score": {"value": 60.0, ...}}`. The mocked data didn't match the actual producer output.

**Solution:**

1. Create integration tests that validate against real producer output
2. Use production data samples as test fixtures
3. Add contract tests between services

```python
# Use actual production data structure in tests
PRODUCTION_PEACH_REPORT = {
    "score": {
        "value": 60.15,
        "regime": "NEUTRAL_ACCUMULATION",
        "components": {...}
    },
    "microstructure": {...}
}

def test_parse_production_structure():
    result = parse_peach_report(PRODUCTION_PEACH_REPORT)
    assert result.score == 60.15
```

**Prevention:**

- Always capture real production data for test fixtures
- Implement integration tests between services
- Use contract testing (Pact, etc.) for service boundaries
- Review test data against actual API responses

**Context:**

- Tool/Version: pytest
- Other relevant context: Particularly important for microservices

**Tags:** `testing` `integration-tests` `schema` `mocking`

---
