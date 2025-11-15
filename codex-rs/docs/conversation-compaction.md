# Conversation Compaction

Codex automatically summarizes old history when a turn approaches the
model’s context window. The compaction logic lives in
`core/src/codex.rs` and `core/src/codex/compact.rs`.

## When compaction runs

1. After each turn, `run_task` looks at the reported `TokenUsage`.
2. If `TokenUsage::tokens_in_context_window()` ≥
   `TurnContext::client.get_auto_compact_token_limit()` (defaults to the
   model’s context window), `token_limit_reached` becomes `true`
   (`core/src/codex.rs:1645-1685`).
3. On the first breach, Codex triggers
   `compact::run_inline_auto_compact_task`. If the limit is still
   exceeded after compaction, the user is asked to start a new session.

## Summarization flow

`run_inline_auto_compact_task` (`core/src/codex/compact.rs:34-117`) does:

1. Clone the current `ConversationHistory` and append an internal user
   prompt (`SUMMARIZATION_PROMPT`, `tui/templates/compact/prompt.md`)
   asking the model to summarize the conversation.
2. Stream the summarization turn just like any other turn. If the prompt
   still exceeds the context window, it drops the oldest response items
   one by one and retries until it fits (`history.remove_first_item()`).
3. Once the model returns a summary, grab:
   - `summary_text`: latest assistant message after summarization
   - `user_messages`: all prior user messages (via
     `collect_user_messages`)
4. Build a “history bridge” using an Askama template:
   - `initial_context`: user instructions + environment context
     (`Session::build_initial_context`)
   - Bridge text includes a concatenated snapshot of user messages
     (trimmed to `COMPACT_USER_MESSAGE_MAX_TOKENS * 4` bytes) plus the
     summary (`HistoryBridgeTemplate`).
5. Replace the session history with `[initial_context, bridge]` so the
   next turn starts from a clean, summarized state.

## Algorithm characteristics

- **Trigger condition**: token limit computed per turn, based on the model
  response’s reported usage; no background cron job.
- **Data retention**: newest items are preserved; oldest items are dropped
  only if the summarization prompt itself would overflow.
- **Compression method**: semantic summarization via the model itself,
  not a statistical compressor. The compact prompt asks for a concise
  recap with bullet points explaining prior user actions and assistant
  results.
- **Persistence**: the resulting summary is written to the rollout log as
  `RolloutItem::Compacted`, so offline tools know compaction happened.

See `core/src/codex/compact.rs` for the full implementation, including
retry/backoff behavior and error reporting.
