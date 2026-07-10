# Agent core: runs, events, queues, and tool batches

`packages/agent` is the reusable runtime beneath the coding agent. It knows
about a model, a system prompt, messages, tools, streaming, queues, and
cancellation. It deliberately does not know about JSONL sessions, themes,
project trust, extensions, or built-in filesystem tools.

The central source files are `packages/agent/src/agent.ts`,
`src/agent-loop.ts`, and `src/types.ts`.

## The three layers inside the package

| Layer | Main file | Responsibility |
|---|---|---|
| Data contracts | `types.ts` | Tool/message/event/loop configuration types |
| Stateful controller | `agent.ts` | Current state, queues, aborting, listeners, lifecycle |
| Pure-ish loop orchestration | `agent-loop.ts` | Provider turn, event emission, tool batches, continuation |

This split is useful when debugging. If state appears stale, start at `Agent`.
If a provider turn or tool batch sequence is wrong, start at `agent-loop`. If
the caller cannot supply a valid hook/tool/message shape, start in `types.ts`.

## Agent state and snapshots

The `Agent` holds state including:

- current system prompt, model, thinking level, and active tools;
- durable messages accumulated in this runtime;
- streaming message and pending tool IDs for UI observers;
- active run/abort controller;
- steering and follow-up message queues;
- event listeners and optional caller hooks.

Before a prompt starts, it creates a context snapshot with copies of the
messages and tool arrays. The snapshot is shallow with respect to each message,
but protects the loop from a caller mutating the arrays themselves during a
turn. The loop owns its current context object and appends new messages as it
emits events.

## Starting a prompt versus continuing a context

`Agent.prompt()` normalizes input to one or more `AgentMessage` values. It
rejects a concurrent active run, then calls `runAgentLoop()` with the snapshot.
`Agent.continue()` instead uses the current transcript and requires its last
message not to be an assistant message, except for explicit queued-message
handling paths.

The distinction prevents accidental replay:

```text
prompt(messages)      adds new user/custom messages and begins a run
continue()             asks the model again from existing context
steer(message)         queues content between assistant turns
followUp(message)      queues content after the agent would otherwise stop
```

Coding-agent uses this deliberately. It starts with a user message, then may
call `continue()` after automatic retry/compaction preparation or messages
queued by a post-run extension event.

## Active-run lifecycle and cancellation

`runWithLifecycle()` owns one `AbortController` per run. It marks the state as
streaming, calls the loop, converts an unexpected thrown failure into an
assistant error/aborted message sequence, then clears the active run.

`Agent.abort()` aborts that controller. The signal is passed to the provider
stream and tool execution. A tool/extension that starts child work must forward
the signal itself; abort does not magically cancel a detached fetch/process
whose API was never given the signal.

`waitForIdle()` resolves after the active run and awaited event listeners
finish. In coding-agent, true application idleness can also include session
policy after the low-level loop, such as retry or compaction. This is why
extension authors should use the higher-level `agent_settled` event when they
need Pi rather than only the current loop to be done.

## Event stream as the observable protocol

The loop emits events rather than updating a UI directly. Important event
families are:

| Events | Meaning |
|---|---|
| `agent_start`, `agent_end` | One low-level run begins/ends |
| `turn_start`, `turn_end` | A provider response plus any tool batch |
| `message_start`, `message_update`, `message_end` | User, assistant, and tool-result message lifetime |
| `tool_execution_start`, `tool_execution_update`, `tool_execution_end` | Tool row/process lifetime |

`Agent.processEvents()` first updates internal state, then awaits registered
listeners. This ordering means a listener sees a coherent agent state through
the event it is handling. Coding-agent relies on that property when syncing
session entries and calling extension hooks.

## One provider turn in detail

`streamAssistantResponse()` is the provider boundary in `agent-loop.ts`.

```text
AgentMessage[] context
  -> optional transformContext
  -> convertToLlm
  -> provider Context { systemPrompt, messages, tools }
  -> streamFn(model, context, options)
  -> for await normalized provider events
  -> partial/final AssistantMessage plus message events
```

The low-level loop does not assume a provider has the same message shape as an
agent message. `convertToLlm` belongs to the caller precisely because a coding
agent may have session-only messages while another embedding has a different
message representation.

On a provider `start` event, the loop pushes a partial assistant message into
its current context and emits `message_start`. Text, thinking, and tool-call
deltas replace that partial state and emit `message_update`. A final `done` or
`error` resolves the stream result and emits `message_end`.

## Tool argument preparation and validation

After a final assistant message, the loop finds content blocks with
`type: "toolCall"`. It resolves the requested tool from active context, calls
optional `prepareArguments`, then validates the result against the tool schema.

`prepareArguments` exists for session compatibility. An extension/tool can
adapt legacy stored call shapes into its current strict schema before validation
without weakening the public parameters declaration. It is not a general way to
accept arbitrary invalid input.

An unknown tool, invalid schema, hook failure, or explicit block becomes an
immediate error result rather than a crash. The model receives that error in a
tool-result message and can decide how to recover.

## Sequential and parallel tool execution

`ToolExecutionMode` has two choices:

| Mode | Behavior |
|---|---|
| `sequential` | Prepare, execute, finalize, and persist each tool before the next |
| `parallel` | Preflight calls in source order; run prepared tools concurrently; finalize completion events as they finish; emit final result messages in source order |

The `Agent` default is `parallel`. Parallel does not mean all events arrive in
arbitrary order: preflight remains deterministic. It does mean a tool hook
cannot assume a sibling's result has finished or been persisted. Use queues or
turn-end state when tools need coordination.

Some tool calls can request sequential execution through their definition or
the batch's execution policy. Inspect the active tool behavior rather than
assuming a single global mode if you build an advanced extension.

## Before and after tool hooks

The generic loop accepts two caller callbacks:

- `beforeToolCall` receives assistant message, tool call, validated args, and
  current context. It may block with a reason.
- `afterToolCall` receives the executed result/error state and may replace
  content, details, error flag, or terminate hint.

Coding-agent connects these callbacks to `ExtensionRunner.emitToolCall` and
`emitToolResult`. That bridge keeps the reusable agent package free of
extension types while allowing extension hooks to participate in execution.

## Termination, steering, and follow-up

When a tool batch completes, the loop creates final `toolResult` messages and
appends them to context. Unless every result has `terminate: true`, it performs
another provider turn. The all-results rule avoids one tool silently stopping a
batch whose other tools expect a model follow-up.

After each completed turn, the loop asks for steering messages. Steering
messages enter before the next provider request. When there are no more tool
calls and no steering messages, it asks for follow-up messages. Follow-ups
restart the inner loop only after the agent would otherwise stop.

```text
assistant/tool batch complete
  -> steering queue?  yes: inject before next model call
  -> no tools/steering: follow-up queue? yes: begin another turn
  -> neither: agent_end
```

This behavior is the technical meaning behind interactive Enter vs Alt+Enter
while Pi is running.

## Next-turn refresh hook

The loop can call `prepareNextTurn` after `turn_end`. It may replace context,
model, or thinking level for the next provider call. Coding-agent uses this to
refresh the current system prompt and active tool list from session state.

This matters when an extension dynamically changes tools or model state during
a run. The current turn remains internally consistent; the next provider call
sees the refreshed session snapshot rather than a stale configuration.

## Error semantics are intentionally normalized

The `StreamFn` contract says ordinary request/model/runtime failures should be
represented by a returned assistant event stream whose final message has
`stopReason: "error"` or `"aborted"`, rather than a rejected promise. The
agent also protects itself by converting unexpected throws into a final
assistant error message.

Tool failures are different: execute throws become an error `toolResult`, so a
model can read output/error and choose a recovery. This difference is important
for tests and UI: a provider failure ends the run; a tool failure usually lets
the model continue.

## Embedding the agent package

If you use `packages/agent` outside coding-agent, you must provide policy that
coding-agent normally supplies:

- initial `AgentState` with a valid model/system prompt/tools;
- `convertToLlm` for your message representation;
- provider `streamFn` and credentials or key resolver;
- listeners that persist/render state if needed;
- context transformation/queue behavior appropriate to your application;
- tool definitions and any permission gate.

The package gives a rigorous loop, not a complete coding application. That is
the reason Pi can support the interactive CLI, SDK examples, and other hosts
without coupling all of them to terminal/session decisions.

## Agent-core debugging checklist

When a turn misbehaves, capture these facts before editing code:

1. What did `convertToLlm` produce?
2. What system prompt and active tools were in the context snapshot?
3. Which normalized provider events arrived, and was a final message emitted?
4. Did the assistant end with tool calls, error, aborted, length, or stop?
5. Which tool arguments passed validation/preparation?
6. Was the batch sequential or parallel, and what event ordering occurred?
7. Did steering/follow-up queues inject another message?
8. Did a `prepareNextTurn` refresh change model/tools/context?

This reduces a complex agent issue to a small number of observable protocol
boundaries.
