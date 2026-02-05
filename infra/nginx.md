# Nginx

Lessons learned for Nginx web server configuration and deployment.

---

## SSL Certificate Bootstrap (Chicken-and-Egg Problem)

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
- Tool/Version: Nginx 1.x, Certbot 1.x
- OS: Ubuntu
- Other relevant context: Common during server migrations

**Tags:** `nginx` `ssl` `certbot` `letsencrypt`

---
