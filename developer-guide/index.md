# Welcome to the Pi codebase

This guide is for a developer who can read Python and JavaScript, but does not
yet use TypeScript every day. Its job is practical: by the end, you should be
able to trace a prompt through Pi, find the right layer for a change, and build
an extension without guessing at the API.

This is a guide to the repository as checked out, not a replacement for the
published user documentation. File paths throughout are repository-relative.
They are deliberately specific so that you can open the implementation beside
this book. When a section simplifies a control flow, it says so; the named
source files are the authority for edge cases.

## What Pi is

Pi is a TypeScript monorepo for an agent harness and a self-extensible coding
agent. The root package is an npm workspace. The interactive `pi` command is
implemented by `@earendil-works/pi-coding-agent`; it composes a terminal UI,
an agent runtime, and a provider-neutral AI API.

The four primary packages are:

| Package | Role | Start reading at |
|---|---|---|
| `packages/tui` | Reusable terminal UI primitives and differential rendering | `src/index.ts`, `src/tui.ts` |
| `packages/ai` | Models, provider dispatch, streaming API adapters, authentication primitives | `src/index.ts`, `src/types.ts`, `src/compat.ts` |
| `packages/agent` | Provider-independent agent loop, message state, tool execution | `src/agent.ts`, `src/agent-loop.ts`, `src/types.ts` |
| `packages/coding-agent` | CLI, interactive application, sessions, built-in tools, configuration, extensions | `src/main.ts`, `src/core/sdk.ts`, `src/core/agent-session.ts` |

`packages/orchestrator` is an experimental package depending on the coding
agent. It is useful when you specifically need its process orchestration
surface, but it is not on the normal interactive prompt path described here.

## A mental model in one diagram

```text
terminal input
    |
    v
coding-agent interactive mode ---------------------+
    |                                               |
    v                                               v
AgentSession: commands, sessions, resources    extension runtime
    |                                               |
    v                                               |
agent-core: context, streaming, tools <-------------+
    |
    v
pi-ai: model dispatch and provider stream
    |
    v
provider API (OpenAI, Anthropic, Google, ...)
```

The arrows point in the usual runtime direction. The extension runtime has
hooks at several boundaries; it does not bypass the agent loop merely because
it can add tools or alter a request.

## Reading order and outcomes

1. [TypeScript used by Pi](01-typescript-for-pi.md) makes the type syntax and
   module conventions readable.
2. [Architecture](02-architecture.md) maps packages, ownership, persistence,
   and runtime entry points.
3. [Prompt lifecycle](03-prompt-lifecycle.md) follows one prompt through every
   major runtime boundary.
4. [Extensions](04-extensions.md) explains discovery, hooks, tools, UI, state,
   packaging, and a complete minimal extension.
5. [Customization and workflow](05-customization-and-workflow.md) helps you
   choose an extension, configuration, SDK use, or core change and validates
   your work safely.

The [glossary](glossary.md) is a quick reference for terms that recur across
the codebase.

## Source map used for this guide

The guide was checked against the package manifests, the extension examples,
the coding-agent extension documentation, and these implementation seams:

- CLI composition: `packages/coding-agent/src/main.ts` and
  `src/core/agent-session-services.ts`.
- Session construction and prompt preprocessing:
  `packages/coding-agent/src/core/sdk.ts` and `src/core/agent-session.ts`.
- Low-level loop and tool execution: `packages/agent/src/agent.ts`,
  `src/agent-loop.ts`, and `src/types.ts`.
- Provider dispatch: `packages/ai/src/compat.ts`, `src/models.ts`, and
  `src/types.ts`.
- Extension contract and loading: `packages/coding-agent/src/core/extensions/`
  plus `packages/coding-agent/docs/extensions.md`.

The repository evolves. If a symbol named in this book moves, use
`rg "symbolName" packages` from the repository root, then update your mental
model from the current implementation rather than preserving an old path.
