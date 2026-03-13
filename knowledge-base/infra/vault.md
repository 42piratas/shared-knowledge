# HashiCorp Vault

**Category:** infra
**Last Updated:** 2026-03-14
**Entries:** 4

---

## Overview

Lessons learned operating HashiCorp Vault in a self-hosted, VPC-restricted environment.

---

## Entries

### Entry 1: Vault Listening Port (8282) {#entry-1}

**Critical:** The Vault instance on `shroom-x` listens on **port 8282**, not the default 8200.

- **Symptoms:**
  - `vault status` returns `dial tcp 127.0.0.1:8200: connect: connection refused`
  - GitHub Actions fail with `connection refused` or timeout
- **Correction:**
  - Always set `export VAULT_ADDR='http://127.0.0.1:8282'` before running CLI commands
  - Check `/etc/vault.d/vault.hcl` for authoritative configuration
  - Use `netstat -tlnp | grep vault` to confirm listening port

**Tags:** `vault` `port` `configuration` `cli`

---

### Entry 2: Unsealing Procedure {#entry-2}

Vault seals automatically on restart (e.g., DigitalOcean maintenance, manual `systemctl stop vault`). Threshold: 3 of 5 keys required.

**Steps (SSH into shroom-x):**

```bash
ssh -i ~/.ssh/42b-do-key-auto root@104.248.89.150
export VAULT_ADDR="https://10.50.0.3:8282"
export VAULT_SKIP_VERIFY=true     # self-signed cert, trusted private network
vault status                      # confirm Sealed: true
vault operator unseal             # paste key 1, Enter
vault operator unseal             # paste key 2, Enter
vault operator unseal             # paste key 3, Enter
# Sealed: false confirms success
```

**Notes:**
- Vault runs as `systemd` — use `systemctl stop/start vault`, NOT `docker stop`
- `VAULT_SKIP_VERIFY=true` is correct for internal VPC access (Tailscale mTLS handles network security)
- No tunnel needed — connect directly via SSH, run unseal on-server
- Unseal keys are held by the operator (not stored on server or in repo)

**Tags:** `vault` `unseal` `restart` `operations`

---

### Entry 3: VPC Firewall Restriction Breaks CI/CD Vault Auth — Diagnose and Fix {#entry-3}

**Problem:**
All CI/CD deploy jobs suddenly fail with a timeout at the Vault authentication step, across all services simultaneously:

```
Error writing data to auth/approle/login: Put "http://{PUBLIC_VAULT_HOST}:8282/v1/auth/approle/login": dial tcp {IP}:8282: i/o timeout
```

**Root Cause:**
A Terraform/IAC commit correctly restricted Vault's firewall rule from `0.0.0.0/0` to VPC-only (`10.x.x.x/16`). The deploy workflows were authenticating with Vault from GitHub Actions runners (internet-facing) — now blocked. The firewall rule takes effect when `terraform apply` runs, not when the IAC commit is pushed. This creates a confusing delay: IAC commit lands, a few deploys still succeed (Terraform not yet applied), then all deploys silently break.

**Diagnosis steps:**

1. Check recent IAC commits: `git log --oneline` in the IaC repo — look for "firewall", "vpc", "restrict"
2. Confirm the error is `i/o timeout` (network blocked) vs `403 Forbidden` (wrong credentials)
3. Verify Vault is actually running: `ssh {droplet} "curl -s http://{VPC_PRIVATE_IP}:8282/v1/sys/health"`
4. If health check returns JSON from inside the droplet but the runner times out → VPC firewall is the cause

**Solution:**
Move all Vault authentication and secret fetching inside the SSH deploy step. The droplet is inside the VPC and can reach Vault over private networking.

Use `appleboy/ssh-action`'s `envs:` parameter to inject `VAULT_ROLE_ID` and `VAULT_SECRET_ID` into the remote shell — no secrets are exposed in logs:

```yaml
- name: SSH and deploy
  uses: appleboy/ssh-action@master
  with:
    host: ${{ secrets.DROPLET_HOSTNAME }}
    username: ${{ secrets.DROPLET_USER }}
    key: ${{ secrets.DEPLOY_KEY_PRIVATE }}
    envs: VAULT_ROLE_ID,VAULT_SECRET_ID,GH_PAT,GITHUB_ACTOR,IMAGE_NAME
    script: |
      set -e

      # Install Vault CLI on server (idempotent)
      if ! command -v vault &> /dev/null; then
        wget -q https://releases.hashicorp.com/vault/1.9.0/vault_1.9.0_linux_amd64.zip
        unzip -q vault_1.9.0_linux_amd64.zip
        mv vault /usr/local/bin/vault
        rm vault_1.9.0_linux_amd64.zip
      fi

      # Authenticate using VPC-internal address (hardcoded private IP)
      export VAULT_ADDR="http://{VPC_PRIVATE_IP}:8282"
      VAULT_TOKEN=$(vault write -field=token auth/approle/login \
        role_id="$VAULT_ROLE_ID" secret_id="$VAULT_SECRET_ID")
      [ -z "$VAULT_TOKEN" ] && echo "ERROR: Vault auth failed" && exit 1
      export VAULT_TOKEN

      # Fetch secrets and write .env
      MY_SECRET=$(vault kv get -field=my_secret secret/myservice)
      cat > .env <<EOF
MY_SECRET=$MY_SECRET
EOF
```

**Key rules:**

- `VAULT_ADDR` inside the SSH script must be the **VPC-internal private IP** — hardcoded, not `${{ secrets.VAULT_ADDR }}` (that secret holds the public URL, now firewalled)
- `envs:` lists variable names (not values) — they become `$VAR` in the remote shell
- Delete the runner-side steps: `Install Vault CLI`, `Authenticate with Vault`, `Fetch secrets from Vault`
- The SCP step (copying `docker-compose.yml`) is unaffected — it uses SSH port 22, not Vault port

**Prevention:**

- When restricting Vault firewall rules to VPC-only, immediately audit all CI/CD workflows — any runner-side Vault call will break on the next `terraform apply`
- Store the VPC-internal Vault address in the infra playbook so any agent can find it without secrets
- Never authenticate to Vault from a CI/CD runner if Vault is VPC-restricted

**Context:**

- Versions: HashiCorp Vault 1.9.0, GitHub Actions, `appleboy/ssh-action`, DigitalOcean VPC
- First documented: 2026-02-20
- Source: Session `260220-1930-toadette-fix-daisy-tests-vault-ci-plan`

**Tags:** `vault` `vpc` `firewall` `ci-cd` `github-actions` `approle` `server-side-auth`

---

### Entry 4: AppRole 403 on Service-Specific Secret Path Despite Correct Policy {#entry-4}

**Problem:**
Deploy workflow gets 403 `permission denied` reading `secret/data/{service}` even though:
- The Vault policy grants `read` on `secret/data/{service}`
- The AppRole's `token_policies` include the service policy
- Reading the same path with a freshly generated AppRole token (via `vault write auth/approle/role/{service}/secret-id`) works fine

```
Error reading secret/data/bowser: Error making API request.
URL: GET https://10.50.0.3:8282/v1/secret/data/bowser
Code: 403. Errors: * 1 error occurred: * permission denied
```

**Root Cause:**
The `VAULT_ROLE_ID` and `VAULT_SECRET_ID` stored in GitHub Actions secrets may be stale — they were set when the AppRole was first created but Terraform may have recreated the AppRole since then (e.g., policy changes, `for_each` modifications), generating new IDs. The old IDs either authenticate as a different/expired role or get a token with outdated policy attachments.

**Solution:**
1. **Immediate workaround:** If the secret can logically live in `secret/common`, move it there — all AppRoles have `common` policy and it always works.
2. **Proper fix:** Regenerate the AppRole role_id and secret_id from Vault, update the GH Actions secrets:
   ```bash
   vault read -field=role_id auth/approle/role/{service}/role-id
   vault write -field=secret_id -f auth/approle/role/{service}/secret-id
   # Update VAULT_ROLE_ID and VAULT_SECRET_ID in GH repo secrets
   ```

**Prevention:**
- After any `terraform apply` that touches `vault_approle_auth_backend_role`, verify GH Actions secrets still match
- Prefer `secret/common` for secrets shared across services (e.g., `jwt_secret` used by both Wario and Bowser)
- Test AppRole access before relying on it in deploy: `vault write auth/approle/login role_id=... secret_id=...` then read the target path

**Context:**
- Versions: Vault 1.15.6, Terraform-managed AppRoles via `for_each`
- First documented: 2026-03-14
- Source: Block 32-05 (Bowser auth middleware deploy)

**Tags:** `vault` `approle` `403` `permission-denied` `github-actions` `stale-credentials`

---

## Common Issues

- **CI/CD Failure — Vault sealed (503):** The server restarted. Unseal manually (see Entry 2).
- **CI/CD Failure — Connection refused:** Check `VAULT_ADDR` is set to port 8282.
- **CI/CD Failure — i/o timeout:** Vault port is VPC-restricted. See Entry 3.
- **CI/CD Failure — 403 on service secret despite correct policy:** Stale AppRole credentials in GH Actions. See Entry 4.

---

## Changelog

| Date       | Change                                                       | Source              |
| ---------- | ------------------------------------------------------------ | ------------------- |
| 2026-02-20 | Reformatted, added Entry 3 (VPC firewall / server-side auth) | Session 260220-1930 |
| 2026-03-03 | Entry 2 updated: fix http→https, add VAULT_SKIP_VERIFY, note systemd vs Docker, unseal threshold | Session 260303-2100 |
| 2026-03-14 | Entry 4 added: AppRole 403 on service-specific secret path despite correct policy | Block 32-05 |
