# Lessons Learned Template

**Instructions for Project Agents:** Use this template to document lessons learned during a session. Create one file per session with filename format `YYMMDD-HHMM-lessons-learned.md` in the project's designated lessons folder (e.g., `{project}/lessons/`).

---

## Session Metadata

- **Date:** 2026-02-06
- **Project:** Ethereum World
- **Agent/Model:** Claude Sonnet 4
- **Session Focus:** Method 3 Phase 1 implementation and RSS parsing data quality fixes

---

## Lessons Learned

### Lesson: RSS Feed Author Field XML Parsing in Atom Feeds

**Category:** frameworks

**Topic:** rss-parsing

**Problem:**
Reddit RSS feeds (Atom format) were displaying raw XML in author fields instead of clean usernames. Users saw `<name>/u/AbdulRoosetrane</name><uri>https://www.reddit.com/user/AbdulRoosetrane</uri>` instead of `/u/AbdulRoosetrane`.

```
Author field content: "<name>/u/Civil-Insurance1931</name><uri>https://www.reddit.com/user/Civil-Insurance1931</uri>"
Expected: "/u/Civil-Insurance1931"
```

**Root Cause:**
RSS parser's `extractAuthor()` function only handled simple string authors and `dc:creator` fields. It didn't parse nested XML structures common in Atom feeds where author information is wrapped in `<name></name>` tags.

**Solution:**
Enhanced `extractAuthor()` function to detect and parse XML tags in author strings with case-insensitive matching.

```typescript
if (typeof author === "string") {
  // Handle nested XML in author field (common in Atom feeds)
  if (
    author.toLowerCase().includes("<name>") &&
    author.toLowerCase().includes("</name>")
  ) {
    const nameMatch = author.match(/<name>(.*?)<\/name>/i);
    if (nameMatch) {
      return nameMatch[1]; // Extract clean username
    }
  }
  return author;
}
```

**Prevention:**
- Test RSS parsers against both RSS and Atom feed formats
- Include XML parsing test cases for author fields
- Use case-insensitive regex matching for robustness

**Context:**
- RSS Parser: Custom TypeScript implementation
- Feed Type: Atom feeds from Reddit
- OS: Vercel Functions environment (Node.js)

**Tags:** `rss` `xml-parsing` `atom-feeds` `author-extraction`

---

### Lesson: Vercel Functions Read-Only Filesystem Constraints

**Category:** infra

**Topic:** vercel

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

---

### Lesson: RSS Source Reliability Assessment Accuracy

**Category:** tooling

**Topic:** monitoring

**Problem:**
Technical debt documentation incorrectly stated "15 failed RSS sources" when actual investigation revealed only 3 permanently broken sources, leading to misallocated development priorities.

```
Documented: "15 out of 42 RSS sources failing"
Actual: "3 broken, 5 disabled for quality, 34 active (81% uptime)"
```

**Root Cause:**
Source failure assessment was based on incomplete testing and confusion between temporarily disabled sources (for content quality) versus permanently broken RSS feeds.

**Solution:**
Implemented systematic source testing with HTTP status validation and clear categorization of failure types.

```bash
# Systematic testing approach
curl -o /dev/null -s -w "%{http_code}" "https://example.com/feed.rss"
# 200 = working, 404 = broken, 429 = rate limited, etc.
```

**Prevention:**
- Implement automated source health monitoring with clear failure categorization
- Document distinction between "disabled" vs "broken" vs "temporarily unavailable"
- Regular verification of technical debt items with actual system testing
- Use monitoring dashboards to track source reliability over time

**Context:**
- Tool: curl for HTTP testing
- Sources: 42 RSS feeds across various domains
- Environment: Manual testing from command line

**Tags:** `monitoring` `rss-feeds` `reliability` `technical-debt`

---

## Session Notes

- JSON configuration approach proved more reliable than database for source metadata in serverless environment
- Comprehensive test suites essential for validating complex multi-tier filtering systems
- Gradual implementation (Phase 1, 2, 3) reduces risk while establishing foundation for future enhancements
- Cost monitoring should be built-in from the start to prevent budget overruns in production

---

## Checklist Before Submission

- [x] All lessons are generalizable (not project-specific)
- [x] No sensitive data (API keys, internal URLs, passwords)
- [x] Error messages are included where applicable
- [x] Solutions are complete and actionable
- [x] Categories and topics are correctly assigned
- [x] File saved as `YYMMDD-HHMM-lessons-learned.md`
