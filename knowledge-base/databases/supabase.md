# Supabase

Lessons learned for Supabase database and backend services.

---

## DNS Failure After Project Unpause

**Problem:**
Application failed to connect to Supabase database after the project was "Unpaused" following a period of inactivity. The error manifested as `getaddrinfo ENOTFOUND` (DNS failure) and later as `ETIMEDOUT` when attempting workarounds.

**Root Cause:**
1. **Broken DNS Record:** The DNS record for the database connection pooler (`db.[project-ref].supabase.co`) was not recreated by Supabase's infrastructure when the project was unpaused.
2. **IPv4 vs IPv6 Incompatibility:** Vercel serverless functions run on IPv4. Supabase's Direct Connection is IPv6-only. The standard workaround (Connection Pooler) relies on the DNS record that was missing.
3. **Cloudflare Blocking:** Attempting to use the "Project URL" (`[project-ref].supabase.co`) failed because it resolves to Cloudflare, which proxies HTTP (80/443) but blocks database ports (6543/5432).

**Failed Workarounds:**
Attempting to bypass broken DNS by connecting directly to the AWS regional endpoint (`aws-0-us-east-2.pooler.supabase.com`) established a network connection, but authentication failed with `"Tenant or user not found"`. The internal routing for the project's pooler is tightly coupled to the DNS alias.

**Solution:**
This failure mode (missing DNS after Unpause) cannot be fixed by the user.

**Action:** Contact Supabase Support to manually restore the DNS records for the project.

**Prevention & Detection:**
- **Verify DNS First:** Before debugging code, verify the database hostname resolves using `dig [hostname]`. If it returns `ANSWER: 0`, no code change will fix it.
- **Restart != Reset:** Using "Restart Project" in Supabase reboots the database daemon but does **not** re-provision infrastructure/DNS records.
- **Dashboard Trust:** Do not assume the connection string shown in the dashboard works if the project was recently unpaused. Validate it externally.

**Context:**
- Tool/Version: Supabase (managed service), Vercel serverless
- Note: May be linked to known Supabase incidents; check status page

**Tags:** `supabase` `dns` `connection-pooler` `vercel` `unpause`

---
