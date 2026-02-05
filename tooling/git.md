# Git

Lessons learned for Git version control.

---

## GPG Signing Causes Git Commands to Hang

**Problem:**
`git tag` or `git commit` hangs indefinitely with no output.

**Root Cause:**
GPG signing is enabled globally, and the GPG agent is waiting for a passphrase prompt that never displays in non-interactive environments (CI, AI assistants, scripts).

**Solution:**
Disable GPG signing for the affected operations:

```bash
# Check current settings
git config --global tag.gpgsign
git config --global commit.gpgsign

# Disable globally
git config --global tag.gpgsign false
git config --global commit.gpgsign false

# Or per-command
git tag -a v1.0.0 --no-sign -m "Release v1.0.0"
```

**Prevention:**
- Disable GPG signing in automated/AI environments
- Use repository-local config for personal signing preferences
- Document signing requirements in project contributing guide

**Context:**
- Tool/Version: git 2.x
- OS: macOS, Linux
- Note: Common when using AI coding assistants

**Tags:** `git` `gpg` `signing` `automation`

---
