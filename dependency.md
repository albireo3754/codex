# Codex Frontend Dependency Graph

## Data Source
- `cargo tree -p codex-cli --depth 2`
- `cargo tree -p codex-exec --depth 2`
- `cargo tree -p codex-tui --depth 2`

## High-Level Relationships

```mermaid
graph TD
    codex-cli["codex-cli (bin)"]
    codex-exec["codex-exec (bin/lib)"]
    codex-tui["codex-tui (lib)"]
    codex-core["codex-core"]
    codex-common["codex-common"]
    codex-arg0["codex-arg0"]
    codex-protocol["codex-protocol"]
    codex-ollama["codex-ollama"]
    codex-app-server["codex-app-server"]
    codex-login["codex-login"]

    codex-cli --> codex-tui
    codex-cli --> codex-exec
    codex-cli --> codex-core
    codex-cli --> codex-common
    codex-cli --> codex-arg0
    codex-cli --> codex-app-server
    codex-cli --> codex-login

    codex-exec --> codex-core
    codex-exec --> codex-common
    codex-exec --> codex-arg0
    codex-exec --> codex-protocol
    codex-exec --> codex-ollama

    codex-tui --> codex-core
    codex-tui --> codex-common
    codex-tui --> codex-arg0
    codex-tui --> codex-protocol
    codex-tui --> codex-ollama
    codex-tui --> codex-login

    codex-common --> codex-core
    codex-common --> codex-protocol
```

## Notes
- **Shared core**: both the interactive (`codex-tui`) and headless (`codex-exec`) frontends depend on `codex-core` for conversation/session management and execution pipelines.
- **CLI multiplexer**: the primary `codex` binary (`codex-cli`) embeds both frontends; it dispatches to `codex-tui` when no subcommand is given and to `codex-exec` for `codex exec`.
- **Runtime helpers**: `codex-arg0` provides arg0-based dispatch used by both binaries to expose additional entrypoints (e.g., Seatbelt/Linux sandbox).
- **Protocol layer**: `codex-protocol` and `codex-common` carry shared configuration structures, ensuring consistent CLI flags across binaries.
- **Model setup**: OSS model bootstrap lives in `codex-ollama`, which is pulled by both frontends when `--oss` is enabled.
- **App integrations**: `codex-cli` alone depends on higher-level tools such as `codex-app-server` and login helpers to expose additional subcommands beyond the two core frontends.
