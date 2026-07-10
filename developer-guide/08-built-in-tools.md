# Built-in tools: filesystem and shell execution

The coding agent ships seven named tools: `read`, `bash`, `edit`, `write`,
`grep`, `find`, and `ls`. They are not special to the agent loop. Each is a
TypeBox-schema-backed tool definition wrapped into an `AgentTool`, which is why
extensions can observe, replace, or add tools through the same lifecycle.

Implementation lives in `packages/coding-agent/src/core/tools/`. The public
construction helpers are re-exported from the coding-agent package. This
chapter focuses on behavior that matters when using, extending, or debugging
the tools, not on teaching shell commands.

## Tool registry and default selection

`core/tools/index.ts` is the tool factory map. It defines the `ToolName` union
and dispatches each name to a `create*ToolDefinition` or `create*Tool` helper.

```text
tool definition: schema + description + execute + optional renderers
       |
       v
wrapToolDefinition(...)
       |
       v
agent-core AgentTool: validated arguments + execution callback
```

The coding-agent SDK creates all definitions so extensions can see metadata,
but the initial active set is the four coding tools: `read`, `bash`, `edit`,
and `write`. `grep`, `find`, and `ls` are available definitions but not part of
that default active list. Explicit tool selection, `--no-builtin-tools`,
extensions, or `pi.setActiveTools()` can change what the model receives.

## Shared design principles

Every built-in tool follows the same broad contract:

1. A TypeBox schema validates the model's arguments in `agent-loop.ts`.
2. `execute` receives an ID, parsed params, an optional abort signal, and an
   optional incremental-update callback.
3. Success returns model-facing content plus optional structured `details`.
4. Failure throws; the agent loop converts it into an error tool result.
5. Optional renderers turn call/result data into a compact TUI row.

Tool descriptions and `promptGuidelines` are not decoration. Coding-agent uses
active tools' snippets/guidelines while rebuilding the system prompt. For
example, `read` tells the model to inspect files with `read` rather than `cat`
or `sed`; `write` says it is for new files or complete rewrites.

## Output limits protect the next provider turn

Large tool output can consume the same context window needed for reasoning and
response. `truncate.ts` defines a shared 50 KB / 2,000 line limit, along with
head/tail/line helpers and metadata describing why output was truncated.

The correct truncation direction depends on the tool:

| Tool use | Useful part | Typical policy |
|---|---|---|
| File content or search result | Beginning/ordered matches | Keep head |
| Long-running command log | Latest state/error | Keep tail |
| Individual match line | Prefix is usually enough | Truncate line |

Built-in tools report truncation to the model and provide an actionable next
step. A custom tool must do the same. Quietly dropping output makes a model
repeat a command or assume a partial result is complete.

## `read`: text ranges and image attachments

`read.ts` accepts a required `path` plus optional 1-indexed `offset` and
`limit`. It resolves the path relative to the session cwd, checks readability,
then branches on MIME detection.

For text files it splits lines, applies the requested range, and then applies
shared truncation. The continuation messages are deliberately specific:

```text
Showing lines 1-2000 of 6500. Use offset=2001 to continue.
```

If a single first line exceeds the byte limit, it reports that condition and
suggests a bounded shell fallback. That avoids pretending the complete line
was read.

For supported images (JPEG, PNG, GIF, WebP, BMP), it reads binary data,
processes/resizes it subject to settings, and returns an image content block
alongside a text note. The tool checks the current model's input capabilities
and adds a note when the model cannot receive images. The global
`images.blockImages` setting adds another later defense in the conversion path.

### Read operations are pluggable

`ReadOperations` has `readFile`, `access`, and optional MIME detection. The
default uses the local filesystem. An extension can build a read tool backed by
SSH, a container, or another remote system by providing those operations. The
schema and TUI behavior stay the same; only the I/O boundary changes.

That is preferable to reimplementing the full read tool merely to redirect
filesystem access.

## `write`: create or replace a complete file

`write.ts` accepts `path` and full `content`. It creates parent directories
recursively, overwrites existing content, and returns a byte count. Its prompt
guideline explicitly reserves it for new files or complete rewrites.

The implementation wraps the complete operation in `withFileMutationQueue`:

```text
resolve target path
  -> wait for same-file queue
  -> check abort
  -> mkdir parent
  -> check abort
  -> write file
  -> check abort
  -> release queue
```

The queue is keyed by a resolved, canonical path. Existing paths are passed
through `realpath`, so symlink aliases share a queue; missing target paths use
the resolved absolute path. Different files still execute in parallel.

Notice the abort design: the write tool does not reject from an abort-event
listener and release the queue while a filesystem operation is still in flight.
It checks `signal.aborted` between awaited operations, keeping the queue held
until the current operation has settled. A custom mutating tool should follow
the same model.

## `edit`: exact replacement and queued mutation

`edit.ts` is for targeted changes to an existing file. Its schema is built
around a path and edit blocks with exact old/new text. The exact-match approach
lets Pi detect ambiguous or stale replacements rather than applying a fuzzy
patch to the wrong location.

Like `write`, `edit` resolves the target and wraps the full read-modify-write
window in `withFileMutationQueue`. Queue only the final write and two parallel
tools can both read old content, derive conflicting replacements, then lose one
change. The lock must cover reading, validation, transformation, and write.

Use `edit` rather than `write` when preserving unrelated content is important.
Use `write` when creating a file or deliberately replacing its entirety. The
model prompt guidelines encode this distinction to reduce destructive rewrites.

## `bash`: process execution with streamed tail output

`bash.ts` accepts a command and optional timeout in seconds. It uses
`createLocalBashOperations()` by default, which resolves the configured shell,
verifies the cwd, starts a child process with stdout/stderr pipes, tracks a
detached process tree where the platform supports it, and kills that tree for
cancellation or timeout.

The tool can apply a configured `shellCommandPrefix` and an extension-provided
`spawnHook` that changes command, cwd, or environment. This is the intended
seam for sandbox/container/remote routing; it avoids a brittle string rewrite
inside a generic `tool_call` hook.

### Streaming output

`OutputAccumulator` collects stdout and stderr. While a process runs, `bash`
throttles partial `onUpdate` events to roughly one every 100 ms. The TUI can
show a live preview while the model waits for final output.

Final command output keeps the tail. If output was truncated, the tool saves
the full output to a temporary file and includes that path in `details` and a
model-facing notice. Nonzero exit status, timeout, and abort become thrown
errors with accumulated output appended where available. The agent loop then
returns that failure as an error tool result to the model.

This is why a failed command can still be useful: the model receives the log
and exit status rather than merely an exception string.

## `grep`, `find`, and `ls`: bounded repository discovery

These search/listing tools use the cwd and respect ignore policy. Their
schemas make the intended scope explicit:

| Tool | Important inputs | Default limit | Result shape |
|---|---|---:|---|
| `grep` | pattern, optional path/glob, literal/case/context options | 100 matches | File path and line-numbered matching lines |
| `find` | glob pattern, optional path | 1,000 results | Matching paths relative to search directory |
| `ls` | optional directory path | 500 entries | Alphabetical listing, directory suffix `/`, includes dotfiles |

All three also use the shared byte cap. `grep` additionally limits long output
lines. The tools are better suited to agent context than raw `rg`/`find` shell
output because their limits and formatting are predictable.

They are exposed as built-in names and can be registered/activated by an
extension. If you override one, retain its result details and renderer contract
unless you intentionally own every affected display/session behavior.

## Paths, cwd, and remote operations

Tools must distinguish what the model typed from the actual target path. The
built-ins resolve relative paths against the session cwd, not an arbitrary
process cwd, and format paths for display relative to the session when useful.

Remote execution does not mean every tool must become shell text. The exported
operations interfaces make I/O replaceable per tool:

```text
ReadOperations    readFile + access + optional MIME detection
WriteOperations   mkdir + writeFile
EditOperations    read/write/mkdir-style file mutation operations
BashOperations    exec with data callback, cwd, signal, timeout, env
Grep/Find/Ls      operation interfaces for filesystem/search behavior
```

An extension can create a tool with remote operations, register it under the
same name, and leave Pi's validation/rendering behavior intact. The extension
documentation's SSH and sandbox examples demonstrate this pattern.

## User shell commands are separate from model `bash`

In interactive mode, input beginning with `!` runs a user shell command;
`!!` marks its result excluded from LLM context. The extension `user_bash`
event can provide custom operations or a complete result for that path.

That is not identical to an assistant's `bash` tool call. The user is directly
requesting execution, while a tool call is model-generated and is represented
in the assistant/tool-result protocol. A sandbox extension that wants
consistent behavior often handles both: it supplies operations for built-in
tools and for `user_bash`.

## Tool rendering is a first-class interface

Built-in definitions include `renderCall` and `renderResult` functions. The
interactive tool row receives current arguments, partial/final state, expanded
flag, prior component, execution time, and an `invalidate()` callback. This
permits syntax-highlighted read/write previews, compact default rendering with
an expand hint, streamed shell output, and error-specific presentation.

An extension that overrides built-in execution can omit a renderer and inherit
the built-in renderer per slot. That convenience depends on keeping the
expected call args/result details shape. If an override changes those shapes,
provide compatible renderers or a self-contained rendering shell.

## Tool lifecycle and extension interception

The agent loop emits this sequence for each tool call:

```text
tool_execution_start
  -> schema validation and tool_call extension gate
  -> execute / tool_execution_update ...
  -> tool_result extension patch
  -> tool_execution_end
  -> final toolResult message events
```

In default parallel mode, start/preflight is source-ordered, execution/update
events can interleave, and final tool-result messages are emitted in assistant
source order. Do not build a custom tool that assumes another tool's result is
already in `ctx.sessionManager` during its own `tool_call` handler.

## A safe custom file-mutation skeleton

The following is a shape, not a complete product tool. It shows the required
ordering for an extension that changes one file.

```ts
import { withFileMutationQueue } from "@earendil-works/pi-coding-agent";
import { readFile, writeFile } from "node:fs/promises";
import { resolve } from "node:path";

async function replaceOnce(cwd: string, path: string, oldText: string, newText: string, signal?: AbortSignal) {
  const target = resolve(cwd, path);
  return withFileMutationQueue(target, async () => {
    if (signal?.aborted) throw new Error("Operation aborted");
    const current = await readFile(target, "utf8");
    if (signal?.aborted) throw new Error("Operation aborted");
    if (!current.includes(oldText)) throw new Error("Expected text was not found");
    await writeFile(target, current.replace(oldText, newText), "utf8");
    if (signal?.aborted) throw new Error("Operation aborted");
  });
}
```

In a real tool, add a TypeBox schema, precise description, result content,
truncation where relevant, and tests for no-match, abort, and concurrent calls.

## Tool-author review questions

Before exposing a custom tool to a model, answer:

1. Does the TypeBox schema constrain the actual safe input shape?
2. Is the tool active only when its prompt guidance is appropriate?
3. Does every large output have a bounded, actionable representation?
4. Does cancellation reach all long-running work?
5. Are writes serialized across aliases of the same target?
6. Does a thrown error give the model enough detail to recover?
7. Are `details` durable and renderer-compatible?
8. Does the behavior work in parallel with built-in tools?
9. Is a policy gate better implemented as `tool_call` interception than inside
   the tool itself?

Treat tools as a public protocol between the model, session, UI, and extension
ecosystem—not as a private helper function.
