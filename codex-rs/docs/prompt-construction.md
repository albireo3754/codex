# Prompt Construction & Serialization

This summarizes how Codex builds the `instructions`/`prompt` payload for a
Responses API turn and how the data is serialized before sending it to the
model.

## Instruction sources

- **Base instructions** – Derived from the selected `ModelFamily`; see
  `model.base_instructions`.
- **Overrides** – `TurnContext.base_instructions` (per-session override) or
  the prompt-specific `Prompt.base_instructions_override`.
- **Apply-patch helper** – If the model needs special apply_patch guidance and
  the tool is not present, `Prompt::get_full_instructions` appends the canned
  appendix (`core/src/client_common.rs:28-70`).
- The resulting string is `Prompt::get_full_instructions(&model)` and becomes
  `ResponsesApiRequest.instructions`.

## Prompt/input assembly

1. `run_task` gathers the full conversation snapshot:
   - `sess.history_snapshot()` (includes user msgs, assistant replies, tool
     calls/outputs, environment context) unless in review mode.
   - Any pending inline user input is appended first
     (`core/src/codex.rs:1590-1619`).
2. `Prompt.input` is set to that `Vec<ResponseItem>`. Each item matches the
   OpenAI Responses API schema (`codex_protocol::models::ResponseItem`).
3. `Prompt::get_formatted_input()` performs additional tweaks:
   - If a Freeform `apply_patch` tool is present, shell outputs are
     reserialized into the structured, multi-section format so they’re easier
     for the model to read (`core/src/client_common.rs:71-149`).
4. `Prompt.tools` is filled from the `ToolRouter` specs
   (`core/src/codex.rs:1785-1805`).
5. Other flags: `parallel_tool_calls`, `output_schema`, reasoning/text controls
   are carried verbatim from `TurnContext`.

## Serialization into Responses API

- `ModelClient::stream` builds a `ResponsesApiRequest<'a>` with references into
  the `Prompt` (`core/src/client.rs:224-237`).
- `ResponsesApiRequest` fields:
  - `model`: e.g., `gpt-5`
  - `instructions`: full string computed above
  - `input`: reference to the `Vec<ResponseItem>`
  - `tools`: `Vec<Value>` describing each tool (function/local_shell/web_search)
  - `tool_choice`: typically `"auto"`
  - `parallel_tool_calls`: bool
  - `reasoning`, `store`, `stream`, `include`, `prompt_cache_key`, `text` (verbosity/output schema)
- The struct derives `Serialize`, so `serde_json::to_value` converts it into a
  standard JSON object that the Responses API expects.
- Before sending, Codex may attach item IDs for Azure endpoints
  (`attach_item_ids`), but the wire format remains JSON.

## Transport

Finally, `attempt_stream_responses` posts the JSON to the provider endpoint
with `Accept: text/event-stream`, receiving SSE events that are fed back into
`ResponseEvent`s (`core/src/client.rs:224-865`). See
`docs/conversation-transport.md` for transport details.
