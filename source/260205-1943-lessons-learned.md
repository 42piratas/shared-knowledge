# Lessons Learned

**Date:** 2026-02-05
**Project:** alfred-01
**Agent/Model:** Claude Sonnet 4
**Session Focus:** Fixing CI/CD pipeline Tailscale connectivity issues

---

## Lessons Learned

### Lesson: Tailscale ACL Grants vs SSH Rules

**Category:** infra

**Topic:** tailscale

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
- Other relevant context: GitHub Actions CI/CD environment

**Tags:** `tailscale` `acl` `cicd` `github-actions` `networking`

---

### Lesson: Tailscale Tag Separation for CI/CD

**Category:** infra

**Topic:** tailscale

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

| Key Purpose | Tag | Used By |
|:------------|:----|:--------|
| Server authentication | `tag:server` | Droplet/VM (in Terraform/cloud-init) |
| CI/CD runner | `tag:ci` | GitHub Actions (in secrets) |

Then configure ACLs to allow `tag:ci` → `tag:server` access.

**Prevention:**
- Always use separate tags for different roles (servers vs clients vs CI runners)
- Document which auth key goes where and what tag it uses
- Never combine multiple role tags on a single auth key

**Context:**
- Tool/Version: Tailscale Admin Console, Auth Keys feature
- OS: N/A (Tailscale SaaS)
- Other relevant context: Auth keys expire after 90 days maximum

**Tags:** `tailscale` `auth-keys` `cicd` `security` `best-practices`

---

### Lesson: Debugging Tailscale CI/CD Connectivity

**Category:** infra

**Topic:** tailscale

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
- Other relevant context: SSH uses Tailscale IP, not public IP

**Tags:** `tailscale` `cicd` `debugging` `github-actions` `troubleshooting`

---

## Session Notes

- The tailscale/github-action warns that `authkey` input is deprecated in favor of OAuth clients, but OAuth clients are not available on all Tailscale plans. Auth keys still work.
- Tailscale ephemeral nodes (CI runners) are automatically removed when they disconnect, keeping the machines list clean.
- When Tailscale IP changes (e.g., after server rebuild), both `DO_DROPLET_TAILSCALE_IP` and `SSH_KNOWN_HOSTS_TAILSCALE` GitHub secrets must be updated.

---

## Checklist Before Submission

- [x] All lessons are generalizable (not project-specific)
- [x] No sensitive data (API keys, internal URLs, passwords)
- [x] Error messages are included where applicable
- [x] Solutions are complete and actionable
- [x] Categories and topics are correctly assigned
- [x] File saved as `YYMMDD-HHMM-lessons-learned.md`
