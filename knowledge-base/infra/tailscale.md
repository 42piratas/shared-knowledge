# Tailscale

**Category:** infra
**Last Updated:** 2026-02-22
**Entries:** 11

---

## Overview

Lessons learned for Tailscale VPN implementation — focusing on authentication, ACL configuration, CI/CD connectivity, cloud-init deployment, and hostname resolution.

---

## Entries

### Entry 1: Auth Keys Must Have Matching Tags {#entry-1}

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

### Entry 2: ACL Grants vs SSH Rules {#entry-2}

**Problem:**
GitHub Actions CI/CD pipeline failing with SSH connection timeout when trying to connect to a Tailscale-networked server, despite having valid auth keys and SSH rules configured.

```
ssh: connect to host 100.x.x.x port 22: Connection timed out
Error: Process completed with exit code 255.
```

The `tailscale status` output from the CI runner showed the target server was not visible, even though SSH ACL rules allowed access.

**Root Cause:**
Tailscale ACLs have two distinct permission systems:

1. **SSH rules** - Control who can SSH to whom
2. **Grants rules** - Control network visibility (who can see whom on the network)

The ACL had SSH rules allowing `tag:ci` to SSH to `tag:server`, but the `grants` section only allowed `autogroup:admin` to see all machines. Without a grant rule, `tag:ci` machines could not even see `tag:server` machines on the network, causing connection timeouts before SSH was even attempted.

**Solution:**
Add a network visibility grant in the ACL for CI/CD access:

```json
{
  "grants": [
    {
      "src": ["autogroup:admin"],
      "dst": ["*"],
      "ip": ["*:*"]
    },
    {
      "src": ["tag:ci"],
      "dst": ["tag:server"],
      "ip": ["*:*"]
    }
  ],
  "ssh": [
    {
      "action": "accept",
      "src": ["tag:ci"],
      "dst": ["tag:server"],
      "users": ["root"]
    }
  ]
}
```

**Prevention:**

- Always configure both `grants` AND `ssh` rules when setting up Tailscale ACLs for CI/CD
- Verify network visibility by checking `tailscale status` output from the connecting machine
- If target machine is not visible in status output, it's a grants issue, not an SSH issue

**Context:**

- Tool/Version: Tailscale 1.94.1, tailscale/github-action@v2
- OS: Linux (Ubuntu GitHub runner, Debian droplet)
- Note: GitHub Actions CI/CD environment

**Tags:** `tailscale` `acl` `cicd` `github-actions` `networking`

---

### Entry 3: Tag Separation for CI/CD {#entry-3}

**Problem:**
Confusion about how many Tailscale auth keys are needed and what tags each should have for a CI/CD setup connecting to a server.

**Root Cause:**
Tailscale tags represent roles/purposes, not just labels. In a CI/CD scenario:

- The **server** receives connections and should be tagged accordingly
- The **CI runner** initiates connections and needs a different tag
- ACLs use these tags to control access patterns

Using the same tag for both, or putting both tags on one key, breaks the security model and can cause ACL matching failures.

**Solution:**
Create two separate auth keys with distinct tags:

| Key Purpose           | Tag          | Used By                              |
| :-------------------- | :----------- | :----------------------------------- |
| Server authentication | `tag:server` | Droplet/VM (in Terraform/cloud-init) |
| CI/CD runner          | `tag:ci`     | GitHub Actions (in secrets)          |

Then configure ACLs to allow `tag:ci` → `tag:server` access.

**Prevention:**

- Always use separate tags for different roles (servers vs clients vs CI runners)
- Document which auth key goes where and what tag it uses
- Never combine multiple role tags on a single auth key

**Context:**

- Tool/Version: Tailscale Admin Console, Auth Keys feature
- Note: Auth keys expire after 90 days maximum

**Tags:** `tailscale` `auth-keys` `cicd` `security` `best-practices`

---

### Entry 4: Debugging CI/CD Connectivity {#entry-4}

**Problem:**
Difficulty diagnosing why CI/CD pipeline cannot connect to Tailscale-networked server. Multiple potential failure points: auth key, tags, ACLs, SSH keys, network.

**Root Cause:**
Tailscale connectivity issues in CI/CD have a specific diagnostic flow that wasn't followed systematically, leading to multiple failed fix attempts.

**Solution:**
Follow this diagnostic checklist in order:

1. **Check CI workflow logs for "Tailscale status" output**
   - Does it show the target server? → If NO, it's an ACL grants issue
   - Does it show the CI runner itself? → If NO, auth key is invalid

2. **Verify auth key in Tailscale Admin Console**
   - Is the key valid (not expired)?
   - Does it have the correct tag (`tag:ci` for CI runners)?

3. **Verify target server in Tailscale Admin Console**
   - Is it connected (green dot)?
   - Does it have the correct tag (`tag:server`)?

4. **Check ACL configuration**
   - Is there a `grants` rule for `tag:ci` → `tag:server`?
   - Is there an `ssh` rule for `tag:ci` → `tag:server`?

5. **Check SSH configuration**
   - Is `SSH_KNOWN_HOSTS` secret correct for current server IP?
   - Is `SSH_PRIVATE_KEY` matching server's authorized_keys?

**Prevention:**

- Add verbose logging in CI workflows to output `tailscale status`
- Document expected Tailscale status output for healthy state
- Create runbook for CI/CD Tailscale troubleshooting

**Context:**

- Tool/Version: GitHub Actions, tailscale/github-action@v2
- OS: Ubuntu (GitHub runner)
- Note: SSH uses Tailscale IP, not public IP. When Tailscale IP changes (e.g., after server rebuild), both IP and known_hosts secrets must be updated.

**Tags:** `tailscale` `cicd` `debugging` `github-actions` `troubleshooting`

---

### Entry 5: Race Condition in Cloud-Init with UFW {#entry-5}

**Problem:**
Complete SSH lockout after `terraform apply` when Tailscale and UFW are configured in the same cloud-init script. SSH connections timeout, Tailscale admin shows machine as connected but SSH fails, DigitalOcean Console becomes unresponsive.

**Root Cause:**
`tailscale up` is asynchronous — it starts authentication but may not complete immediately. The cloud-init script configures UFW rules for `tailscale0` interface BEFORE Tailscale authentication completes:

```bash
# Problematic order:
tailscale up --ssh --advertise-tags=tag:server --authkey=${var.tailscale_auth_key}
# ... immediately after ...
ufw allow in on tailscale0 to any port 22
ufw --force enable
```

The `tailscale0` interface may not exist when UFW rules are applied. UFW enables and blocks all SSH before Tailscale is ready, resulting in complete lockout with no fallback access.

**Solution:**
Do NOT configure Tailscale and UFW together in cloud-init. Implement in phases:

1. **Phase 1:** Provision droplet with public SSH only (secured by SSH key)
2. **Phase 2:** Install and verify Tailscale works (manually or via configuration management)
3. **Phase 3:** Add UFW rules incrementally, testing after each step
4. **Phase 4:** Keep at least TWO independent SSH access methods active

```bash
# In cloud-init: Install Tailscale but don't start it
curl -fsSL https://tailscale.com/install.sh | sh

# DO NOT run tailscale up in cloud-init
# DO NOT configure UFW in cloud-init

# Let droplet boot successfully first
# Configure Tailscale and UFW post-deployment
```

**Prevention:**

- Never enable UFW in cloud-init (race conditions with network services)
- Add security layers incrementally AFTER basic connectivity is verified
- Always maintain at least TWO independent SSH access methods
- Test security changes on non-production environment first
- Use DigitalOcean Recovery Console (VNC) as emergency fallback — not Droplet Console (which uses network SSH)
- Wait for Tailscale authentication to complete before configuring firewall rules

**Context:**

- Versions affected: Tailscale (any), UFW (any)
- OS: Ubuntu 22.04 LTS (DigitalOcean)
- First documented: 2026-02-07
- Source: `260207-1800-lessons-learned-ssh-tailscale.md`

**Tags:** `tailscale` `ufw` `cloud-init` `race-condition` `ssh-lockout` `terraform`

---

### Entry 6: Auth Key Expiry and Lifecycle Management {#entry-6}

**Problem:**
Tailscale authentication fails silently during provisioning, leading to incomplete setup and connection failures. Multiple failure modes: key expired, single-use key already consumed, or key not tagged correctly.

**Root Cause:**
Auth keys have several failure modes that all present as silent authentication failure:

1. **Expiry:** Keys expire after a configured period (1-90 days)
2. **Single-Use:** Non-reusable keys are consumed by the first machine that uses them — subsequent provisioning attempts fail
3. **ACL Permissions:** Key not tagged correctly or ACLs don't permit the tagged device (see Entry 1)

**Solution:**
Use reusable, long-lived auth keys with proper tagging:

1. Generate auth key with:
   - **Reusable:** YES
   - **Expiry:** 90 days (maximum)
   - **Tags:** matching the `--advertise-tags` value (e.g., `tag:server`)
   - **Pre-approved:** YES (skip device approval step)

2. Set expiry reminders at 80 days (before 90-day expiry)

3. Test auth key before using in automation:

```bash
# Test on temporary machine
tailscale up --authkey=${TEST_KEY} --advertise-tags=tag:server
tailscale status  # Verify connection
```

4. Clean up stale Tailscale machines from admin console after failed provisioning attempts

**Prevention:**

- Use reusable auth keys for automation (single-use keys fail on retry)
- Set 90-day expiry to minimize rotation frequency
- Monitor expiry dates proactively (80-day reminder)
- Store auth keys in secure secrets management (GitHub Secrets, HashiCorp Vault)
- Test auth keys in non-production before using in production IaC
- Clean up old Tailscale machines from admin console (stale entries from failed provisioning)

**Context:**

- Versions affected: Tailscale (all)
- OS: all
- First documented: 2026-02-07
- Source: `260207-1800-lessons-learned-ssh-tailscale.md`

**Tags:** `tailscale` `auth-keys` `expiry` `lifecycle` `automation`

---

### Entry 7: Hostname Resolution Failures on macOS {#entry-7}

**Problem:**
SSH connections using Tailscale hostname (e.g., `ssh root@batcave-a-1`) fail or timeout on macOS, even though Tailscale shows device as connected. Tailscale IP works but hostname doesn't resolve.

**Root Cause:**
macOS DNS resolution may not properly query Tailscale's MagicDNS for `.ts.net` domains. Additionally, there can be a mismatch between the machine name in Tailscale admin and the hostname used for SSH:

- **Droplet hostname:** Set in Terraform/cloud-init (e.g., `batcave-a`)
- **Tailscale hostname:** Advertised to Tailscale network (e.g., `batcave-a-1`)

These can differ, causing confusion.

**Solution:**
Use Tailscale IP instead of hostname for reliability:

```bash
# Get Tailscale IP from admin console or:
tailscale status | grep batcave

# Use IP for SSH:
ssh root@100.x.x.x
```

Update SSH config with the Tailscale IP:

```bash
# ~/.ssh/config
Host batcave-a
    HostName 100.x.x.x  # Tailscale IP
    User root
    IdentityFile ~/.ssh/key
```

If hostname is required:

1. Verify Tailscale hostname in admin console matches what you're using
2. Ensure MagicDNS is enabled in Tailscale admin
3. Check macOS DNS settings
4. Restart Tailscale client: `sudo tailscale down && sudo tailscale up`

**Prevention:**

- Document the Tailscale IP in infrastructure playbook for quick reference
- Use SSH config with IP instead of relying on hostname resolution
- Prefer Tailscale IP over hostname for automation (CI/CD, scripts)
- Keep Tailscale machine name and droplet hostname synchronized to avoid confusion
- Test hostname resolution after provisioning before relying on it

**Context:**

- Versions affected: Tailscale (any), macOS (any)
- OS: macOS (client-side)
- First documented: 2026-02-07
- Source: `260207-1800-lessons-learned-ssh-tailscale.md`

**Tags:** `tailscale` `macos` `dns` `hostname` `magicdns`

---

### Entry 8: Tailscale Serve — Expose Local Services to Tailnet Only {#entry-8}

**Problem:**
Need to access a local service (e.g., an AI agent gateway on `127.0.0.1:18789`) from other devices without exposing it to the public internet.

**Root Cause:**
Opening ports in the firewall exposes services to the entire internet. Traditional VPNs add complexity. What's needed is identity-based access restricted to trusted devices only.

**Solution:**
Use Tailscale Serve to proxy a local port over the tailnet with automatic HTTPS:

```bash
# Expose local port 18789 to your tailnet
tailscale serve 18789

# Output:
# Available within your tailnet:
# https://your-machine.tailnet-name.ts.net
# |-- / proxy http://127.0.0.1:18789
```

Now any device on your tailnet can access the service at `https://your-machine.tailnet-name.ts.net` — with automatic HTTPS certificates, no port forwarding, and no public exposure.

**For applications with native Tailscale support** (e.g., OpenClaw), configure in the app's config:

```json
{
  "gateway": {
    "bind": "loopback",
    "tailscale": {
      "mode": "serve"
    }
  }
}
```

- `"serve"` — tailnet-only HTTPS (recommended for private use)
- `"funnel"` — public HTTPS (see Entry 9)
- `"off"` — no Tailscale automation (default)

**Important:** `gateway.bind` must stay `"loopback"` when Serve/Funnel is enabled.

**Check status and stop:**

```bash
tailscale serve status    # What's being served
tailscale serve off       # Stop serving
```

**Prevention:**

- Always prefer Tailscale Serve over opening firewall ports for internal services
- Use `loopback` binding when Serve is enabled — the service should not listen on public interfaces
- Verify access works from another tailnet device before removing alternative access methods

**Context:**

- Tool/Version: Tailscale 1.50+
- Docs: https://tailscale.com/kb/1312/serve

**Tags:** `tailscale` `serve` `private-access` `https` `reverse-proxy`

---

### Entry 9: Tailscale Funnel — Public Access When Needed {#entry-9}

**Problem:**
Need to expose a local service to the **entire internet** (e.g., for webhook endpoints like Gmail Pub/Sub, GitHub webhooks, Sentry) without revealing the server's real IP.

**Root Cause:**
Some external services require a publicly accessible HTTPS URL for push notifications or webhooks. Traditional solutions (public IP + HTTPS cert + firewall rules) add complexity and expose infrastructure.

**Solution:**
Use Tailscale Funnel to expose a local port publicly through a Tailscale relay URL:

```bash
# Expose local port 8788 publicly
tailscale funnel 8788

# Output:
# Available on the internet:
# https://your-machine.tailnet-name.ts.net
# |-- / proxy http://127.0.0.1:8788
```

Traffic is routed through Tailscale's relay servers, hiding the server's real IP.

**Limitations:**

- Funnel only works on ports 443, 8443, and 10000
- Bandwidth is limited (relay-based, not direct peer-to-peer)
- Requires `funnel` node attribute in your tailnet policy file
- Not suitable for high-throughput or latency-sensitive services

**Prevention:**

- Use Funnel only for webhook endpoints, not primary application traffic
- Enable authentication/authorization on any Funneled service
- Prefer Tailscale Serve (Entry 8) for internal access — only use Funnel when public access is required
- Monitor Funnel usage — it's relay-based and adds latency

**Context:**

- Tool/Version: Tailscale 1.50+
- Docs: https://tailscale.com/kb/1223/funnel

**Tags:** `tailscale` `funnel` `public-access` `webhooks` `https`

---

### Entry 10: Locking Down SSH to Tailscale with UFW {#entry-10}

**Problem:**
Need to restrict SSH access to only Tailscale-connected devices while maintaining emergency fallback access and keeping public HTTPS for web services.

**Root Cause:**
Leaving SSH (port 22) open to the public internet invites brute-force bot attacks. Moving SSH behind Tailscale provides identity-based access control, but must be done carefully to avoid lockout.

**Solution:**
After verifying Tailscale SSH works, configure UFW incrementally:

```bash
# 1. FIRST verify Tailscale SSH works (connect via Tailscale IP!)
ssh root@100.x.y.z

# 2. Configure UFW rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0          # All traffic over Tailscale
sudo ufw allow 443/tcp                    # HTTPS for public web services
sudo ufw allow 80/tcp                     # HTTP for cert renewal / redirects
sudo ufw allow 41641/udp                  # Tailscale direct connections (improves perf)

# 3. Optional: emergency SSH from a specific IP
# sudo ufw allow from <your-static-home-ip> to any port 22 proto tcp comment 'Emergency SSH'

# 4. Remove public SSH
sudo ufw delete allow 22/tcp

# 5. Enable
sudo ufw enable
sudo ufw reload
sudo service ssh restart

# 6. Test: exit and reconnect
exit
ssh root@100.x.y.z          # Should work
ssh root@<public-ip>        # Should timeout
```

**Do NOT allow port 22 from anywhere** unless you have a specific emergency-access IP. `ufw allow in on tailscale0` handles all SSH access over Tailscale.

**Do NOT expose application ports** (e.g., 18789 for an AI gateway). Use Tailscale Serve (Entry 8) instead. For Docker inter-container communication, use the `trustedProxies` config:

```json
{
  "gateway": {
    "bind": "lan",
    "trustedProxies": ["172.28.0.0/16"]
  }
}
```

**Emergency fallback:** If Tailscale access is lost, use DigitalOcean's **Recovery Console** (Recovery tab → Boot from Recovery ISO → Power cycle) — not the Droplet Console (which uses network SSH and will be blocked).

**Prevention:**

- Always verify Tailscale SSH works BEFORE restricting public SSH
- Keep at least two independent access methods active until Tailscale is proven stable
- Test emergency fallback (Recovery Console) before you need it
- Never configure UFW in the same cloud-init script as Tailscale (see Entry 5)
- Document the UFW rules in your infrastructure playbook

**Context:**

- Tool/Version: UFW on Ubuntu 22.04+, Tailscale any version
- First documented: 2026-02-22
- Source: `alfred-01/docs/openclaw-lessons-learned.md`

**Tags:** `tailscale` `ufw` `ssh` `lockdown` `security` `firewall`

---

### Entry 11: Subnet Routers for Multi-Droplet Docker Access {#entry-11}

**Problem:**
Need devices on the tailnet to reach Docker containers on a remote Droplet's private network (e.g., `172.28.0.0/16`) without installing Tailscale inside every container.

**Root Cause:**
Docker containers use an internal bridge network that is not routable from outside the host. Tailscale connects devices at the host level, but the container network is invisible to other tailnet members.

**Solution:**
Configure the Droplet as a Tailscale subnet router to advertise its Docker network:

```bash
# 1. Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# 2. Advertise the Docker subnet
sudo tailscale set --advertise-routes=172.28.0.0/16

# 3. Approve the route in Tailscale Admin Console
#    Machines → your Droplet → "..." menu → Edit route settings → Approve
```

After approval, other tailnet devices can reach Docker containers on that Droplet's `172.28.0.0/16` network directly.

**Use case:** Multi-droplet architectures where a centralized service (e.g., a dashboard on Droplet A) needs to reach agent containers on Droplet B without public port exposure.

**Prevention:**

- Only advertise subnets you actually need exposed — don't advertise `0.0.0.0/0`
- Document which Droplet advertises which subnet
- Approve routes explicitly in the admin console — don't auto-approve
- Test connectivity from another tailnet device after approval

**Context:**

- Tool/Version: Tailscale any version, Linux with IP forwarding
- Docs: https://tailscale.com/kb/1019/subnets
- Note: Subnet routes must be approved in the Tailscale Admin Console

**Tags:** `tailscale` `subnet-router` `docker` `multi-droplet` `networking`

---

## Quick Reference: Tailscale Commands

```bash
# Status and connectivity
tailscale status                        # All devices, connection type (direct/relay)
tailscale ip -4                         # This device's Tailscale IPv4
tailscale ping <hostname>               # Test connectivity to a peer
tailscale version                       # Installed version

# Serve (private tailnet access)
tailscale serve 18789                   # Expose local port to tailnet
tailscale serve status                  # Check what's being served
tailscale serve off                     # Stop serving

# Funnel (public internet access)
tailscale funnel 8788                   # Expose local port publicly
tailscale funnel off                    # Stop funnel

# Administration
sudo tailscale down                     # Disconnect from tailnet
sudo tailscale up --ssh                 # Reconnect with SSH enabled
sudo tailscale set --advertise-routes=172.28.0.0/16   # Advertise subnet

# Troubleshooting
tailscale netcheck                      # Network connectivity check
tailscale bugreport                     # Generate debug report
journalctl -u tailscaled --since "1h ago"  # Service logs
```

---

## Cross-References

- [infra/digitalocean.md](infra/digitalocean.md) — SSH lockout and recovery console
- [infra/github-actions.md](infra/github-actions.md) — CI/CD pipeline with Tailscale connectivity
- [infra/terraform.md](infra/terraform.md) — IaC provisioning with Tailscale

---

## External Resources

- [Tailscale Official Docs](https://tailscale.com/kb/)
- [Tailscale ACL Documentation](https://tailscale.com/kb/1018/acls/)
- [Tailscale Auth Keys](https://tailscale.com/kb/1085/auth-keys/)

---

## Changelog

| Date       | Change                                                                              | Source                                                             |
| ---------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 2026-02-05 | Initial creation with 4 entries                                                     | `260205-1724-lessons-learned.md`, `260205-1943-lessons-learned.md` |
| 2026-02-18 | Added entries #5-7 (cloud-init race, auth key lifecycle, macOS hostname)            | `260207-1800-lessons-learned-ssh-tailscale.md`                     |
| 2026-02-22 | Added entries #8-11 (Serve, Funnel, UFW lockdown, subnet routers) + quick reference | `alfred-01/docs/openclaw-lessons-learned.md`                       |
