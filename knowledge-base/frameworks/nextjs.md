# Next.js

**Category:** frameworks
**Last Updated:** 2026-02-22
**Entries:** 6

---

## Overview

Lessons learned for Next.js framework deployment, configuration, and tooling — focusing on production issues, reverse proxy integration, and build system changes.

---

## Quick Reference

| Task               | Command/Pattern                                     |
| ------------------ | --------------------------------------------------- |
| Dev with Turbopack | `next dev --turbopack`                              |
| Dev with Webpack   | `next dev` (remove `--turbopack` from package.json) |
| Standalone build   | Set `output: 'standalone'` in `next.config.js`      |

---

## Entries

### Entry 1: Next.js 16 Turbopack Default Conflicts with Custom Webpack Config {#entry-1}

**Problem:**
`npm run dev` fails locally with an error about custom webpack configuration being incompatible with Turbopack:

```
This build is using Turbopack, with a `webpack` config and no `turbopack` config.
```

**Root Cause:**
Next.js 16 enables Turbopack by default for development. If `next.config.js` contains custom webpack configuration (e.g., for standalone output, polyfills, or custom loaders), Turbopack does not support it and will error or warn.

**Solution:**
Either explicitly opt in to Turbopack (which acknowledges the webpack config is ignored for dev), or remove the flag to fall back to webpack:

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build"
  }
}
```

Note: The `--turbopack` flag applies only to `next dev`. Production builds (`next build`) always use webpack, so custom webpack config in `next.config.js` still applies at build time.

**Prevention:**

- Check Next.js upgrade guides for default behavior changes (like Turbopack becoming default)
- Separate build-time config (webpack) from dev-time config if possible
- Test both `dev` and `build` commands after upgrading Next.js

**Context:**

- Versions affected: Next.js 16+
- OS: all
- First documented: 2026-02-09
- Source: `260209-2230-lessons-learned-ui-deployment.md`

**Tags:** `nextjs` `turbopack` `webpack` `dev-server`

---

### Entry 2: Subpath Proxying Fails for Multiple Next.js Apps on Same Domain {#entry-2}

**Problem:**
When hosting two Next.js applications on the same domain using path-based routing (e.g., Dashboard at `/` and LiteLLM Admin UI at `/litellm/`), the subpath app fails to load — blank page with 404 errors and MIME type mismatches for all CSS and JavaScript assets.

```
GET /_next/static/chunks/main-abc123.js 404 (Not Found)
Refused to apply style... MIME type ('text/html') is not a supported stylesheet MIME type
```

**Root Cause:**
Three factors combine to make this unworkable:

1. **Asset path collision:** Both Next.js apps hardcode their static assets to `/_next/`. When sharing a domain, Nginx cannot distinguish which app an `/_next/...` request belongs to.

2. **Root-relative asset paths:** The subpath app generates HTML with root-relative URLs (e.g., `<script src="/_next/static/...">`). From the browser at `/litellm/`, these resolve to the domain root, hitting the other app's route handler.

3. **Framework limitations:** LiteLLM's `SERVER_ROOT_PATH` environment variable (designed to fix this) is known to be buggy in v1.81.0+ (GitHub issues #20011, #16934). Next.js `basePath` works but requires the app to be built with it — not possible for third-party pre-built apps.

Nginx `rewrite` rules cannot fix this because the asset paths are hardcoded in client-side JavaScript bundles.

**Solution:**
Use subdomains instead of subpaths. Each app gets its own domain root, eliminating all path conflicts:

```
alfred.42labs.io   → Dashboard (Next.js)    → /_next/ assets resolve correctly
llm.42labs.io      → LiteLLM (Next.js)      → /_next/ assets resolve correctly
```

Implementation:

1. **DNS:** Add A record for subdomain pointing to the same server IP
2. **Nginx:** Separate `server` blocks per subdomain:

```nginx
server {
    listen 443 ssl http2;
    server_name alfred.example.com;
    location / {
        proxy_pass http://dashboard:3000;
    }
}

server {
    listen 443 ssl http2;
    server_name llm.example.com;
    location = / {
        return 301 https://$server_name/ui/;
    }
    location / {
        proxy_pass http://litellm:4000;
    }
}
```

3. **SSL:** Expand certificate to SAN covering both domains:

```bash
certbot certonly --standalone --expand \
  -d alfred.example.com -d llm.example.com
```

**Prevention:**

| Scenario                 | Subpath                             | Subdomain      |
| ------------------------ | ----------------------------------- | -------------- |
| Different frameworks     | ✅ Usually OK                       | ✅ OK          |
| Same framework (Next.js) | ❌ High risk — asset path collision | ✅ Recommended |
| Third-party pre-built UI | ❌ Cannot control basePath          | ✅ Required    |

Guidelines:

- Never host multiple Next.js apps on the same domain via subpaths unless both are built with distinct `basePath` values
- Before relying on a framework's subpath/proxy feature (e.g., `SERVER_ROOT_PATH`), search its GitHub Issues for known bugs
- Subdomain strategy costs nothing — same IP, just an additional DNS record
- Use SAN certificates (`--expand` flag) for multi-domain SSL on a single server

**Context:**

- Versions affected: Next.js 16+, LiteLLM v1.81.0+
- OS: all (Nginx reverse proxy)
- First documented: 2026-02-11
- Source: `260211-0009-lessons-learned-litellm-subpath-proxy.md`, `260211-0027-lessons-learned-litellm-subpath-proxy.md`, `260211-1259-lessons-learned-litellm-subpath-routing.md`, `260211-1543-lessons-learned-litellm-subdomain-strategy.md`

**Tags:** `nextjs` `nginx` `subpath` `subdomain` `reverse-proxy` `litellm` `ssl`

### Entry 3: Recharts Horizontal Bar Chart Layout Confusion {#entry-3}

**Problem:**
A "Horizontal Bar Chart" (bars going left-to-right) renders with invisible or extremely thin bars, even though data is present. Tooltips show correct values but labels like "0 Score: 88%" (showing axis ticks instead of categories).

Code using the intuitive-but-wrong configuration:

```tsx
<BarChart layout="horizontal">
  <XAxis type="number" />
  <YAxis type="category" />
  ...
</BarChart>
```

**Root Cause:**
In Recharts, the `layout` prop defines the **direction of the dependent axis**, not the visual orientation of the bars in the way most expect.

- `layout="horizontal"` (default): Vertical bars. X-axis is the independent variable (category), Y-axis is the dependent variable (number).
- `layout="vertical"`: Horizontal bars. Y-axis is the independent variable (category), X-axis is the dependent variable (number).

Using `layout="horizontal"` with swapped axis types (`XAxis type="number"` + `YAxis type="category"`) causes the rendering engine to calculate bar dimensions incorrectly (often resulting in 0-height bars).

**Solution:**
For a horizontal bar chart (sideways bars):

1. Set `layout="vertical"` on the `BarChart` component.
2. Ensure `XAxis` is `type="number"` (the value scale).
3. Ensure `YAxis` is `type="category"` (the labels).

```tsx
// Correct Horizontal Bar Chart
<BarChart data={data} layout="vertical">
  <XAxis type="number" domain={[0, 100]} />
  <YAxis dataKey="name" type="category" width={150} />
  <Bar dataKey="value" />
</BarChart>
```

**Prevention:**

- Remember Recharts logic: `layout="vertical"` = Horizontal Bars.
- If bars are missing but tooltips work, check `layout` prop first.
- Always match `layout` to the axis types:
  - `horizontal`: X=category, Y=number
  - `vertical`: Y=category, X=number

**Context:**

- Versions affected: Recharts 2.x
- Source: `260218-0130-fix-confidence-chart.md` (Session Log)

**Tags:** `recharts` `charts` `visualization` `bug`

### Entry 4: API Route Responses Need Explicit Cache-Control Headers {#entry-4}

**Problem:**
A Next.js App Router API route (`route.ts`) returns correct JSON data when tested via `curl`, but the browser serves stale data from a previous response. The UI shows outdated values even after page refresh.

**Root Cause:**
Next.js API route handlers (`GET` in `route.ts`) do not set `Cache-Control` headers by default. Without explicit headers, the browser applies heuristic caching — it may reuse a previous response without revalidating with the server.

This is especially dangerous for dynamic data (e.g., status endpoints, real-time metrics) where stale data causes incorrect UI state. The `Vary` header that Next.js adds (`rsc, next-router-state-tree, ...`) does not prevent browser-level fetch caching.

**Solution:**
Add explicit `Cache-Control` headers to every dynamic API route response:

```typescript
return NextResponse.json(
  { status: result.rows[0].status },
  { headers: { "Cache-Control": "no-store, no-cache, must-revalidate" } },
);
```

For the client-side fetch, also add `cache: "no-store"` as a belt-and-suspenders measure:

```typescript
const response = await fetch(`/api/agents/${id}/status`, {
  cache: "no-store",
});
```

**Prevention:**

- **Every dynamic API route** should include `Cache-Control: no-store, no-cache, must-revalidate` in its response
- Add `cache: "no-store"` to client-side `fetch()` calls for dynamic data
- When debugging "UI shows stale data", check response headers with `curl -sI` before investigating React state
- Static data (rarely changing) can use `Cache-Control: public, max-age=300` instead

**Context:**

- Versions affected: Next.js 15+ App Router (route handlers)
- OS: all (browser caching behavior)
- First documented: 2026-02-21
- Source: alfred-01 session — dashboard status badge showing stale "unknown" response

**Tags:** `nextjs` `api-routes` `cache-control` `browser-caching` `stale-data`

---

### Entry 5: Zustand Persist Middleware Causes Premature Auth Redirects in SSR {#entry-5}

**Problem:**
In a Next.js application using Zustand with `persist` middleware for authentication state, users are redirected to `/login` on every page reload — even though `isAuthenticated: true` is correctly saved in `localStorage`.

```
// User logs in → state persisted → reload page → immediately redirected to /login
```

**Root Cause:**
Zustand's `persist` middleware rehydrates state from `localStorage` **asynchronously**. During SSR and the first client-side render, the store initializes with default values (`isAuthenticated: false`). A `useEffect` hook checking authentication runs before hydration completes, triggering a premature redirect:

```typescript
// BROKEN — runs before Zustand hydration finishes
const { isAuthenticated } = useAuthStore();

useEffect(() => {
  if (!isAuthenticated) {
    router.push("/login"); // Redirects before localStorage is read
  }
}, [isAuthenticated]);
```

**Solution:**
Implement a hydration guard pattern — wait for client-side mount and Zustand hydration before evaluating persisted state:

```typescript
const { isAuthenticated } = useAuthStore();
const [isHydrated, setIsHydrated] = useState(false);

// 1. Wait for client-side mount
useEffect(() => {
  setIsHydrated(true);
}, []);

// 2. Only check auth after hydration is confirmed
useEffect(() => {
  if (isHydrated && !isAuthenticated) {
    router.push("/login");
  }
}, [isAuthenticated, router, isHydrated]);

// 3. Prevent flash of unauthenticated content
if (!isHydrated || !isAuthenticated) {
  return null; // Or a loading spinner
}
```

**Prevention:**

- When using persisted client-side state in an SSR framework (Next.js, Remix, Nuxt), always assume the initial render uses server defaults (empty/false)
- Use a `useHydrated` hook or local state flag to defer logic until hydration completes
- Never access `localStorage` directly during render — causes hydration mismatch errors
- Implement backend-side bypasses for development credentials rather than frontend state hacks

**Context:**

- Versions affected: Next.js 14+ (App Router), Zustand v4+ with `persist` middleware
- Storage: `localStorage`
- OS: all
- First documented: 2026-02-09
- Source: `260209-1552-lessons-learned.md`, `260208-2100-lessons-learned.md`

**Tags:** `nextjs` `zustand` `persist` `ssr` `hydration` `authentication` `state-management`

---

### Entry 6: Recharts Crashes with DecimalError on NaN or Null Data {#entry-6}

**Problem:**
A page using Recharts crashes with an unhandled runtime error when rendering chart components:

```
Unhandled Runtime Error: [DecimalError] Invalid argument: NaN
```

The error originates from `recharts-scale` during axis domain calculation.

**Root Cause:**
The Recharts library (specifically `recharts-scale`) throws a hard error when calculating axis domains if any data point contains `NaN` or `undefined`. When the API returns `null` for numeric fields and the component performs math on them (e.g., `value * 100`), the result is `NaN`, which crashes the chart renderer. Unlike many React components, Recharts does not gracefully handle non-numeric values — it throws instead of ignoring.

**Solution:**
Sanitize all data at the component boundary before passing it to Recharts:

```typescript
// Sanitize each value before chart consumption
value:
  rawValue && !isNaN(rawValue)
    ? Math.round(rawValue * 100)
    : 0,
```

For a more robust approach, use a utility function:

```typescript
function chartSafe(
  value: number | null | undefined,
  transform?: (v: number) => number,
): number {
  if (value === null || value === undefined || isNaN(value)) return 0;
  return transform ? transform(value) : value;
}

// Usage
const chartData = apiData.map((d) => ({
  name: d.label,
  value: chartSafe(d.score, (v) => Math.round(v * 100)),
}));
```

**Prevention:**

- Never assume API data is numeric — always sanitize at the component boundary
- Use a Zod schema or utility function to transform API responses into "chart-safe" formats (replace `null`/`NaN` with `0` or omit the data point)
- Wrap chart components in a React Error Boundary to prevent the entire page from crashing due to a single bad data point
- Test chart components with `null`, `undefined`, and `NaN` data explicitly

**Context:**

- Versions affected: Recharts 2.x (uses `recharts-scale` internally)
- OS: all (browser)
- First documented: 2026-02-08
- Source: `260208-2100-lessons-learned.md`

**Tags:** `recharts` `charts` `nan` `data-validation` `error-handling` `nextjs`

---

## Cross-References

- [infra/nginx.md](infra/nginx.md) — Nginx routing collision with Next.js client routes
- [infra/openclaw.md](infra/openclaw.md) — OpenClaw deployment and configuration

---

## External Resources

- [Next.js Documentation — basePath](https://nextjs.org/docs/app/api-reference/next-config-js/basePath)
- [Next.js Documentation — Turbopack](https://nextjs.org/docs/architecture/turbopack)
- [LiteLLM GitHub Issue #20011](https://github.com/BerriAI/litellm/issues/20011) — SERVER_ROOT_PATH bug
- [Let's Encrypt — SAN Certificates](https://letsencrypt.org/docs/faq/#can-i-issue-a-certificate-for-multiple-domain-names)

---

## Changelog

| Date       | Change                                                  | Source                               |
| ---------- | ------------------------------------------------------- | ------------------------------------ |
| 2026-02-22 | Entry 5: Zustand hydration guard; Entry 6: Recharts NaN | 42bros lessons-learned consolidation |
| 2026-02-21 | Entry 4: API route Cache-Control headers                | alfred-01 session log 260221-0158    |
| 2026-02-18 | Initial creation with 2 entries                         | Multiple TMP sources consolidated    |
