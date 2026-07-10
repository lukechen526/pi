# Failure and recovery reference

Pi turns many failures into explicit protocol data instead of allowing an
exception to escape through every layer. This reference connects symptoms to
the layer that owns recovery. Use it during debugging and extension design.

## Failure flow by runtime layer

```text
configuration/trust failure
  -> startup diagnostic or ignored project resource

model/provider failure
  -> final assistant message: error or aborted
  -> AgentSession retry/compaction policy may decide next action

tool validation/execution failure
  -> error ToolResultMessage
  -> model receives the error and can choose a recovery action

extension hook failure
  -> runner reports error; tool_call fails safe by blocking

rendering failure
  -> fallback renderer or interactive diagnostic path
```

The different outcomes are intentional. A provider failure means there is no
trusted assistant result to continue from. A tool failure is part of an
assistant turn and is often useful information for the model's next request.

## Startup and configuration failures

| Symptom | Owning layer | Expected behavior | First recovery action |
|---|---|---|---|
| Malformed `settings.json` or `models.json` | Settings/model registry | Diagnostic/load error; built-ins remain available where possible | Correct JSON; inspect reported scope/path |
| Project extension unavailable | Trust/resource loader | Project resources ignored when untrusted | Save/override trust intentionally, then restart/reload |
| Extension factory fails | Resource loader/application | Startup diagnostics; Pi suggests retry without extensions | Run with `-ne` or `-e` to isolate path/import failure |
| No model selected | Session setup | Prompt is rejected before agent loop | Configure/choose model and auth |
| Auth missing | Model registry/session | Prompt is rejected with provider guidance | `/login`, configure key, or select authenticated model |

Do not work around a trust failure by making a global extension read an
untrusted project's configuration. That moves a deliberate security boundary
into unreviewed custom code.

## Provider failures

Provider adapters report normal failure/abort outcomes through the assistant
stream contract. `Agent.runWithLifecycle()` also converts unexpected throws to
a final assistant message with `stopReason: "error"` or `"aborted"`, followed
by turn/agent end events. The transcript and extensions therefore see a normal
terminal event sequence even when the network fails.

`AgentSession` evaluates the final assistant message. Depending on settings and
error classification it may prepare an agent-level retry, auto-compact/retry
after an overflow, or settle. Provider SDK retries and agent-level retries are
separate knobs; an excessively long provider retry can hide useful control from
the agent/session layer.

For an extension, the correct response to an `agent_end` error is normally to
observe/report it, not to immediately create another competing prompt. Use
`agent_settled` or a deliberate command action once the built-in policy has
finished.

## Tool failures

Tool processing has several failure points:

| Point | Result returned to model |
|---|---|
| Tool name is not active/found | Error result naming the missing tool |
| `prepareArguments` or schema validation fails | Error result containing the failure reason |
| `tool_call` returns block | Error result with block reason |
| Tool execute throws | Error result with thrown error text |
| Abort occurs before/during work | Error result indicating cancellation |
| `tool_result` hook throws | Error result rather than a malformed patched result |

The loop emits a tool-result message after finalization. A model can inspect
the result, modify its arguments, call another tool, or explain the failure.
Make tool error strings actionable: include an invalid path, missing exact
text, exit code, timeout, or next command when safe. Do not expose secrets in
a tool error just because the model can read it.

## Cancellation versus timeout

An abort signal expresses user/application cancellation. A timeout expresses a
tool/provider policy deadline. They can lead to similar text in the transcript,
but extension code should preserve the distinction:

- Forward abort signals into `fetch`, process execution, and model helpers.
- Clean up partial state deterministically when cancellation happens.
- Use a tool timeout only when the work has a clear bounded deadline.
- Do not convert user cancellation into an automatic retry.

The built-in bash tool kills its process tree for abort and timeout, collects
available output, and returns an error with the appropriate status. A custom
tool that launches a child process should provide equivalent cleanup rather
than leaving a detached process behind.

## Recovery after compaction and tree movement

Compaction/retry is recovery from context pressure, not from every provider
error. A custom compaction extension must return a summary tied to the
preparation's `firstKeptEntryId` and token count; an arbitrary summary string
without this boundary cannot correctly rebuild session context.

Tree navigation changes the active leaf. An extension that reconstructs state
from `getEntries()` instead of the active branch may accidentally recover state
from an abandoned path. This can look like a failure after `/tree` even though
session persistence behaved correctly.

## UI and renderer fallback

Renderers are presentation code. If a custom tool/message renderer throws, Pi
falls back to basic rendering for that slot where supported. Treat a fallback
as a diagnostic, not evidence that result data is correct. Inspect the original
content/details and renderer context, then make the renderer handle partial,
error, and expanded states explicitly.

For a TUI-only issue, test the same data at a narrow terminal width and with
the tool output collapsed/expanded. Many apparent rendering failures are width
or ANSI-visible-width mistakes rather than agent/session failures.

## Incident checklist

For a reliable bug report or regression test, record:

1. Mode, cwd, trust state, session ID/path, and selected model.
2. The input/event that triggered the problem.
3. Whether it was idle, streaming, compacting, or replacing a session.
4. The relevant final stop reason/tool result error.
5. Sanitized provider status/headers if applicable.
6. Active tools and extension/resource provenance.
7. Whether Esc/reload/new/resume changes the behavior.
8. A focused test, session tree sketch, or terminal capture proving the result.

This checklist establishes a factual boundary before a fix is proposed. It is
the operational counterpart to the source tours in the previous chapter.
