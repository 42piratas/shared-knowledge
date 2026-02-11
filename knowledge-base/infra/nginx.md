# Nginx

**Category:** infra
**Last Updated:** 2026-02-18
**Entries:** 3

---

## Overview

Lessons learned for Nginx web server configuration and deployment — focusing on SSL bootstrapping, reverse proxy routing, and CI/CD deployment patterns.

---

## Entries

### Entry 1: SSL Certificate Bootstrap (Chicken-and-Egg Problem) {#entry-1}

**Problem:**
Nginx fails to start because SSL certificate paths don't exist, but Certbot can't run because Nginx isn't serving port 80 for the ACME challenge.

```
cannot load certificate "/etc/letsencrypt/live/domain.xyz/fullchain.pem": No such file or directory
```

**Root Cause:**
Nginx configuration references SSL certificate paths before Certbot has obtained them. Certbot requires HTTP access (port 80) to complete the challenge, which requires Nginx to be running.

**Solution:**
Two-stage deployment:

1. Deploy HTTP-only config first (no SSL blocks)
2. Run Certbot to obtain certificates
3. Deploy HTTPS config with SSL paths

```
# Stage 1: Deploy HTTP config
cp nginx-http-only.conf /etc/nginx/sites-available/
nginx -t && systemctl reload nginx

# Stage 2: Get certificates
certbot --nginx -d domain.xyz -d www.domain.xyz --non-interactive --agree-tos

# Stage 3: Deploy full HTTPS config
cp nginx-https.conf /etc/nginx/sites-available/
nginx -t && systemctl reload nginx
```

**Prevention:**

- Always use two-stage SSL deployment
- Keep HTTP-only configs as templates for bootstrap
- Automate the sequence in IaC provisioners

**Context:**

- Versions affected: Nginx 1.x, Certbot 1.x
- OS: Ubuntu
- First documented: 2026-02-03
- Source: `260203-1500-retroactive-lessons-learned.md`

**Tags:** `nginx` `ssl` `certbot` `letsencrypt`

---

### Entry 2: Routing Collision Between Nginx Proxy Paths and Next.js Client Routes {#entry-2}

**Problem:**
A third-party service (LiteLLM) embedded in an iframe at `/litellm` loads the host application itself recursively, resulting in a blank or infinite loading state.

**Root Cause:**
The Next.js app has a client-side route `/litellm`. The Nginx reverse proxy has a location block `/litellm/` (with trailing slash) that proxies to an upstream service. The iframe URL was set without a trailing slash (`https://domain.com/litellm`), so Nginx saw `/litellm` (no trailing slash), which did NOT match the `/litellm/` proxy location block. It fell through to the default `/` location, which proxies to the Next.js dashboard — loading the dashboard inside itself.

**Solution:**
Add a trailing slash to the URL: `NEXT_PUBLIC_LITELLM_URL=https://domain.com/litellm/`

This ensures the request matches the Nginx proxy location block exactly, routing to the upstream service instead of the Next.js app.

**Prevention:**

- Always distinguish between client-side routes and API proxy paths — avoid naming collisions
- Use trailing slashes consistently for Nginx proxy locations
- Verify environment variables used for internal routing match the exact Nginx location block syntax
- Test iframe/embed URLs match the proxy path exactly

**Context:**

- Versions affected: Nginx (any), Next.js 16+
- OS: all
- First documented: 2026-02-09
- Source: `260209-2230-lessons-learned-ui-deployment.md`

**Tags:** `nginx` `nextjs` `reverse-proxy` `trailing-slash` `routing`

---

### Entry 3: Missing Nginx Configuration File in CI/CD Deployment {#entry-3}

**Problem:**
Nginx container restarts repeatedly after CI/CD deployment. Ports 80/443 are not accessible. Logs show configuration file not found or mount errors about missing files.

**Root Cause:**
The CI/CD "Copy configuration files" step only copied application-specific configurations (e.g., app config, LiteLLM config) but missed the nginx configuration file. The nginx service was defined in docker-compose.yml but its configuration dependency was not being deployed.

**Solution:**
Add nginx directory creation and configuration file copying to the CI/CD deployment script:

```yaml
- name: Copy configuration files
  run: |
    ssh "mkdir -p /opt/project/nginx"
    scp nginx/nginx.conf root@server:/opt/project/nginx/nginx.conf
```

**Prevention:**

- When adding a new service to docker-compose.yml, audit ALL configuration file dependencies and add them to the CI/CD deployment script
- Maintain a complete inventory of required configuration files alongside the docker-compose.yml
- Test configuration file presence before attempting service startup in the deployment script
- Add a pre-start validation step: `test -f /opt/project/nginx/nginx.conf || (echo "Missing nginx config" && exit 1)`

**Context:**

- Versions affected: Nginx (any), GitHub Actions CI/CD
- OS: all
- First documented: 2026-02-08
- Source: `260208-2207-lessons-learned-phase6-deployment.md`

**Tags:** `nginx` `ci-cd` `deployment` `configuration`

---

## Cross-References

- [frameworks/nextjs.md](frameworks/nextjs.md) — Next.js subpath proxying failure (use subdomains)
- [infra/github-actions.md](infra/github-actions.md) — CI/CD environment variable and deployment gaps

---

## External Resources

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Let's Encrypt — Getting Started](https://letsencrypt.org/getting-started/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

---

## Changelog

| Date       | Change                                            | Source                                       |
| ---------- | ------------------------------------------------- | -------------------------------------------- |
| 2026-02-05 | Initial creation with 1 entry                     | `260203-1500-retroactive-lessons-learned.md` |
| 2026-02-18 | Added entries #2-3 (routing collision, CI/CD gap) | Multiple TMP sources                         |
