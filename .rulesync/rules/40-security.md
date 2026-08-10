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
