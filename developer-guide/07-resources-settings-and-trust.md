# Resources, settings, and project trust

Before the first prompt, Pi assembles a runtime from configuration, credentials,
models, extensions, skills, prompts, themes, and context files. This chapter
explains that startup surface so you can change configuration intentionally and
know when a project-local change will be ignored.

Primary implementation files are
`packages/coding-agent/src/core/settings-manager.ts`,
`src/core/resource-loader.ts`, `src/core/trust-manager.ts`, and
`src/core/agent-session-services.ts`. The user-facing configuration reference
is `packages/coding-agent/docs/settings.md`.

## The startup services boundary

`createAgentSessionServices()` creates the cwd-bound services needed for a
session:

```text
cwd + agent directory
  -> SettingsManager
  -> AuthStorage
  -> ModelRegistry
  -> DefaultResourceLoader
       -> reload resources subject to trust
  -> apply pending provider registrations from extension factories
```

This separation is deliberate. A resumed session can have a different cwd than
the process's initial cwd, so settings, project resources, and models must be
created for the effective session cwd. The `AgentSessionRuntime` rebuilds this
service bundle on a session replacement rather than pretending that settings
from the old project apply to the new one.

## Global and project settings

Pi reads JSON settings from two conventional scopes:

| File | Scope | Relative resource paths resolve from |
|---|---|---|
| `~/.pi/agent/settings.json` | All projects for the current user | `~/.pi/agent` |
| `.pi/settings.json` | The current project | `.pi` |

Project settings override global settings. Nested objects are merged, so a
project can adjust only one compaction field while retaining the global
default for the rest:

```json
// global
{ "compaction": { "enabled": true, "reserveTokens": 16384 } }

// project
{ "compaction": { "reserveTokens": 8192 } }
```

The effective configuration has `enabled: true` and `reserveTokens: 8192`.
Use this pattern for small project adjustments rather than copying a full
settings file that becomes stale as Pi adds defaults.

## Trust is an execution policy, not a cosmetic prompt

Project-local settings can declare extensions/packages and project skills can
instruct a model to execute commands. Pi therefore does not automatically run
project resources in an unknown folder.

Interactive startup checks whether a project contains trust-requiring resources
and whether a saved decision applies to that folder or a parent. In the absence
of a decision it can ask. Trust enables project `.pi/settings.json`, project
resources, missing project package installation, and project extension code.

Noninteractive modes cannot ask. They follow global `defaultProjectTrust`:

| Value | Noninteractive behavior without saved decision |
|---|---|
| `"ask"` | Ignore project resources because there is no UI prompt |
| `"never"` | Ignore project resources |
| `"always"` | Trust project resources |

`--approve`/`-a` and `--no-approve`/`-na` override trust for one run. `/trust`
saves a decision for future starts but does not retroactively reload the
current runtime; restart or use the appropriate workflow after changing trust.

Extensions receive `project_trust` before project-local extensions are loaded.
Only global/user or CLI-provided extensions are eligible at this point. A
handler returns `trusted: "yes" | "no" | "undecided"`; the first yes/no result
owns the decision. This prevents a project extension from approving its own
execution before Pi has decided to load it.

## Resource categories

The resource loader assembles several independently useful kinds of content:

| Resource | What it changes | Typical discovery location |
|---|---|---|
| Extension | Runtime behavior, tools, commands, UI, providers | `extensions/`, configured paths |
| Skill | Reusable instruction bundle and optional `/skill:name` command | `skills/`, `.agents/skills` |
| Prompt template | User-invoked prompt expansion | `prompts/`, configured paths |
| Theme | Terminal styling data | `themes/`, configured paths |
| Context file | Instructions added to system-prompt construction | `AGENTS.md` and related loader results |
| Package | Bundle containing the above | npm/git/local source |

These are not interchangeable. A skill is model instruction; an extension is
executable code; a theme is presentation. If a customization only needs to
tell the model how to work, use a skill/prompt/context file rather than an
extension with full process permissions.

## Explicit resource paths and patterns

The settings keys `extensions`, `skills`, `prompts`, and `themes` accept local
file paths or directories. `packages` adds package sources. Arrays support
glob patterns plus filtering syntax:

| Syntax | Meaning |
|---|---|
| `pattern` | Include matching paths |
| `!pattern` | Exclude matching paths |
| `+path` | Force-include this exact package-relative path |
| `-path` | Force-exclude this exact package-relative path |

Keep path resolution in mind when moving configuration between scopes: a
relative global path and a relative project path have different base
directories. Use absolute paths or `~` only when the resource truly belongs to
one machine, not to a repository.

## Skills and prompt templates on the input path

`AgentSession.prompt()` gives extension commands first chance to handle slash
input. It then emits the extension `input` hook. If input is not handled,
Pi expands skill commands and prompt templates. That means a raw `/skill:name`
is not necessarily the message sent to the model; it can become the skill's
content plus user arguments.

`enableSkillCommands` controls whether skills are registered as commands. A
skill can still be loaded as system-prompt resource even if its direct command
surface is disabled, depending on how it is supplied. When debugging prompt
content, inspect both `ctx.getSystemPromptOptions()` and the actual context
event rather than searching only the typed text.

## Context files and system-prompt construction

The loader provides loaded context files and skills to `AgentSession`.
`_rebuildSystemPrompt()` creates structured `BuildSystemPromptOptions` from:

- cwd;
- selected tool names, snippets, and guidelines;
- custom/append system-prompt CLI options;
- loaded skills;
- loaded context files.

`buildSystemPrompt()` turns those options into provider-independent system
prompt text. The system prompt is rebuilt when the active tool set changes and
when session resources reload. Extensions that replace the prompt in
`before_agent_start` receive the already-built prompt and can chain a change,
but should avoid discarding these inputs unintentionally.

## Settings that alter runtime behavior

Settings are more than UI preferences. The following groups affect runtime
semantics and should be considered part of a reproducible project setup.

### Model and thinking

`defaultProvider`, `defaultModel`, `defaultThinkingLevel`,
`enabledModels`, and `thinkingBudgets` influence startup selection, model
cycling, and provider options. A saved session model takes precedence during a
resume if it is still configured and authenticated; defaults are a fallback.

### Compaction and branch summary

`compaction.enabled`, `reserveTokens`, and `keepRecentTokens` determine when
and how much conversation is summarized. `branchSummary.reserveTokens` and
`skipPrompt` affect tree navigation. These settings change what reaches a
provider, so they are not merely transcript retention preferences.

### Retry and delivery

The `retry` block controls agent-level retry and provider request settings.
Provider retries default to zero; agent-level retry is separate. Keeping them
distinct avoids hiding a long provider retry delay from the session-level logic.

`steeringMode` and `followUpMode` select whether queued messages drain all at
once or one at a time. `transport`, `httpIdleTimeoutMs`, and
`websocketConnectTimeoutMs` influence provider transport behavior.

### Shell, images, and terminal

`shellPath`, `shellCommandPrefix`, and `npmCommand` affect process execution
and package management. Image settings control resize/block behavior before a
provider request. Terminal image and rendering settings affect display but do
not make an image model-capable; model input capabilities still control whether
an image can be sent.

## Project configuration that an extension should respect

An extension with its own project-level configuration should:

1. Use `CONFIG_DIR_NAME` from coding-agent rather than hardcoding `.pi`.
2. Resolve the path relative to `ctx.cwd`.
3. Call `ctx.isProjectTrusted()` before reading that configuration.
4. Validate parsed JSON as untrusted data.
5. Define which state is global, project-local, and session-local.

For example, a global extension may store user preferences in the agent
directory but read a project allowlist only after trust. That matches Pi's own
security model and avoids creating an unreviewed bypass.

## Packages are resource delivery, not a new execution mode

Pi packages distribute extensions, skills, prompts, and themes from npm, git,
or a local path. A package can declare a `pi` manifest in `package.json` or use
conventional subdirectories. The package manager records sources in settings;
the resource loader then finds the declared resources.

Package scope follows settings scope:

- Global installs are user-wide and live under the agent directory.
- Project installs are stored under `.pi` and are trust-gated.
- A local path is loaded in place, so edits affect the next load/reload.

`pi install`, `pi remove`, `pi list`, and `pi update --extensions` operate on
this package layer. They do not change the fact that an extension is regular
code with the privileges of the Pi process.

## Themes and the TUI boundary

Theme paths load JSON themes. Themes influence colors/formatting used by
interactive components and extension renderers; they do not modify keybindings
or component logic. An extension can query/change themes through `ctx.ui`, but
should use semantic theme keys (`accent`, `warning`, `toolTitle`, and so on)
instead of ANSI constants so the user's theme continues to work.

## Reload semantics

`/reload` reloads extensions, skills, prompts, themes, and context files. It
does not preserve arbitrary extension closures. The documented lifecycle is:

```text
session_shutdown (old extension runtime)
  -> resources reloaded
  -> session_start { reason: "reload" } (new runtime)
  -> resources_discover { reason: "reload" }
```

Code after `await ctx.reload()` still executes in the old command call frame.
Treat reload as terminal for that handler. Rebuild all session-scoped state in
the new `session_start` path.

## Configuration-review checklist

Before committing a project `.pi/settings.json` or package declaration, check:

1. Does this setting execute code or fetch/install dependencies?
2. Does the team intend to trust the repository for that behavior?
3. Are environment references used for secrets instead of literal credentials?
4. Are relative paths intentional for this scope?
5. Does a nested override preserve the global fields you expect?
6. Will noninteractive CI receive the required trust decision?
7. Can the change be represented as a skill/theme/template instead of code?

These questions make configuration review an architectural step rather than
an afterthought.
