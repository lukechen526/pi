# Extensions: change Pi without forking it

An extension is a TypeScript or JavaScript module that Pi loads at runtime with
`jiti`. Its default export is a factory receiving `ExtensionAPI`. The factory
registers hooks, tools, commands, shortcuts, flags, renderers, and providers;
Pi later invokes those registrations at defined lifecycle boundaries.

Extensions execute with the permissions of the Pi process. They are trusted
code, not a sandbox. Review a package before installing it and treat an
extension's filesystem, process, network, and credential access as you would a
normal local program.

## Choose the right customization mechanism

Before writing code, use the smallest mechanism that owns the behavior:

| Need | Prefer |
|---|---|
| Persist a model instruction or reusable task procedure | `AGENTS.md`, a skill, or a prompt template |
| Change colors and visual style | Theme |
| Change model/provider selection or static behavior | Settings or a custom provider |
| Add a command, tool, lifecycle gate, state, custom UI, or request hook | Extension |
| Reuse Pi from another Node program | Coding-agent SDK (`createAgentSession`) |
| Change generic loop/provider/TUI behavior for every consumer | Core contribution in the owning package |

An extension is powerful but should not become an unreviewed fork of the
interactive application. Favor public `ExtensionAPI` methods over importing
coding-agent internals.

## Discovery, trust, and loading

Pi automatically discovers a single-file extension or directory `index.ts` in
these places:

| Scope | Files discovered |
|---|---|
| Global | `~/.pi/agent/extensions/*.ts`, `~/.pi/agent/extensions/*/index.ts` |
| Project | `.pi/extensions/*.ts`, `.pi/extensions/*/index.ts` |

Project-local resources are loaded only after the project is trusted. You can
also add local paths in `settings.json`, or try a path for one invocation with
`pi -e ./path/to/extension.ts`. The latter is ideal for a quick test, but
`/reload` hot reload is intended for auto-discovered locations.

At startup, the resource loader loads extension factories before session start.
An async factory is awaited before startup continues. That makes it suitable
for one-time initialization such as discovering models and calling
`pi.registerProvider()`. Do not start timers, watchers, processes, or sockets
in the factory: commands such as `--list-models` can load factories without
starting a session. Start long-lived session resources in `session_start` and
close them idempotently in `session_shutdown`.

## The lifecycle you can rely on

```text
factory runs
  -> project_trust (eligible global/CLI extensions only)
  -> session_start
  -> resources_discover

prompt
  -> extension commands, then input
  -> before_agent_start -> agent_start
  -> for each model/tool turn:
       turn_start -> context -> provider hooks -> streamed messages
       -> tool execution hooks -> turn_end
  -> agent_end -> agent_settled

reload/switch/fork/quit -> session_shutdown
```

The timeline is condensed. The authoritative event names, return values, and
parallel-tool ordering are in `packages/coding-agent/docs/extensions.md` and
the type definitions in `packages/coding-agent/src/core/extensions/types.ts`.
Use the types for autocomplete and the documentation for behavioral contracts.

### Event groups and common use

| Event group | Representative events | Use it for |
|---|---|---|
| Trust/resources | `project_trust`, `resources_discover` | Decide project trust or contribute paths for skills/prompts/themes |
| Session | `session_start`, `session_shutdown`, `session_before_switch`, `session_before_compact` | Initialize/clean resources, persist/rebuild state, gate a session operation |
| Input/agent | `input`, `before_agent_start`, `agent_start`, `agent_end`, `agent_settled` | Transform input, add prompt context, track the true idle state |
| Turn/messages | `turn_start`, `context`, `message_update`, `turn_end` | Adjust context or render/measure streaming activity |
| Provider | `before_provider_headers`, `before_provider_request`, `after_provider_response` | Tracing, a request rewrite, response metadata |
| Tool | `tool_execution_start`, `tool_call`, `tool_execution_update`, `tool_result`, `tool_execution_end` | Observe progress, block/mutate calls, patch results |
| Model | `model_select`, `thinking_level_select` | Update model-aware UI/state |
| User shell | `user_bash` | Route `!` and `!!` through a different execution backend |

`tool_call` is a policy enforcement point: it runs after schema validation and
can block a tool. Its input is intentionally mutable, so a mutation affects
execution and later hook handlers see the changed input. Pi does not validate
again after that mutation. Keep any mutation narrow and validate your own
invariants before relying on it.

`context` receives a deep copy of messages that an extension may replace. It
is a safer place to prune/inject LLM context than modifying session storage.
`before_provider_request` is lower-level still: it sees the provider-specific
payload after serialization, so changes there are not reflected by
`ctx.getSystemPrompt()`.

## A complete first extension

This extension adds a model-callable `hello` tool, a `/hello-pi` command, and a
startup notification. It intentionally has no dependencies beyond Pi's bundled
extension APIs.

1. Create the file in an auto-discovered global location:

```bash
mkdir -p ~/.pi/agent/extensions
```

2. Save this as `~/.pi/agent/extensions/hello-pi.ts`:

```ts
import { Type } from "@earendil-works/pi-ai";
import { defineTool, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

const helloTool = defineTool({
  name: "hello",
  label: "Hello",
  description: "Greet a person by name",
  parameters: Type.Object({
    name: Type.String({ description: "The person to greet" }),
  }),
  async execute(_toolCallId, params, _signal, _onUpdate, _ctx) {
    return {
      content: [{ type: "text", text: `Hello, ${params.name}!` }],
      details: { greeted: params.name },
    };
  },
});

export default function register(pi: ExtensionAPI) {
  pi.registerTool(helloTool);

  pi.registerCommand("hello-pi", {
    description: "Show a greeting notification",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello, ${args || "world"}!`, "info");
    },
  });

  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("hello-pi extension is ready", "info");
  });
}
```

The tool definition uses `defineTool`, as does the repository's verified
`packages/coding-agent/examples/extensions/hello.ts`. `Type.Object` is a
runtime schema. `defineTool` derives the static type of `params`, and the agent
loop validates model-supplied arguments against that schema before calling
`execute`.

3. Start Pi in any project. The startup notification confirms discovery.
   Run `/hello-pi Ada` to exercise the command. Ask Pi to greet Ada with the
   `hello` tool, or inspect its active tool list, to exercise the model-facing
   tool.

4. Edit the file and run `/reload` in Pi. The reload path emits
   `session_shutdown`, reloads resources, then emits `session_start` with
   `reason: "reload"`. Treat code after `await ctx.reload()` as stale; the
   documented safe pattern is `await ctx.reload(); return`.

For a one-off test outside discovery, use `pi -e ~/.pi/agent/extensions/hello-pi.ts`.
The global file is preferable for the reload workflow.

## What `ExtensionAPI` and context provide

`ExtensionAPI` is the registration/control surface. Its important methods are:

| API | Purpose |
|---|---|
| `pi.on()` | Subscribe to a typed lifecycle event |
| `pi.registerTool()` | Add or override a model-callable tool |
| `pi.registerCommand()` | Add `/name` command with optional argument completion |
| `pi.registerShortcut()` / `pi.registerFlag()` | Add a keybinding action or CLI option |
| `pi.sendMessage()` / `pi.sendUserMessage()` | Put a custom or user message into agent flow |
| `pi.appendEntry()` | Persist extension-only session data outside LLM context |
| `pi.registerMessageRenderer()` / `pi.registerEntryRenderer()` | Render custom message/entry types in the transcript |
| `pi.getActiveTools()` / `pi.setActiveTools()` | Inspect or change the currently exposed tool set |
| `pi.setModel()` / `pi.setThinkingLevel()` | Change live session model settings |
| `pi.registerProvider()` / `pi.unregisterProvider()` | Register an endpoint/model provider at runtime |
| `pi.exec()` | Run a process with Pi's helper interface |

Handlers receive a context. The high-value fields are `ctx.cwd`, `ctx.mode`,
`ctx.hasUI`, `ctx.sessionManager`, `ctx.modelRegistry`, `ctx.model`, `ctx.signal`,
and `ctx.ui`. Use `ctx.signal` for cancelable nested work. Use `ctx.mode ===
"tui"` before requiring terminal-only UI behavior and `ctx.hasUI` before
prompting the user.

`ExtensionCommandContext` adds session-changing methods such as `newSession`,
`fork`, `navigateTree`, `switchSession`, `waitForIdle`, and `reload`. They are
available only in command handlers because changing a session from an ordinary
event handler can deadlock or leave stale state.

## Tools: make the model contract explicit

A custom tool needs a unique `name`, human-facing `label`, model-facing
`description`, TypeBox `parameters`, and `execute`. The return `content` goes
to the LLM; `details` is structured metadata for renderers and state recovery.

Add `promptSnippet` when the tool should have a one-line entry in Pi's default
system prompt. Add `promptGuidelines` for model instructions while the tool is
active. Each guideline must name its tool because Pi flattens all active tool
guidelines into one section.

For robust tools:

- Throw from `execute` to report an error to the model.
- Forward `signal` into `fetch`, process helpers, and long computations.
- Use `onUpdate` for incremental result rendering.
- Truncate large output with the exported truncation helpers; built-in limits
  are 50 KB or 2,000 lines.
- Use `StringEnum` from `@earendil-works/pi-ai` for string enums exposed to
  Google-compatible APIs.
- Use `withFileMutationQueue(absolutePath, callback)` around the *entire*
  read-modify-write window for a file-mutating tool.

An extension may override a built-in tool by registering the same name. That is
an advanced compatibility commitment: preserve the built-in result `details`
shape expected by session and UI code. If the override omits a renderer, Pi can
reuse the built-in renderer slot, but prompt metadata is not inherited.

## Commands, UI, and rendering

Commands are best for direct user actions and configuration. If multiple
extensions register the same name, Pi keeps both and gives them numeric
invocation suffixes in load order. Do not assume a command name is globally
unique after loading third-party packages.

`ctx.ui` supports notifications, select/confirm/input/editor dialogs, footer
status, widgets, editor text, themes, custom autocomplete, tool expansion, and
replacement editors. For a multi-key interaction, `ctx.ui.custom()` temporarily
replaces the editor with a `@earendil-works/pi-tui` component. In RPC and
noninteractive modes, many TUI-specific operations are no-ops or return
defaults; check mode first.

Use a message renderer for a custom message created with `pi.sendMessage()`;
that content participates in LLM context. Use an entry renderer for a custom
entry created with `pi.appendEntry()`; that is durable transcript/UI state but
does not reach the LLM. This is the key display-versus-context choice.

## State, sessions, and queues

In-memory variables vanish on reload, resume, fork, and restart. Reconstruct
state in `session_start` by examining the active branch. For tool-backed state,
store a recoverable snapshot in that tool result's `details`; it follows the
conversation branch naturally. Use `appendEntry()` for state that should not
enter model context.

`pi.sendMessage()` accepts delivery modes: `steer` inserts at the next steering
boundary, `followUp` waits until the agent would finish, and `nextTurn` waits
for the next user prompt. `sendUserMessage()` creates a real user message and
requires a delivery mode while streaming. Do not start a second direct agent
run from a hook; use these queues.

## Distribute an extension as a Pi package

For a reusable package, declare explicit resources in `package.json`:

```json
{
  "name": "@acme/pi-team-tools",
  "keywords": ["pi-package"],
  "peerDependencies": {
    "@earendil-works/pi-ai": "*",
    "@earendil-works/pi-agent-core": "*",
    "@earendil-works/pi-coding-agent": "*",
    "@earendil-works/pi-tui": "*",
    "typebox": "*"
  },
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

Put third-party runtime packages in `dependencies`, not `devDependencies`:
Pi package installation uses production installs by default. Core Pi packages
are bundled by Pi, so list them as `peerDependencies` with the documented `"*"`
range rather than bundling a competing copy. Package conventions also discover
`extensions/`, `skills/`, `prompts/`, and `themes/` when a `pi` manifest is not
present, but an explicit manifest is clearer and supports filtering.

Users can install a package from npm or git with `pi install`, or list it under
the `packages` array in settings. Project-installed packages are trust-gated
just like project-local extensions.

## Debugging extensions

Start with the smallest observation point:

1. Confirm discovery with a `session_start` notification.
2. Use `pi -e path/to/file.ts` to isolate a one-run load failure.
3. Run `/reload` after edits in an auto-discovered location.
4. Use `ctx.ui.notify()` in TUI mode and concise diagnostic logging while
   developing.
5. Use the hidden `/debug` command for the rendered TUI lines and the last
   messages sent to the LLM; it writes `~/.pi/agent/pi-debug.log`.
6. Inspect `ctx.sessionManager` state rather than guessing what a branch or
   compaction contributed.

Errors in extension handlers are logged and Pi continues, except a
`tool_call` handler error fails safe by blocking that tool call. Test policy
hooks for both allow and deny paths. Test `session_shutdown` cleanup by reload,
new session, resume, and exit rather than only by startup.
