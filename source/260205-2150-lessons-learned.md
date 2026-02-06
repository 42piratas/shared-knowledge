# Lessons Learned Template

## Session Metadata

- **Date:** 2026-02-05
- **Project:** Ethereum World
- **Agent/Model:** Claude Opus 4
- **Session Focus:** Resolving Supabase database connectivity issues for Vercel serverless functions

---

## Lessons Learned

### Lesson: Supabase Connection Pooler Selection for IPv4 Environments

**Category:** databases

**Topic:** supabase

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

| Type | Host Pattern | Port | IPv4 | Best For |
|------|--------------|------|------|----------|
| Direct | `db.[ref].supabase.co` | 5432 | ❌ | VMs, long-lived servers |
| Dedicated Pooler | `db.[ref].supabase.co` | 6543 | ❌ | Paid tier, performance |
| Session Pooler | `[region].pooler.supabase.com` | 5432 | ✅ | Persistent + IPv4 |
| Transaction Pooler | `[region].pooler.supabase.com` | 6543 | ✅ | Serverless functions |

**Context:**
- Tool/Version: Supabase (managed PostgreSQL), Vercel Functions (Node.js serverless)
- OS: N/A (cloud services)
- Other relevant context: Free tier Supabase project, Vercel Pro plan

**Tags:** `supabase` `postgresql` `vercel` `serverless` `ipv4` `connection-pooler` `supavisor`

---

## Session Notes

- The "Tenant or user not found" error is specific to Supavisor (Supabase's connection pooler) and indicates the pooler cannot route the connection to the correct project
- This can happen when using the wrong hostname/username format, not just when credentials are incorrect
- Previous troubleshooting session incorrectly attributed the issue to DNS/infrastructure failure after a Supabase incident — always verify connection string format before assuming infrastructure problems

---

## Checklist Before Submission

- [x] All lessons are generalizable (not project-specific)
- [x] No sensitive data (API keys, internal URLs, passwords)
- [x] Error messages are included where applicable
- [x] Solutions are complete and actionable
- [x] Categories and topics are correctly assigned
- [x] File saved as `YYMMDD-HHMM-lessons-learned.md`
