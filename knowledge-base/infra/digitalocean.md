# DigitalOcean

**Category:** infra
**Last Updated:** 2026-02-22
**Entries:** 8

---

## Overview

Lessons learned for DigitalOcean cloud infrastructure — covering networking restrictions, firewall layers, console access, resource management, and common deployment pitfalls.

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

## SMTP Is Permanently Blocked on All Droplets

**Problem:**
Email sending via SMTP silently fails from DigitalOcean Droplets. The agent or application may report success, but no email is ever delivered.

**Root Cause:**
DigitalOcean blocks **all** outbound SMTP traffic on ports 25, 465, and 587 across every Droplet. This is a platform-wide anti-spam policy. There is no way to unblock it — not even by contacting support. Reserved (formerly Floating) IPs and IPv6 addresses are also subject to the same block.

**Solution:**
Use an HTTP-based email API over HTTPS (port 443), which is never blocked:

- **Brevo** — `POST https://api.brevo.com/v3/smtp/email` (free: 300 emails/day)
- **SendGrid** — `POST https://api.sendgrid.com/v3/mail/send`
- **Postmark** — `POST https://api.postmarkapp.com/email`
- **Mailgun** — `POST https://api.mailgun.net/v3/{domain}/messages`
- **Amazon SES** — `POST https://email.{region}.amazonaws.com/`

**Prevention:**

- Never attempt SMTP-based email sending from cloud-hosted instances
- Always use HTTP-based email APIs for cloud deployments
- Test email delivery from the Droplet before assuming it works
- This applies to AWS Lightsail, GCP, and other cloud providers as well

**Context:**

- Affected: All DigitalOcean Droplets, all regions, all IP types
- DO docs: https://docs.digitalocean.com/support/why-is-smtp-blocked/

**Tags:** `digitalocean` `smtp` `email` `blocked-ports` `brevo` `sendgrid`

---

## Network Traffic Restrictions

**Problem:**
Certain network operations fail unexpectedly on DigitalOcean Droplets with no clear error message.

**Root Cause:**
DigitalOcean enforces several network restrictions beyond SMTP blocking:

| Restriction                | Details                                                    |
| :------------------------- | :--------------------------------------------------------- |
| SMTP (25, 465, 587)        | Blocked on all Droplets, all IPs, all protocols            |
| TCP/UDP port 11211 inbound | Blocked (Memcached amplification prevention)               |
| Multicast traffic          | Blocked                                                    |
| IP/MAC spoofing            | Traffic not matching Droplet's IP/MAC is dropped           |
| Network throughput         | 2 Gbps max (regular), 10 Gbps (premium CPU), 25 Gbps (GPU) |

**Solution:**

- For SMTP: use HTTP-based email APIs (see entry above)
- For IP/MAC restrictions: use userspace tunnels (e.g., Tailscale/WireGuard) instead of raw packet forwarding
- For throughput limits: plan capacity accordingly for high-traffic services

The IP/MAC restriction means you cannot use the Droplet as a traditional site-to-site VPN gateway or configure direct server return. Tailscale works fine because it uses a userspace WireGuard tunnel — it doesn't need raw packet forwarding.

**Prevention:**

- Review DO network restrictions before designing network architecture
- Test connectivity patterns from the Droplet, not just locally
- Use userspace networking tools for overlay networks

**Context:**

- DO docs: https://docs.digitalocean.com/products/droplets/details/limits/

**Tags:** `digitalocean` `networking` `restrictions` `traffic` `firewall`

---

## Cloud Firewalls vs. UFW — Use Both

**Problem:**
Confusion about which firewall layer to use on DigitalOcean Droplets, or relying on only one layer.

**Root Cause:**
DigitalOcean offers two distinct firewall layers that serve different purposes:

1. **Cloud Firewalls** — network-level, configured in the DO control panel, applied before traffic reaches the Droplet. Free. Stateful. Block everything not explicitly allowed.
2. **UFW (Uncomplicated Firewall)** — OS-level, configured on the Droplet itself via `ufw` commands. Runs inside the Droplet.

**Solution:**
Use both layers for defense in depth:

- **Cloud Firewalls** for coarse rules: allow 443 for HTTPS, allow 80 for HTTP/cert renewal, block everything else publicly
- **UFW on the Droplet** for fine-grained control: allow all traffic on `tailscale0` interface, restrict SSH to Tailscale only, per-port rules for internal services

**Prevention:**

- Never rely on a single firewall layer
- Never enable UFW in cloud-init / Terraform provisioning scripts alongside Tailscale (race condition — see `infra/tailscale.md` Entry #5)
- Test firewall rules from outside the network after configuration
- Always maintain at least two independent SSH access methods

**Context:**

- Cloud Firewalls: https://docs.digitalocean.com/products/networking/firewalls/

**Tags:** `digitalocean` `firewall` `ufw` `cloud-firewall` `security`

---

## Docker Memory Management on Small Droplets

**Problem:**
Docker containers are OOM-killed or exhibit degraded performance on small (2–4GB) DigitalOcean Droplets running multiple services.

**Root Cause:**
Small Droplets have limited RAM. Running multiple containers (e.g., an AI agent gateway + database + reverse proxy + monitoring) without memory limits leads to one container consuming all available memory, triggering the kernel OOM killer on random processes.

**Solution:**
Set explicit Docker memory limits in `docker-compose.yml`:

```yaml
deploy:
  resources:
    limits:
      memory: 2048M
    reservations:
      memory: 1024M
```

**Monitoring:**

```bash
# Real-time per-container memory usage
docker stats --no-stream

# Check for OOM kills
dmesg | grep -i "out of memory\|oom"

# Check total system memory
free -h
```

**Prevention:**

- Always set memory limits for every container in production
- Set reservations to guarantee minimum memory availability
- Monitor with `docker stats` and check `dmesg` for OOM events
- Plan capacity: sum all container reservations and ensure they fit within Droplet RAM minus ~512MB for OS overhead
- Disable memory-hungry optional features (e.g., browser tools in AI agents) on small Droplets

**Context:**

- Affected: Any Droplet size, but most critical on ≤4GB
- Note: Docker's default is unlimited memory per container

**Tags:** `digitalocean` `docker` `memory` `oom` `small-droplets` `resource-limits`

---

## Changelog

| Date       | Change                                                                                            | Source                                       |
| ---------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| 2026-02-05 | Initial creation (geo-blocking, SSH lockout, console types)                                       | Infrastructure lessons learned               |
| 2026-02-22 | Added SMTP blocked, network restrictions, Cloud Firewalls vs UFW, Docker memory on small Droplets | `alfred-01/docs/openclaw-lessons-learned.md` |

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
