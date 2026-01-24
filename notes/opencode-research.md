# OpenCode research notes

## Summary
- The OpenCode CLI supports continuing or resuming prior work via `--continue` (continue the last session) and `--session` (continue a specific session ID). It also exposes a `session` command for listing sessions, plus `export` and `import` for moving sessions between machines or URLs.
- The CLI can attach a terminal to a running OpenCode backend started with `serve` or `web`, enabling long-running servers and reconnectable sessions. The `--attach` flag lets commands connect to a running server, and the `serve` and `web` commands start a headless server or server with a web UI.
- The CLI includes a `--dir` flag to start the TUI in a chosen working directory, which is useful for isolating workspaces or sandbox-style workflows. It also supports `OPENCODE_SERVER_PASSWORD` to enable basic auth on `serve`/`web` servers.

## Commands used
- `curl -s https://opencode.ai/docs/cli/ | rg -n "session|serve|web|attach|continue|export|import|dir|OPENCODE_SERVER_PASSWORD" -i`

## Sources
- https://opencode.ai/docs/cli/
