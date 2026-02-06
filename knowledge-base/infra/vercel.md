# Vercel

Lessons learned for deploying and managing applications on Vercel.

---

## Vercel Functions Read-Only Filesystem Constraints

**Problem:**
Attempted to update JSON configuration files directly from Vercel Functions but received filesystem errors preventing file writes.

```
Error: EROFS: read-only file system, open '/var/task/data/feed-sources.json'
```

**Root Cause:**
Vercel Functions run in a read-only filesystem environment. Files cannot be modified at runtime, only read from the deployment bundle.

**Solution:**
Created separate scripts for local development that generate configuration data, then deploy the updated files as part of the build process rather than modifying them at runtime.

```javascript
// Local script approach
function updateConfigFile() {
  const filePath = path.join(__dirname, '..', 'data', 'feed-sources.json');
  const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
  // Apply modifications
  fs.writeFileSync(filePath, JSON.stringify(data, null, 2));
}
```

**Prevention:**
- Use environment variables for runtime configuration changes
- Implement configuration via external services (database, KV store) for dynamic updates
- Design API endpoints to return configuration data rather than modify files
- Use build-time scripts for static configuration updates

**Context:**
- Platform: Vercel Functions
- Runtime: Node.js serverless environment
- File Type: JSON configuration files

**Tags:** `vercel` `serverless` `filesystem` `configuration`

**Source:** 260206-0025-lessons-learned.md

---
