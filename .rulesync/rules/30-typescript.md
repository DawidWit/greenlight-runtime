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
