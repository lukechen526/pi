# Terminal UI and interactive mode

Pi's interactive application is built from a reusable terminal UI package and
a coding-agent-specific composition layer. The separation is worth learning:
an extension normally customizes the coding-agent UI surface, while a
contribution to `packages/tui` changes primitives that could serve another
terminal application.

Relevant files include `packages/tui/src/tui.ts`, `src/terminal.ts`,
`src/keybindings.ts`, `src/autocomplete.ts`, and
`packages/coding-agent/src/modes/interactive/interactive-mode.ts`.

## The TUI package owns terminal mechanics

The public barrel `packages/tui/src/index.ts` exports the `TUI` class,
component/container interfaces, text/markdown/editor/list components, terminal
implementations, key parsing/matching, keybindings, autocomplete, image
support, and ANSI-aware text utilities.

The `TUI` class is responsible for the component tree, focus, layout, input
routing, overlays, terminal lifecycle, and differential screen updates. A
component returns lines for an available width and can invalidate itself when
its rendered state changes. This is a retained component model, not immediate
`console.log` output.

## Differential rendering in practical terms

Terminal rendering is expensive and visually disruptive if every update clears
the whole screen. The TUI tracks prior lines and writes only the changes needed
to transform the terminal into the newly rendered view.

```text
component tree changes
  -> render visible lines with ANSI styling
  -> compare with previous terminal frame
  -> emit minimal cursor movement/text changes
```

That design enables token-by-token assistant updates, tool-output streaming,
and autocomplete without a full redraw for every byte. It also explains why
components should invalidate only when necessary and reuse component instances
for high-frequency partial updates where the API supports it.

## Components are composition, not application policy

The TUI exports general components such as `Text`, `Box`, `Container`,
`Markdown`, `Editor`, `Input`, `SelectList`, `SettingsList`, `Loader`, and
`Image`. Their job is layout/presentation/input behavior. They do not know
what a session, provider, or tool result means.

Coding-agent builds a chat-specific component tree around those primitives:

```text
InteractiveMode
  -> header/status regions
  -> chat transcript container
       -> user/assistant/thinking/tool components
  -> pending messages/tool area
  -> widgets above/below editor
  -> editor or extension replacement editor
  -> footer and overlays/dialogs
```

This means a custom transcript renderer should return a TUI `Component`; it
does not need to manage raw terminal escape sequences itself.

## Terminal lifecycle and capabilities

`ProcessTerminal` and terminal support code manage stdin/stdout, raw mode,
size information, terminal capabilities, and cleanup. Image rendering is
capability-dependent: Pi can choose Kitty, iTerm2, or fallback behavior based
on what the terminal supports.

Interactive mode initializes terminal/theme behavior before starting its event
loop and stops/restores terminal state at shutdown. If you add a feature that
opens an external editor, subprocess, or custom modal, follow the existing
interactive-mode lifecycle rather than leaving raw terminal state altered.

## Keyboard input is parsed before it becomes a binding

`keys.ts` parses terminal input into key semantics and `keybindings.ts` maps
semantic keybinding IDs to user-configurable keys. A binding ID is stable,
namespaced policy such as:

```text
tui.input.submit
tui.editor.cursorWordLeft
tui.select.confirm
app.interrupt
app.tools.expand
app.model.cycleForward
```

An extension should display hints with `keyHint("app.tools.expand", "to expand")`
or inspect the injected `KeybindingsManager`. It should not hardcode raw
`ctrl+x` checks. Hardcoding bypasses user configuration and breaks the mapping
between the keybindings file, help text, and actual behavior.

## Editor submission and queue semantics

The interactive editor is not just a text box. Its submission handler handles:

- built-in slash commands;
- `!`/`!!` user shell commands;
- compaction-time message holding;
- steering/follow-up delivery while an agent is active;
- normal queued input for the main `run()` loop;
- editor history, multiline text, image paste, autocomplete, and external edit.

The high-level outcome is simple: an idle normal submission resolves
`getUserInput()`, whose loop calls `session.prompt(text)`. While streaming,
regular Enter sends a steering prompt; Alt+Enter sends a follow-up prompt.
The detailed handling stays in `InteractiveMode` so `AgentSession` can serve
print/RPC clients without importing terminal components.

## Transcript rendering from agent events

`InteractiveMode.subscribeToAgent()` receives coding-agent session events.
`handleEvent()` dispatches behavior by type:

| Event | UI responsibility |
|---|---|
| `agent_start` | Clear pending tool bookkeeping and begin activity state |
| `message_start` | Create/update the appropriate transcript row |
| `message_update` | Refresh partial assistant/thinking/tool-call display |
| `message_end` | Finalize transcript component and session-visible output |
| tool execution events | Create/update/complete a tool row |
| `turn_end` / `agent_end` | Final activity/footer/pending-message updates |

The UI works from events rather than polling the model. This preserves the
same visual behavior whether an assistant emits text, thinking, tool calls,
errors, or a synthetic retry/compaction message.

## Tools in the transcript

Tool rows are composed by the interactive components around a tool definition's
`renderCall` and `renderResult` slots. The context passed to those renderers
includes current args, a row-local mutable state object, optional last component,
partial/final flags, expanded state, show-images preference, cwd, timing, and
an `invalidate()` method.

This produces a useful layering:

```text
tool definition knows how to render its call/result data
interactive mode knows where the row belongs in chat and when it updates
TUI knows how to layout/diff-render the resulting components
```

For an extension tool, return a compact default component and make details
available when expanded. For a high-frequency partial result, reuse
`context.lastComponent` if safe instead of creating a new large component on
every stream update.

## Extension UI surface

`ctx.ui` is an adapter owned by `InteractiveMode`. It gives extensions a
controlled surface rather than direct access to private chat/editor fields:

| API family | Examples |
|---|---|
| Dialogs | `select`, `confirm`, `input`, `editor` |
| Notices/status | `notify`, `setStatus`, working message/indicator controls |
| Editor/widgets | `setEditorText`, `pasteToEditor`, `setWidget`, custom editor |
| Transcript/rendering | message/entry renderers, tool expansion controls |
| Look and completion | themes, autocomplete provider, custom footer |
| Complex interaction | `custom(factory)` and experimental overlays |

Use `ctx.hasUI` for methods that require user interaction and `ctx.mode ===
"tui"` before terminal-only behavior. In print/JSON modes, a dialog cannot
block for input. In RPC mode, Pi can map some UI operations to protocol events
but custom terminal components are not an equivalent surface.

## Custom components and focus

`ctx.ui.custom()` temporarily replaces the editor with a factory that receives
the TUI instance, current theme, a keybindings manager, and a `done(value)`
callback. The component handles input until it calls `done`.

```ts
const answer = await ctx.ui.custom<boolean>((_tui, _theme, keybindings, done) => {
  const component = new Text("Enter confirms; Escape cancels", 1, 1);
  component.onKey = (key) => {
    if (keybindings.matches(key, "tui.select.confirm")) done(true);
    if (keybindings.matches(key, "tui.select.cancel")) done(false);
    return true;
  };
  return component;
});
```

The example uses named bindings, so a user who rebinds confirm/cancel receives
the expected behavior. More complex overlays have focus/visibility APIs; use
them only when an editor replacement/dialog cannot express the interaction.

## Autocomplete composition

The TUI supplies autocomplete interfaces and a combined provider. Pi's
interactive editor layers slash-command and path completion. An extension can
add a provider by receiving the current provider and delegating when its own
trigger syntax does not apply.

The reliable pattern is:

1. Inspect text before the cursor.
2. If extension syntax matches, return an explicit prefix and candidate list.
3. Otherwise call the previous provider.
4. Delegate completion application unless custom insertion is necessary.
5. Preserve existing file-completion decisions unless deliberately changing it.

This prevents an extension's `#issue` or `$variable` completion from disabling
normal slash/path completion elsewhere in the line.

## Themes and semantic styling

Theme functions provide semantic styles such as `accent`, `muted`, `dim`,
`success`, `warning`, `error`, and `toolTitle`. Use `theme.fg(name, text)` plus
style helpers like `theme.bold(text)` in renderer code.

Do not embed fixed ANSI colors in an extension renderer. A literal color may be
unreadable in a user's light/custom theme and cannot participate in Pi's visual
language. Semantic keys let an extension look native across themes.

## Images and rendered width

Terminal width is measured in display cells, not JavaScript string length.
Wide CJK characters, ANSI escape sequences, hyperlinks, and image rows make
naive `text.length` calculations wrong. The TUI exports utilities such as
`visibleWidth`, `truncateToWidth`, and ANSI-aware wrapping. Use them in custom
components rather than slicing strings manually.

Likewise, terminal image support is capability- and cell-size-dependent. Use
the TUI `Image` component/terminal image helpers rather than trying to print a
provider image's base64 data as text.

## Debugging visual issues

When an interactive feature is wrong, isolate its layer:

1. Capture the agent/session event sequence; verify data before rendering.
2. Inspect the tool/message renderer output as a component.
3. Check available width and ANSI-aware visible width.
4. Confirm the component invalidates after its state changes.
5. Test narrow terminal widths and long/CJK/ANSI-rich content.
6. Test focus and cancel paths for custom UI.
7. Use `/debug` for rendered TUI lines and the last model messages.

The correct fix is usually at the narrowest layer that owns the defect: a tool
renderer for tool presentation, interactive mode for chat composition, or the
TUI package for a general terminal layout/input primitive.
