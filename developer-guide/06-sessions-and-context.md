# Sessions, branches, and durable context

Pi does not treat a chat as a flat array that is overwritten on every new
turn. The coding agent persists an append-only JSONL session tree and rebuilds
the current conversation from a selected leaf. That design supports resume,
fork, branch navigation, compaction, labels, and extension state without
requiring a database.

This chapter is grounded in `packages/coding-agent/src/core/session-manager.ts`,
`packages/coding-agent/src/core/messages.ts`,
`packages/coding-agent/src/core/agent-session-runtime.ts`, and
`packages/coding-agent/docs/session-format.md`.

## Why persistence is a tree

Suppose a user asks for an implementation, then navigates back to an earlier
message and explores a different plan. A linear transcript either loses one
path or duplicates a large prefix into another file. Pi instead stores entries
with an `id` and `parentId`. One file can represent both paths.

```text
header (not a tree node)
  |
  +-- user: "Investigate auth"
        |
        +-- assistant: plan A
              |
              +-- tool result: read auth.ts
                    |
                    +-- user: "Implement A"              <- old leaf
                    |
                    +-- branch summary: "A explored ..."
                          |
                          +-- user: "Try plan B instead" <- current leaf
```

The active branch is the chain from the current leaf to the root, read in
chronological order. Other children are not provider context merely because
they exist on disk. This is a critical distinction when debugging a surprising
model response: inspect the active path rather than the whole session file.

## Files, headers, and locations

By default, coding-agent sessions live under the agent directory, arranged by
working directory. Each persisted session is a `.jsonl` file. The first line is
a `session` header with metadata such as a UUID, timestamp, format version,
and cwd; it has no `id`/`parentId` and is not part of the tree.

The session location can be overridden in settings or by CLI/environment
configuration. The documented precedence is:

```text
--session-dir  >  PI_CODING_AGENT_SESSION_DIR  >  settings.json sessionDir
```

Use `SessionManager.inMemory(cwd?)` when embedding Pi where persistence is not
desired. An in-memory manager still has branches and context construction; it
simply has no session-file path.

## The entry vocabulary

Every non-header entry has a `type`, an ID, a parent ID, and a timestamp.
The following table shows why each entry exists and whether it becomes model
context through `buildSessionContext()`.

| Entry type | Meaning | Becomes LLM context? |
|---|---|---|
| `message` | A standard `AgentMessage`: user, assistant, tool result, or supported coding-agent message | Yes, subject to branch/compaction selection |
| `model_change` | Selected provider/model at that point in history | Used to restore state, not a chat message |
| `thinking_level_change` | Selected thinking level | Used to restore state, not a chat message |
| `compaction` | Summary plus the first entry retained after summarization | Yes, as a compaction-summary message |
| `branch_summary` | Summary of the path left during tree navigation | Yes, as a branch-summary message |
| `custom` | Extension-specific durable data | No |
| `custom_message` | Extension message with display/details | Yes |
| `label` | Named marker attached to an entry | No |
| `session_info` | User-visible session name | No |

This table yields a simple rule for extension authors: use `custom` for state
that Pi or the user needs, and use `custom_message`/`pi.sendMessage()` only
when the model should receive it.

## Message shapes at the provider boundary

The `message` entry wraps an `AgentMessage`. The core roles are:

```text
user          user text and optional images
assistant     streamed text, thinking, and/or toolCall blocks
toolResult    result for a specific assistant tool call
```

Coding-agent adds message kinds for local interaction and synthesized context,
including user bash execution, custom messages, compaction summaries, and
branch summaries. `convertToLlm` and the agent loop decide how these richer
messages become the provider's normalized message types.

An assistant's content is a list of typed blocks. A tool call is one such
block, containing an ID, name, and argument object. The corresponding
`toolResult` refers to the call ID. Retaining that link is essential: a model
provider expects a tool result to answer a particular tool call, not merely a
similarly named tool.

## `SessionManager` as the persistence boundary

The class exposes both static constructors/listing methods and instance
methods. The most useful API groupings are:

| Intent | Methods |
|---|---|
| Start/open | `create`, `open`, `continueRecent`, `inMemory`, `forkFrom` |
| Discover | `list`, `listAll` |
| Append transcript | `appendMessage`, `appendModelChange`, `appendThinkingLevelChange` |
| Append summaries/state | `appendCompaction`, `appendCustomEntry`, `appendCustomMessageEntry`, `appendLabelChange`, `appendSessionInfo` |
| Navigate tree | `getLeafId`, `getBranch`, `getTree`, `getChildren`, `branch`, `resetLeaf`, `branchWithSummary` |
| Build context | `buildContextEntries`, `buildSessionContext` |
| Inspect metadata | `getHeader`, `getSessionName`, `getCwd`, `getSessionFile`, `isPersisted` |

The manager owns the JSONL representation. Do not have an extension write a
second, unrelated copy of the session file: it will not participate in branch
or compaction semantics and may become inconsistent with the transcript.

## Building context from a selected leaf

`buildContextEntries()` is the intermediate view used by the interactive
transcript and session context construction. At a high level it does this:

```text
all persisted entries
       |
       v
walk parent links from active leaf to root
       |
       v
reverse into chronological active path
       |
       v
apply the latest effective compaction boundary
       |
       v
context entries for rendering and conversion
```

When there is no compaction, the active path is the selected context. With a
compaction entry, Pi retains the compaction summary and only the entries from
that entry's `firstKeptEntryId` onward, plus later entries. It does not throw
away the earlier raw records; they remain in the JSONL file for tree history,
but they are intentionally not sent to the LLM.

`buildSessionContext()` goes a step further. It scans the full path to restore
the current model and thinking level, then maps selected entries to context
messages. Custom entries are omitted. This is why storing a private extension
cache in `appendEntry()` does not inflate context length.

## Compaction is a controlled replacement of old context

Context windows are finite. Pi's automatic compaction condition is conceptually:

```text
estimated context tokens > model context window - reserveTokens
```

The default reserve is 16,384 tokens and the default recent-history target is
20,000 tokens, both configurable under `compaction` in settings. The exact
token calculation and cut-point behavior are implemented in the agent
harness; coding-agent coordinates the session operation, extension events, UI,
and retry behavior.

The standard flow is:

1. Locate a valid cut point near the recent-history token target.
2. Collect the older messages to summarize, keeping tool-result relationships
   valid.
3. Ask a model for a structured summary, including previous-summary context
   when necessary.
4. Append a compaction entry with summary, token count, and first kept entry.
5. Rebuild the session context from the summary plus retained recent entries.

Compaction is not the same as deleting history. Its purpose is to preserve an
actionable description of old work while freeing provider context. An extension
can intercept `session_before_compact` to cancel or provide a custom summary,
and observes the final result through `session_compact`.

### Split turns and tool continuity

Normally a compaction cut sits at a turn boundary. A turn begins with a user
message and includes the assistant's response and resulting tool calls until
the next user message. Pi avoids cutting between an assistant tool call and
its tool result because that would make the remaining context internally
incoherent for the provider.

An exceptionally large turn can exceed the recent-history budget by itself.
The compaction design has a split-turn path that summarizes the early prefix
and preserves the later portion. This is why an extension should not assume
compaction always removes only whole, old user turns.

## Branch navigation and branch summaries

`/tree` changes the leaf to a different entry. The old branch may contain work
that is relevant to the new branch but is no longer on its parent chain. Pi can
summarize that abandoned path and append a `branch_summary` at the navigation
point. That summary becomes context on the new path.

```text
old path:    common ancestor -> A -> B -> C
new target:  common ancestor -> D -> E

optional summary of A/B/C is appended near E so the new conversation has
a concise account of the abandoned work without replaying all old messages.
```

`branchSummary.reserveTokens` controls the summarizer's response reserve.
`branchSummary.skipPrompt` controls whether the interactive UI asks before
making a summary. Extensions can provide/cancel summaries with
`session_before_tree` and observe them with `session_tree`.

## Session replacement is not a mutation of the old runtime

New/resume/fork flows use `AgentSessionRuntime`. It owns both the current
`AgentSession` and cwd-bound services such as settings, resources, models, and
auth. A replacement flow deliberately tears down the old runtime and creates a
new one for the destination session.

```text
request /resume or extension command
  -> session_before_switch (may cancel)
  -> session_shutdown on old extension runtime
  -> old AgentSession disposed
  -> services/session created for target cwd
  -> extension runtime rebound
  -> session_start on the new extension instance
  -> optional withSession callback receives fresh context
```

An extension command can call `ctx.newSession`, `ctx.switchSession`, or
`ctx.fork`. Its `withSession` callback receives a fresh replacement context.
Captured old `pi`, command context, or `SessionManager` objects are stale after
the replacement and must not be reused. Capture plain strings/IDs before the
switch, then use only the callback context for session-bound operations.

## Session names, labels, and human navigation

`/name` and `pi.setSessionName()` append `session_info`. The selector then
shows the stored name rather than deriving a name from the first user message.
Labels are per-entry bookmarks, added by `pi.setLabel(entryId, label)`, and
appear in tree navigation. They are lightweight metadata: setting a label does
not edit a historic message or change model context.

Use labels for checkpoints, handoff points, and experiment names. Use a session
name for the overall task. This division makes a large session tree easier to
scan without placing extra prose in provider context.

## Extension persistence patterns

### Stateful tool results

When state is the consequence of a tool call, return a serializable snapshot in
`details` and reconstruct from current-branch tool results in `session_start`.
This keeps state branch-aware.

```ts
pi.on("session_start", (_event, ctx) => {
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type !== "message" || entry.message.role !== "toolResult") continue;
    if (entry.message.toolName !== "project_notes") continue;
    // Rebuild extension state from entry.message.details after validating it.
  }
});
```

### Private durable data

Use `pi.appendEntry("my-extension", data)` when the model does not need the
state. On next `session_start`, scan `getEntries()` or the branch for custom
entries with that `customType`. Register an entry renderer if users should see
the data in the transcript.

### Persistent model context

Use `pi.sendMessage({ customType, content, display })` when the message should
be retained *and* given to the model. Register a message renderer if the
default transcript presentation is insufficient. Do not encode secrets or
verbose debug data in a custom message because it becomes conversation context.

## Session debugging checklist

When a resumed session behaves unexpectedly, answer these in order:

1. Which JSONL file is active (`getSessionFile()`)?
2. Which leaf is selected (`getLeafId()`)?
3. Which entries are in `buildContextEntries()`?
4. Is a compaction or branch summary affecting context?
5. Which model/thinking-level entry is last on the full path?
6. Is the extension reading the current branch or every historic entry?
7. Did a replacement flow leave a stale object captured in a closure?

This approach is faster than treating session persistence as an opaque feature:
the entire model context is explainable as a deterministic view of the active
JSONL tree.
