# HashiCorp Vault

**Category:** infra
**Last Updated:** 2026-02-20
**Entries:** 3

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

Vault seals automatically on restart (e.g., DigitalOcean maintenance).

**Steps:**

1. SSH to `shroom-x.42bros.xyz`
2. Set correct address: `export VAULT_ADDR='http://127.0.0.1:8282'`
3. Check status: `vault status`
4. Unseal with 3 keys:
   ```bash
   vault operator unseal <key1>
   vault operator unseal <key2>
   vault operator unseal <key3>
   ```
5. Verify `Sealed: false` in output

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

## Common Issues

- **CI/CD Failure — Vault sealed (503):** The server restarted. Unseal manually (see Entry 2).
- **CI/CD Failure — Connection refused:** Check `VAULT_ADDR` is set to port 8282.
- **CI/CD Failure — i/o timeout:** Vault port is VPC-restricted. See Entry 3.

---

## Changelog

| Date       | Change                                                       | Source              |
| ---------- | ------------------------------------------------------------ | ------------------- |
| 2026-02-20 | Reformatted, added Entry 3 (VPC firewall / server-side auth) | Session 260220-1930 |
