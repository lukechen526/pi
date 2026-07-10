# Run modes, SDK embedding, and operational boundaries

The `pi` binary is one host for the coding-agent runtime. The same session
construction and event model can also serve print, JSON, RPC, and programmatic
SDK use. Understanding the modes prevents terminal assumptions from leaking
into automation and helps you select an integration surface deliberately.

Start with `packages/coding-agent/src/main.ts`, `src/modes/`, and
`src/core/sdk.ts`. For SDK examples, see
`packages/coding-agent/examples/sdk/` and `docs/sdk.md`.

## How Pi chooses a mode

`main.ts` resolves application mode from explicit arguments and terminal state:

| Mode | Typical trigger | Primary host |
|---|---|---|
| Interactive | TTY stdin/stdout without print/RPC/JSON selection | `InteractiveMode` |
| Print | `-p`, non-TTY stdin/stdout, or piped prompt | `runPrintMode` |
| JSON | `--mode json` | JSON-oriented output path |
| RPC | `--mode rpc` | `runRpcMode` JSON protocol |

The distinction is not cosmetic. It changes whether Pi can prompt a user, how
output is emitted, and which extension UI methods are meaningful. All modes
still rely on the coding-agent session/runtime instead of implementing separate
provider/tool loops.

## Extension mode behavior

An extension context exposes `mode` and `hasUI`:

| Application surface | `ctx.mode` | `ctx.hasUI` | Design implication |
|---|---|---|---|
| Interactive terminal | `"tui"` | true | Full components, dialogs, editor behavior |
| RPC | `"rpc"` | true | Protocol-backed dialogs/notices; custom terminal UI is limited |
| JSON | `"json"` | false | Do not wait for a user prompt or render components |
| Print | `"print"` | false | Agent runs, then process exits; UI calls are no-ops/fallbacks |

Guard user interaction explicitly. A policy tool that requires confirmation
needs a noninteractive decision, such as blocking with a clear reason, rather
than awaiting `ctx.ui.confirm()` in a mode without UI.

## Print mode as a single-run host

Print mode is appropriate for scripts that want one response rather than a
long-lived terminal session. `main.ts` prepares initial text/images from
arguments/files/stdin, then passes that input to `runPrintMode`. The normal
`AgentSession.prompt()` preprocessing remains active: extensions, context,
tools, provider selection, and session policy can still participate.

An extension in print mode should avoid assuming the process will stay alive
after a response. Background resources, delayed notifications, and UI-only
state are poor fits. Return model-facing content or durable session data if the
automation needs an observable result.

## JSON mode and machine-readable automation

JSON mode emits structured activity rather than terminal chat formatting. It
is useful for a caller that wants to observe events without parsing ANSI output.
Keep stdout clean in extensions: logging arbitrary text to stdout can corrupt a
machine-readable protocol. Prefer the documented event/UI mechanisms or stderr
for controlled diagnostics when appropriate.

The same caution applies to provider payload tracing. A raw JSON dump may
contain private context and may interfere with automation output. Build a
sanitized, opt-in diagnostic command instead.

## RPC mode as a persistent process boundary

RPC mode keeps a process alive for command/request exchange. Coding-agent has
RPC types, JSONL framing, client code, and an entry point under `src/modes/rpc/`.
The runtime and session still own model/tool state; RPC is a transport around
that behavior, not an alternate agent implementation.

Extensions can use `ctx.hasUI` in RPC mode for operations that map to protocol
UI messages, but `ctx.ui.custom()` cannot become a terminal component for a
remote client. Design commands/tools with a structured noninteractive fallback
when they might run through RPC.

## The public SDK entry point

`createAgentSession(options?)` in `core/sdk.ts` is the composition entry point
for a programmatic host. It can create defaults or accept caller-provided cwd,
agent directory, model, session manager, settings manager, model registry,
resource loader, scoped models, tool restrictions, custom tools, and a session
start event.

The SDK does several pieces of application policy on purpose:

```text
resolve cwd and agent directory
  -> auth/model/settings/session services
  -> resource loader (unless caller supplies one)
  -> restore/choose model and thinking level
  -> create Agent with coding-agent conversion/provider hooks
  -> create AgentSession and extension runtime
```

The result gives an `AgentSession`, extension load result, and possible model
fallback message. A host can subscribe to the session, call `prompt`, provide
custom tools, inspect sessions, or embed its own UI.

## Why SDK callers should not reimplement `main.ts`

It is tempting to instantiate `Agent` directly because its constructor is
small. That skips coding-agent semantics including resource loading, session
persistence, model restore/default selection, built-in tool definitions,
extension runner integration, system-prompt building, retry, and compaction.

Use `Agent` directly only when you deliberately own all of those policies.
Use `createAgentSession` when you want Pi's coding-agent behavior in a new host.
Use the CLI/RPC surface when a separate process boundary is preferable.

## Custom tools supplied by an SDK host

`createAgentSession` accepts custom tool definitions alongside built-in-tool
selection. These tools enter the same registry/extension wrapping flow as an
extension-registered tool. Give them a clear source identity, schema, durable
details shape, and prompt metadata where appropriate.

An SDK host can use this for application-specific operations without requiring
an extension file on disk. The tradeoff is scope: a tool supplied by one host
does not automatically appear in a user's ordinary `pi` invocation.

## Initial messages and attachments

The CLI's initial-message machinery can combine explicit prompt text, piped
stdin, file arguments, and supported images. Interactive mode feeds the
resulting message/images through the same session prompt path used after the
editor submits text. This keeps extension `input`/`before_agent_start` behavior
consistent between startup prompts and later user prompts.

When embedding, construct user content as text/image blocks and let the
session/agent conversion path handle the provider representation. Do not
pre-serialize provider-specific image payloads in a generic SDK host.

## Operational safety boundaries

Pi runs with the operating-system permissions of its process. It does not add a
built-in filesystem/process/network permission sandbox. The root README
recommends containerization or sandbox patterns when stronger isolation is
needed.

This has mode-specific consequences:

- A print/CI invocation can execute the same tools as an interactive one.
- Trust gates project extensions/resources, but do not sandbox global code.
- A custom provider can transmit context to its configured endpoint.
- `bash`, `write`, and extension `exec` run with host process authority.

Use OS/container boundaries for security enforcement. Use extension tool gates
for workflow policy and user confirmation. Do not confuse a model instruction
or a theme with a security control.

## Operational observability

The runtime includes timing, diagnostics, and output-guard behavior. Startup
diagnostics are collected during service/session construction and reported by
the application layer. Extension load failures are treated as startup errors
with a hint to retry without extensions. Settings/model load issues surface as
warnings/errors appropriate to the mode.

For a production-like integration, capture:

- selected mode and cwd;
- selected model/provider/API type, without credentials;
- session ID/path and persistence state;
- extension/resource load diagnostics;
- agent/turn/tool timing and completion reason;
- provider status metadata when safely available;
- cancellation/retry/compaction decisions.

Those observations map directly to the package boundaries in this guide and
make a remote incident diagnosable without collecting the full user prompt.

## Choosing an integration approach

| Goal | Best starting point |
|---|---|
| Human coding in a terminal | Interactive CLI |
| One scripted answer | Print mode |
| Machine-readable event/result stream | JSON mode |
| Long-lived external controller | RPC mode |
| Node application sharing Pi semantics | `createAgentSession` SDK |
| New model protocol/provider behavior | Provider extension or `packages/ai` contribution |
| Custom execution policy/UI in existing Pi | Extension |

Choosing the boundary early is an architectural decision. It determines who
owns terminal state, persistence, authentication, user confirmation, and
process lifetime.

## Operational checklist

Before putting Pi into automation or another host:

1. Choose the mode/SDK boundary intentionally.
2. Set a trust policy appropriate for noninteractive projects.
3. Select or restrict models/tools explicitly if policy requires it.
4. Store credentials in supported auth/config mechanisms, not prompt text.
5. Ensure output/logging does not corrupt JSON/RPC protocols.
6. Forward cancellation from the host into Pi's abortable work.
7. Decide session persistence and cleanup behavior.
8. Add an OS/container sandbox if untrusted code or high-risk tools are in scope.

These operational decisions are the final layer of the architecture: Pi's
internal boundaries are valuable only when the host uses them deliberately.
