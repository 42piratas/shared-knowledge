# Persistent Storage

Lessons learned for managing persistent storage in cloud infrastructure.

---

## Stateful Services on Ephemeral Storage (Data Loss Risk)

**Problem:**
Critical data lost when droplet is destroyed or resized via Terraform.

**Root Cause:**
Stateful service (e.g., secrets manager, database) was configured to store data on the droplet's local disk, which is ephemeral. Running `terraform apply` to resize or replace the droplet permanently deletes all data.

**Solution:**
1. Create persistent block storage volume
2. Mount volume before service starts
3. Configure service to use mounted path
4. Add `prevent_destroy = true` lifecycle rule

```hcl
resource "digitalocean_volume" "service_data" {
  name   = "service-data-prod"
  region = var.do_region
  size   = 10

  lifecycle {
    prevent_destroy = true
  }
}
```

**Prevention:**
- Never run stateful services on ephemeral storage
- Always use persistent volumes for databases and secret stores
- Test backup/restore before any infrastructure changes
- Use lifecycle rules to prevent accidental destruction

**Context:**
- Tool/Version: DigitalOcean Volumes, Terraform
- Note: Cost is minimal (~$1/month for 10GB)

**Tags:** `storage` `persistence` `terraform` `digitalocean` `data-loss`

---
