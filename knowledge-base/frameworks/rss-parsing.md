# RSS Parsing

Lessons learned for parsing RSS and Atom feeds.

---

## RSS Feed Author Field XML Parsing in Atom Feeds

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
