# GitHub Actions

Lessons learned for GitHub Actions CI/CD workflows.

---

## Secrets Typo Causes Silent Deployment Failure

**Problem:**
GitHub Actions workflow fails silently or with cryptic Vault connection errors.

```
connection refused: localhost:8200
```

**Root Cause:**
A typo in GitHub repository secrets (e.g., `VAULT_ADDRVAULT_ADDR` instead of `VAULT_ADDR`) causes the environment variable to be undefined. The workflow defaults to localhost, failing to reach the actual Vault server.

**Solution:**
1. Audit all repository secrets for typos
2. Delete incorrect secrets
3. Re-add with correct names
4. Add validation step in workflow

```yaml
- name: Validate secrets
  run: |
    if [ -z "${{ secrets.VAULT_ADDR }}" ]; then
      echo "ERROR: VAULT_ADDR secret is missing"
      exit 1
    fi
```

**Prevention:**
- Use consistent naming conventions across repos
- Add secret validation step at start of workflows
- Document required secrets in repository README
- Periodically audit secrets across all repos

**Context:**
- Tool/Version: GitHub Actions
- Note: Discovered after weeks of silent failure

**Tags:** `github-actions` `secrets` `debugging`

---
