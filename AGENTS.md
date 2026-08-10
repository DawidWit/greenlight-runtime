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
