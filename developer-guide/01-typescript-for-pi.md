# TypeScript used by Pi

Pi is written as native ECMAScript modules in TypeScript. The root
`package.json` sets `"type": "module"`, and source imports include a `.ts`
suffix. The root `tsconfig.json` maps package names to their source entry
points during repository type checking; published packages instead expose
compiled `dist/` files through their package manifests.

You do not need to learn every TypeScript feature before contributing. The
small set below covers the syntax that appears repeatedly in the runtime and
extension API.

## Modules and imports

An ES module exports named values or one default value. Import values normally;
prefix an import with `type` when it exists only to describe data at compile
time. A type-only import disappears from JavaScript output.

```ts
import { Type } from "@earendil-works/pi-ai";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function register(pi: ExtensionAPI) {
  // Type is a runtime value; ExtensionAPI is erased after type checking.
  void pi;
  void Type;
}
```

This is similar to Python importing a protocol only for static checking, except
TypeScript makes the runtime/type distinction explicit. Pi follows this style
throughout; for example, `packages/coding-agent/examples/extensions/hello.ts`
imports `ExtensionAPI` as a type.

When reading repository source, relative imports normally include `.ts`:

```ts
import { Agent } from "./agent.ts";
```

Keep that convention in source packages. Do not introduce dynamic imports just
to avoid a type cycle; Pi's project rules require top-level imports.

## Types describe values; they do not validate values

An `interface` describes the fields a value must have. A `type` can name an
interface-like shape, a union, or a computed type. Neither emits a runtime
validator.

```ts
interface RunOptions {
  timeoutMs?: number;
  signal?: AbortSignal;
}

type RunState = "idle" | "streaming";
```

`?` means optional. The equivalent Python intuition is a dataclass field that
may be omitted, but TypeScript callers can still supply arbitrary JSON unless
you validate it at a boundary. Pi validates LLM tool arguments using schemas,
not just TypeScript annotations.

## Narrowing unions with a discriminant

Pi models events and content as discriminated unions: every variant has a
literal tag such as `type` or `role`. An `if` or `switch` on that tag narrows
the remaining fields safely.

```ts
type Content =
  | { type: "text"; text: string }
  | { type: "image"; mediaType: string; data: string };

function describe(content: Content): string {
  switch (content.type) {
    case "text":
      return content.text;
    case "image":
      return `image: ${content.mediaType}`;
  }
}
```

In Pi, the important real examples are `AgentEvent` in
`packages/agent/src/types.ts`, assistant content blocks in
`packages/ai/src/types.ts`, and session entries in
`packages/coding-agent/src/core/session-manager.ts`. The low-level agent loop
switches on streamed event types such as `"text_delta"`, `"toolcall_end"`,
and `"done"` in `packages/agent/src/agent-loop.ts`.

This pattern is safer than checking for a field with `in`: the tag documents
the protocol and enables exhaustiveness checking when a new variant is added.

## Generics: preserve a relationship between inputs and outputs

Generic parameters are placeholders supplied by callers. Read `T` as "some
type chosen later," not as a runtime value. Pi uses generics heavily to keep a
tool's schema, parsed parameters, and implementation aligned.

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

const result = first(["a", "b"]); // string | undefined
```

The `readonly` modifier says the function promises not to mutate the array.
The result includes `undefined` because TypeScript does not assume an array is
non-empty. Handle it before using a string method:

```ts
if (result !== undefined) {
  result.toUpperCase();
}
```

`Static<typeof schema>` is a common Pi-specific generic pattern. TypeBox
creates a runtime schema and derives a matching static TypeScript type from it.
The `defineTool` helper used in the shipped `hello.ts` example infers the
`params` type from `parameters`, so the execute function sees `params.name` as
a string.

## `unknown` is the honest type for untrusted data

Use `unknown` for data whose shape has not been proven: JSON, extension data,
or a caught error. You must narrow it before using it. Avoid `any`; it turns
off checking and can conceal a bad provider payload or tool result.

```ts
function messageOf(value: unknown): string {
  return value instanceof Error ? value.message : String(value);
}
```

This exact defensive shape occurs in many Pi error paths. It matters most at
boundaries: provider responses, parsed configuration, session migration, and
extension code loaded with `jiti`.

## Async code and cancellation

An `async` function returns a `Promise<T>`, so its caller uses `await` to get
the eventual `T`. Pi streams model output with asynchronous iterables:

```ts
for await (const event of response) {
  // Consume a provider event as it arrives.
}
```

`AbortSignal` is Pi's cancellation token. The agent supplies it to provider
streams and tool execution; an extension should pass it into APIs that support
it rather than inventing a second cancellation mechanism.

```ts
const response = await fetch(url, { signal });
```

The agent loop converts tool exceptions into error results, and its stream
contract represents ordinary provider failures as final stream messages. That
is why error handling belongs at the boundary rather than in a broad,
unstructured `try/catch` around the whole application.

## Objects, spreads, and immutable snapshots

Pi frequently makes a shallow copy before changing an object:

```ts
const next = { ...previous, thinkingLevel: "high" };
const copiedMessages = messages.slice();
```

The spread expression copies own properties; nested arrays and objects remain
shared. This is not deep immutability. In the agent package, snapshots of
message and tool arrays protect loop state from accidental array mutation; in
the extension API, some event payloads are intentionally mutable. Follow the
contract for the specific event rather than assuming every object is frozen.

## Functions are values and callbacks define lifecycle seams

Handlers passed to `pi.on`, tool `execute`, renderer functions, and queue
callbacks are ordinary functions. The API calls them later. Arrow functions
capture surrounding variables; class methods use `this` and must keep their
receiver context.

```ts
pi.on("session_start", async (_event, ctx) => {
  ctx.ui.notify("Ready", "info");
});
```

The underscore is a naming convention for an intentionally unused parameter.
It does not change runtime behavior.

## Classes and public fields

The TUI and `Agent` use classes because they own long-lived mutable state,
methods, subscriptions, and cleanup. You will see explicit fields and
constructor assignment rather than TypeScript parameter properties. That is
intentional: the repository uses TypeScript syntax that Node can erase without
a JavaScript emit transform.

```ts
class Counter {
  private value: number;

  constructor(initial: number) {
    this.value = initial;
  }

  increment(): number {
    this.value += 1;
    return this.value;
  }
}
```

`private` is a TypeScript compile-time access restriction. It is not a security
boundary. Prefer a small public API over reaching into a coding-agent class's
private state.

## TypeBox: runtime schemas for tool inputs

The model produces JSON-like arguments, so Pi needs a runtime schema as well
as a static type. `typebox` supplies schemas such as `Type.Object` and
`Type.String`. Pi re-exports `Type` from `@earendil-works/pi-ai`, and validates
tool calls in `packages/agent/src/agent-loop.ts` before execution.

```ts
import { Type } from "@earendil-works/pi-ai";

const parameters = Type.Object({
  name: Type.String({ description: "Name to greet" }),
});
```

For string choices used in a tool exposed to Google-compatible APIs, use
`StringEnum` from `@earendil-works/pi-ai`; the extension documentation records
that `Type.Union` plus `Type.Literal` is not compatible with Google's API.

## A compact reading checklist

When TypeScript looks dense, answer these questions in order:

1. What is the runtime value, and what is type-only?
2. Which tag (`type`, `role`, or another literal) narrows this union?
3. Which values may be missing (`?` or `| undefined`)?
4. Is the data external or model-produced and therefore runtime-validated?
5. Is this callback invoked now, or saved for a future lifecycle event?
6. Does the function receive an `AbortSignal` that must be forwarded?

That checklist is enough to read the following architecture and lifecycle
chapters without treating types as magic.
