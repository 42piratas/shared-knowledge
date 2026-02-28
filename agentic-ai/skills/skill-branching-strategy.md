# Skill: Branching Strategy — Branch-Per-Task Workflow

Branch-per-task workflow for CI-gated, auto-merge deployments. Every code change flows through a short-lived branch → PR → CI → auto-merge → deploy. `main` is always deployable.

---

## When to Use

Any project that uses CI/CD and wants to gate code changes before they reach `main`. Applies to all repos where a push to `main` triggers a deploy or has infrastructure consequences. Repos with no deploy risk (documentation, meta, knowledge bases) remain direct-to-main.

---

## Principles

1. **`main` is always deployable.** No exceptions. If it's on `main`, it passed CI.
2. **Branches are ephemeral.** One task, one branch, merge same day. Delete after merge.
3. **CI is the gate.** Auto-merge on green CI. CI quality depends on test coverage — this skill wires the gate, not the tests.
4. **Agents stay autonomous.** The agent creates the branch, pushes, opens PR, monitors CI, fixes failures, and validates deploy — no human in the loop unless the agent can't self-recover.
5. **Incremental adoption.** Start with CI gating and auto-merge. Required reviewers, environment gates, and staging deploys are future enhancements when stakes increase.

---

## Branch Naming Convention

```/dev/null/branch-naming.md#L1-3
feat/{short-desc}     — new functionality
fix/{short-desc}      — bug fixes
chore/{short-desc}    — maintenance, deps, config
```

Rules:

- Lowercase, hyphens only
- No ticket/block numbers (keep readable)
- Cross-repo tasks: **identical branch name on every affected repo**

---

## Repo Classification

Every repo in the project falls into one of these categories. The project's agent docs define which repos belong to each.

| Category        | Workflow                                             | Branch Protection | Auto-Merge |
| :-------------- | :--------------------------------------------------- | :---------------- | :--------- |
| Deploy-tracked  | Branch → PR → CI (`test.yml`) → auto-merge → deploy | ✅                 | ✅          |
| IaC             | Branch → PR → validate CI → merge → user applies    | ✅                 | ✅          |
| Docs / meta     | Direct-to-main                                       | ❌                 | N/A        |

---

## Engineer Workflow

### Session Start

- [ ] **Determine branch name** from block plan or task description (use naming convention above)
- [ ] Create and checkout branch in each affected protected repo: `git checkout -b {branch-name}`
- [ ] If resuming a prior branch from handover: `git checkout {branch-name}` and `git pull`

### During Session

- All `git push` operations target the feature branch, not `main`
- Push with: `git push -u origin {branch-name}`
- Docs/meta repos remain direct-to-main — no branch needed

### Session End

- [ ] For each affected protected repo:
  1. `git add`, `commit` (Conventional Commits), push to branch
  2. Open PR with auto-merge: `gh pr create --fill --base main && gh pr merge --auto --squash`
  3. Wait for CI: `gh run list --repo {org}/{repo} --limit 1 --json status,conclusion`
  4. If CI fails → fix, push to same branch, CI re-runs automatically
  5. Once CI passes → auto-merge triggers → deploy triggers (if deploy-tracked)
  6. Branch auto-deletes after merge (repo setting)
- [ ] Docs/meta repos: push directly to main as before
- [ ] If any repo is deploy-tracked → follow the project's deploy validation procedure after merge

### Engineer Return / Handover

Include in the Engineer Return message:

- Branch name used this session
- PR status per repo (merged / open / failed)

Include in handover (if branch is still open):

- Branch name, so the next session can resume

---

## Cross-Repo Ordering

When a task touches a shared library AND its consumers, merging order matters. The shared library must be merged, released, and pinned before consumers claim compatibility.

**Generic sequence:**

- [ ] 1. Merge the shared library PR first
- [ ] 2. Release the new version (tag, publish, or whatever the project's release mechanism is)
- [ ] 3. Update the version pin in each consumer's dependency file to the new release
- [ ] 4. **Only then** create branches and open PRs on consumer repos (with the updated pin)
- [ ] 5. If the shared library PR fails CI → do NOT proceed to steps 2–4. Fix first.

Agent MUST NOT open consumer PRs before the shared library is merged AND released. Consumer CI may pass against the old version, but correctness requires the new release to exist before consumers reference it.

Each project defines its specific release mechanism (git tags, package registry, etc.) in the project's Engineer agent doc.

---

## Branch Timeout Policy

- [ ] Branch open > **2 sessions** without merging → escalate to user with reason (CI flaky? blocked dependency? scope creep?)
- [ ] CI flaky (passes/fails intermittently on same code) → do NOT retry more than 3 times. Log the flaky test name, escalate to user. Flaky tests are a test quality issue, not a branching issue.
- [ ] CI timeout → check CI provider status page. If no outage, retry once. If still timing out, escalate to user.
- [ ] Abandoned branches (escalated but no user response) → escalate again each session until user responds.

---

## TRON Integration

When an orchestrator (TRON or equivalent) manages the session:

**Session Plan:**
- Include branch name (from handover or derived from task)

**Agent Spawn:**
- Pass branch name to the engineer spawn instruction

**Engineer Returns:**
- Track PR status per repo (merged / open / failed)
- If any PR still open → note in session summary with reason and age (session count)

**Session End:**
- Verify all PRs merged before marking session complete
- If any PR still open: flag in session log and handover with reason and age

---

## Reviewer Integration

When a code reviewer operates alongside the engineer:

- **With a branch:** scope review to the PR diff (`gh pr diff --repo {org}/{repo}`) instead of timestamp-based `git log`
- **Without a branch (fallback):** use timestamp-based scope from the last review log

---

## IaC-Specific Notes

IaC repos use the same branch workflow but with different CI and a manual apply step:

- CI runs structural validation only (e.g., `terraform fmt -check` + `terraform validate`) — not state-changing operations
- `plan` and `apply` remain manual (user-executed). CI catches syntax and structure errors before merge.
- The user reviews the plan output **before merge** to decide whether to approve the PR
- Emergency exception: admin bypass allows direct-to-main hotfixes (document reason in session log)

---

## Branch Protection Configuration

When setting up branch protection (typically a one-time setup task):

- Require status checks to pass before merging (required for auto-merge to function — without this, PRs merge immediately)
- Do NOT require pull request reviews (unless the project has required reviewers)
- Allow force pushes: No
- Allow deletions: No
- **Allow administrators to bypass protection: Yes** — self-lock mitigation for solo developers
- Enable auto-merge at repo level
- Enable auto-delete of head branches after merge

**Self-lock mitigation:** With no second person to approve overrides, a misconfigured rule can lock the repo. Admin bypass is the escape hatch. Verify it works immediately after enabling protection.

**Emergency rollback** (remove protection entirely):

```/dev/null/emergency-rollback.sh#L1-1
gh api -X DELETE repos/{org}/{repo}/branches/main/protection
```

---

## Project-Specific Extensions

Each project extends this skill by defining:

1. **Repo classification table** — which repos are deploy-tracked, IaC, or docs-only
2. **Exact CI workflow names** — what status checks to require (e.g., `test.yml`, `validate.yml`)
3. **Cross-repo release mechanism** — git tags, package registry, or other
4. **Cross-repo ordering details** — exact dependency graph and pin update procedure
5. **Deploy validation procedure** — what to check after merge triggers a deploy

These extensions belong in the project's Engineer agent file or project-specific skills — not in this shared skill.

---

**Last Updated:** 2026-02-28

---

## Changelog

| Date       | Change                                                                                                |
| :--------- | :---------------------------------------------------------------------------------------------------- |
| 2026-02-28 | Initial creation — extracted branch-per-task workflow from 42Bros branching strategy block plan        |
