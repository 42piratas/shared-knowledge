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
