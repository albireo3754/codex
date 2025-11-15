# Conversation Transport Protocol

Codex talks to the model via the OpenAI **Responses API** using standard
HTTP + Server-Sent Events (SSE). Each turn packages the entire prompt into
JSON and streams the reply. Key pieces live in `core/src/client.rs` and
`core/src/client_common.rs`.

## Request construction

- `Prompt` (conversation history + tool list + instructions) is converted
  into `ResponsesApiRequest` (`core/src/client_common.rs:208-237`).
- The request JSON includes:
  - `model`, `instructions`, `input` (full history)
  - `tools` and `tool_choice`
  - Reasoning / text controls (`reasoning`, `text`)
  - `stream: true` and `include` fields, so the server emits incremental
    events.
- `ModelClient::attempt_stream_responses` sends a `POST` with
  `Content-Type: application/json` and `Accept: text/event-stream` using
  `reqwest` (`core/src/client.rs:224-310`). No custom binary protocol is
  involved—just HTTPS + JSON.

## Streaming transport

- The HTTP response body is an SSE stream. `process_sse` wraps the body
  with `eventsource_stream` and processes each JSON event
  (`core/src/client.rs:720-865`).
- Event kinds such as `response.output_item.done`,
  `response.output_text.delta`, `response.completed`, etc., are parsed
  and mapped into `ResponseEvent` values so the rest of Codex can react
  incrementally.

In short: every turn sends a JSON payload over HTTPS to the provider’s
Responses endpoint and reads back a text/event-stream. There is no
special binary or gRPC layer—just standard HTTP POST + SSE.
