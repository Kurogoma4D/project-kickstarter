---
name: spec-builder
description: |
  Interview the user to gather project requirements and consolidate them into a
  structured `spec.md` at the repository root. Use when starting a new project or
  feature and the requirements are not yet written down. Invoke with `/spec-builder`.
  Produces the input consumed by the `spec-to-issues` skill.
allowed-tools:
  - Read
  - Write
  - AskUserQuestion
---

# Spec Builder

You gather requirements for the **{{PROJECT_NAME}}** project through a structured
interview and consolidate them into a single `spec.md` file at the repository root.
The resulting `spec.md` is the source of truth for the `spec-to-issues` skill, which
breaks it into GitHub issues.

## Workflow

### Step 1 — Check for an existing spec

- Look for `spec.md` at the repository root.
- If it exists, read it and ask the user whether to **update** it or **start over**.
- If updating, treat the existing content as the baseline and only fill gaps / apply changes.

### Step 2 — Interview the user

Conduct the interview using the `AskUserQuestion` tool. Ask in focused rounds rather
than dumping every question at once. Cover the following areas, skipping any the user
has already answered:

1. **Overview** — What is being built and why? Who are the users? What problem does it solve?
2. **Goals & success criteria** — What does "done" look like? How is success measured?
3. **Scope** — Core features in scope for this iteration.
4. **Out of scope** — Explicitly what will *not* be built now (prevents scope creep).
5. **Functional requirements** — Concrete behaviors, user flows, inputs/outputs.
6. **Non-functional requirements** — Performance, security, accessibility, scalability, i18n.
7. **Supply-chain security** — How dependencies, the build pipeline, and tooling are kept
   trustworthy (see below). Capture this so `/supply-chain-guard` can enforce it later.
8. **Constraints** — Tech stack, platforms, deadlines, dependencies, existing systems.
9. **Open questions / risks** — Unknowns to resolve before or during implementation.

When covering **Supply-chain security**, gather the policy the project wants to hold itself
to so it can be set up during environment construction rather than bolted on later:

- Package manager and lockfile policy (committed lockfiles, frozen/locked installs,
  exact-version pinning).
- Install-time script policy (whether `postinstall`-style scripts are allowed, and for which
  packages) and the allowed package registries (to prevent dependency confusion).
- Which audit/scanning tools run and at what severity gate (e.g. `npm audit`, `pip-audit`,
  `osv-scanner`, `cargo audit`, `govulncheck`).
- CI hardening expectations (SHA-pinned actions, least-privilege tokens,
  Dependabot/Renovate).
- MCP servers and other agent tooling that are trusted, and how new ones get vetted.

If the user has no policy yet, propose sensible defaults (commit lockfiles, frozen installs,
pin actions to SHA, run an audit in CI) and record what they accept.

When covering **Overview** and **Constraints**, also gather the project metadata that the
`template-setup` skill needs to fill the `.claude/` placeholders, so the same interview
serves both purposes. Capture concretely:

- Project name and a one-sentence description (→ `PROJECT_NAME`, `PROJECT_DESCRIPTION`).
- GitHub owner/org and repository name, if known (→ `GITHUB_OWNER`, `GITHUB_REPO`).
- Directory / module structure of the codebase (→ `PROJECT_STRUCTURE`).
- Language and version, key libraries/frameworks (→ `LANGUAGE_VERSION_NOTE`,
  `KEY_DEPENDENCIES`, `TECH_STACK`).
- Build / lint / format / test commands run before a PR (→ `QA_COMMANDS`).

Record these in the **Constraints** section so `template-setup` can read them back later.

Rules for the interview:

- Ask one area at a time and adapt follow-ups to the answers.
- When the user is vague, propose concrete options or sensible defaults and confirm.
- Do not invent requirements the user did not state — record genuine unknowns under "Open questions".
- Stop interviewing once each section has enough detail to be actionable.

### Step 3 — Write `spec.md`

Write the gathered requirements to `spec.md` at the repository root using the template below.
Omit sections that genuinely do not apply, but keep the heading order.

```markdown
# {{PROJECT_NAME}} — Specification

> Status: Draft · Last updated: <YYYY-MM-DD>

## 1. Overview
<What is being built, for whom, and the problem it solves.>

## 2. Goals & Success Criteria
- <Goal / measurable success criterion>

## 3. Scope
- <In-scope feature>

## 4. Out of Scope
- <Explicitly excluded item>

## 5. Functional Requirements
### FR-1: <Title>
- <Behavior, user flow, inputs/outputs, acceptance criteria>

## 6. Non-Functional Requirements
- <Performance / security / accessibility / scalability / ...>

## 7. Supply-Chain Security
- Dependencies: <committed lockfiles, frozen/locked installs, exact-version pinning>
- Install scripts & registries: <postinstall policy, allowed registries>
- Auditing: <tools and severity gate, e.g. osv-scanner / npm audit in CI>
- CI hardening: <SHA-pinned actions, least-privilege tokens, Dependabot/Renovate>
- Trusted tooling: <vetted MCP servers / agents and how new ones are approved>

## 8. Constraints
### Project metadata
- Repository: `<github-owner>/<github-repo>` (if known)
- Structure: <directory / module layout>

### Tech stack
- Language: <language and version>
- Frameworks / key dependencies: <list>
- Tooling (QA commands): <build / lint / format / test commands>

### Other constraints
- <Platform, deadline, existing system, ...>

## 9. Open Questions & Risks
- <Unresolved question or risk>
```

Number functional requirements (`FR-1`, `FR-2`, …) so the `spec-to-issues` skill can
reference them when creating issues.

### Step 4 — Review with the user

- Show a summary of what was written and the path to `spec.md`.
- Ask the user to confirm or request changes; iterate until they approve.
- Once approved, tell the user they can run `/spec-to-issues` to break the spec into GitHub issues.

## Rules

- Write `spec.md` in the same language the user used during the interview.
- Keep the spec concise and actionable — it feeds an automated issue pipeline, not a marketing doc.
- Never start implementation; this skill only produces the specification.
- Always confirm the final content with the user before considering the task complete.
