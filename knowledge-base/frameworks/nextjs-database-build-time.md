# Next.js Static Generation with Direct Database Access

**Category:** Frameworks
**Topic:** Next.js App Router & Database Connections
**Date:** 2026-02-21

## Context

In Next.js App Router (versions 13+), pages are statically generated (SSG) by default unless they use dynamic functions (like `cookies()`, `headers()`) or are explicitly marked as dynamic.

When a server component fetches data directly from a database (e.g., using `pg`, `mysql2`, or Prisma) instead of via an HTTP `fetch()` call, Next.js attempts to execute this code during the build process (`next build`) to generate the initial HTML.

## The Problem

If your application connects directly to a database in a Server Component:

1.  **Build Failures:** The build process will fail if the database is not accessible from the build environment (common in Docker multi-stage builds or CI/CD pipelines where the database service isn't running).
2.  **Stale/Empty Data:** If you wrap the database call in a `try/catch` block to prevent build failures, Next.js may successfully build the page with the "fallback" data (e.g., `null` or empty arrays). This page is then **cached as static HTML**. At runtime, users will see this empty state indefinitely, and the server will **not** attempt to fetch fresh data, because the page is considered static.

### Example Scenario

```typescript
// app/dashboard/page.tsx
import { db } from "@/lib/db";

export default async function Dashboard() {
  // Next.js runs this at build time!
  // If DB is unreachable, it might return [] if error handling suppresses the error.
  const data = await db.query("SELECT * FROM stats");

  return <div>Stats: {data.length}</div>;
}
```

If the build environment cannot reach the DB, `data` might be empty. The deployed page will permanently show "Stats: 0" regardless of the actual database state.

## Solutions

### 1. Force Dynamic Rendering (Recommended for Dashboards)

If the page relies on real-time data that changes frequently or depends on a database not available at build time, force dynamic rendering. This opts out of Static Site Generation (SSG) and ensures the code runs on the server for every request.

```typescript
// Add this line to the page file
export const dynamic = "force-dynamic";

export default async function Page() { ... }
```

### 2. Revalidate (ISR)

If you want the performance benefits of static generation but need updates, use Incremental Static Regeneration. However, this still requires the database to be reachable during the initial build (or the first request if `fallback` strategies are used), or the initial build will freeze the "error" state for `revalidate` seconds.

```typescript
export const revalidate = 60; // Revalidate every 60 seconds
```

### 3. Build-Time Environment Configuration

If you *must* have static generation, ensure the build environment has access to the database (e.g., using docker-compose for the build step), but this complicates CI/CD pipelines significantly.

## Key Takeaway

Always explicitly define the rendering strategy (`dynamic = "force-dynamic"`) for Server Components that connect directly to external infrastructure (databases, caches, queues) that may not be present during the Docker build phase.
