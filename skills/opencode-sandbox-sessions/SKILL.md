---
name: opencode-sandbox-sessions
description: Run OpenCode in sandbox-like workspaces with resumable, long-running sessions. Use when you need durable OpenCode servers, session continuation, or isolated working directories for agents.
---
# OpenCode sandbox and resumable sessions

## Scope
- Keep OpenCode running in long-lived servers and reconnect later.
- Continue or attach to existing sessions using CLI flags.
- Isolate work into dedicated directories (sandbox-style workspaces).

## Workflow
1. Decide whether to run the OpenCode TUI directly or a headless server (`serve`) for long-running sessions.
2. Start OpenCode in a dedicated working directory using `--dir` when you need per-project isolation.
3. For long-lived sessions, run `opencode serve` (or `opencode web`) in a persistent terminal session (for example, `tmux`) so the server survives disconnects.
4. Reattach to the running server using `opencode attach` or CLI commands with `--attach`.
5. Continue a session with `--continue` for the most recent session or `--session` for a specific ID.
6. Use `opencode session list` to discover session IDs, and `opencode export`/`opencode import` to move sessions across machines or restore work.
7. If the server is shared or exposed, set `OPENCODE_SERVER_PASSWORD` for basic auth on `serve`/`web`.

## Outputs
- A documented runbook for keeping OpenCode sessions alive and reconnectable.
- A standard command set for listing, continuing, exporting, and importing sessions.
- Clear guidance on when to use dedicated working directories for sandbox-style isolation.
