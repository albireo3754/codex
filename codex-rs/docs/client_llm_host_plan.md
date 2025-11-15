# Client LLM Host Architecture Plan

This document captures a high-level map of the existing Codex workspace and a concrete plan for building a Codex-style client LLM host. It is intended as a living reference while iterating on a new host implementation.

## Goals

- Preserve a snapshot of the Codex Rust workspace structure that is relevant to an LLM host.
- Identify the core architectural concepts (conversation engine, sandboxing, UI frontends, backend integrations, and protocols).
- Lay out a phased plan for developing a similar client-side LLM host with clear milestones.

## Codex Workspace Snapshot

| Area | Purpose | Notes |
| --- | --- | --- |
| `core/` (`codex-core`) | Central business logic for managing conversations, tools, task queueing, prompts, sandbox abstractions, and interaction with remote models through the `ModelClient` trait. | Re-exports protocol types, enforces sandbox + apply_patch constraints, manages conversation state and CLI/TUI integration. |
| `protocol/` (`codex-protocol`) | Shared protocol definitions, config enums, response/item types, SSE payload models. | Keeps transport-agnostic data structures in sync between host and backend. |
| `cli/`, `exec/`, `tui/` | Frontend binaries (multitool CLI, headless `exec`, Ratatui TUI). | Wrap `codex-core`, inject UI-specific behavior, configure logging/telemetry. |
| `backend-client/` & `codex-backend-openapi-models/` | HTTP client and OpenAPI models for Codex cloud backends. | Encapsulate auth, retries, SSE streaming, and typed responses. |
| `app-server*/`, `responses-api-proxy/`, `cloud-tasks*/` | Additional services and utilities (app server, responses proxy, task UI). | Demonstrate server-side integrations and CLIs for auxiliary workflows. |
| `async-utils/`, `common/`, `utils/*`, `feedback/`, `keyring-store/` | Shared utilities (async patterns, config helpers, tokenization, caching, telemetry, credential storage). | Provide reusable building blocks for the rest of the workspace. |
| Sandbox & execution (`linux-sandbox/`, `windows-sandbox-rs/`, `execpolicy/`, `arg0/`, `apply-patch/`, `exec/`) | Platform-specific sandbox launchers, policy parsers, `arg0` runner, and the virtual `apply_patch` command. | Ensure commands run under least privilege and that apply_patch tooling behaves consistently. |
| Integration surfaces (`mcp-server/`, `mcp-types/`, `mcp-server/tests`) | Model Context Protocol server implementation so Codex can act as a tool for other agents. | Highlights how to expose Codex capabilities externally. |
| Dev tooling (`ollama/`, `login/`, `rmcp-client/`, `file-search/`, `feedback/`) | Experimental client/tool crates that integrate with external services (e.g., Ollama, OAuth). | Illustrate additional host responsibilities such as auth enrollment and local model support. |

### Data/Control Flow Overview

1. **Frontend entrypoint** (`codex-cli`, `codex-exec`, or `codex-tui`) parses CLI args, loads config from `docs/config.md` schema via `codex-core::config_loader`, and instantiates UI-specific adapters.
2. **`codex-core` orchestration** wires together:
   - `ConversationManager`, `CodexConversation`, and `tasks::*` to drive turn-by-turn operation.
   - `ModelClient` implementations (OpenAI, Azure, OSS) that speak `codex-protocol` payloads using SSE streaming (`chat_completions`, `response_processing`).
   - Sandbox + exec environment integration (`seatbelt`, `landlock`, `unified_exec`, `shell`, `exec_env`) plus the virtual `apply_patch` command.
   - Optional integrations (MCP client/server, custom prompts, rollout recorder, telemetry).
3. **Backend connectivity** is encapsulated inside `backend-client` with typed requests/responses and SSE streaming helpers. Authentication is sourced via `auth`, `keyring-store`, or CLI-provided tokens.
4. **Tooling + MCP**: `mcp` modules let the CLI connect to configured MCP servers; `mcp-server` exposes Codex as a server.
5. **Support services** like `app-server` or `responses-api-proxy` demonstrate how to embed Codex within other processes or provide proxies for constrained environments.

This snapshot should act as a reference when choosing which components to replicate versus replace.

## Plan for Building a Codex-Style Client Host

### Phase 0 – Foundation & Workspace Setup

1. Mirror the Cargo workspace layout: start with crates for `core`, `protocol`, and at least one frontend (`cli`). Provide shared crates for utilities, config parsing, and sandbox helpers.
2. Define protocol models early (messages, responses, events, streaming envelopes) so both the core and any backend/service layers share the same schema.
3. Establish configuration loading + override strategy inspired by `common/config_override.rs` and `docs/config.md`, ensuring TOML-based config plus environment overrides.

### Phase 1 – Core Conversation Engine

1. Port the `ConversationManager` concept: maintain sessions, history, prompts, and instructions that can be shared across frontends.
2. Implement a `ModelClient` trait mirroring Codex’s `client.rs` so different providers (OpenAI, Azure, local) can be swapped without touching UI code.
3. Include task abstractions (`tasks::*`) that represent user requests (interactive, review, exec). Start with a single task type, but keep the API open to add review/compact tasks later.

### Phase 2 – Execution Environment & Safety

1. Integrate sandbox controls similar to `seatbelt`, `landlock`, and `execpolicy`: define policies for macOS, Linux, and Windows and enforce them whenever shell commands run.
2. Ship a built-in `apply_patch` analog so the host can apply patches deterministically. Leverage the `apply-patch` crate design (parser, CLI shim).
3. Build command review tooling (`command_safety`, `user_shell_command`) to intercept risky operations and confirm with the user.

### Phase 3 – Frontends & UX

1. Deliver a headless binary (`exec`) for automation first; follow with a multipurpose CLI (`cli`) and eventually a Ratatui-based TUI (`tui`).
2. Share UI utilities via a `terminal` or `tui-utils` module to keep presentation layers thin. Reuse formatting helpers and diff renderers from Codex’s `turn_diff_tracker`/`tui` conventions.
3. Implement snapshot-based tests for TUI components (via `insta`) and pretty assertions for non-UI modules to ensure regressions are caught early.

### Phase 4 – Integrations & Extensibility

1. Add MCP client/server support so the host can both consume and expose tools, following the implementations in `codex-core::mcp` and the separate `mcp-server` crate.
2. Provide plugin points for custom prompts (`custom_prompts`), rollout logging, and telemetry (`otel`) similar to Codex’s architecture.
3. Consider auxiliary services (responses proxy, cloud task viewer) once the main loop is stable.

### Phase 5 – Distribution, Auth, and Observability

1. Package binaries per target platform, bundling sandbox launchers (`linux-sandbox`, `windows-sandbox-rs`) and ensuring the `arg0` runner is installed to set the correct execution context.
2. Implement secure credential storage (`keyring-store`) and OAuth flows (`login`, `rmcp-client`) for managed backends.
3. Enable tracing/metrics pipelines (OpenTelemetry) to observe model latency, sandbox failures, and network issues.

### Phase 6 – Hardening & Feedback Loops

1. Build regression suites similar to `core/tests` and `app-server/tests` that cover SSE streaming, sandbox enforcement, and command execution.
2. Integrate feedback capture (`feedback/`) and prompt rollout tooling so changes to prompts or safety logic can be audited.
3. Regularly sync documentation in `docs/` with user-facing behavior, mirroring Codex’s comprehensive `docs/getting-started.md` and `docs/advanced.md`.

## Next Actions

1. Decide which Codex crates map directly to the new host (e.g., reusing `codex-protocol` vs. writing a bespoke protocol layer).
2. Start scaffolding the foundational crates (Phase 0) with placeholder modules and documentation to unblock concurrent development.
3. Record updates to this plan as architecture choices evolve; treat it as the central reference for the client host project.

