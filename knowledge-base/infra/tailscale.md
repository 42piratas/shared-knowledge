# Tailscale

Lessons learned for Tailscale networking and configuration.

---

## Auth Keys Must Have Matching Tags

**Problem:**
Tailscale authentication via auth key failed silently during cloud-init. The droplet never appeared in the Tailscale admin panel despite the auth key being valid.

```
+ tailscale up --ssh --advertise-tags=tag:server --authkey=tskey-auth-...
+ tailscale ip -4
no current Tailscale IPs; state: NeedsLogin
```

**Root Cause:**
The auth key was created with `tag:ci` permission only, but the `tailscale up` command used `--advertise-tags=tag:server`. Auth keys can only create machines with tags they are explicitly authorized for.

**Solution:**
Create a new auth key with the correct tag permissions:
1. Go to Tailscale Admin → Settings → Keys
2. Generate new key with **Tags: `tag:server`** (or both `tag:ci` and `tag:server` if needed)
3. Ensure "Reusable" is enabled for infrastructure-as-code use cases

```bash
# The --advertise-tags must match a tag the auth key is authorized for
tailscale up --ssh --advertise-tags=tag:server --authkey=${TAILSCALE_AUTH_KEY}
```

**Prevention:**
- Always verify auth key tags match the `--advertise-tags` value in scripts
- Document which tags each auth key is authorized for
- Test auth key permissions before using in automated deployments

**Context:**
- Tool/Version: Tailscale 1.94.1
- OS: Ubuntu 22.04
- Note: The `state: NeedsLogin` error is misleading — it suggests the key is invalid, but the actual issue may be tag permission mismatch

**Tags:** `tailscale` `auth-key` `tags` `cloud-init`

---
