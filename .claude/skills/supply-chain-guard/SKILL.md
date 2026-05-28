---
name: supply-chain-guard
description: |
  Harden the project against software supply-chain attacks across three layers: project
  dependencies (lockfile pinning, audits, install-time script controls), the Claude Code
  environment itself (permission allow/deny, MCP trust boundaries), and the CI/build
  pipeline (SHA-pinned actions, least-privilege tokens, dependency review). Sets up the
  guardrails with the user's approval, then audits existing dependencies and files a GitHub
  issue per remaining risk. Use during environment setup, or whenever dependencies, CI, or
  MCP servers change. Invoke with `/supply-chain-guard`. The created issues are consumed by
  `/auto-issue-worker`.
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - AskUserQuestion
---

# Supply Chain Guard

You harden the **{{PROJECT_NAME}}** repository (`{{GITHUB_OWNER}}/{{GITHUB_REPO}}`) against
software supply-chain attacks. You do two things:

1. **Establish guardrails** — apply preventive configuration (dependency pinning, install
   script controls, Claude Code permission boundaries, CI hardening) so a compromised or
   malicious dependency, action, or MCP server has a much smaller blast radius.
2. **Audit and triage** — scan what already exists for known-vulnerable or unpinned
   dependencies and file one GitHub issue per remaining risk so `/auto-issue-worker` can
   drive the fixes.

This is defensive, authorized work on the user's own project. Every change that touches a
config file or files an issue goes through the user's approval first.

Project tech stack (use it to pick the right package manager, registry, and CI specifics):

{{TECH_STACK}}

## Workflow

### Step 1 — Detect the stack and current posture

Identify what the project uses so later steps target the right tooling. Do **not** modify
anything yet.

- Detect package manager(s) and ecosystems from manifests and lockfiles:
  `package.json` + (`package-lock.json` | `pnpm-lock.yaml` | `yarn.lock` | `bun.lockb`),
  `pyproject.toml` / `requirements*.txt` + (`poetry.lock` | `uv.lock`), `Cargo.toml` +
  `Cargo.lock`, `go.mod` + `go.sum`, `Gemfile` + `Gemfile.lock`, etc.
- Note which lockfiles exist and whether they are committed (`git ls-files`).
- Detect CI: list `.github/workflows/*.yml` and any existing `dependabot.yml` /
  `renovate.json`.
- Inspect the Claude Code config: read `.claude/settings.json` and `.claude/settings.local.json`
  if present, and any configured MCP servers (`.mcp.json` / settings `mcpServers`).
- Summarize the current posture to the user before proposing changes.

### Step 2 — Layer 1: project dependency guardrails

Propose the applicable items below for the detected ecosystems, show the exact diffs, and
apply only what the user approves.

- **Commit and verify lockfiles.** Every ecosystem with a lockfile must have it committed.
  In CI and local installs, use frozen/locked installs so resolution cannot drift:
  `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable`,
  `pip install --require-hashes` (or `uv sync --frozen`), `cargo build --locked`,
  `go mod verify`.
- **Pin exact versions.** Discourage floating ranges for direct deps where practical
  (e.g. npm `.npmrc` with `save-exact=true`; document the policy for others).
- **Control install-time scripts.** Arbitrary `postinstall` scripts are a primary supply-chain
  vector. For npm/pnpm/yarn, propose `ignore-scripts=true` in `.npmrc` (or pnpm
  `onlyBuiltDependencies` allowlist) and document which packages legitimately need scripts.
- **Pin the registry.** Set the registry explicitly in `.npmrc` / `pip.conf` / `.cargo/config.toml`
  to prevent dependency-confusion via alternate registries. For private scopes, bind the
  scope to its registry.
- **Run an audit** appropriate to the stack and capture results for Step 5:
  `npm audit` / `pnpm audit` / `yarn npm audit`, `pip-audit`, `osv-scanner -r .`,
  `cargo audit`, `govulncheck ./...`, `bundle audit`. If a tool is not installed, note it
  and prefer `osv-scanner` as the cross-ecosystem fallback (or recommend installing it).

Example `.npmrc` hardening (confirm before writing):

```ini
save-exact=true
ignore-scripts=true
audit-level=moderate
registry=https://registry.npmjs.org/
```

### Step 3 — Layer 2: Claude Code environment guardrails

The agent itself can be a supply-chain vector: it may install packages, run untrusted code,
or talk to MCP servers. Tighten `.claude/settings.json` (project-scoped) with the user's
approval — never weaken existing rules without asking.

- **Permission boundaries.** Propose a `permissions` block that denies the high-risk,
  exfiltration- and remote-code-prone patterns and requires confirmation for installs. For
  example, deny piping remote scripts into a shell and reaching arbitrary hosts, and put
  package installs behind `ask`:

  ```json
  {
    "permissions": {
      "deny": [
        "Bash(curl:*| sh)",
        "Bash(curl:*| bash)",
        "Bash(wget:*| sh)",
        "Read(./.env)",
        "Read(./.env.*)",
        "Read(./**/*.pem)"
      ],
      "ask": [
        "Bash(npm install:*)",
        "Bash(pnpm add:*)",
        "Bash(pip install:*)",
        "Bash(cargo add:*)"
      ]
    }
  }
  ```

  Treat this as a starting point and tailor it to the project's package manager and secret
  file locations. Explain that deny/ask rules are matched as configured by the harness, so
  the exact tokens matter.
- **MCP trust boundary.** Review every configured MCP server with the user: who publishes
  it, what it can access, and whether it is pinned to a specific version/image rather than a
  floating tag. Recommend removing or sandboxing any server that is unvetted or grants broad
  filesystem/network access. Document the vetted set so additions are a conscious decision.
- **Agent/skill hygiene.** Confirm the project's agents and skills do not auto-install
  untrusted packages or execute fetched-at-runtime code without review. Flag any that do.

### Step 4 — Layer 3: CI / build pipeline guardrails

Harden GitHub Actions (or the project's CI) with the user's approval. Show diffs first.

- **Pin actions to a full commit SHA**, not a moving tag — `uses: actions/checkout@<40-char-sha>`
  with the human-readable version in a trailing comment. Floating tags (`@v4`, `@main`) let a
  compromised action push malicious code into your pipeline.
- **Least-privilege tokens.** Add a top-level `permissions: { contents: read }` and grant more
  only per-job where needed. Avoid `write-all`.
- **Don't leak the checkout token.** Set `persist-credentials: false` on `actions/checkout`
  unless the job genuinely needs to push.
- **Verify lockfiles in CI.** Use the frozen-install commands from Step 2 so CI fails if the
  lockfile and manifest disagree.
- **Automated dependency updates + review.** Propose `dependabot.yml` (or Renovate) so updates
  are reviewable PRs, and add `actions/dependency-review-action` on pull requests to block
  known-vulnerable or disallowed-license dependencies from entering.
- **Optional:** `step-security/harden-runner` for egress control, and OpenSSF Scorecard for an
  ongoing posture score. Offer these; do not force them.

### Step 5 — Triage findings and confirm with the user

Consolidate everything that the guardrails could not fix automatically — known
vulnerabilities from the audits, unpinned actions you could not safely rewrite, unvetted MCP
servers, missing lockfiles in ecosystems you do not control. For each:

- Assign a **severity** (Critical / High / Medium / Low) by impact and exploitability.
- De-duplicate items sharing one root cause into a single issue.
- Present a table and get approval on what to file before creating any issue:

| # | Title | Severity | Layer (dep / claude / ci) | Location | Fix |
|---|-------|----------|---------------------------|----------|-----|

- Do not create issues until the user approves the list.

### Step 6 — Create the issues

For each approved finding, create an issue with `gh`, keeping the body compatible with
`/auto-issue-worker`:

```bash
gh issue create \
  --repo {{GITHUB_OWNER}}/{{GITHUB_REPO}} \
  --title "[supply-chain] <concise risk title>" \
  --label "supply-chain" \
  --body "$(cat <<'EOF'
## Summary
<The supply-chain risk and its impact, in one or two sentences.>

## Severity
<Critical | High | Medium | Low> — <layer: dependency | claude-env | ci>

## Location
<file:line / manifest / workflow / MCP server affected>

## Details
<Why this is exploitable: e.g. vulnerable version + advisory ID, unpinned action, postinstall risk, unvetted MCP server.>

## Remediation
<Recommended fix — target version, SHA to pin to, config to add.>

## Acceptance Criteria
- [ ] The risk is remediated as described above
- [ ] Lockfiles / pins are committed and CI verifies them
- [ ] The fix does not regress existing functionality
- [ ] E2E behavior is verified and guaranteed at the completion of this task

## Dependencies
- <Depends on #<n>, or "None">
EOF
)"
```

Rules for issue creation:

- Apply a `supply-chain` label (create it with `gh label create supply-chain` if missing,
  after confirming with the user). Add a severity label too if the project uses them.
- Order issues by severity (Critical/High first) so they surface earlier to `/auto-issue-worker`.
- **Redact real secrets** — describe the location, never paste the value.
- Every issue's **Acceptance Criteria** must include
  "E2E behavior is verified and guaranteed at the completion of this task".

### Step 7 — Report

- Summarize the guardrails applied (which config files changed) and the issues filed
  (number/URL per finding).
- Note anything you recommended but the user declined, and any audit tool that was missing.
- Tell the user they can run `/auto-issue-worker` to drive the remaining fixes.

## Rules

- **Authorized, defensive scope only**: harden this repository and its tooling. Do not probe
  third-party systems.
- **Approve before applying**: show the exact diff for every config change and the full list
  before filing issues. Never weaken an existing security rule without explicit confirmation.
- **Non-destructive audits**: run audit/scan tools in read-only mode; do not auto-upgrade
  dependencies without approval.
- Use the same language as `spec.md` for issue titles and bodies.
- Do not commit changes unless the user explicitly asks.
- If `gh` fails (auth, permissions, missing repo), report the error clearly and stop.
