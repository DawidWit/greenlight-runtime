# Greenlight Runtime Design

**Status:** Approved for specification review  
**Date:** 2026-08-10  
**Repository:** `DawidWit/greenlight-runtime`

## Summary

Greenlight Runtime is a local-first control plane for AI coding agents. It lets an agent investigate a repository and prepare changes inside an isolated Git worktree while a deterministic policy engine controls which actions may proceed, which require human approval, and which are unavailable.

The product makes an agent run inspectable rather than opaque. A developer can watch a normalized event timeline, review requested commands, inspect the final diff and verification results, and either accept the prepared work for handoff or leave it isolated for later inspection.

The flagship value is the boundary between probabilistic AI behavior and deterministic authorization: model output can request an action, but it can never grant itself permission.

## Goals

- Run a single coding task in an isolated Git worktree.
- Stream a durable timeline of agent activity to a local web interface.
- Enforce approval decisions in Go code outside the model context.
- Produce a final evidence bundle containing the task, changes, commands, test results, errors, and approval decisions.
- Recover inspectable state after a process crash or cancellation.
- Support model providers through documented APIs and a provider-neutral interface.
- Provide a polished, understandable demonstration of Go, TypeScript, applied AI, event-driven design, safety boundaries, and testing.

## Non-goals for the MVP

- Multi-agent coordination.
- Remote or cloud execution.
- Team accounts, authentication, billing, or hosted persistence.
- Autonomous pushes, pull requests, releases, or other external mutations.
- A plugin marketplace or arbitrary third-party tools.
- Transparent wrapping or automation of undocumented consumer AI interfaces.
- A hardened operating-system sandbox. The MVP is a local developer tool whose trust boundary is enforced at the application and workspace layers.

## Core user journey

1. The developer starts Greenlight Runtime locally and selects an existing Git repository.
2. The developer describes a coding task and chooses a configured model provider.
3. Greenlight records the run as `queued`, creates a dedicated Git worktree, and transitions the run to `running` before agent execution begins.
4. The built-in coding agent reads and searches the worktree, proposes edits, and requests commands through typed tools.
5. Greenlight evaluates every tool request against deterministic policy.
6. Reads, searches, and edits contained within the worktree may proceed automatically. Shell commands pause until the developer approves or rejects the exact command.
7. Each request, decision, result, error, and state transition is appended to the run event log and streamed to the dashboard.
8. The developer reviews the final diff and verification evidence.
9. The developer marks the run accepted for manual handoff, cancels it, or leaves it preserved for later inspection.
10. Worktree deletion is always a separate, explicit action.

## Architecture

The repository is a monorepo with a Go service and a React/TypeScript web application.

```text
greenlight-runtime/
├── cmd/greenlightd/       Go application entry point
├── internal/
│   ├── runs/              Run state machine and orchestration
│   ├── agent/             Provider-neutral coding-agent loop
│   ├── providers/         Documented model API adapters
│   ├── tools/             Typed read, search, edit, and command tools
│   ├── policy/            Authorization and approval decisions
│   ├── workspace/         Git worktree lifecycle
│   ├── events/            Normalized event definitions and streaming
│   └── storage/           SQLite persistence
├── web/                   React and TypeScript dashboard
├── contracts/             API and event schemas
├── docs/                  Design and architecture documentation
├── .rulesync/rules/       Canonical coding-agent instructions
└── rulesync.jsonc         Rulesync targets and feature configuration
```

### Trusted service

The Go service is the trusted application boundary. It owns repository access, worktree creation, model calls, tool execution, authorization, persistence, redaction, and event publication. The web application never reads arbitrary filesystem paths or executes commands directly.

### Agent runtime

The MVP contains a small built-in coding-agent loop instead of attempting to supervise undocumented third-party agent internals. A provider adapter accepts normalized messages and tool definitions and returns normalized model responses. Provider credentials are supplied through process environment variables and are never persisted in SQLite or returned to the browser.

A deterministic fake provider implements the same interface for repeatable integration tests and the public demo.

### Tool boundary

The initial tool set is intentionally narrow:

- `read_file`: read a UTF-8 text file inside the worktree.
- `list_files`: list paths inside the worktree with bounded results.
- `search_text`: search text inside the worktree with bounded results.
- `apply_patch`: apply a structured patch inside the worktree.
- `run_command`: request a command with an explicit working directory, arguments, timeout, and output limit.

All paths are canonicalized and checked against the worktree root before use. Symbolic links that resolve outside the worktree are rejected. The model cannot register new tools at runtime.

### Policy engine

Policy receives a typed action request and returns one of three decisions:

- `allow`: safe worktree-contained reads, searches, and edits.
- `require_approval`: every shell command in the MVP.
- `deny`: access outside the worktree, unsupported tools, external publishing, and malformed requests.

Approval is bound to the exact normalized tool request. Editing the command, arguments, working directory, or relevant execution options invalidates the approval and creates a new request.

The model may explain why it wants an action, but model text is never an input that can override policy.

### Event flow

```text
Task submitted
  -> queued run recorded
  -> worktree created
  -> run marked running
  -> agent requests tool
  -> policy decides
       -> allow and execute
       -> pause for exact approval
       -> deny
  -> result appended as event
  -> event streamed to dashboard
  -> final diff and evidence produced
```

Server-Sent Events carry the live run timeline because the server is the primary event producer. Approval and cancellation use ordinary HTTP commands. A reconnecting client supplies the last observed event sequence and receives missed events from SQLite before resuming the live stream.

## Run states

A run has one of the following states:

- `queued`
- `running`
- `waiting_for_approval`
- `completed`
- `failed`
- `cancelled`
- `interrupted`

State transitions are validated by the run domain model and persisted before their corresponding external effect. A process restart marks previously active `running` or `waiting_for_approval` runs as `interrupted`; the MVP does not silently resume model execution.

Cancellation stops further agent steps and attempts to terminate an active child command. The worktree, existing events, command output, and diff remain available.

## Persistence

SQLite stores:

- Registered project metadata and canonical local paths.
- Runs, current states, and timestamps.
- Ordered events with monotonic per-run sequence numbers.
- Tool requests and redacted results.
- Approval requests and human decisions.
- Verification results and artifact references.

Large command logs and generated artifacts may be stored as bounded files in Greenlight's application data directory, referenced from SQLite. Provider credentials and raw environment variables are never persisted.

Events are append-only. Mutable summary tables may be rebuilt from the event log where practical, but the MVP does not aim to implement a general-purpose event-sourcing framework.

## Local API

The initial API exposes only the operations required by the dashboard:

- Register and list local projects.
- Create, inspect, cancel, and list runs.
- Stream ordered run events.
- Approve or reject an exact pending tool request.
- Read a run's final diff and evidence summary.
- Explicitly delete a finished or interrupted worktree.

API and event payloads are described by versioned schemas in `contracts/`. TypeScript types are generated from those schemas so the frontend does not maintain handwritten copies of backend contracts.

## Dashboard

The dashboard has three primary views:

1. **New run:** repository selection, task entry, and provider selection.
2. **Run timeline:** current state, ordered events, pending approvals, bounded command output, and cancellation.
3. **Review:** changed files, unified diff, verification results, errors, approval history, and handoff status.

The dashboard labels AI-generated explanations as model output. Policy decisions, execution results, and test outcomes have visually distinct system provenance.

## Failure handling

- Invalid paths and malformed tool requests are denied and recorded without execution.
- Provider failures use bounded retries only for transient errors; exhausted retries fail the run with the provider error redacted for secrets.
- Command execution has a mandatory timeout and output cap. Timeout or truncation is explicit in the event record.
- A storage failure stops the run before another tool is executed because the durable audit trail is part of the safety boundary.
- Event-stream disconnection does not affect the run; the browser catches up from persisted sequence numbers.
- Worktree creation failure leaves the source repository untouched and fails the run before model execution.
- Cleanup failure preserves the worktree record and surfaces a retryable cleanup action.

## Security and provider compliance

- Greenlight uses official, documented model APIs with user-provided credentials.
- Credentials remain server-side and are sourced from the local process environment.
- The agent cannot make arbitrary network requests through its tool set in the MVP. Network access used by configured model providers is owned by the trusted service.
- Common credential formats are redacted before events or output are persisted.
- Filesystem operations reject traversal and escapes from the canonical worktree root.
- External Git mutations, pull requests, and releases are unavailable in the MVP.
- The product documentation states that a Git worktree is not a complete OS security sandbox and that users must review approved shell commands.
- Greenlight does not bypass model safeguards, automate undocumented consumer authentication, or claim compatibility beyond documented provider APIs.

## Verification strategy

### Go tests

- Table-driven unit tests cover valid and invalid run-state transitions.
- Policy tests prove that model text cannot change authorization outcomes.
- Path-boundary tests cover traversal, absolute paths, and symbolic-link escapes.
- Workspace integration tests use temporary Git repositories and verify that the source checkout remains unchanged.
- Agent integration tests use the fake provider to exercise approval, rejection, cancellation, provider failure, and completion.
- Storage tests verify ordered event replay and crash-state recovery.

### Contract and frontend tests

- Contract tests verify that generated TypeScript types match versioned backend schemas.
- Component tests cover pending approvals, run-state presentation, timeline catch-up, and review evidence.
- One browser-level test exercises the complete fake-provider path from task creation through approval to final review.

### Continuous integration

CI runs Go formatting and static analysis, Go tests, TypeScript formatting and linting, type checking, frontend tests, the browser smoke test, schema-generation drift checks, and rulesync drift checks.

## Rulesync configuration

The repository uses the `dyoshikawa/rulesync` Node CLI already used in the owner's other work. It is pinned as a development dependency and configured with:

- Targets: `claudecode` and `codexcli`.
- Feature: `rules`.
- Canonical sources in `.rulesync/rules/`.
- Generated `CLAUDE.md` and `AGENTS.md` committed to the repository.
- A check command in CI that fails when generated files drift from their canonical sources.

Canonical rule files are:

```text
.rulesync/rules/
├── 00-project.md
├── 10-architecture.md
├── 20-go.md
├── 30-typescript.md
├── 40-security.md
└── 50-testing.md
```

`00-project.md` contains the stable project description, goals, non-goals, vocabulary, and trust model. The remaining files add focused rules without duplicating the project description.

## Repository presentation

The repository is public under the MIT license. Its GitHub description is:

> Local-first control plane for AI coding agents with isolated workspaces, approval gates, and auditable evidence.

The initial README will lead with a short product demonstration, an architecture diagram, the safety model, local setup, and an explicit project-status section. It will avoid claiming production-grade sandboxing or provider support that has not been verified.

## MVP acceptance criteria

The MVP is complete when all of the following are demonstrated from a clean local checkout:

1. A developer can register a temporary Git repository and submit a task.
2. Greenlight creates a separate worktree without changing the source checkout.
3. The fake provider can read, search, and edit a file through typed tools.
4. A requested shell command pauses the run until the exact request is approved or rejected.
5. Events remain ordered and can be replayed after reconnecting the dashboard.
6. Cancellation or process interruption preserves the run, worktree, and evidence.
7. Final review shows the task, diff, tool activity, approval history, and verification result.
8. Worktree deletion requires an explicit user action.
9. Unit, integration, contract, frontend, and browser smoke tests pass in CI.
10. Rulesync generation and drift checking pass in CI.
