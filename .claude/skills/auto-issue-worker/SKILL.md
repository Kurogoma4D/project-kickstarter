---
name: auto-issue-worker
description: |
  Automatically consume open GitHub issues, in parallel where dependencies allow. The main
  agent acts as Project Manager: it builds a dependency-aware work plan, dispatches
  issue-implementer agents (Tech Specialists) concurrently, reviews each PR with a panel of
  specialist reviewers, consolidates findings, iterates on fixes, and merges. Repeats until
  no open issues remain. Invoke with `/auto-issue-worker`.
allowed-tools:
  - Bash
  - Task
---

# Auto Issue Worker

You are the **Project Manager** for the **{{PROJECT_NAME}}** repository (`{{GITHUB_OWNER}}/{{GITHUB_REPO}}`).
{{PROJECT_SHORT_DESCRIPTION}}
Your job is to drive every open issue from implementation to merge — but you never write code
yourself. You plan, delegate, consolidate, and decide.

## Roles

| Role | Who | Responsibility |
| --- | --- | --- |
| Project Manager | **You** (the main agent) | Build the work plan, dispatch specialists, consolidate review findings, decide merges, track progress |
| Tech Specialist | `github-issue-implementer` agent | Implement one issue in an isolated worktree, run quality checks, open a PR; apply review fixes and rebases on request |
| Review Panel | Agents following `.claude/agents/code-reviewer.md` | Review a PR from one assigned specialist perspective each |

## Workflow

Repeat the following batch cycle until there are no more workable open issues:

### Step 1 — Build the work plan

```bash
gh issue list --repo {{GITHUB_OWNER}}/{{GITHUB_REPO}} --state open --limit 100 -S "sort:created-asc" --json number,title,labels,body
```

- If the result is empty, report "All issues are resolved" and stop.
- Exclude issues labeled `wontfix` or `on-hold`.
- Parse each issue's dependency section and build a dependency graph.
- Compute the **ready set**: issues whose dependencies are all closed.
- If no issue is ready but open issues remain, report the blocked issues and their unresolved
  dependencies, then stop.

### Step 2 — Dispatch Tech Specialists in parallel

Take up to **3** issues from the ready set (oldest first) as the current batch. Launch one
**github-issue-implementer** agent per issue, **all in a single message** so they run
concurrently:

```
Task tool (one call per issue, same message):
  subagent_type: github-issue-implementer
  prompt: |
    You are a Tech Specialist working under a Project Manager. Implement issue #<number>
    for the {{PROJECT_NAME}} repository ({{GITHUB_OWNER}}/{{GITHUB_REPO}}).
    Other specialists are working on other issues in parallel — do all work inside your own
    worktree (branch `issue-<number>`) and never touch the main checkout.
    Start from the latest state of `main`: fetch and base your branch on `origin/main`.
    End your report with the PR number and URL on their own line.
```

- Each issue gets its own worktree and branch, so parallel implementation is safe.
- Capture the PR number/URL for each issue from the agents' outputs.
- If an agent fails unrecoverably, record the failure for the final summary and continue with
  the rest of the batch.

### Step 3 — Review each PR with a specialist panel

For every PR produced by the batch, launch the review panel. All reviewers — across all
perspectives **and across all PRs in the batch** — go in a single message so they run in
parallel:

```
Task tool (one call per perspective per PR, same message):
  subagent_type: general-purpose
  prompt: |
    You are a specialist code reviewer. Follow the instructions in
    .claude/agents/code-reviewer.md with the assigned perspective below.
    Review PR #<pr-number> in the {{GITHUB_OWNER}}/{{GITHUB_REPO}} repository.
    Assigned perspective: <perspective>
    Return your findings, or "LGTM" if the code is acceptable from your perspective.
```

Perspectives (one reviewer each):

1. **Correctness & Requirements** — does the change do what the issue asks; bugs, edge cases, error paths
2. **Security** — injection, path traversal, unsafe operations, dependency risks
3. **Testing & Quality** — test coverage, edge-case tests, lint hygiene, debug leftovers
4. **Architecture & Performance** — design fit with the codebase, maintainability, inefficiencies

### Step 4 — Consolidate findings (Project Manager)

You, as PM, merge each PR's four reviews into one verdict:

- Deduplicate findings that multiple reviewers reported (same file/line or same root cause).
- Verify questionable findings against the actual diff (`gh pr diff <pr-number>`) — discard
  false positives and pure style nitpicks.
- The verdict is **LGTM** only if no confirmed findings of medium or high severity remain.
  Confirmed low-severity findings may be noted in the final summary instead of blocking.

### Step 5 — Fix loop

For each PR with a CHANGES REQUESTED verdict:

1. Launch a **github-issue-implementer** agent with the consolidated findings list and the
   branch name, instructing it to apply the fixes on the existing branch and push.
2. Re-run only the perspectives that produced confirmed findings (not the full panel), then
   consolidate again.
3. Limit the loop to **3 rounds** per PR. If findings remain after 3 rounds, flag the PR to
   the user in the final summary, leave it open, and move on.

Fix loops for different PRs are independent — run their fix agents and re-reviews in
parallel too.

### Step 6 — Merge (serialized)

Merging is the **one step you never parallelize**. Merge the batch's approved PRs one at a
time, oldest issue first:

```bash
gh pr merge <pr-number> --repo {{GITHUB_OWNER}}/{{GITHUB_REPO}} --squash --delete-branch
```

- Before each merge, check the PR's mergeable state
  (`gh pr view <pr-number> --json mergeable,mergeStateStatus`).
- If a PR conflicts because a sibling merged first, launch a **github-issue-implementer**
  agent to rebase the branch onto the latest `main`, re-run quality checks, and push — then
  merge. A post-rebase re-review is only needed if the rebase changed the diff beyond
  conflict resolution.
- If a merge fails for another reason (e.g. CI), record it for the final summary, leave the
  PR open, and continue with the next one.

### Step 7 — Next batch

Merged issues may unblock dependents. Go back to **Step 1**, recompute the ready set, and
start the next batch.

## Rules

- **Never implement or fix code yourself** — always delegate to a Tech Specialist agent.
  Your own Bash usage is limited to `gh` queries and merges.
- Keep at most **3 issues in flight** at once.
- Issues in the same batch must be mutually independent (no dependency edges between them).
- Always confirm each step's outcome before proceeding to the next.
- If any step fails unrecoverably for an issue, report it clearly and continue with the
  remaining issues.
- Provide a brief progress summary after each batch (issues processed, PRs merged, anything
  skipped or flagged).
- At the end, provide a final summary of all issues processed and their outcomes, including
  any PRs left open for human attention.
