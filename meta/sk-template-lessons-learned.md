# Lessons Learned Template

**Instructions for Project Agents:** Use this template to document lessons learned during a session. Create one file per session with filename format `YYMMDD-HHMM-lessons-learned.md` in the project's designated lessons folder (e.g., `{project}/lessons/`).

---

## Session Metadata

- **Date:** YYYY-MM-DD
- **Project:** {project_name}
- **Agent/Model:** {model_name}
- **Session Focus:** {brief description of what was being worked on}

---

## Lessons Learned

<!-- 
Copy the template below for EACH lesson learned in this session.
Delete any unused templates before finalizing.
Include ONLY generalizable lessons — not project-specific business logic.
-->

### Lesson: {Brief Descriptive Title}

**Category:** {infra | tooling | languages | frameworks | databases | messaging | secrets}

**Topic:** {specific tool/technology, e.g., "digitalocean", "poetry", "python", "terraform"}

**Problem:**
<!-- What went wrong? What error was encountered? Be specific. Include error messages if applicable. -->

```
{Paste exact error message or describe the unexpected behavior}
```

**Root Cause:**
<!-- Why did this happen? What was the underlying issue? -->

**Solution:**
<!-- How was it fixed? Include specific commands, code changes, or configuration updates. -->

```
{Include code/commands if applicable}
```

**Prevention:**
<!-- How can this be avoided in the future? Any checks or practices to adopt? -->

**Context:**
<!-- Version numbers, OS, specific conditions that matter -->
- Tool/Version: 
- OS: 
- Other relevant context:

**Tags:** `{tag1}` `{tag2}` `{tag3}`

---

### Lesson: {Title}

**Category:** 

**Topic:** 

**Problem:**

```
```

**Root Cause:**

**Solution:**

```
```

**Prevention:**

**Context:**
- Tool/Version: 
- OS: 
- Other relevant context:

**Tags:** 

---

### Lesson: {Title}

**Category:** 

**Topic:** 

**Problem:**

```
```

**Root Cause:**

**Solution:**

```
```

**Prevention:**

**Context:**
- Tool/Version: 
- OS: 
- Other relevant context:

**Tags:** 

---

## Session Notes

<!-- 
Optional: Any additional observations that don't fit the lesson format but might be useful.
Examples: "This tool seems unstable", "Documentation was outdated", "Consider alternative X"
-->

---

## Checklist Before Submission

- [ ] All lessons are generalizable (not project-specific)
- [ ] No sensitive data (API keys, internal URLs, passwords)
- [ ] Error messages are included where applicable
- [ ] Solutions are complete and actionable
- [ ] Categories and topics are correctly assigned
- [ ] File saved as `YYMMDD-HHMM-lessons-learned.md`
