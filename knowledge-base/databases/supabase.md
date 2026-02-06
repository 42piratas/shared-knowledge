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

## Supabase Connection Pooler Selection for IPv4 Environments

**Problem:**
Application deployed on Vercel Functions could not connect to Supabase database. The error returned was:

```
"Tenant or user not found"
```

This error occurred despite Supabase support confirming the database was fully functional. The connection string from the Supabase dashboard appeared valid.

**Root Cause:**
The Supabase dashboard was showing the **Dedicated Pooler** connection string by default:

```
postgres://postgres:[PASSWORD]@db.[PROJECT-REF].supabase.co:6543/postgres
```

This format uses:

- Hostname: `db.[ref].supabase.co` (Dedicated Pooler)
- Port: `6543` (transaction mode)

The Dedicated Pooler requires **IPv6 connectivity**. Vercel Functions (and many serverless platforms) are **IPv4-only**, causing the connection to fail at the routing/tenant identification layer.

**Solution:**
Use the **Session Pooler** (Shared Pooler) connection string instead, which supports IPv4:

```
postgresql://postgres.[PROJECT-REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:5432/postgres
```

Key differences:

- **Username:** `postgres.[PROJECT-REF]` (project reference embedded in username)
- **Hostname:** `aws-0-[REGION].pooler.supabase.com` (shared pooler infrastructure)
- **Port:** `5432` (session mode)

In the Supabase dashboard, click "Connect" and look for the "Session Pooler" option, which explicitly states "IPv4 compatible".

**Prevention:**

1. **Always verify IPv4 requirements** before selecting a connection method for serverless platforms
2. **Look for the IPv4 warning** in the Supabase Connect dialog - if it says "Not IPv4 compatible", you need a different connection type
3. **Document the connection type used** in environment variable comments or infrastructure docs
4. **Test connectivity immediately** after configuring connection strings, before assuming the database is broken

**Supabase Connection Types Reference:**

| Type               | Host Pattern                   | Port | IPv4 | Best For                |
| ------------------ | ------------------------------ | ---- | ---- | ----------------------- |
| Direct             | `db.[ref].supabase.co`         | 5432 | ❌   | VMs, long-lived servers |
| Dedicated Pooler   | `db.[ref].supabase.co`         | 6543 | ❌   | Paid tier, performance  |
| Session Pooler     | `[region].pooler.supabase.com` | 5432 | ✅   | Persistent + IPv4       |
| Transaction Pooler | `[region].pooler.supabase.com` | 6543 | ✅   | Serverless functions    |

**Context:**

- Tool/Version: Supabase (managed PostgreSQL), Vercel Functions (Node.js serverless)
- OS: N/A (cloud services)
- Other relevant context: Free tier Supabase project, Vercel Pro plan

**Tags:** `supabase` `postgresql` `vercel` `serverless` `ipv4` `connection-pooler` `supavisor`

---
