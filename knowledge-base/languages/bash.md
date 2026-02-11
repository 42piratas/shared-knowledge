# Bash / Shell Scripting

**Category:** languages
**Last Updated:** 2026-02-18
**Entries:** 1

---

## Overview

Lessons learned for POSIX shell and Bash scripting patterns, focusing on correctness, portability, and CI validation.

---

## Quick Reference

| Task | Command/Pattern |
|------|-----------------|
| Lint shell scripts | `shellcheck script.sh` |
| Log all arguments as string | `echo "Command: $*"` |
| Pass all arguments to command | `exec "$@"` |

---

## Entries

### Entry 1: `$@` vs `$*` — Use `$*` for String Context, `$@` for Argument Passing {#entry-1}

**Problem:**
A shell entrypoint script failed CI validation with shellcheck error SC2145:

```
In infrastructure/docker/openclaw/entrypoint.sh line 24:
echo "[entrypoint] Executing command: $@"
                                      ^-- SC2145 (error): Argument mixes string and array. Use * or separate argument.
```

**Root Cause:**
`$@` expands to separate words (one per positional parameter) when unquoted, or to separate quoted strings when inside double quotes. Using it inside a string context (`echo "text $@"`) mixes a string with an array, which is ambiguous behavior.

- `"$@"` expands to `"$1" "$2" "$3"` — correct for passing arguments to commands
- `"$*"` expands to `"$1 $2 $3"` (single string with IFS separator) — correct for display/logging

**Solution:**

```bash
# WRONG — mixing string and array
echo "[entrypoint] Executing command: $@"

# CORRECT — $* for display as a single string
echo "[entrypoint] Executing command: $*"

# CORRECT — $@ for passing arguments to exec
exec "$@"
```

**Prevention:**
- Use `$*` when you need all arguments as a single string (logging, display, string interpolation)
- Use `$@` (always quoted as `"$@"`) when passing arguments to another command
- Run `shellcheck` on all shell scripts before committing — add it to CI validation
- Rule of thumb: if it's inside `echo` or a string, use `$*`; if it's an argument to `exec` or another program, use `"$@"`

**Context:**
- Versions affected: All POSIX sh and Bash versions
- OS: all
- First documented: 2026-02-08
- Source: `260208-0035-lessons-learned.md`

**Tags:** `bash` `shellcheck` `posix-sh` `SC2145`

---

## Cross-References

- [infra/docker.md](infra/docker.md) — Docker entrypoint scripts commonly use `$@` for CMD passthrough

---

## External Resources

- [ShellCheck Wiki — SC2145](https://www.shellcheck.net/wiki/SC2145)
- [Bash Reference Manual — Special Parameters](https://www.gnu.org/software/bash/manual/html_node/Special-Parameters.html)

---

## Changelog

| Date | Change | Source |
|------|--------|--------|
| 2026-02-18 | Initial creation with 1 entry | `260208-0035-lessons-learned.md` |
