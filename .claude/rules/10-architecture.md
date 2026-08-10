---
paths:
  - '**/*'
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
