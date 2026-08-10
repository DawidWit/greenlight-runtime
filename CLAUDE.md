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
