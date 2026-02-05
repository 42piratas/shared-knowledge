# Lessons Learned: Supabase DNS Failure After Project Unpause

**Date:** 2026-02-05
**Topic:** Infrastructure / Database Connectivity
**Status:** Critical / Unresolved (Requires Support)

## Problem
The application failed to connect to the Supabase database after the project was "Unpaused" following a period of inactivity. The error manifested initially as `getaddrinfo ENOTFOUND` (DNS failure) and later as `ETIMEDOUT` (Network blockage) when attempting workarounds.

## Root Cause
1.  **Broken DNS Record:** The DNS record for the database connection pooler (`db.[project-ref].supabase.co`) was not recreated by Supabase's infrastructure when the project was unpaused. This was linked to a known incident on Feb 4, 2026.
2.  **IPv4 vs IPv6 Incompatibility:** Vercel serverless functions run on IPv4. Supabase's Direct Connection is IPv6-only. The standard workaround (Connection Pooler) relies on the DNS record that was missing.
3.  **Cloudflare Blocking:** Attempting to use the "Project URL" (`[project-ref].supabase.co`) failed because it resolves to Cloudflare, which proxies HTTP (80/443) but blocks the database port (6543/5432).

## Failed Workarounds
We attempted to bypass the broken DNS by connecting directly to the AWS regional endpoint (`aws-0-us-east-2.pooler.supabase.com`). While this established a network connection, authentication consistently failed with `"Tenant or user not found"`. This indicates that the internal routing for the project's pooler is tightly coupled to the DNS alias or resides in a different cluster than the one implied by the region "East US (Ohio)".

## Prevention & Detection
1.  **Verify DNS First:** Before debugging code, verify the database hostname resolves using `dig [hostname]`. If it returns `ANSWER: 0`, no code change will fix it.
2.  **Restart != Reset:** Using "Restart Project" in Supabase reboots the database daemon but does **not** re-provision the infrastructure/DNS records.
3.  **Dashboard Trust:** Do not assume the connection string shown in the dashboard works if the project was recently unpaused. Validate it externally.

## Solution
This specific failure mode (Missing DNS after Unpause) cannot be fixed by the user.
**Action:** Contact Supabase Support to manually restore the DNS records for the project.
