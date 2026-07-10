# Guided source tours and architecture exercises

The earlier chapters describe Pi's architecture. This chapter turns that map
into repeatable reading exercises. Each tour has a question, an entry point,
the important control-flow boundary, and a concrete outcome. Work through them
with the source open; the purpose is to learn how the modules connect, not to
memorize file names.

## Tour 1: What launches when `pi` starts?

**Question:** Which module turns process arguments into an interactive agent?

Start at `packages/coding-agent/src/cli.ts`. It sets process title/environment,
configures the HTTP dispatcher early, and calls `main(process.argv.slice(2))`.
Then follow `main.ts`.

Trace these checkpoints:

1. `parseArgs` turns raw argv into a typed argument object.
2. `resolveAppMode` selects RPC, JSON, print, or interactive behavior from
   explicit flags and TTY state.
3. Startup settings/trust/session inputs are prepared.
4. `createAgentSessionRuntime` creates services and a session.
5. `InteractiveMode`, `runRpcMode`, or `runPrintMode` receives that runtime.

**Outcome:** You should be able to answer why a piped stdin invocation may use
print mode and why interactive mode can keep the caller's cwd while resolving
Pi's own assets from its package configuration.

## Tour 2: Why can a session change cwd safely?

**Question:** What is recreated when `/resume` opens a session from another
working directory?

Start at `core/agent-session-runtime.ts`. Read `switchSession`, then the
shared teardown/apply helpers. Follow the `createRuntime` factory passed from
`main.ts`.

Draw this before reading implementation:

```text
old session + old cwd services
  -> session shutdown/dispose
  -> SessionManager.open(target)
  -> target cwd services
  -> target AgentSession
  -> rebind interactive UI
```

Check which objects are cwd-bound: `SettingsManager`, `DefaultResourceLoader`,
and the runtime's session/model configuration all depend on the effective cwd.
The shared global auth storage can remain reusable, but its resolution is still
consulted by the new model registry.

**Outcome:** You should understand why extension code must not reuse an old
session context after a replacement and why `withSession` exists.

## Tour 3: How do resources reach a system prompt?

**Question:** Where do `AGENTS.md`, skills, prompts, and tool guidance join?

Start at `core/resource-loader.ts`, then find callers of
`resourceLoader.getSystemPrompt`, `getSkills`, and `getAgentsFiles` in
`core/agent-session.ts`. Read `_rebuildSystemPrompt` and then
`core/system-prompt.ts`.

Write the data as a structured object before it becomes text:

```text
cwd
context files
skills
custom/append prompt
selected tools
tool snippets
tool guidelines
      -> BuildSystemPromptOptions -> string
```

Now inspect `setActiveToolsByName`. It rebuilds the prompt because a changed
tool selection must change the instructions available to the model.

**Outcome:** You can explain why adding a tool has both a runtime registry
effect and a prompt/context effect.

## Tour 4: What happens to a user message before a provider sees it?

**Question:** Which transformations can change user text?

Read `AgentSession.prompt` top to bottom. List each return/branch in order:

1. Extension command resolution.
2. `input` event and possible transform/handled result.
3. Skill/template expansion.
4. Streaming steer/follow-up queue behavior.
5. Auth/model checks and possible compaction.
6. User/custom message construction.
7. `before_agent_start` message/system-prompt modifications.
8. `_runAgentPrompt`.

Then read the `Agent` call path and `streamAssistantResponse` to see the later
context transform plus `convertToLlm` boundary.

**Outcome:** You can choose `input`, `before_agent_start`, `context`, or
`before_provider_request` based on the representation you need to change.

## Tour 5: Follow one streamed token to the terminal

**Question:** How does a provider delta become a changed terminal row?

Start in `packages/agent/src/agent-loop.ts` at `streamAssistantResponse`.
Find how `text_delta`, `thinking_delta`, and `toolcall_delta` replace the
partial assistant message and emit `message_update`. Follow the event callback
to `Agent.processEvents` in `agent.ts`, then to coding-agent's session and
interactive subscribers.

Finally find `InteractiveMode.handleEvent`. Do not try to read the entire file
linearly; search for the event type and the component method it calls.

**Outcome:** You can locate a rendering bug as one of four categories:
provider event normalization, agent partial state, session event bridge, or
interactive component update.

## Tour 6: Follow one tool call through validation

**Question:** Why cannot an extension tool execute arbitrary model JSON?

Start at `executeToolCalls` in `agent-loop.ts`. Follow one call through:

```text
assistant content block
  -> active tool lookup
  -> prepareArguments
  -> validateToolArguments
  -> beforeToolCall
  -> execute
  -> afterToolCall
  -> ToolResultMessage
```

Next open `packages/coding-agent/examples/extensions/hello.ts` and identify
where its TypeBox schema becomes both a static parameter type and a runtime
validator. Compare it with `core/tools/read.ts`, which adds operations and
renderers but retains the same fundamental definition shape.

**Outcome:** You can explain why a schema belongs in an extension even when
TypeScript already knows the desired params type.

## Tour 7: Study parallel tool execution without guessing

**Question:** What ordering is guaranteed when an assistant asks for two tools?

Read `executeToolCallsParallel` and contrast it with the sequential path.
Write a timeline for two calls A and B:

```text
start(A), validate/gate(A)
start(B), validate/gate(B)
execute(A) and execute(B) concurrently
end events in completion order
final ToolResultMessage(A), ToolResultMessage(B) in source order
```

Compare that with the extension documentation's tool-execution guarantees. Then
inspect `withFileMutationQueue` to see how same-file writes gain a narrower
serialization guarantee without making unrelated tools sequential.

**Outcome:** You can design an extension tool that works correctly under the
default mode rather than accidentally relying on timing.

## Tour 8: Explain a model request without opening a network trace

**Question:** Where do key, headers, model metadata, and payload come from?

Start at the stream function in `core/sdk.ts`. It asks the model registry for
auth, combines environment/header settings, emits `before_provider_headers`,
then calls `streamSimple`.

Follow into `packages/ai/src/compat.ts`. It chooses a built-in model dispatcher
or resolves the API provider by `model.api`. Open one adapter such as
`api/openai-completions.ts` or `api/anthropic-messages.ts` and identify its
serialization and event-stream parser entry points.

**Outcome:** You can distinguish configuration bugs (registry), protocol bugs
(adapter), and runtime-loop bugs (agent) without a speculative rewrite.

## Tour 9: Trace a resume-context discrepancy

**Question:** Why did a resumed model refer to an old branch or summary?

Open `core/session-manager.ts` and read `buildContextEntries` plus
`buildSessionContext`. Use a small hand-drawn tree with a compaction entry and
a branch summary. Trace the active leaf backwards, then identify which entries
become context messages and which are metadata only.

Cross-check with `docs/session-format.md`. Use `getBranch`,
`buildContextEntries`, and `buildSessionContext` from an extension command to
inspect an actual session instead of parsing JSONL manually at first.

**Outcome:** You can prove whether a context difference is persistence policy,
compaction, branch navigation, or an extension's reconstruction mistake.

## Tour 10: Add a narrowly scoped extension feature

**Question:** How do you add a feature without coupling to internals?

Pick one small behavior such as a status widget, a custom command, or a
read-only tool. Start from the closest example. Name the lifecycle event/API
that owns it, then write a one-file extension under a global auto-discovery
directory. Verify startup, action, reload, and noninteractive behavior.

Use this decision table:

| Desired result | Narrow public seam |
|---|---|
| Ask model to use a new operation | `registerTool` |
| Let user invoke a task | `registerCommand` |
| Block/mutate a tool call | `on("tool_call")` |
| Add context per user prompt | `on("before_agent_start")` |
| Change provider endpoint/models | `registerProvider` |
| Store private session state | `appendEntry` |
| Render a custom persistent record | `registerEntryRenderer` |
| Add a small always-visible indicator | `ctx.ui.setStatus` or widget |

**Outcome:** You practice the extension API as a set of explicit seams rather
than as permission to reach into `InteractiveMode` or `AgentSession` private
state.

## A weekly reading routine for new contributors

Large codebases become legible through repeated, bounded questions. A practical
routine is:

| Session | Focus | Deliverable to yourself |
|---|---|---|
| 1 | CLI startup and services | One startup diagram with package boundaries |
| 2 | Agent loop and messages | One provider/tool turn timeline |
| 3 | Sessions/compaction | One active-branch/context sketch |
| 4 | Built-in tools | One tool contract comparison |
| 5 | Extensions | A working one-file extension and reload proof |
| 6 | TUI | A captured interactive event/render trace |
| 7 | Provider config | A local/custom provider configuration review |

The guide is intentionally organized so each exercise has a corresponding
chapter. Revisit the prompt lifecycle after every tour: it is the shortest
integration model of the entire project.
