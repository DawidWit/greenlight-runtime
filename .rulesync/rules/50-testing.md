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
