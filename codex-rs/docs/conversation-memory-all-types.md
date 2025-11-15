# Conversation Memory – Included Item Types

Codex replays **every API-visible `ResponseItem`** when constructing the
next prompt, so prior tool executions (e.g., “read file X”) and their
outputs stay in context. The relevant logic lives in
`core/src/codex.rs` and `core/src/conversation_history.rs`.

## Recorded items

`ConversationHistory::record_items` accepts any `ResponseItem` that
`is_api_message` returns `true` for (`core/src/conversation_history.rs:36-53`,
`core/src/conversation_history.rs:348-371`). That includes:

- User/assistant messages (`role != "system"`)
- `FunctionCall` / `FunctionCallOutput`
- `CustomToolCall` / `CustomToolCallOutput`
- `LocalShellCall` (local tool execution metadata)
- `Reasoning` summaries and `WebSearchCall` markers

System messages and `ResponseItem::Other` are ignored so they don’t bloat
the replayed prompt.

## Tool executions in history

1. When the model asks to run a tool, its `FunctionCall` (or
   `CustomToolCall`) is streamed through `process_sse`, converted into a
   `ResponseItem`, and recorded via `Session::record_conversation_items`
   (`core/src/codex.rs:1590-1619`).
2. After Codex executes the tool (e.g., reads a file), the result is
   wrapped in a `FunctionCallOutput`/`CustomToolCallOutput` and also
   appended to history (`core/src/codex.rs:1660-1667` inside
   `process_items`).
3. Before the next turn, `run_task` calls `sess.history_snapshot()` so the
   full sequence—user prompt → tool call → tool output—gets replayed to
   the model (`core/src/codex.rs:1601-1619`).

Because both the call and its output are in the transcript, the model can
reference earlier tool results (like file contents) in follow-up prompts.

### “Explored … Read/Search …” entries in Codex TUI

- Those UI rows come from `LocalShellCall` + `FunctionCallOutput` records
  that were already persisted to history; the TUI simply formats them via
  `ExecCell::exploring_display_lines` (`tui/src/exec_cell/render.rs:222-317`).
- `ConversationHistory` explicitly keeps `LocalShellCall` variants (see
  `is_api_message` in `core/src/conversation_history.rs:348-371`), so the
  underlying tool command and its formatted output remain in the replayed
  prompt even though the UI shows a condensed “Explored / Read …” label.
- As a result, every “explore/search/list” action you see in the TUI is
  already part of the model-facing transcript; the front-end visualization
  does not add extra memory beyond what the model receives.

## Invariant helpers

- `ConversationHistory::ensure_call_outputs_present` inserts synthetic
  outputs if the model never returned one, so every recorded call has a
  matching output (`core/src/conversation_history.rs:84-187`).
- `remove_orphan_outputs` does the inverse: if an output appears without a
  matching call, it’s pruned (`core/src/conversation_history.rs:189-233`).

These guards ensure the “call → output” pairs that capture tool execution
results remain valid, which is critical for keeping the conversation
memory coherent across turns.
