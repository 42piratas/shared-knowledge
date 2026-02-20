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

---

## UFW Changes Must Be in IaC

**Problem:**
Manual UFW changes made via SSH caused configuration drift. When droplet was rebuilt, the manual changes were lost, but the expectation remained that Tailscale-only SSH would work.

**Root Cause:**
During security hardening, UFW rules were modified manually:

```bash
# Run manually on server - BAD
ufw delete allow 22/tcp
ufw allow in on tailscale0 to any port 22 proto tcp
```

These changes were not reflected in Terraform's cloud-init user_data script, so subsequent rebuilds had different UFW configurations.

**Solution:**
All UFW configuration must be in Terraform user_data (cloud-init):

```hcl
user_data = <<-EOF
#!/bin/bash
# ... other setup ...

# UFW configuration - ALL rules here, no manual changes
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow in on tailscale0 to any port 22 proto tcp comment 'SSH via Tailscale'
ufw allow from 10.0.0.0/8 to any port 22 proto tcp comment 'Emergency SSH'
ufw allow 41641/udp comment 'Tailscale'
ufw --force enable
EOF
```

**Prevention:**

- Never run UFW commands manually on production servers
- All firewall rules in Terraform cloud-init or configuration management
- Add prominent warnings in documentation: "NEVER modify UFW rules manually"
- If emergency manual changes are needed, immediately update IaC and redeploy

**Context:**

- Tool/Version: Terraform 1.x, UFW
- OS: Ubuntu 22.04

**Tags:** `terraform` `ufw` `iac` `configuration-drift`

---

## Vault Provider `address` Overrides `VAULT_ADDR` Env Var

**Problem:**
After restricting a firewall to VPC-only access, `terraform plan` fails with `context deadline exceeded` even though `VAULT_ADDR` env var is set to `http://127.0.0.1:8282` (SSH tunnel). Terraform ignores the env var and connects to the public IP, which is now blocked.

```
Error: failed to lookup token, err=context deadline exceeded
  with data.vault_kv_secret_v2.some_secret
```

**Root Cause:**
When the Vault provider explicitly sets `address` via a Terraform variable, it takes precedence over the `VAULT_ADDR` environment variable:

```hcl
# variables.tf
variable "vault_addr" {
  default = "http://vault.example.com:8282"  # Public DNS → public IP
}

# main.tf
provider "vault" {
  address = var.vault_addr  # This OVERRIDES VAULT_ADDR env var
}
```

The `VAULT_ADDR` env var only applies if the provider config omits the `address` field entirely. Setting the env var while the provider has an explicit `address` does nothing.

**Solution:**

1. Change the default in `variables.tf` to localhost (tunnel-first, secure default):

```hcl
variable "vault_addr" {
  default = "http://127.0.0.1:8282"
}
```

2. Use SSH tunnel for all Terraform operations:
   - Terminal 1: `ssh -i {KEY} -L 8282:localhost:8282 -N root@{SERVER}`
   - Terminal 2: `export VAULT_TOKEN={TOKEN} && terraform plan`

3. For emergency direct access, override at runtime:

```bash
terraform plan -var="vault_addr=http://{PUBLIC_IP}:8282"
```

**Prevention:**

- Default `vault_addr` to localhost in `variables.tf` — never default to a public address
- Use a passphrase-free SSH key for tunnel scripts to avoid interactive prompts breaking automation
- Document the tunnel requirement in your infrastructure playbook
- If firewall changes block Vault, use cloud provider console to temporarily add an inbound rule, apply Terraform, then let Terraform remove the temporary rule on next apply

**Context:**

- Tool/Version: Terraform 1.x, Vault provider ~> 3.0, DigitalOcean Cloud Firewalls
- DigitalOcean firewalls operate at the hypervisor level — restricting a port to VPC CIDR blocks all external access including traffic that would arrive via SSH tunnel if the client resolves to the public IP

**Tags:** `terraform` `vault` `firewall` `ssh-tunnel` `provider-config`

---
