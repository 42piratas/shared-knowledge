# Testing

Lessons learned for software testing patterns and practices.

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
