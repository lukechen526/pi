# Customization patterns and contributor workflow

This chapter turns the architecture into decisions. It also calls out the
mistakes that most often create an extension which appears to work once but
fails on reload, in parallel tool mode, or outside the interactive terminal.

## Configuration before code

Pi configuration is layered. The coding-agent package uses a global agent
directory (normally `~/.pi/agent`) and project-local `.pi` resources. The
`piConfig.configDir` package setting makes the config directory rebrandable, so
extension code should import `CONFIG_DIR_NAME` when constructing a project path
instead of hardcoding `.pi`.

Use settings for stable choices such as providers, model selection, themes,
keybindings, trusted packages, and resource locations. Use an extension when a
choice must react to events, require a tool, retain state, or manipulate UI.

Project configuration and resources are an execution boundary: untrusted
projects do not load project-local extensions, packages, skills, or related
dynamic resources. A global extension should call `ctx.isProjectTrusted()`
before honoring project-local configuration of its own.

## Pattern: a safety gate around a built-in tool

This is appropriate for a team policy that requires confirmation before a
destructive command. It observes the existing `bash` tool instead of replacing
all shell behavior.

```ts
import { isToolCallEventType, type ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function register(pi: ExtensionAPI) {
  pi.on("tool_call", async (event, ctx) => {
    if (!isToolCallEventType("bash", event)) return;
    if (!event.input.command.includes("rm -rf")) return;

    const allowed = await ctx.ui.confirm("Dangerous command", "Allow rm -rf?");
    if (!allowed) return { block: true, reason: "Blocked by user" };
  });
}
```

The call is already schema-validated when the hook runs. The `isToolCallEventType`
guard provides a typed `event.input` for a built-in tool. This pattern must
handle non-TUI modes deliberately: `ctx.hasUI` can be false, so choose a safe
default or a noninteractive policy rather than assuming a dialog exists.

Do not mutate a tool's input merely to implement policy when blocking is more
honest. If you do mutate, remember no second schema validation occurs.

## Pattern: inject focused context, not a second system

`before_agent_start` is the correct point to add per-prompt instructions. It
can add a custom message and/or replace the chained system prompt. The custom
message is stored in the session and reaches the LLM; a notification is not.

```ts
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function register(pi: ExtensionAPI) {
  pi.on("before_agent_start", async (event) => {
    if (!event.prompt.startsWith("review ")) return;
    return {
      message: {
        customType: "review-mode",
        content: "Prioritize correctness, regressions, and concise findings.",
        display: true,
      },
    };
  });
}
```

Multiple handlers see a chained system prompt. Prefer appending a narrow,
conditional instruction over replacing the entire system prompt; replacement
can accidentally discard skills, tools, context files, and user-provided
instructions. For a message-level context transformation before *every*
provider request, use the `context` event instead.

## Pattern: state that survives restart and branching

For state derived from tool actions, put a serializable snapshot in each tool
result's `details`, then reconstruct from `ctx.sessionManager.getBranch()` in
`session_start`. A branch sees its own prior results, so it naturally restores
the state associated with that branch.

```text
tool result details  -> durable session entry -> branch reload -> extension rebuilds state
```

For state that should be visible in the transcript but never sent to the
model, use `pi.appendEntry("my-state", data)` and
`pi.registerEntryRenderer("my-state", renderer)`. Do not use `sendMessage`
for private bookkeeping: custom messages are model context.

## Pattern: dynamic tools and active-tool policy

`pi.registerTool()` may be called after startup. Pi refreshes it immediately
for the current session; it appears in `pi.getAllTools()` and can be enabled
with `pi.setActiveTools()`. To avoid accidentally removing useful tools, start
from the current selection:

```ts
const active = pi.getActiveTools();
pi.setActiveTools([...new Set([...active, "my_tool"])]);
```

Changing active tools affects the next turn and rebuilds the system prompt.
`setActiveTools(["read", "bash"])` is a real policy change, not just a UI
filter. Use it with a command/shortcut that clearly signals the new mode.

## Pattern: provider customization

Use `pi.registerProvider()` to add a proxy, endpoint, model catalog, OAuth
flow, or nonstandard stream implementation. Use a provider hook only for a
small cross-cutting change such as tracing headers or payload inspection.

An async factory is the right location when the provider must query a local
model service before registration. Pi waits for that factory, then applies
queued registrations while it creates services. Registering from a later
command takes effect immediately, but it cannot make the model available to
earlier startup operations such as model listing.

Provider configuration has security impact. Environment-variable references
for `apiKey` keep secrets out of source. Avoid logging headers, payloads, or
`ctx.getSystemPromptOptions()` because they can contain credentials, context
files, skills, and user instructions.

## Pattern: custom TUI that still works outside the TUI

Use `ctx.ui.setStatus` for a small persistent footer line, `setWidget` for
editor-adjacent information, and renderers for transcript content. Use
`ctx.ui.custom()` only when you need an interactive component with its own
keyboard behavior.

An extension may run in four modes: `"tui"`, `"rpc"`, `"json"`, and `"print"`.
Dialogs and custom component behavior do not have equivalent semantics in every
mode. Design a noninteractive fallback first, then add an interactive dialog:

```ts
if (!ctx.hasUI) {
  return { block: true, reason: "This action requires interactive approval" };
}
```

For custom editors and UI components, use injected keybinding managers and
named keybinding IDs. Do not hardcode raw control-key tests; that bypasses user
configuration.

## Pattern: package a team customization

An extension package can bundle extensions, skills, prompts, and themes. Put
runtime dependencies in `dependencies`; Pi package installation uses a
production-style install. Declare Pi core libraries as documented peer
dependencies. Share a versioned package with explicit `pi` manifest paths when
the team needs repeatability, rather than copying a large extension into every
repository.

Use a project package for project-specific behavior only after the team agrees
to trust that source. Use a global package for behavior that is independent of
the current project. The scope determines both discovery and the security
boundary.

## Common mistakes and the correct boundary

| Mistake | Why it fails | Better approach |
|---|---|---|
| Start a file watcher in the extension factory | Factory may run for a non-session command | Start in `session_start`; close in `session_shutdown` |
| Keep all extension state in a closure | Reload/resume/fork loses it | Rebuild from session entries or tool `details` |
| Call session replacement from a normal event | It can deadlock and leave stale contexts | Use a command and `ExtensionCommandContext` |
| Use `sendMessage` for private metadata | The model receives it | Use `appendEntry` and an entry renderer |
| Ignore `AbortSignal` | Esc cannot stop your nested work | Forward `ctx.signal`/tool signal |
| Mutate files in a custom tool without a queue | Parallel sibling calls can lose writes | Use `withFileMutationQueue` around read-modify-write |
| Override `read` or `bash` with a different result shape | Built-in UI/session behavior expects details | Preserve the documented built-in contract |
| Assume tool hooks observe sibling results | Parallel tools finish independently | Persist/coordinate after the batch or avoid the dependency |
| Assume `agent_end` means completely idle | Retry/compaction/follow-up may still run | Use `agent_settled` or command `waitForIdle()` |
| Test only `pi -e` | It does not prove discovery/reload behavior | Test from auto-discovery plus `/reload` |

## A practical verification ladder

Use evidence appropriate to what you changed:

1. Type-check the extension with the same TypeScript version/API contracts used
   by Pi, especially tool parameters and event return values.
2. Load it with `pi -e path/to/extension.ts` to isolate load errors.
3. Move it to an auto-discovered location and verify `session_start`.
4. Exercise the happy path and the relevant failure/deny/cancel path.
5. Run `/reload`, then repeat. Confirm no duplicate resource, stale closure, or
   leaked status remains.
6. For persistence, start a new process, resume/fork a session, and verify the
   intended branch state is restored.
7. For tools, test a malformed call/schema failure and a cancellation. For
   mutating tools, test concurrent sibling calls if applicable.
8. If you change repository code, run the repository-prescribed relevant check
   rather than a real-provider test. The coding-agent suite uses a faux provider
   under `packages/coding-agent/test/suite/harness.ts` for deterministic tests.

The repository ships many extension examples in
`packages/coding-agent/examples/extensions/`. Read the smallest example that
matches your need first: `hello.ts` for a tool, `permission-gate.ts` for a gate,
`input-transform.ts` for input routing, `entry-renderer.ts` for transcript-only
state, `dynamic-tools.ts` for runtime tool changes, and `subagent/` for a
larger composition.
