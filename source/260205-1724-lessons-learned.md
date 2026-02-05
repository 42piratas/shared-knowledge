# Lessons Learned: Tailscale Auth & UFW Lockout

## Session Metadata

- **Date:** 2026-02-05
- **Project:** alfred-01
- **Agent/Model:** Claude Opus 4.5
- **Session Focus:** Infrastructure rebuild after complete droplet lockout

---

## Lessons Learned

### Lesson: Tailscale Auth Keys Must Have Matching Tags

**Category:** infra

**Topic:** tailscale

**Problem:**
Tailscale authentication via auth key failed silently during cloud-init. The droplet never appeared in the Tailscale admin panel despite the auth key being valid.

```
+ tailscale up --ssh --advertise-tags=tag:server --authkey=tskey-auth-kwMxtvV3RJ11CNTRL-...
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
- Other relevant context: Using auth key in Terraform cloud-init user_data

**Tags:** `tailscale` `auth-key` `tags` `cloud-init`

---

### Lesson: Never Have Single-Point-of-Failure SSH Access

**Category:** infra

**Topic:** digitalocean

**Problem:**
Complete droplet lockout with no recovery path. SSH timed out on both public IP and Tailscale IP. DigitalOcean Droplet Console also timed out.

```
ssh: connect to host 100.69.22.51 port 22: Operation timed out
ssh: connect to host 198.211.101.121 port 22: Operation timed out
```

**Root Cause:**
UFW was configured to only allow SSH via the `tailscale0` interface:
```bash
ufw allow in on tailscale0 to any port 22 proto tcp
```

When Tailscale service failed to start (due to auth key issue), there was no backup SSH access method. The DigitalOcean Droplet Console uses network SSH, so it was also blocked.

**Solution:**
Configure UFW with emergency access paths that don't depend on Tailscale:

```bash
# Primary access (Tailscale)
ufw allow in on tailscale0 to any port 22 proto tcp comment 'SSH via Tailscale - PRIMARY'

# Emergency access via DigitalOcean private network
ufw allow from 10.0.0.0/8 to any port 22 proto tcp comment 'SSH via DO private network - EMERGENCY'

# Emergency access via DigitalOcean internal interface
ufw allow in on eth1 to any port 22 proto tcp comment 'SSH via DO internal interface - EMERGENCY'
```

**Prevention:**
- Always maintain at least two independent SSH access methods
- Test emergency access paths after configuration
- Use DigitalOcean Recovery Console (boots from ISO) as last resort, not Droplet Console
- Document emergency access procedures in infrastructure playbook

**Context:**
- Tool/Version: UFW on Ubuntu 22.04, DigitalOcean Droplet
- OS: Ubuntu 22.04
- Other relevant context: DigitalOcean Droplet Console requires network SSH access; Recovery Console uses VNC and boots from ISO

**Tags:** `digitalocean` `ufw` `ssh` `lockout` `emergency-access`

---

### Lesson: DigitalOcean Droplet Console vs Recovery Console

**Category:** infra

**Topic:** digitalocean

**Problem:**
Attempted to use DigitalOcean "Console" to access locked droplet, but it timed out. Assumed this was a direct console access but it wasn't working.

**Root Cause:**
There are two different console types in DigitalOcean:

1. **Droplet Console** - Uses network SSH. Subject to firewall rules. Will timeout if SSH is blocked.
2. **Recovery Console** - VNC-based, boots from Recovery ISO. Works even if network/SSH is completely broken.

The "Console" button in the droplet dashboard is the Droplet Console, not the Recovery Console.

**Solution:**
To access a completely locked droplet:
1. Go to Droplet → **Recovery** tab (not Access)
2. Click **Boot from Recovery ISO**
3. Power cycle the droplet
4. Click **Recovery Console**
5. Login with credentials shown on screen
6. Mount filesystem: `mount /dev/vda1 /mnt`
7. Inspect logs: `cat /mnt/var/log/cloud-init-output.log`

**Prevention:**
- Document the difference between Droplet Console and Recovery Console
- Use Recovery Console for network/SSH issues
- Use Droplet Console only when network access is working

**Context:**
- Tool/Version: DigitalOcean Dashboard
- OS: N/A (cloud provider feature)
- Other relevant context: Recovery Console requires power cycle after enabling Recovery ISO boot

**Tags:** `digitalocean` `console` `recovery` `lockout`

---

### Lesson: UFW Changes Must Be in IaC

**Category:** infra

**Topic:** terraform

**Problem:**
Manual UFW changes made via SSH caused configuration drift. When droplet was rebuilt, the manual changes were lost, but the expectation remained that Tailscale-only SSH would work.

**Root Cause:**
During Phase 3 security hardening, UFW rules were modified manually:
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
- Other relevant context: IaC-first mandate in project meta-instructions

**Tags:** `terraform` `ufw` `iac` `configuration-drift`

---

## Session Notes

- Tailscale's `state: NeedsLogin` error is misleading - it suggests the key is invalid, but the actual issue was tag permission mismatch
- DigitalOcean's dashboard naming is confusing - "Console" vs "Recovery Console" should be clearer
- The combination of Tailscale auth failure + UFW lockdown created a complete lockout with no obvious recovery path
- Recovery Console is the only reliable way to access a completely locked DigitalOcean droplet

---

## Checklist Before Submission

- [x] All lessons are generalizable (not project-specific)
- [x] No sensitive data (API keys, internal URLs, passwords)
- [x] Error messages are included where applicable
- [x] Solutions are complete and actionable
- [x] Categories and topics are correctly assigned
- [x] File saved as `260205-1724-lessons-learned.md`
