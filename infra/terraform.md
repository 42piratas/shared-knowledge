# Terraform

Lessons learned for Terraform infrastructure-as-code.

---

## Terraform/Vault Circular Dependency (DR Vulnerability)

**Problem:**
Terraform cannot plan/apply because it depends on Vault for secrets, but Vault runs on infrastructure managed by Terraform. If Vault is destroyed, Terraform is deadlocked.

```
context deadline exceeded
Error reading from Vault: connection refused
```

**Root Cause:**
Terraform Vault provider reads secrets during the planning phase. If Vault is unavailable (destroyed, unreachable), Terraform cannot even generate a plan to recreate it.

**Solution:**
1. Comment out Vault provider and all `data.vault_kv_secret_v2` blocks
2. Add defaults to Vault variables: `default = ""`
3. Store bootstrap secrets (DO token) outside Vault for DR
4. Run `terraform apply` to recreate infrastructure
5. Restore Vault from backup
6. Uncomment Vault integration and reapply

```
# In variables.tf during DR
variable "vault_role_id" {
  default = ""  # Allow empty during DR
}
```

**Prevention:**
- Store critical bootstrap secrets (cloud provider tokens) in environment variables OR separate secret manager
- Maintain offline backup of DO API token
- Document DR procedure in playbook
- Consider blue-green infrastructure for migrations

**Context:**
- Tool/Version: Terraform 1.x, Vault 1.x
- Discovered during region migration from nyc3 to ams3

**Tags:** `terraform` `vault` `disaster-recovery` `secrets`

**Source:** 260203-1500-retroactive-lessons-learned.md
