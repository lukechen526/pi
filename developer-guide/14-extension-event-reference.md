# Extension event and API reference

This chapter is a practical reference for choosing extension hooks. It does not
replace the source types in `packages/coding-agent/src/core/extensions/types.ts`
or the exhaustive `packages/coding-agent/docs/extensions.md`; use those when
writing code. Its goal is to make event choice, return value, context, and
lifecycle safety explicit before implementation.

## Choose an event by the representation you need

The same user action appears in several representations. Select the latest
boundary that still contains the information you need, because later hooks are
more specific and less likely to disturb unrelated behavior.

| Need to inspect/change | Event/API | Representation |
|---|---|---|
| Raw submitted text before skills/templates | `input` | Text/images plus source/delivery mode |
| Prompt-specific system context | `before_agent_start` | Expanded prompt and system-prompt options |
| Message history before every LLM call | `context` | Deep-copied agent messages |
| Final HTTP headers | `before_provider_headers` | Mutable header map |
| Provider wire payload | `before_provider_request` | Adapter-specific payload |
| Provider status/response headers | `after_provider_response` | HTTP metadata before stream consumption |
| Validated tool arguments | `tool_call` | Tool name and mutable input |
| Tool execution output | `tool_result` | Content/details/error patch opportunity |
| User typed `!` command | `user_bash` | User command and context-exclusion choice |
| Durable session operation | Session events or command context | Session metadata/tree/replacement methods |

The rows describe different capabilities, not aliases. For example, a context
filter cannot reliably rewrite a provider's serialized protocol payload, and a
provider hook cannot alter Pi's own system-prompt string reported to other
extensions.

## Startup and trust events

### `project_trust`

This event occurs before Pi decides whether project-local dynamic resources may
run. It is visible only to user/global and CLI-provided extensions, not project
extensions. The handler must return a trust decision:

```ts
pi.on("project_trust", async (event, ctx) => {
  if (!ctx.hasUI) return { trusted: "undecided" };
  const yes = await ctx.ui.confirm("Trust project?", event.cwd);
  return yes ? { trusted: "yes", remember: true } : { trusted: "undecided" };
});
```

Return `"undecided"` when your extension does not own the policy. A yes/no
decision suppresses later handlers and the built-in flow. Only use `remember`
when it is appropriate to persist a global trust decision for this user.

### `resources_discover`

This fires after `session_start` so an extension can return additional skill,
prompt, and theme paths. It receives `reason: "startup" | "reload"`. It is a
discovery hook, not an instruction to read arbitrary project paths without
checking trust. Use the surrounding extension/session context's trust state for
project-local behavior.

## Session events

### `session_start` and `session_shutdown`

Use `session_start` to initialize state that requires an active session:
restore durable data, add UI status, start a watcher, or open a connection. Its
reason identifies startup, reload, new, resume, or fork. Use
`session_shutdown` to close that same resource. Cleanup must be idempotent
because reload, replacement, and quit may all trigger it.

Do not start long-lived resources in the factory. A factory may run for a
command that never creates a session, while session events make lifecycle
ownership explicit.

### `session_before_switch` and `session_before_fork`

These are cancellation gates. A switch handler sees whether the operation is
`new` or `resume` plus an optional target path. A fork handler sees entry ID and
position (`before` for fork or `at` for clone). Return `{ cancel: true }` to
block a user action after a confirmation/policy check.

Do not call a session-changing method from this ordinary event handler. If an
extension must initiate a switch, use a command handler with command-context
APIs described later in this chapter.

### Compaction and tree hooks

`session_before_compact` can cancel compaction or return a custom summary plus
the required kept-entry/token metadata. `session_compact` observes the durable
result and whether it came from an extension. `session_before_tree` can cancel
or provide a branch summary; `session_tree` observes the selected leaf and
summary outcome.

Use these hooks for policy/summary customization. Use `context` if the goal is
only to hide/filter data for a provider call; a context hook does not change the
durable compaction/tree semantics.

### Session information

`session_info_changed` fires after `/name` or `pi.setSessionName`. It is a good
place to update a title/status integration. It is not a model message and does
not require rebuilding the system prompt.

## Input and agent-run events

### `input`

Input handlers run after extension command lookup and before skill/template
expansion. The result controls the next step:

| Result | Effect |
|---|---|
| no result / `continue` | Leave input unchanged |
| `transform` with text/images | Pass modified input into later expansion |
| `handled` | Stop input processing; no agent run starts |

Transforms chain in extension load order. Keep transforms simple and make
source-aware decisions with `event.source` (`interactive`, `rpc`, or
`extension`). During streaming, `event.streamingBehavior` distinguishes steer
and follow-up queue delivery.

### `before_agent_start`

This runs after normal input expansion but before the low-level agent loop. It
is the best prompt-scoped seam. A result may add a custom persistent message
and/or replace the chained system prompt. Later handlers receive system-prompt
changes from earlier handlers.

The event exposes `systemPromptOptions`, including custom prompt, selected
tools, snippets/guidelines, cwd, context files, and skills. Treat it as
sensitive extension-local data. It may contain full instructions that should
not be copied into command descriptions or telemetry.

### `agent_start`, `agent_end`, and `agent_settled`

`agent_start` and `agent_end` correspond to one low-level agent loop. The end
does not prove application idleness: coding-agent may retry, compact/retry, or
start queued continuations. `agent_settled` means no such automatic work
remains. Use settled for a status integration that must not clear itself too
early.

### `turn_start`, `turn_end`, and messages

A turn is one assistant response plus its tool calls/results. `turn_end`
includes the assistant message and final tool results. Message events give
fine-grained lifecycle: user/assistant/tool-result messages emit start/end;
assistant streaming also emits update events. `message_end` can replace the
finalized message while preserving its role.

Message replacement is powerful. Preserve required fields and do not change a
role merely to repurpose a transcript; provider conversion and session logic
rely on role semantics.

## Provider events

### `before_provider_headers`

Pi assembles outgoing headers, then gives handlers a mutable map. Assign a
string to add/override, or `null` to delete. The hook runs once per provider
request; retries reuse the same headers rather than invoking it again.

Use it for a stable request correlation header. Do not use it for a credential
refresh protocol that must run anew on every retry unless you explicitly own
that behavior.

### `before_provider_request`

This sees the adapter-specific payload after Pi has converted context/system
prompt/tools. Return `undefined` to keep it or another value to replace it for
later handlers and the actual request. The event is suitable for a narrow
serialization compatibility patch or a sanitized development trace.

Because it is provider-specific, code here is coupled to API type. Guard logic
by model/provider when necessary, avoid assuming every adapter uses the same
JSON shape, and test against the provider's expected protocol.

### `after_provider_response`

This fires after the HTTP response arrives and before its body stream is
consumed. It receives normalized status and headers when available. Some
provider transports abstract away headers; absence is a provider limitation,
not an extension failure.

## Tool events and parallelism

`tool_execution_start`, `tool_execution_update`, and `tool_execution_end`
describe the execution row. `tool_call` is the mutable/blocking policy hook.
`tool_result` is the result-patching hook. The default parallel execution order
is subtle:

```text
source-order starts and preflight
  -> concurrent execute/update
  -> completion-order tool_result/end
  -> source-order final toolResult messages
```

Therefore, a `tool_call` handler sees session state synchronized through the
assistant message but not necessarily sibling tool results. Use a typed helper
for built-ins:

```ts
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

pi.on("tool_call", (event) => {
  if (!isToolCallEventType("read", event)) return;
  // event.input.path is typed for the built-in read tool.
});
```

For a custom tool, export its `Static<typeof schema>` input and supply explicit
generic parameters to the guard. Direct `event.toolName === name` narrowing is
not sufficient because arbitrary custom names overlap built-in literals.

## Extension context reference

All handlers receive a context with these recurring values:

| Field/method | Practical meaning |
|---|---|
| `cwd` | Current session working directory |
| `mode` / `hasUI` | Execution surface and interaction availability |
| `ui` | Controlled interactive UI adapter |
| `sessionManager` | Read-only access to durable tree/context state |
| `modelRegistry` / `model` | Available/current model and authentication view |
| `signal` | Active agent signal during turn-related work, otherwise usually undefined |
| `isIdle`, `abort`, `hasPendingMessages` | Flow-control observations/actions |
| `getContextUsage` | Context-window usage/estimate |
| `compact` | Trigger compaction with callbacks |
| `getSystemPrompt` | Current Pi system-prompt string at this lifecycle point |

The scope of `signal` is important. A command handler executed while idle does
not receive an active turn signal. A `tool_result` handler usually does. Pass it
to nested asynchronous work only when it is defined and relevant.

## Command context reference

Command handlers receive `ExtensionCommandContext`, which adds operations that
would be unsafe from arbitrary events:

| Method | Safe use |
|---|---|
| `waitForIdle` | Finish automatic work before a session mutation |
| `newSession` | Create a replacement session, optional setup/handoff |
| `fork` | Fork/clone from an entry |
| `navigateTree` | Move to a tree point with optional summary |
| `switchSession` | Open another session path |
| `reload` | Run `/reload` lifecycle |
| `getSystemPromptOptions` | Inspect base prompt inputs in a user-triggered command |

Replacement methods accept `withSession`. Use only the callback's context for
session-bound work. The old extension instance may have already run shutdown
cleanup, and captured old session objects will throw or represent old state.

## Registration API quick decisions

| API | Use when | Avoid when |
|---|---|---|
| `registerTool` | Model needs a new operation | User-only interaction is enough |
| `registerCommand` | User needs an explicit action | Input syntax is a natural transform |
| `registerShortcut` | User needs a configurable keyboard action | You can use a command/menu instead |
| `registerFlag` | Startup behavior needs a CLI option | State can live in settings |
| `sendMessage` | Model needs a custom context message | State should remain private |
| `sendUserMessage` | Agent should receive a genuine user message | You only need a silent state update |
| `appendEntry` | Durable data should not affect LLM context | You need model-visible instructions |
| `registerProvider` | Endpoint/models/auth/streaming change | A header-only trace is enough |

## Safety rules for extension authors

1. Start background resources in `session_start`, clean them in
   `session_shutdown`.
2. Treat project-local config as untrusted until `isProjectTrusted()` says
   otherwise.
3. Forward cancellation signals.
4. Serialize file mutations with the shared queue.
5. Truncate model-facing output.
6. Use mode/hasUI guards for user prompts and custom components.
7. Reconstruct persistent state from session entries.
8. Use fresh context after session replacement/reload.
9. Prefer public APIs/types over internal imports.

These rules cover the lifecycle errors that are hardest to discover from a
happy-path extension demo.
