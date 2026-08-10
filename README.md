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

The architecture and safety boundaries above define the current product scope.

## Project status

Greenlight Runtime is in the design and repository-foundation stage. The README describes the approved target architecture, not a production-ready release. In particular, Git worktrees provide change isolation but are not a hardened operating-system sandbox.

## Development rules

Canonical coding-agent instructions live in `.rulesync/rules/` and generate tool-native files for Codex and Claude Code.

Requires Node.js 24 and pnpm 11.20.0.

```bash
pnpm install
pnpm rules:generate
pnpm rules:check
pnpm rules:doctor
```

## License

MIT
