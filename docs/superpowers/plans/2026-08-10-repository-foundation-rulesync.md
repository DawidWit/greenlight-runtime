# Repository Foundation and Rulesync Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Turn the design-only public repository into a polished project foundation with pinned Rulesync tooling, canonical project instructions, generated Codex and Claude rules, drift-checking CI, and recruitment-ready presentation.

**Architecture:** A private root `package.json` owns repository tooling without making the Go runtime a Node application. Canonical Markdown under `.rulesync/rules/` is the only hand-edited source for coding-agent instructions; Rulesync 16.9.1 generates `AGENTS.md`, `CLAUDE.md`, and Claude's scoped rule files. CI installs the pinned toolchain and fails when generated instructions drift.

**Tech Stack:** pnpm 11.20.0, Rulesync 16.9.1, GitHub Actions, Markdown, JSONC

## Global Constraints

- The repository remains public at `DawidWit/greenlight-runtime` under the MIT license.
- The GitHub description remains: `Local-first control plane for AI coding agents with isolated workspaces, approval gates, and auditable evidence.`
- Rulesync targets are `claudecode` and `codexcli`; the only enabled feature is `rules`.
- Generated coding-agent files are committed, but `.rulesync/rules/` remains their canonical source.
- AI output never grants itself permission; deterministic Go policy will own authorization.
- The MVP is local-first and does not claim hardened OS sandboxing, multi-agent support, remote execution, or autonomous publishing.
- Provider integrations use official documented APIs and user-provided credentials only.
- Do not scaffold runtime application code in this plan; the Go service and React dashboard receive separate implementation plans.

---

## Planned file structure

```text
.
├── .github/workflows/rulesync.yml       Rulesync drift CI
├── .rulesync/rules/
│   ├── 00-project.md                    Root description and invariants
│   ├── 10-architecture.md               Module boundaries and dependency direction
│   ├── 20-go.md                         Go-specific conventions
│   ├── 30-typescript.md                 Web-specific conventions
│   ├── 40-security.md                   Trust-boundary and secret-handling rules
│   └── 50-testing.md                    Verification expectations
├── .claude/rules/*.md                   Generated scoped Claude rules
├── AGENTS.md                            Generated Codex rules
├── CLAUDE.md                            Generated Claude root rules
├── LICENSE                              MIT license
├── README.md                            Recruitment-oriented project overview
├── package.json                         Pinned repository tooling and scripts
├── pnpm-lock.yaml                       Reproducible tooling resolution
└── rulesync.jsonc                       Rulesync targets and features
```

### Task 1: Pin repository tooling

**Files:**
- Create: `package.json`
- Create: `pnpm-lock.yaml` via `pnpm install`
- Create: `.gitignore`

**Interfaces:**
- Consumes: pnpm 11.20.0 installed on the development machine.
- Produces: `pnpm rules:generate`, `pnpm rules:check`, and `pnpm rules:doctor`; later tasks and CI call these exact scripts.

- [ ] **Step 1: Create the root tooling manifest**

Create `package.json` with exactly:

```json
{
  "name": "greenlight-runtime",
  "version": "0.0.0",
  "private": true,
  "description": "Local-first control plane for AI coding agents with isolated workspaces, approval gates, and auditable evidence.",
  "license": "MIT",
  "packageManager": "pnpm@11.20.0",
  "scripts": {
    "rules:generate": "rulesync generate",
    "rules:check": "rulesync generate --check",
    "rules:doctor": "rulesync doctor"
  },
  "devDependencies": {
    "rulesync": "16.9.1"
  }
}
```

- [ ] **Step 2: Create repository ignores**

Create `.gitignore` with:

```gitignore
node_modules/
coverage/
dist/

.env
.env.*
!.env.example

*.db
*.db-shm
*.db-wal

rulesync.local.jsonc
.DS_Store
```

Do not ignore `AGENTS.md`, `CLAUDE.md`, `.claude/rules/`, or other Rulesync-generated project instructions because drift checking requires committed outputs.

- [ ] **Step 3: Install the pinned dependency**

Run:

```bash
pnpm install
```

Expected: `pnpm-lock.yaml` is created and `rulesync` resolves to exactly `16.9.1`.

- [ ] **Step 4: Verify the tool version**

Run:

```bash
pnpm exec rulesync --version
```

Expected output: `16.9.1`.

- [ ] **Step 5: Commit the tooling foundation**

```bash
git add package.json pnpm-lock.yaml .gitignore
git commit -m "chore: pin repository tooling"
```

### Task 2: Define canonical project instructions

**Files:**
- Create: `rulesync.jsonc`
- Create: `.rulesync/rules/00-project.md`
- Create: `.rulesync/rules/10-architecture.md`
- Create: `.rulesync/rules/20-go.md`
- Create: `.rulesync/rules/30-typescript.md`
- Create: `.rulesync/rules/40-security.md`
- Create: `.rulesync/rules/50-testing.md`

**Interfaces:**
- Consumes: Rulesync scripts from Task 1 and the approved design at `docs/superpowers/specs/2026-08-10-greenlight-runtime-design.md`.
- Produces: one root project rule plus five focused rules from which Task 3 generates tool-native instructions.

- [ ] **Step 1: Configure Rulesync**

Create `rulesync.jsonc`:

```jsonc
{
  "$schema": "https://github.com/dyoshikawa/rulesync/releases/latest/download/config-schema.json",
  "targets": ["claudecode", "codexcli"],
  "features": ["rules"],
  "delete": true
}
```

- [ ] **Step 2: Write the root project description**

Create `.rulesync/rules/00-project.md`:

```markdown
---
root: true
targets:
  - "*"
description: "Greenlight Runtime project purpose, vocabulary, and non-negotiable product boundaries"
globs:
  - "**/*"
---

# Greenlight Runtime

Greenlight Runtime is a local-first control plane for AI coding agents. It runs a coding task in an isolated Git worktree, records a durable event timeline, requires human approval for sensitive actions, and presents the final diff and verification evidence for review.

## Product invariants

- Model output may request an action but can never authorize it.
- Deterministic Go policy decides whether a tool request is allowed, requires approval, or is denied.
- The Go service is the trusted boundary; the web application never accesses arbitrary filesystem paths or executes commands directly.
- Every run remains inspectable after failure, cancellation, or interruption.
- Worktree deletion is always an explicit user action.
- Provider credentials remain server-side and are never persisted or returned to the browser.
- Use official documented model APIs only; do not automate undocumented consumer authentication.
- Do not claim hardened OS sandboxing, remote execution, multi-agent support, or autonomous publishing unless those capabilities are implemented and verified.

## Vocabulary

- **Project:** a registered local Git repository.
- **Run:** one coding task and its complete lifecycle.
- **Workspace:** the isolated Git worktree created for a run.
- **Tool request:** a typed action requested by the model.
- **Approval:** a human decision bound to one exact normalized tool request.
- **Event:** an append-only record of a state transition, request, decision, result, or error.
- **Evidence bundle:** the task, final diff, tool activity, approval history, verification results, and errors shown at review.

## Repository workflow

- Edit canonical agent instructions only in `.rulesync/rules/`.
- Regenerate tool-native instructions with `pnpm rules:generate`.
- Verify generated instructions with `pnpm rules:check` and `pnpm rules:doctor`.
- Keep the approved design in `docs/superpowers/specs/2026-08-10-greenlight-runtime-design.md` consistent with architectural changes.
```

- [ ] **Step 3: Write architecture boundaries**

Create `.rulesync/rules/10-architecture.md`:

```markdown
---
root: false
targets:
  - "*"
description: "Greenlight Runtime architecture and dependency boundaries"
globs:
  - "**/*"
---

# Architecture

- `cmd/greenlightd/` contains composition and process startup only.
- `internal/runs/` owns the run state machine and orchestration.
- `internal/agent/` owns the provider-neutral agent loop.
- `internal/providers/` adapts documented model APIs to agent interfaces.
- `internal/tools/` defines typed tool requests and execution results.
- `internal/policy/` is the only module that returns allow, require-approval, or deny decisions.
- `internal/workspace/` owns Git worktree creation, containment, and cleanup.
- `internal/events/` owns normalized event contracts and publication.
- `internal/storage/` persists projects, runs, events, approvals, and artifact references in SQLite.
- `contracts/` is the source for versioned HTTP and event schemas; generate TypeScript types rather than maintaining handwritten copies.
- `web/` communicates with the Go service through the local API and never accesses repositories directly.

Keep dependency direction toward domain interfaces. Filesystem, Git, process execution, SQLite, HTTP, and model providers stay behind narrow adapters. Do not import a concrete adapter into domain logic.

Prefer small files with one responsibility. A package should be understandable through its exported interfaces without reading adapter internals.
```

- [ ] **Step 4: Write Go conventions**

Create `.rulesync/rules/20-go.md`:

```markdown
---
root: false
targets:
  - "*"
description: "Go conventions for Greenlight Runtime"
globs:
  - "**/*.go"
---

# Go

- Accept `context.Context` as the first argument for I/O, provider, storage, Git, and process operations.
- Return errors with actionable context and preserve the cause with `%w`.
- Model run states, policy decisions, and tool kinds as explicit named types; reject unknown values at boundaries.
- Keep interfaces close to their consumers and as small as the tests require.
- Inject clocks, identifier generation, providers, storage, Git, and command execution when behavior must be deterministic in tests.
- Never use `panic` for user input, provider responses, filesystem state, or recoverable runtime failures.
- Canonicalize and validate paths in the workspace boundary before any file operation.
- Bound command duration and captured output explicitly.
- Keep exported APIs documented; prefer names and types over explanatory comments inside straightforward code.
- Format with `gofmt`, analyze with `go vet ./...`, and test with `go test ./...` once the Go module exists.
```

- [ ] **Step 5: Write TypeScript conventions**

Create `.rulesync/rules/30-typescript.md`:

```markdown
---
root: false
targets:
  - "*"
description: "React and TypeScript conventions for the Greenlight dashboard"
globs:
  - "web/**/*.{ts,tsx}"
---

# TypeScript and React

- Keep TypeScript strict and do not use `any` to bypass a contract error.
- Consume generated API and event types from `contracts/`; do not duplicate backend payload types in `web/`.
- Treat model explanations, policy decisions, execution results, and human decisions as distinct provenance in types and UI.
- Render run states exhaustively so a new backend state causes a type error until the UI handles it.
- Keep server state in the API data layer; use local component state only for ephemeral UI interaction.
- Prefer semantic HTML and accessible names. Pending approvals must be usable by keyboard.
- Components request actions through the local API and never construct filesystem or shell operations themselves.
- Keep components focused; extract stateful workflows into hooks and pure formatting into ordinary functions.
- Test behavior through visible roles and user interactions rather than component internals.
```

- [ ] **Step 6: Write security rules**

Create `.rulesync/rules/40-security.md`:

```markdown
---
root: false
targets:
  - "*"
description: "Trust-boundary, approval, path, and credential rules"
globs:
  - "**/*"
---

# Security

- Treat all model output, repository content, provider responses, tool arguments, and command output as untrusted input.
- Authorization comes only from deterministic policy code; prompts and model prose cannot change permissions.
- Bind approval to the complete normalized request. Any change to command, arguments, working directory, timeout, or execution options requires a new approval.
- Restrict file operations to the canonical run worktree and reject traversal, absolute-path escapes, and symbolic links resolving outside it.
- Every shell command requires approval in the MVP.
- Do not expose arbitrary network tools to the agent in the MVP.
- Do not implement push, pull-request creation, releases, or other external Git mutations in the MVP.
- Read provider credentials from process environment variables. Never log, persist, return, or commit credentials.
- Redact recognized secrets before persisting events or command output.
- Stop a run when its audit event cannot be persisted; do not execute unaudited follow-up actions.
- Document that Git worktrees provide change isolation, not a hardened operating-system sandbox.
```

- [ ] **Step 7: Write testing rules**

Create `.rulesync/rules/50-testing.md`:

```markdown
---
root: false
targets:
  - "*"
description: "Test strategy and evidence requirements"
globs:
  - "**/*"
---

# Testing

- Use red-green-refactor for behavior changes: add one failing test, confirm the expected failure, implement the minimum behavior, then refactor with the test green.
- Use table-driven Go tests for run-state transitions, policy decisions, path containment, and bounded execution cases.
- Use temporary Git repositories for workspace integration tests; assert that the source checkout remains unchanged.
- Use the deterministic fake provider for agent integration and browser smoke tests. CI must not require paid model credentials.
- Test approval and rejection against the exact normalized tool request.
- Test interruption, cancellation, timeout, output truncation, provider failure, event replay, and cleanup failure explicitly.
- Frontend tests assert accessible behavior and visible provenance, not implementation details.
- Contract drift and Rulesync drift are test failures.
- Never weaken or delete a failing safety test merely to make CI pass; fix the violated invariant.
```

- [ ] **Step 8: Validate source formatting and configuration**

Run:

```bash
pnpm rules:doctor
```

Expected: Rulesync recognizes six rule sources and reports no configuration errors. Missing generated outputs may still be reported before Task 3.

- [ ] **Step 9: Commit canonical rules**

```bash
git add rulesync.jsonc .rulesync/rules
git commit -m "docs: define canonical project rules"
```

### Task 3: Generate and verify tool-native instructions

**Files:**
- Create: `AGENTS.md` via Rulesync
- Create: `CLAUDE.md` via Rulesync
- Create: `.claude/rules/10-architecture.md` via Rulesync
- Create: `.claude/rules/20-go.md` via Rulesync
- Create: `.claude/rules/30-typescript.md` via Rulesync
- Create: `.claude/rules/40-security.md` via Rulesync
- Create: `.claude/rules/50-testing.md` via Rulesync

**Interfaces:**
- Consumes: `rulesync.jsonc`, `.rulesync/rules/*.md`, and scripts from Tasks 1-2.
- Produces: committed native instructions read by Codex and Claude Code plus a reproducible drift check for Task 4.

- [ ] **Step 1: Confirm drift before generation**

Run:

```bash
pnpm rules:check
```

Expected: exit code 1 because generated project instructions do not exist yet.

- [ ] **Step 2: Generate native instructions**

Run:

```bash
pnpm rules:generate
```

Expected: Rulesync creates `AGENTS.md`, `CLAUDE.md`, and the five focused files under `.claude/rules/`.

- [ ] **Step 3: Inspect the generated root files**

Run:

```bash
sed -n '1,220p' AGENTS.md
sed -n '1,160p' CLAUDE.md
```

Expected: both identify Greenlight Runtime as a local-first control plane. `AGENTS.md` contains the composed project rules, and `CLAUDE.md` contains the root project description while referring Claude Code to its scoped rule files through native discovery.

- [ ] **Step 4: Verify idempotence and health**

Run:

```bash
pnpm rules:check
pnpm rules:doctor
git diff --check
```

Expected: all commands exit 0 and a second `pnpm rules:generate` produces no diff.

- [ ] **Step 5: Commit generated instructions**

```bash
git add AGENTS.md CLAUDE.md .claude/rules
git commit -m "chore: generate agent instructions"
```

### Task 4: Add recruitment presentation and drift CI

**Files:**
- Create: `README.md`
- Create: `LICENSE`
- Create: `.github/workflows/rulesync.yml`

**Interfaces:**
- Consumes: repository scripts from Task 1, generated instructions from Task 3, and the approved design specification.
- Produces: a public landing page, explicit license, and CI enforcement for the canonical/generated rule relationship.

- [ ] **Step 1: Write the repository landing page**

Create `README.md`:

```markdown
# Greenlight Runtime

> A local-first control plane for AI coding agents with isolated workspaces, approval gates, and auditable evidence.

Greenlight Runtime is an early-stage developer tool for making AI-assisted code changes inspectable. An agent works in an isolated Git worktree, requests sensitive actions through deterministic policy, and produces a review bundle containing its diff, commands, approvals, tests, and failures.

## Why this exists

Coding agents can produce useful changes, but their execution is often difficult to inspect and their authorization boundaries are easy to blur. Greenlight separates those concerns:

- AI proposes actions.
- Go policy authorizes, pauses, or denies them.
- Humans approve exact sensitive requests.
- An append-only timeline preserves the evidence.

The model can request permission; it can never grant permission to itself.

## Planned MVP

1. Select a local Git repository and submit one coding task.
2. Create an isolated worktree for the run.
3. Stream normalized agent and tool events to a React dashboard.
4. Pause every shell command for exact human approval.
5. Review the final diff, verification results, errors, and approval history.
6. Preserve interrupted work and delete worktrees only through an explicit action.

## Architecture

```text
Task -> Worktree -> Agent -> Tool request -> Go policy
                                          |-> allow
                                          |-> approval
                                          `-> deny

Every transition -> SQLite event log -> SSE timeline -> Review evidence
```

The trusted Go service owns model calls, policy, Git, filesystem access, command execution, persistence, and redaction. The TypeScript web application is an interface over the local HTTP and event APIs; it never touches repositories directly.

See the [approved design](docs/superpowers/specs/2026-08-10-greenlight-runtime-design.md) for the complete product boundary and acceptance criteria.

## Project status

Greenlight Runtime is in the design and repository-foundation stage. The README describes the approved target architecture, not a production-ready release. In particular, Git worktrees provide change isolation but are not a hardened operating-system sandbox.

## Development rules

Canonical coding-agent instructions live in `.rulesync/rules/` and generate tool-native files for Codex and Claude Code.

```bash
pnpm install
pnpm rules:generate
pnpm rules:check
pnpm rules:doctor
```

## License

MIT
```

- [ ] **Step 2: Add the MIT license**

Create `LICENSE`:

```text
MIT License

Copyright (c) 2026 Dawid Witczak

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 3: Add Rulesync drift CI**

Create `.github/workflows/rulesync.yml`:

```yaml
name: Rulesync

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  check:
    name: Check generated agent rules
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up pnpm
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Diagnose Rulesync configuration
        run: pnpm rules:doctor

      - name: Check generated files
        run: pnpm rules:check
```

- [ ] **Step 4: Run repository verification**

Run:

```bash
pnpm install --frozen-lockfile
pnpm rules:doctor
pnpm rules:check
git diff --check
git status --short
```

Expected: dependency installation and both Rulesync checks exit 0; `git diff --check` reports no whitespace errors; status lists only `README.md`, `LICENSE`, and `.github/workflows/rulesync.yml` before staging.

- [ ] **Step 5: Commit presentation and CI**

```bash
git add README.md LICENSE .github/workflows/rulesync.yml
git commit -m "docs: add project presentation and rules CI"
```

### Task 5: Verify and publish the repository foundation

**Files:**
- Verify only; no new files expected.

**Interfaces:**
- Consumes: all deliverables from Tasks 1-4.
- Produces: a clean, pushed `main` branch whose public landing page and CI reflect the approved design.

- [ ] **Step 1: Run the complete local verification suite**

```bash
pnpm exec rulesync --version
pnpm rules:doctor
pnpm rules:check
git diff --check
git status --short --branch
```

Expected: Rulesync reports `16.9.1`; all checks exit 0; the worktree is clean and on `main` ahead of `origin/main` by the new commits.

- [ ] **Step 2: Inspect the commit series**

```bash
git log --oneline --decorate -5
```

Expected newest commits, in order:

```text
docs: add project presentation and rules CI
chore: generate agent instructions
docs: define canonical project rules
chore: pin repository tooling
docs: add Greenlight Runtime design
```

- [ ] **Step 3: Push the verified commits**

```bash
git push origin main
```

Expected: `main` is updated on `https://github.com/DawidWit/greenlight-runtime`.

- [ ] **Step 4: Verify the public repository and workflow**

Using `gh-axi`, inspect repository metadata and the newest workflow run:

```bash
gh-axi repo view --repo=DawidWit/greenlight-runtime
gh-axi run list --repo=DawidWit/greenlight-runtime
```

Expected: the repository is public with the approved description, and the `Rulesync` workflow completes successfully for the pushed `main` commit.

