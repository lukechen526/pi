# Glossary

**Agent**

: The stateful class in `packages/agent` that owns a model, system prompt,
  messages, active tools, queues, and one active run.

**AgentSession**

: Coding-agent policy around an `Agent`: prompt preprocessing, resources,
  extension bridge, system prompt, sessions, retries, and compaction.

**AgentSessionRuntime**

: Owner for one live `AgentSession` and its cwd-bound services. It replaces the
  runtime during new/resume/fork flows.

**Assistant message event stream**

: Async stream from `pi-ai` that normalizes provider streaming events into
  start, deltas, completion, and error states.

**Compaction**

: Summarizing older context into a durable session entry so recent context can
  fit in a model window. It is not simple transcript deletion.

**Context**

: The provider-facing bundle of system prompt, converted messages, and active
  tools. It is distinct from the broader persisted session.

**Extension**

: A runtime-loaded TypeScript/JavaScript factory module that receives
  `ExtensionAPI` and registers behavior.

**Extension context**

: The `ctx` passed to hooks/tools/commands, containing mode, cwd, UI, session,
  model registry, and potentially an abort signal.

**Follow-up**

: A queued user/custom message delivered after the agent would otherwise
  finish, rather than between tool turns.

**Model registry**

: Coding-agent service that knows models, providers, authentication, OAuth,
  and dynamic provider registration.

**Resource loader**

: Coding-agent service that discovers/loads extensions, skills, prompt
  templates, themes, and context files subject to trust policy.

**SessionManager**

: JSONL-tree persistence and context reconstruction API for a coding-agent
  session.

**Steering**

: A queued message delivered after the current assistant turn and tool batch,
  before the next provider request.

**Tool**

: A model-callable operation with a TypeBox parameter schema and async execute
  function. Built-in and extension tools share the agent-loop boundary.

**Tool result**

: A durable message produced by a tool execution. Its content goes to the
  model; details carry structured metadata for renderers/state.

**TUI**

: Terminal user interface. Pi's reusable TUI primitives live in
  `packages/tui`; coding-agent composes them into the interactive application.
