# Skill: Worktree Workflow — Parallel Agent Isolation

Git worktrees for parallel agent sessions. Each agent works in an isolated checkout tied to its feature branch. The main checkout stays on `main` and is never used for work.

---

## When to Use

Any project where multiple agents may operate on the same repository concurrently. Applies only to **protected repos** (repos with branch protection that require the branch-per-task workflow from `skill-branching-strategy.md`). Direct-to-main repos (`meta/`, `shared-knowledge/`, or equivalent) are excluded — no worktree needed.

---

## Principles

1. **Main checkout is read-only.** The primary clone stays on `main`. No agent commits, stages, or modifies files there. It exists solely as the base for creating worktrees.
2. **One worktree per branch.** Each feature branch gets exactly one worktree. Two agents must not share a worktree.
3. **Worktrees are ephemeral.** Created at session start, removed after PR merge. The only exception is WIP handover — the worktree survives until the next agent resumes and completes the work.
4. **No destructive git operations.** Worktrees share the same `.git` directory. Operations like `git gc`, `git prune`, `git rebase` across worktrees, or `git worktree prune` while other agents are active can corrupt shared state. These are prohibited during any session.

---

## Naming Convention

Worktree directory: `{repo}--{branch-name}`

The worktree is created as a sibling directory to the main checkout, at the same level in the filesystem.

```/dev/null/worktree-naming.md#L1-5
42bros-mario/                      ← main checkout (stays on main, read-only)
42bros-mario--feat/B03-05B-fixes/  ← worktree for agent A
42bros-mario--fix/adhoc-hotfix/    ← worktree for agent B (different branch)
```

**Rules:**

- Double-dash `--` separates the repo name from the branch name
- The branch name portion preserves the slash (e.g., `feat/B03-05B-fixes`) — git worktree handles this natively
- If the filesystem doesn't support slashes in directory names, replace `/` with `--` (e.g., `42bros-mario--feat--B03-05B-fixes`)

---

## Agent Workflow

### Session Start — Create Worktree

When an agent is assigned work on a protected repo:

- [ ] 1. `cd` into the main checkout of the repo
- [ ] 2. Ensure main is up to date: `git fetch origin`
- [ ] 3. **New branch:** create worktree + branch in one step:
  ```/dev/null/create-worktree-new.sh#L1-1
  git worktree add ../{repo}--{branch-name} -b {branch-name} origin/main
  ```
- [ ] 4. **Resuming an existing branch** (from handover): create worktree from existing remote branch:
  ```/dev/null/create-worktree-existing.sh#L1-1
  git worktree add ../{repo}--{branch-name} {branch-name}
  ```
  If the local branch doesn't exist yet: `git worktree add ../{repo}--{branch-name} -b {branch-name} origin/{branch-name}`
- [ ] 5. `cd` into the worktree directory for all subsequent work on that repo
- [ ] 6. All file reads, edits, commits, and pushes happen inside the worktree — never in the main checkout

### During Session

- All `git` operations (`add`, `commit`, `push`) happen inside the worktree directory
- The agent's tool file paths must reference the worktree directory, not the main checkout
- Multiple worktrees across different repos can be active simultaneously (one per repo per branch)

### Session End — Cleanup or Handover

**If PR was merged (work complete):**

- [ ] 1. `cd` out of the worktree (back to main checkout or any other directory)
- [ ] 2. Remove the worktree:
  ```/dev/null/remove-worktree.sh#L1-2
  cd {repo}
  git worktree remove ../{repo}--{branch-name}
  ```
- [ ] 3. Delete the local branch (squash-merge deletes the remote branch, but the local one lingers):
  ```/dev/null/delete-local-branch.sh#L1-2
  cd {repo}
  git branch -D {branch-name}
  ```
- [ ] 4. Verify clean: `git worktree list` shows only the main checkout, `git branch` shows only `main`

**If WIP handover (work incomplete, branch still open):**

- [ ] 1. Commit and push all changes to the branch from within the worktree
- [ ] 2. **Keep the worktree alive** — do not remove it
- [ ] 3. Record the worktree path in the handover so the next agent can resume:
  ```/dev/null/handover-worktree-note.md#L1-2
  Worktree: ../{repo}--{branch-name}
  Branch: {branch-name}
  ```
- [ ] 4. The next agent `cd`s into the existing worktree and continues — no need to create a new one

**If session ends with no commit and no WIP:**

- Something went wrong. The agent should flag this in the session log and remove the worktree + delete the local branch to avoid stale state.

**Stale branch scan (every session end, regardless of what was done this session):**

- [ ] For every repo the agent touched this session: `git fetch --prune origin && git branch`
- [ ] Any local branch whose remote counterpart no longer exists (and is not the current WIP branch) is stale → delete it: `git branch -D {branch-name}`
- [ ] If stale branches from prior sessions are found → clean them now and note in the session log

⛔ **A session cannot end with stale worktrees or stale local branches.** Cleaning them is mandatory, not advisory.

---

## Prohibited Operations

These operations affect the shared `.git` directory and can corrupt state for other agents working in parallel worktrees on the same repo:

| Operation                         | Why it's dangerous                                                         |
| :-------------------------------- | :------------------------------------------------------------------------- |
| `git gc`                          | Repacks objects that other worktrees may be reading                        |
| `git prune`                       | Removes objects that other worktrees may reference                         |
| `git worktree prune`              | Removes worktree metadata — may invalidate another agent's active worktree |
| `git rebase` across worktrees     | Rewrites commits that another worktree's branch may depend on              |
| `git clean -fdx` in main checkout | May remove worktree link files from `.git/worktrees/`                      |

If any of these are needed (e.g., repo maintenance), they must be done when **no other agent is active** on the repo.

---

## Edge Cases

### Agent crashes or session aborts

Worktree survives on disk. Next agent (or user) can either resume it or clean it up:

```/dev/null/cleanup-stale.sh#L1-3
cd {repo}
git worktree list          # identify stale worktrees
git worktree remove ../{repo}--{branch-name} --force
```

### Two agents need the same repo but different branches

This is the normal case — each gets its own worktree. No conflict.

### Two agents need the same branch on the same repo

This should not happen. If it does, the workflow has a coordination problem upstream (TRON or user assigned overlapping work). The second agent must stop and flag the conflict.

### Worktree directory already exists

If `git worktree add` fails because the directory exists, the agent should check `git worktree list` to determine if it's a valid active worktree (possibly from a prior session's WIP handover) or stale. If stale → remove and recreate. If active from handover → resume it.

---

## Project-Specific Extensions

Each project extends this skill by defining:

1. **Which repos require worktrees** — typically all protected repos from the branching strategy's repo classification table
2. **Worktree base path** — where worktrees are created (default: sibling to main checkout)
3. **Handover format** — how worktree paths are recorded in the project's handover mechanism
4. **Agent-specific rules** — which agents create worktrees (typically only agents that write code) vs. which agents are read-only and don't need them

---

**Last Updated:** 2026-03-01

---

## Changelog

| Date       | Change                                                            |
| :--------- | :---------------------------------------------------------------- |
| 2026-03-01 | Initial creation — worktree isolation for parallel agent sessions |
