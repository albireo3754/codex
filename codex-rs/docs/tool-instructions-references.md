# Tool Instruction References

Use this cheat-sheet when designing function-calling / command tools for an
LLM-powered workflow like Codex.

## Core references

- `core/src/client_common.rs:208-237` – structure of `ResponsesApiRequest`
  (where tool specs are serialized into the outgoing JSON).
- `core/src/tools/spec.rs` – `ToolsConfig` parsing + `build_specs`, showing how
  TOML tool definitions become `ToolSpec` values.
- `core/src/tools/router.rs` – `ToolRouter` dispatch path, including how
  `FunctionCall` items are converted into concrete invocations.
- `config/tools/*.toml` – real-world tool definitions (names, descriptions,
  schema). Great starting point for your own command tool.
- `core/src/function_tool` and `core/src/unified_exec` – examples of server-side
  handlers that validate arguments and run commands before emitting
  `FunctionCallOutput`.

## Designing tool instructions

1. **Describe intent clearly** – Fill the tool `description` with short,
   imperative guidance (e.g., “Run shell commands inside the workspace. Provide
   `cmd` as an array of strings…”).
2. **Enforce schema** – Use JSON Schema features (`enum`, `pattern`, `minItems`)
   so the model can only produce valid calls. Codex sets `strict: true` for
   function tools, instructing the API to validate arguments.
3. **Return structured output** – Emit deterministic text blocks so the model
   can quote prior results. See `build_structured_output` in
   `core/src/client_common.rs:150-189` for the shell output format (exit code,
   wall time, etc.).
4. **Surface safety context** – If the command is dangerous or privileged,
   mention constraints in the description and enforce them server-side before
   running anything.

## Example flow

1. Define your tool in `config/tools/your_tool.toml`.
2. `ToolsConfig` loads it at session start, `ToolRouter::from_config` turns it
   into `ToolSpec`.
3. Each turn, the `Prompt` carries that `ToolSpec` in the `tools` array, so the
   LLM knows how to call it.
4. When the model emits a function call, `ToolRouter::build_tool_call` parses
   the payload and dispatches to your handler, which executes the command and
   returns a `FunctionCallOutput`.

Use these references to guide your own function-calling system, adjusting
descriptions and schemas to match the commands you want the LLM to run.
