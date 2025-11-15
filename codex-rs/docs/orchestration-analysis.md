# Codex Orchestration Analysis (Draft)

This draft gathers the minimum context needed to prototype a new orchestration
application that issues tool calls on top of the Codex Rust workspace. It
focuses on the crates that already coordinate tool execution (`codex-core`),
the two primary frontends (`codex-exec` and `codex-tui`), and calls out the
supporting services that become relevant when you need richer backends.

## High-Level Topology

- `codex-core` owns every piece of business logic: configuration, session
  lifecycle, model I/O, tool orchestration, sandboxing, MCP integration, and
  event emission.
- Frontends (`codex-exec`, `codex-tui`, `codex app-server`, etc.) are thin shells
  that build UI/UX around the `core` abstractions. They all rely on
  `ConversationManager` to spawn and track `CodexConversation` instances.
- Tool execution is mediated by the `tools` module in `core`. Every logical tool
  (shell exec, apply_patch, MCP adapters, etc.) implements a `ToolRuntime`
  compatible API so that the shared `ToolOrchestrator` can enforce approvals,
  sandbox policies, retries, and telemetry.
- Events emitted by `core` follow the schemas in `codex-protocol`. Every
  frontend subscribes to the same stream and renders/relays it differently
  (human terminal output, JSONL, Ratatui widgets, VS Code bridge, …).

```
 ┌────────────┐        ┌────────────────┐        ┌─────────────────────┐
 │  Frontend  │ ─────► │ Conversation   │ ─────► │  codex::Session     │
 │ (exec/tui) │ submit │ Manager        │ spawn  │  (turn loop + tools)│
 └────────────┘        └────────────────┘        └─────────────────────┘
        ▲                       │ events emit                 │
        │ render/serialize      ▼                            ▼
    Human / JSON /     codex-protocol events         Tool runtimes + sandbox
    App-Server UIs
```

## `codex-core` (tool orchestration + backend integration)

### Session & Conversation lifecycle

- `ConversationManager` (`core/src/conversation_manager.rs`) abstracts spawning,
  resuming, forking, and removing `CodexConversation` objects. Each spawn calls
  `Codex::spawn`, which sets up async submission/event channels
  (`core/src/codex.rs`).
- `Codex` is the headless session runner. You push `Submission`s (user prompts,
  approvals, tool responses, shutdown) and read streamed `Event`s. The first
  event is always `SessionConfigured`, which carries the effective config that
  frontends echo back to users.
- Resume/backtrack pathways rely on rollout histories recorded under
  `core/src/rollout`. `ConversationManager::resume_conversation_from_rollout`
  reconstructs the initial history and spawns a new session with it.

### Turn processing pipeline

1. Frontend submits a `Submission` (typically `Op::UserTurn`) built from user
   prompt, attached files/images, and optional overrides.
2. `codex::Session` builds a `TurnContext` that bundles sandbox policy, cwd,
   approvals, telemetry handles, and access to tool registries.
3. Turn orchestration pulls messages off the model stream, dispatches tool
   calls via the `ToolRouter`, and emits protocol events (agent reasoning,
   tool begin/end, diffs, approvals, completion).
4. On `TaskComplete`, frontends clean up local UI state and optionally write the
   final message to disk for scripting (`codex-exec --output-last-message`).

### Tool execution stack

- `core/src/tools` defines the types and registries for every tool.
  - `spec.rs` keeps the declarative list (ids, descriptions, arguments).
  - `registry.rs` wires spec entries to concrete handlers.
  - `router.rs` picks a tool implementation based on model function calls.
- `tools/orchestrator.rs` centralizes the approval + sandbox selection flow:
  - Check whether the tool requires approval for the current policy.
  - Request approval via protocol events when necessary.
  - Select an initial sandbox using `SandboxManager`.
  - Retry without sandbox when denials occur and the policy allows it (with a
    second approval prompt unless the first was session-scoped).
- `tools/runtimes` host low-level execution drivers (e.g., shell, apply_patch,
  unified exec). These runtimes translate abstract requests into concrete
  process launches via `exec_env` and `sandboxing`.
- `unified_exec` adds PTY-based long-running command sessions. Its
  `UnifiedExecSessionManager` tracks active PTYs, clamps output, and streams
  incremental chunks back through the event stream while staying within the
  orchestrator’s approval/sandbox rules.

### Safety, auth, and environment

- `sandboxing/` implements policy evaluation for different platforms (Seatbelt
  on macOS, Landlock/seccomp for Linux). Policies are configured via config
  profiles or CLI overrides (read-only, workspace-write, danger-full-access).
- `command_safety/` contains heuristics that decide whether a command is high
  risk and needs explicit approval, plus platform-specific allowlists.
- `AuthManager` loads all configured credentials (API keys, GitHub/GitLab PATs,
  MCP auth) and hands them to new sessions so tool calls can fetch remote data.
- `mcp/` and `mcp_connection_manager.rs` let Codex act as an MCP client; MCP
  tool invocations produce the same events as built-in tools.
- `config_loader/` + `config_types.rs` map `config.toml` onto rich structs that
  every frontend can extend with overrides (model selection, cwd, approvals,
  extra writable roots, etc.).

### Emitted protocol events

- All events conform to `codex-protocol` and cover:
  - Session lifecycle (`SessionConfigured`, `TaskStarted`, `TaskComplete`).
  - Agent messaging (`AgentReasoning*`, `AgentMessage*`).
  - Tool execution (`ExecCommandBegin/End`, `PatchApply*`, `McpToolCall*`,
    `TurnDiffEvent`, `TurnAbortReason`).
  - Approvals & sandbox status (`AskForApproval`, `WebSearch*`, risk reports).
  - Token usage snapshots for metering/billing.
- Frontends simply match on `EventMsg` to decide how to render or serialize.

## `codex-exec` (headless CLI)

- Entry point (`exec/src/main.rs`) is a thin arg0 dispatcher: the binary can
  either behave as `codex-exec` or as `codex-linux-sandbox` depending on how it
  was invoked, enabling distribution as a single artifact.
- `Cli` (`exec/src/cli.rs`) exposes everything needed for automation:
  prompt input (arg or stdin), optional images, model overrides, sandbox/approval
  flags (`--full-auto`, `--dangerously-bypass-approvals-and-sandbox`), JSON mode,
  last-message file output, resume capabilities, and config profile selection.
- `run_main` (`exec/src/lib.rs`) wires the CLI to `ConversationManager`:
  - Resolves prompt text (argument vs stdin), color mode, and OTEL tracing.
  - Loads configs, enforces auth guards, and seeds the conversation with the
    desired model/sandbox policy.
  - Chooses an `EventProcessor` implementation based on `--json`. The human
    processor renders a readable transcript with colorized sections and keeps a
    short buffer of tool output (20 lines) for portability; the JSON processor
    emits newline-delimited JSON events for downstream automation.
  - On `TaskComplete`, optionally writes the final agent message to disk so
    shell scripts can parse results without scraping stdout.
- Designed for CI or other non-interactive orchestration layers: you can pipe a
  prompt + files into stdin, capture events, and programmatically answer tool
  approvals by feeding additional submissions into the session.

## `codex-tui` (interactive terminal UI)

- Built with [Ratatui](https://ratatui.rs/) and crossterm. The crate enforces
  strict style guidelines (see `tui/styles.md` and `Stylize` helpers) so UI
  components stay consistent.
- `run_main` (`tui/src/lib.rs`) mirrors the CLI surface of `codex-exec` but
  launches the fullscreen TUI instead of emitting text. It:
  - Computes sandbox/approval overrides, handles OSS model shortcuts, resolves
    `cwd`, and configures OTEL/tracing with file sinks when needed.
  - Initializes onboarding flows (trust directories, auth prompts), clipboard
    helpers, and update checks before entering the main app loop.
- `App` (`tui/src/app.rs`) is the central state machine:
  - Owns a shared `ConversationManager` (session source `SessionSource::Cli`).
  - Instantiates a `ChatWidget` configured with the active `Config`,
    `AuthManager`, frame requester, and resume selection.
  - Manages other subsystems: file search previews, bottom pane popups, history
    cells, overlays (diff viewer, pager), feedback hooks, and update prompts.
  - Runs an event loop that merges UI events (`TuiEvent`), backend events
    (`AppEvent`), and async conversation output via Tokio channels.
- UI structure:
  - `chatwidget/` renders the transcript, approvals, tool output, and status
    indicators. It consumes protocol events and converts them into Ratatui lines
    (with heavy snapshot coverage under `tui/src/chatwidget/snapshots`).
  - `bottom_pane/` hosts the composer input, slash commands, approvals modal,
    feedback forms, and command palette.
  - `pager_overlay/`, `diff_render/`, and `history_cell/` compose the diff/share
    views for patches and command output.
  - `updates/`, `onboarding/`, and `status/` display state about versions,
    sandbox trust, and account usage.
- Because every event flows through `AppEventSender`, it is straightforward to
  hook additional orchestration features: subscribe to the same stream and
  mirror how the existing widgets map events to UI state.

## Supporting services & crates to keep in mind

- `codex-app-server`: JSON-RPC 2.0 (streaming over stdio) bridge used by the
  VS Code extension. If your orchestration app needs to serve another UI, reuse
  this harness or its protocol so you stay wire-compatible.
- `cli/`: bundles the subcommands (`codex`, `codex exec`, `codex tui`,
  `codex sandbox …`) into a single binary. Helpful if you need to expose your
  app alongside the official tooling.
- `codex-protocol` + `codex-common`: shared types (events, models, config
  schemas, CLI helpers) relied upon by every crate.
- `backend-client`, `responses-api-proxy`, `ollama`: host integrations with
  backend services and OSS model runners. Only needed if the orchestration app
  must change how Codex talks to remote inference endpoints.
- `linux-sandbox`, `execpolicy`, `apply-patch`: standalone binaries/libs that
  the toolchain shells out to when executing commands under sandbox policies.

## Suggested next steps for orchestration research

1. **Define the orchestration UX**: decide whether your app consumes the JSONL
   event stream (like `codex-exec --json`) or wants direct access to the
   `ConversationManager` for richer control (like the TUI).
2. **Prototype against `codex-core` directly**: instantiate a
   `ConversationManager`, spawn a session with your preferred config, and drive
   it with scripted approvals/tool inputs. This lets you validate the tool call
   flow without committing to a frontend.
3. **Evaluate approval & sandbox hooks**: if your app requires custom approval
   UX, inspect `tools/orchestrator.rs` and `tui/bottom_pane/approval_overlay.rs`
   to see how requests are surfaced to humans today, then decide whether to
   reuse the ReviewDecision flow or intercept it earlier.
4. **Plan integration points**: choose between reusing the app-server protocol,
   extending the CLI, or embedding `codex-core` via an FFI layer depending on
   how remote your orchestrator needs to be.
5. **Document desired tool set**: `codex-core` includes many tool handlers out
   of the box (shell, apply_patch, plan/test helpers, MCP). Identify gaps early
   so you can estimate the effort to add new handlers (tool spec entry +
   runtime implementation + UI surfacing + snapshots).

This draft should unblock deeper code reading sessions. Follow-up work can dive
into specific submodules (e.g., MCP transport, sandbox adapters, approvals UI)
as the orchestration requirements solidify.
