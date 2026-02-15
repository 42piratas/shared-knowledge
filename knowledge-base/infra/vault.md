# HashiCorp Vault

## Lessons Learned

### Vault Listening Port (8282)

**Critical:** The Vault instance on `shroom-x` listens on **port 8282**, not the default 8200.

- **Symptoms:**
  - `vault status` returns `dial tcp 127.0.0.1:8200: connect: connection refused`
  - GitHub Actions fail with `connection refused` or timeout
- **Correction:**
  - Always set `export VAULT_ADDR='http://127.0.0.1:8282'` before running CLI commands
  - Check `/etc/vault.d/vault.hcl` for authoritative configuration
  - Use `netstat -tlnp | grep vault` to confirm listening port

### Unsealing Procedure

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

### Common Issues

- **CI/CD Failure:** If deployments fail with "Vault is sealed" (Error 503), the server likely restarted. Unseal manually.
- **Connection Refused:** Check if `VAULT_ADDR` is set to port 8282.
