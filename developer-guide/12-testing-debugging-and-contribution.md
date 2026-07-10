# Testing, debugging, and contribution workflow

Architecture knowledge becomes useful when it shortens a safe debugging loop.
Pi's repository rules distinguish code checks, targeted tests, interactive
testing, and real-provider behavior. Follow the narrowest useful validation
path; do not make a provider call merely to prove a local state transition.

The authoritative local rules are in `AGENTS.md`. Coding-agent also documents
development behavior in `packages/coding-agent/docs/development.md` and
extension behavior in `packages/coding-agent/docs/extensions.md`.

## Start with a reproducible boundary

For a report such as "Pi ignored my extension" or "a tool result rendered
wrong," identify which boundary is wrong before changing code:

| Symptom | First boundary to inspect |
|---|---|
| Extension is absent | Trust decision, resource loader discovery, factory load error |
| Command runs but prompt is unchanged | Extension command/input/before-agent event path |
| Model gets wrong messages | Session context, `context` hook, `convertToLlm`, provider payload |
| Tool has invalid args | Schema, `prepareArguments`, model tool-call block |
| Tool output is lost | Truncation, tool result, parallel ordering, renderer |
| Provider call fails | Model registry/auth, adapter response, retry configuration |
| UI is stale or clipped | Session event, renderer component, TUI invalidation/width |
| Resume/fork loses state | Session branch/context, extension reconstruction, stale replacement closure |

This map prevents a common waste of time: editing the TUI when the agent never
emitted a message event, or editing a provider adapter when an extension never
loaded due to project trust.

## Static checking versus tests

After code changes, repository policy calls for `npm run check`. It formats and
checks the workspace, verifies pinned dependencies/import rules/shrinkwrap
state, and runs the root TypeScript check. It is a broad code-quality gate, not
the test suite.

Tests prove behavioral expectations. Do not use `npm run build` as a proxy for
tests, and do not run the full Vitest suite casually: the repository warns that
some end-to-end tests can activate when provider/auth environment variables are
present. For broad non-LLM coverage, use `./test.sh` from the repository root;
for a targeted coding-agent test, run the Vitest CLI from the package root as
described by `AGENTS.md`.

## Test at the correct package

| Change area | First test location/type |
|---|---|
| Provider serialization/stream parsing | `packages/ai/test/` unit/regression tests |
| Agent tool scheduling/message events | `packages/agent` tests or harness tests |
| TUI width, key, component behavior | `packages/tui/test/` |
| Coding-agent sessions/config/extensions | `packages/coding-agent/test/` |
| Agent behavior without a paid provider | `packages/coding-agent/test/suite/` with faux provider |

The coding-agent suite harness is intentional. For tests under
`packages/coding-agent/test/suite/`, use `test/suite/harness.ts` and the faux
provider. Real model keys introduce cost, nondeterminism, rate limits, and a
different failure surface than the behavior being tested.

## High-value test cases by subsystem

### Extensions

- Discovery from global/project/CLI paths and trust gating.
- Async factory completion before provider/model use.
- Correct event ordering and return semantics.
- Tool block/allow behavior for policy hooks.
- Reload cleanup and state reconstruction.
- Mode fallback when `ctx.hasUI` is false.
- Session replacement with a fresh `withSession` context.

### Tools

- Schema validation and legacy `prepareArguments` shape.
- Success, thrown error, abort, and timeout.
- Bounded output and continuation/full-output notices.
- Parallel sibling behavior and same-file mutation queue.
- Renderer partial/final/expanded behavior.

### Sessions and compaction

- JSONL loading/migration and malformed input.
- Tree path selection and label/session-info changes.
- Context rebuild with compaction and branch summaries.
- Model/thinking restoration.
- Resume/fork/new runtime cleanup and extension lifecycle.

### Provider/model registry

- Custom models/overrides merge semantics.
- Auth source precedence and command/env configuration.
- Header merging and `authHeader` behavior.
- Dynamic register/unregister restoration.
- Compatibility flags for a protocol-specific regression.

## Interactive TUI smoke testing

`AGENTS.md` documents a controlled tmux pattern for testing the interactive
application. The important principle is to use a detached terminal with known
dimensions, capture the pane after startup, send text/keys, then cleanly kill
the session. This produces a repeatable artifact for editor, command,
streaming, and rendering behavior without relying on the developer's current
terminal state.

For a UI reproduction, record terminal dimensions, theme and keybinding
overrides, pasted text/image input, exact keys, model/streaming state, and
captured terminal lines or a `/debug` log. That information often explains a
visual issue that a screenshot alone cannot reproduce.

## `/debug` and diagnostic data

The hidden `/debug` command writes `~/.pi/agent/pi-debug.log` with rendered TUI
lines containing ANSI codes and the last messages sent to the LLM. It is a
source of truth for a rendering/context mismatch, but it may contain sensitive
prompts, file contents, and tool results. Treat it as local sensitive data; do
not paste it into a public issue unreviewed.

Extension diagnostics should be equally disciplined. Log an event name, safe
IDs, timing, and sanitized state—not API keys, auth headers, complete provider
payloads, or full context-file content.

## Error classification before retrying

Pi has several retry-like mechanisms with different ownership:

| Failure/condition | Owner | Typical action |
|---|---|---|
| Invalid tool input or tool execution error | Agent loop/model | Return error tool result; model may recover |
| Transport/provider transient error | AgentSession/provider settings | Retry according configured policy |
| Context overflow | AgentSession compaction policy | Compact and retry if applicable |
| User abort | Active run abort controller | Stop abort-aware work; do not auto-retry blindly |
| Extension hook error | Extension runner | Log; tool-call errors fail safe by blocking |
| Project not trusted | Trust manager | Do not load project code/resources |

Misclassifying these leads to bad fixes. Retrying an invalid schema will not
correct model arguments; compacting a missing API key is pointless.

## Source navigation habits

The repository is large enough that broad search results can mislead. A useful
sequence is to find a public symbol with `rg`, read its contract and focused
call sites, identify the package boundary crossed, and then read a nearby test
or example. Change one boundary at a time and add a regression where needed.

Examples are valuable for extensions. Start with
`packages/coding-agent/examples/extensions/hello.ts`, then choose a focused
example such as `permission-gate.ts`, `dynamic-tools.ts`, `entry-renderer.ts`,
or `custom-provider-*` instead of copying a large multi-feature extension.

## Git hygiene in a shared worktree

The repository may have concurrent sessions changing unrelated files. Stage
only explicit paths you changed; never use broad staging commands. Inspect
`git status` before staging/committing, and do not reset, clean, stash, or
checkout unrelated work. The project rules also prohibit commits unless the
user asks.

For generated model metadata, change the generator source rather than editing
`packages/ai/src/models.generated.ts` by hand. That policy preserves the link
between catalog data and its generation process.

## Documentation as a tested interface

This guide is source-grounded documentation. Keep its code examples subject to
the same review questions as implementation:

1. Does every exported symbol exist at the documented package path?
2. Does the example respect actual async/cancellation semantics?
3. Does a lifecycle claim match the runner/agent implementation?
4. Does a configuration example state merge/replace/trust semantics?
5. Does a visual description name the actual renderer/component boundary?

When an API changes, update its focused chapter and the prompt lifecycle
chapter together. The lifecycle is the integration test for the book: it
reveals contradictions between isolated component explanations.

## Pre-review checklist

Before requesting review for a code or extension change:

1. Reproduce the original behavior with a minimal scenario.
2. State the package and public contract that own the fix.
3. Add/run the narrowest regression test or controlled interactive proof.
4. Run the repository-required static check for code changes.
5. Check trust/security implications of new configuration or package loading.
6. Test cancellation, error, and reload/replacement paths where relevant.
7. Inspect diffs for generated/lockfile/dependency changes you did not intend.

This workflow makes Pi's extension power manageable: every change is tied to a
specific runtime boundary and an evidence-producing validation path.
