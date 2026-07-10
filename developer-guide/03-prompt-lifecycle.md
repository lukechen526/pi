# Prompt lifecycle: from Enter to the next render

This chapter traces the normal interactive path. It is source-grounded in
`packages/coding-agent/src/modes/interactive/interactive-mode.ts`,
`packages/coding-agent/src/core/agent-session.ts`,
`packages/agent/src/agent.ts`, `packages/agent/src/agent-loop.ts`, and the
`packages/ai` dispatch layer.

The exact call graph has guards for commands, compaction, retries, queued
messages, and other modes. The numbered path below is verified behavior; its
diagrams omit helper calls that do not alter the data/control boundary being
explained.

## The complete path

```text
Editor Enter
  -> InteractiveMode.getUserInput()
  -> AgentSession.prompt(text)
      -> extension command / input preprocessing / prompt expansion
      -> user + extension custom messages
      -> Agent.prompt(messages)
          -> agentLoop
              -> transform context -> convert messages -> provider stream
              -> assistant stream events -> tool validation/execution
              -> tool-result messages -> repeat or finish
      -> session persistence + extension events + transcript rendering
```

## Stage 1: Terminal input enters the TUI

`InteractiveMode` owns the coding-agent's interactive screen. Its editor
submission handler recognizes built-in slash commands and `!`/`!!` user-shell
commands before normal agent prompting. A normal idle submission is queued or
resolved through `getUserInput()`. The `run()` loop awaits that method, then
calls `await this.session.prompt(userInput)`.

When Pi is already streaming, regular Enter calls `session.prompt(text,
{ streamingBehavior: "steer" })`; Alt+Enter uses `"followUp"`. That is not a
second concurrent agent run. It queues a message for a defined loop boundary.
During compaction, non-command input is held by interactive mode until the
compaction flow completes.

The initial prompt supplied by command-line arguments travels through the same
`session.prompt()` API. Print and RPC modes do not instantiate
`InteractiveMode`, but they converge on `AgentSession` rather than
reimplementing the provider path.

## Stage 2: Commands, keybindings, and input interception

There are two distinct command systems:

- Built-in interactive commands such as `/model`, `/settings`, `/new`, and
  `/compact` are handled in `InteractiveMode` before a normal prompt reaches
  the session.
- Extension commands registered with `pi.registerCommand()` are detected first
  inside `AgentSession.prompt()`. They execute immediately, including while an
  agent is streaming, and do not enter the model context unless their handler
  explicitly sends a message.

For ordinary text, `AgentSession.prompt()` asks the extension runner to emit
an `input` event. The event sees raw text after extension-command lookup and
before skill/template expansion. An extension can continue, transform, or
handle it. If it continues, Pi expands `/skill:name ...` and prompt templates,
then continues with the expanded text.

This order matters for custom syntax. An extension that wants to intercept a
skill invocation must use `input`; one that wants a durable custom command
should use `registerCommand`, not parse slash text manually.

## Stage 3: Agent-session orchestration constructs a turn

Before starting a fresh run, `AgentSession.prompt()` rejects missing model or
credential cases, checks pre-prompt compaction, builds a user `AgentMessage`,
adds pending `nextTurn` custom messages, and emits `before_agent_start`.
Handlers can add custom messages and chain a system-prompt override. The
session then calls its private `_runAgentPrompt(messages)` method.

`_runAgentPrompt()` calls `Agent.prompt()` and may later call `Agent.continue()`
for automatic retry, automatic compaction retry, or messages queued by an
`agent_end` handler. It finally clears the per-run prompt override, flushes
pending user-bash messages, and emits `agent_settled`.

The `Agent` class in `packages/agent/src/agent.ts` owns the active abort
controller, streaming flags, accumulated messages, active tools, and queues.
`Agent.prompt()` captures a context snapshot and drives `runAgentLoop()`.

## Stage 4: Context and system prompt are built at two boundaries

The coding agent first builds a base system prompt in
`AgentSession._rebuildSystemPrompt()`. It combines the working directory,
loaded `AGENTS.md`/context files, loaded skills, custom and appended system
prompts, and selected tools plus their snippets and guidelines.

`buildSystemPrompt()` in `core/system-prompt.ts` converts that structured data
to text. Changing active tools rebuilds this base prompt, so tool prompt
metadata can affect the next model turn.

Immediately before *each* model request, the low-level loop has another
boundary. `streamAssistantResponse()` in `agent-loop.ts` calls the optional
`transformContext` callback on `AgentMessage[]`, then calls `convertToLlm()` to
produce provider-compatible `Message[]`. Coding-agent supplies these callbacks
when creating `Agent` in `core/sdk.ts`:

- `transformContext` dispatches the extension `context` event.
- `convertToLlm` filters/normalizes coding-agent-specific message types and
  applies the image-blocking setting as a defense in depth.

The resulting `Context` has exactly `systemPrompt`, `messages`, and `tools`.
This separation explains why a session can persist rich UI-only entries while
only a selected representation reaches the LLM.

## Stage 5: Model and provider selection

At startup or session restoration, `createAgentSession()` in `core/sdk.ts`
tries the model recorded in a nonempty session, then configured defaults, then
provider defaults through `findInitialModel()`. It clamps the thinking level to
the selected model's capabilities.

The coding agent's `ModelRegistry` owns the model catalog, configured provider
overrides, credential lookup, OAuth state, and request headers. The stream
function installed by `core/sdk.ts` calls
`modelRegistry.getApiKeyAndHeaders(model)`, merges provider-attribution and
request headers, then emits the extension `before_provider_headers` hook.

`packages/ai/src/compat.ts` receives the chosen model and its `api` identifier.
`streamSimple()` either delegates to built-in model support or resolves the
registered API provider and calls that provider's `streamSimple`. Provider
adapters in `packages/ai/src/api/` serialize the normalized context and return
a provider-independent assistant event stream.

Extensions can participate at three precise points:

- `before_provider_headers` mutates assembled request headers.
- `before_provider_request` receives and may replace the provider-specific
  payload after serialization.
- `after_provider_response` observes HTTP status/headers before stream
  consumption when the adapter exposes them.

Those hooks are not substitutes for a custom provider. Use
`pi.registerProvider()` when you must declare a model catalog, endpoint, auth,
or streaming implementation.

## Stage 6: Streaming response becomes agent events and TUI updates

`streamAssistantResponse()` consumes the `AssistantMessageEventStream` with
`for await`. It creates a partial assistant message on `start`, replaces that
partial message as text/thinking/tool-call deltas arrive, and emits
`message_start`, `message_update`, and `message_end` events.

`Agent.processEvents()` reduces those events into state and then awaits all
subscribers. `AgentSession` bridges generic events to persistence, extensions,
and application behavior. `InteractiveMode` updates the visible transcript,
current streaming row, tool rows, footer, and render requests. The TUI package
performs the actual terminal rendering.

The extension lifecycle maps directly onto this path: `agent_start`,
`turn_start`, `message_start`, many `message_update` events, `message_end`,
then later `turn_end` and `agent_end`.

## Stage 7: Tool calls are parsed, validated, gated, and executed

After a final assistant message, `agent-loop.ts` filters content blocks whose
`type` is `"toolCall"`. A message stopped for `"length"` fails all its tool
calls instead of executing possibly truncated JSON arguments.

For each normal tool call, the loop:

1. Finds an active tool by name from the current context.
2. Runs its optional `prepareArguments` compatibility transform.
3. Calls `validateToolArguments()` against the TypeBox schema.
4. Emits `tool_execution_start`.
5. Invokes `beforeToolCall`, which coding-agent wires to extension `tool_call`.
   A handler may mutate validated input or return `{ block: true }`.
6. Calls the tool's `execute(id, args, signal, onUpdate)` function.
7. Converts incremental `onUpdate` values into `tool_execution_update` events.
8. Calls `afterToolCall`, wired to extension `tool_result`, and allows its
   partial result patch.
9. Emits `tool_execution_end` and saves a `toolResult` message.

The configured default is parallel execution: preflight happens in assistant
source order, successful tool executions may run concurrently, and final
`toolResult` messages are emitted in assistant source order. Start/update/end
event ordering differs in parallel mode; see the extension documentation before
using sibling tool results as state.

Built-in tools are supplied by coding-agent. They are ordinary tools at this
boundary, which is why an extension can add a tool or override a built-in one.
Custom filesystem-mutating tools must participate in `withFileMutationQueue()`
to avoid lost writes when sibling tools run in parallel.

## Stage 8: Results create the next agent action

Each final tool result is appended to the in-memory agent context and becomes
the model-facing material for the next turn. If at least one tool call was
executed and the batch was not explicitly terminating, the loop starts another
assistant response. If the model instead returns ordinary final text, the loop
checks steering messages first and follow-up messages when it would otherwise
stop.

```text
user/custom context -> model -> assistant tool calls
                                |
                                v
                         validated tool results
                                |
                                +----> model again
```

A tool can return `terminate: true` to hint that the next model call should be
skipped, but the batch terminates only when every finalized tool result sets
that flag. A tool signals failure by throwing; merely returning a field named
`isError` does not mark it failed.

## Stage 9: Persistence and rendering settle the turn

As message events complete, `AgentSession` records them through its
`SessionManager`. The session is a JSONL tree, so the persisted form can later
rebuild a branch and compact context. Model and thinking-level changes are also
session entries. Extension state added with `appendEntry()` is persisted as a
custom entry but remains outside LLM context.

`InteractiveMode.handleEvent()` responds to the same event stream. Assistant
updates repaint the streaming message; tool lifecycle events manage pending
tool rows; finalized messages become permanent transcript components. Renderer
registration lets an extension replace the display of its custom messages,
entries, or tool calls/results without changing how the model loop works.

## Stage 10: Errors, cancellation, retries, compaction, and shutdown

There are several deliberately different failure paths:

| Situation | What happens |
|---|---|
| Provider/model run throws | `Agent.runWithLifecycle()` turns it into a final assistant message with `stopReason: "error"` or `"aborted"`, then emits normal terminal events. |
| Provider stream reports error/aborted | `agent-loop.ts` ends the turn without executing tools. |
| Tool schema invalid, unknown tool, hook blocks, or execute throws | The loop creates an error tool result and gives that result to the model. |
| User presses Esc / abort is requested | The active `AbortController` is aborted; abort-aware provider/tool/extension work should receive the signal. |
| Retryable assistant error | `AgentSession` may prepare an automatic retry after the low-level run ends. |
| Context pressure | `AgentSession` may compact, write a summary entry, rebuild context, and continue/retry. |
| Session replacement or process exit | The extension runtime receives `session_shutdown`; session-bound resources must be cleaned up. |

`agent_end` means the current low-level loop stopped. `agent_settled` is later:
it means Pi has no automatic retry, compaction retry, or queued continuation
left. Status integrations and cleanup that require true idleness should use
`agent_settled` or `ctx.waitForIdle()` in a command context.

## Trace this yourself

For a live source trace, start at these symbols in order:

1. `InteractiveMode.run()` and `getUserInput()`.
2. `AgentSession.prompt()` and `_runAgentPrompt()`.
3. `Agent.prompt()`, `createLoopConfig()`, and `processEvents()`.
4. `runAgentLoop()`, `streamAssistantResponse()`, and `executeToolCalls()`.
5. `streamSimple()` in `packages/ai/src/compat.ts`.
6. The `streamSimple` function in the selected provider API adapter.

Use the extension event timeline in `packages/coding-agent/docs/extensions.md`
as a second view of the same lifecycle. It documents what extension authors
can rely on; the source symbols above show which layer makes each guarantee.
