# DigitalOcean

Lessons learned for DigitalOcean cloud infrastructure.

---

## External API Geo-Blocking Requires Region Testing

**Problem:**
Service works locally but fails in production with 451 error.

```
ExchangeNotAvailable: binance GET 451 - Service unavailable from a restricted location
```

**Root Cause:**
External APIs (Binance, others) may geo-block certain regions. DigitalOcean NYC3 region is blocked by Binance due to US regulations.

**Solution:**
1. Test API connectivity from target region BEFORE deployment
2. Migrate to unblocked region (e.g., Amsterdam `ams3`)
3. Document region requirements in playbook

```bash
# Test from target region
ssh user@server "curl -I https://api.binance.com/api/v3/ping"
# Expected: HTTP/1.1 200 OK
# Blocked: HTTP/1.1 451 Unavailable For Legal Reasons
```

**Prevention:**
- Research API geo-restrictions before choosing cloud region
- Test API access as part of infrastructure provisioning
- Maintain list of region requirements per external service
- Have migration procedure ready for region changes

**Context:**
- Tool/Version: Binance API, DigitalOcean
- Note: US-based regions commonly blocked by crypto exchanges

**Tags:** `digitalocean` `api` `geo-blocking` `region`

---
