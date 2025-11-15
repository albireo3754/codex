# Conversation Memory Lifecycle

Codex keeps a transcript for every session so the model sees the full
context whenever you send a follow-up prompt. The transcript is stored as
`ResponseItem`s (messages, tool calls, tool outputs, etc.).

## How a turn is built

1. Incoming user input is converted into `ResponseInputItem`s and
   persisted via `Session::record_input_and_rollout_usermsg`
   (`core/src/codex.rs:1041`). This appends the user message (and any
   structured inputs like file previews) to the in-memory
   `ConversationHistory`.
2. Each time `run_task` prepares a turn, it asks the session for a full
   history snapshot (`sess.history_snapshot()` inside
   `core/src/codex.rs:1601-1620`). That snapshot contains:
   - Initial context (user instructions + environment metadata from
     `Session::build_initial_context`)
   - Every prior user/assistant message
   - All function/custom tool calls and their outputs
3. The snapshot becomes the `Prompt.input` passed to `run_turn`
   (`core/src/codex.rs:1790-1805`). `Prompt` is then serialized into a
   `ResponsesApiRequest` and sent through the Responses API stream
   (`core/src/client.rs:224-237`). Because the entire history is replayed,
   the model has the previous “file read” answer available when you ask
   the next question.

## Managing context window pressure

Codex tracks token usage on every turn. After the model finishes, the
reported `TokenUsage` is compared against the auto-compaction threshold in
`run_task` (`core/src/codex.rs:1650-1685`). If the prompt is close to the
model’s context limit:

1. Codex launches the inline compaction flow
   (`compact::run_inline_auto_compact_task` at
   `core/src/codex.rs:1683-1685`).
2. The compaction task replays the history into a summarization prompt
   (`core/src/codex/compact.rs:34-117`). When successful, it replaces the
   transcript with:
   - Fresh environment context
   - A concise bridge that summarizes older user turns and the latest
     assistant response (`core/src/codex/compact.rs:118-166`)
3. Future turns keep streaming new messages on top of that summary, so the
   model retains high-level memory without exceeding the token budget.

## ConversationHistory invariants

`ConversationHistory` (`core/src/conversation_history.rs`) is the backing
store for all of the above. It ensures:

- Only API-visible items (messages, calls, outputs) are replayed to the
  model (`record_items` filters via `is_api_message`).
- Every function/custom tool call has a corresponding output; missing
  counterparts are synthesized so the transcript stays well-formed
  (`ensure_call_outputs_present`).
- When compaction trims old entries, paired call/output items stay in sync
  (`remove_corresponding_for` and `remove_orphan_outputs`).

Together, these pieces answer the original question: yes, when you ask a
follow-up after an expensive “read this file” turn, Codex resends the
entire conversation (subject to automatic compaction) so the model can
reference the earlier file contents and responses.
