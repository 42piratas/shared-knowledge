# DigitalOcean

Lessons learned for DigitalOcean cloud infrastructure.

---

## External API Geo-Blocking Requires Region Testing

**Problem:**
Service works locally but fails in production with 451 error.

```
ExchangeNotAvailable: binance GET 451 - Service unavailable from a restricted location
```

**Root Cause:**
External APIs (Binance, others) may geo-block certain regions. DigitalOcean NYC3 region is blocked by Binance due to US regulations.

**Solution:**

1. Test API connectivity from target region BEFORE deployment
2. Migrate to unblocked region (e.g., Amsterdam `ams3`)
3. Document region requirements in playbook

```bash
# Test from target region
ssh user@server "curl -I https://api.binance.com/api/v3/ping"
# Expected: HTTP/1.1 200 OK
# Blocked: HTTP/1.1 451 Unavailable For Legal Reasons
```

**Prevention:**

- Research API geo-restrictions before choosing cloud region
- Test API access as part of infrastructure provisioning
- Maintain list of region requirements per external service
- Have migration procedure ready for region changes

**Context:**

- Tool/Version: Binance API, DigitalOcean
- Note: US-based regions commonly blocked by crypto exchanges

**Tags:** `digitalocean` `api` `geo-blocking` `region`

---

## Never Have Single-Point-of-Failure SSH Access

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
- Note: DigitalOcean Droplet Console requires network SSH access; Recovery Console uses VNC and boots from ISO

**Tags:** `digitalocean` `ufw` `ssh` `lockout` `emergency-access`

---

## Droplet Console vs Recovery Console

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
- Note: Recovery Console requires power cycle after enabling Recovery ISO boot

**Tags:** `digitalocean` `console` `recovery` `lockout`

---
