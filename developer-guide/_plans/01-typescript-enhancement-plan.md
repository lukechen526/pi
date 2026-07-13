# Enhancement Plan: Beginner Coverage for `01-typescript-for-pi.qmd`

## Purpose

The chapter now has a coherent progression, but several sections still assume
that a beginner can infer terminology from code. This plan improves each
section without changing the overall learning path:

1. first contact with TypeScript
2. modules and shapes
3. unions and ownership
4. functions, generics, and boundaries
5. async work and lifecycle
6. Pi source tours and workflow
7. reference material

The objective is not to add more terminology. It is to make each term earn its
place through a short definition, a contrastive example, and an immediate
checkpoint.

## Cross-cutting standard for every learning section

Every numbered learning section should follow this template:

1. **What you will learn** — 2–4 plain-language outcomes.
2. **Vocabulary before code** — define every new term used in the first major
   example; use the term again immediately.
3. **Example sequence** — at least three examples where appropriate:
   - JavaScript or familiar baseline
   - TypeScript annotation or idiomatic form
   - common mistake versus corrected form
4. **Pi bridge** — one real path and one sentence explaining what to look for.
5. **Checkpoint with answers** — no deferred answer key for the essential skill.

Use stable labels in code blocks: `JavaScript`, `TypeScript`, `Unsafe`, `Safe`,
and `Pi pattern`. Avoid introducing a symbol only in a comment. Define or
remove placeholders such as `pi`, `SessionContext`, `runTool`, `url`, and
`riskyOperation` when they appear in beginner-facing examples.

---

## Opening material

### Title and opening paragraphs

**Current gaps**

- “native ECMAScript modules,” “runtime,” and “type descriptions” appear before
  they are explained.
- The promise is clear, but the reader does not yet see what a first Pi file
  looks like.

**Enhancements**

- Add a two-sentence vocabulary box:
  - *runtime*: the JavaScript that actually executes
  - *type checker*: a tool that reports type mistakes before execution
  - *module*: a file that exports and imports names
- Add a tiny Pi-shaped file map showing one internal `.ts` module and one
  extension entry file.
- State that the first half teaches reading, while later sections teach safe
  authoring patterns.

### “What you should know first”

**Current gaps**

- “Promise” is named but not defined.
- The chapter assumes readers know the difference between a value, a function,
  and a reference.

**Enhancements**

- Add a short “If this is unfamiliar” box with one-line definitions for
  Promise, object, array, and function.
- Add a readiness example with a plain JavaScript function and ask the reader to
  identify its input, output, and side effect.

### “The five questions”

**Current gaps**

- “Boundary,” “schema,” “generic,” “union,” and “cleanup” are used as navigation
  terms before being taught.
- The list is useful but abstract.

**Enhancements**

- Recast the questions around the running status report and answer each one in
  one sentence.
- Add a compact annotated snippet with arrows pointing to the runtime value,
  tag, validation boundary, callback, and signal.

### 0. Set up the workspace and run TypeScript

**Current gap**

The chapter assumes the reader already knows how to install the repository,
execute a `.ts` file, type-check it, or add a dependency. A beginner can
understand the examples but still be unable to reproduce them locally.

**Enhancements**

Add a short practical setup section before language concepts. It should explain:

- how to check the Node.js version required by the repository
- how to install dependencies in this npm workspace
- when to use `npm install --ignore-scripts` for local development and
  `npm ci --ignore-scripts` for a clean, lockfile-based checkout
- what the root workspace packages are and why a package import can resolve to
  another workspace package
- how to run a standalone `.ts` example with Node's type stripping
- how to run a source example with the repository's `tsx`/launcher workflow
- how to type-check without emitting JavaScript (`npx tsgo --noEmit`)
- how `npm run check` differs from a focused type check
- how to add a runtime dependency versus a development dependency with npm
- why dependency installation and TypeScript type checking are separate steps

Use a tiny chapter example and show the complete loop:

```text
install dependencies → run the file → edit it → type-check it → run the project gate
```

Include a troubleshooting table:

| Symptom | First check |
|---|---|
| `command not found: tsgo` | dependencies were not installed; use the workspace command |
| `Cannot find module` | package install, import path, `.ts` suffix, or tsconfig mapping |
| file runs but type-check fails | runtime stripping/transpilation is not type checking |
| dependency works locally but changes the lockfile | inspect the package manifest and lockfile before sharing |

Commands and Node flags must be verified against the current root `package.json`,
workspace scripts, and `tsconfig` files before publication. Add a checkpoint
where the reader identifies which command runs code, which command checks types,
and which command installs packages.

---

## Part 1: First contact with a TypeScript file

### 1. Read a `.ts` file in 60 seconds

**Current gaps**

- “Annotation,” “interface,” “type checker,” “type-only,” “shallow,” and
  “deep clone” arrive quickly.
- The first example jumps directly from JavaScript to an interface without
  showing inference or a primitive annotation.
- The `rest` example is present but not explained as “collect remaining
  arguments.”

**Enhancements**

- Begin with three progressively annotated versions of `greeting`:
  plain JS, parameter-only annotation, and parameter plus return annotation.
- Add a two-column “what exists at runtime / what exists only for checking”
  table.
- Define shallow copy with a nested-object diagram or a two-line identity
  check (`copy.meta === original.meta`).
- Add a second example for destructuring and a second example for rest/spread:
  one function rest parameter and one object rest property.
- Add a small exercise asking the reader to predict `?.`, `??`, `||`, and a
  missing property result.

### 2. Files, exports, and imports

**Current gaps**

- “Module,” “ESM,” “public surface,” “runtime value,” “type-only import,” and
  “cycle” are not defined before use.
- The deep-import and cycle advice is accurate but too advanced for the first
  pass.
- Named/default exports are shown, but the difference in import syntax is not
  explained in words.

**Enhancements**

- Add a “module means one file with imports/exports” definition before code.
- Show a complete two-file example with one named value and one type export.
- Add a named-versus-default table: declaration syntax, import syntax, and
  common Pi use.
- Move circular dependencies into a short “later warning” callout; first teach
  only the distinction between value imports and type imports.
- Add an exercise where the reader chooses the correct import for a runtime
  schema and a type-only API.

### 3. Describe values: annotations, inference, and shapes

**Current gaps**

- “Widening,” “literal,” “interface,” “type alias,” “intersection,”
  “structural typing,” “fresh object literal,” `as const`, and `satisfies` are
  introduced in one section with little repetition.
- The `satisfies` example does not show what problem it solves compared with an
  annotation or `as const`.
- The running report appears late and is not built field by field.

**Enhancements**

- Add a vocabulary table with one sentence and one code token per term.
- Split the section into three micro-lessons:
  1. annotations and inference
  2. object shapes and optional fields
  3. literal contracts and object checking
- Add a three-way contrast:
  - annotation changes the variable’s declared type
  - `as const` narrows and readonly-locks literals
  - `satisfies` checks compatibility while preserving useful inference
- Add examples for `interface` versus `type`, `extends` versus `&`, and
  `field?` versus `field: T | undefined` with matching caller code.
- Build `StatusReport` one field at a time and add a checkpoint that asks which
  errors the checker catches and which it cannot catch.

---

## Part 2: Choose and protect a value’s shape

### 4. Unions, narrowing, and exhaustive branches

**Current gaps**

- “Union,” “narrowing,” “discriminated/tagged union,” “literal tag,”
  “exhaustiveness,” `never`, and “type predicate” are all essential but are
  not defined with a single running story.
- The section contains good examples, but the learner must infer why each
  narrowing tool differs.
- `in`, truthiness, `typeof`, and `instanceof` are listed more than compared.

**Enhancements**

- Add a progression table: test, what it proves, and when to use it.
- Use the same `StatusEvent` for every subsection:
  - `typeof` on a primitive payload
  - tag equality on event objects
  - `switch` with `never`
  - reusable predicate
- Add a “why optional fields are weaker” example with a failed runtime branch.
- Explain `never` in plain language: “after all possible cases are removed,
  no value remains.”
- Add a before/after example where adding a new event variant breaks the
  exhaustive switch.
- Add immediate answer text under each checkpoint question.

### 5. Mutability, snapshots, and copies

**Current gaps**

- “Reference,” “alias,” “shallow,” “deep,” “snapshot,” and “readonly view” are
  terms beginners often confuse.
- The table is useful but does not show object identity visually.
- `structuredClone` is named with no positive example beyond one line.

**Enhancements**

- Add a diagram-like example showing `original.meta === copy.meta` and
  `snapshot[0] === messages[0]`.
- Contrast three goals explicitly:
  - new container only
  - new element objects
  - fully independent clone
- Add a `readonly` versus `as const` side-by-side example and state that
  neither freezes runtime objects.
- Connect `StatusEvent[]` history to Pi message snapshots with a concrete
  “before mutation / after snapshot” sequence.
- Add a checkpoint with predicted output and an answer immediately below it.

---

## Part 3: Functions, reusable types, and data boundaries

### 6. Functions, callbacks, and closures

**Current gaps**

- “Function type,” “callback,” “closure,” “arrow field,” “detached method,” and
  “receiver” are not all defined before use.
- The Pi callback uses undefined illustrative types: `pi`,
  `SessionStartEvent`, and `SessionContext`.
- There are not enough examples distinguishing a callback’s signature from the
  code that invokes it.

**Enhancements**

- Define callback as “a function passed to another function to be called now or
  later.”
- Show three forms:
  1. direct call
  2. callback passed and invoked immediately
  3. callback stored and invoked later
- Define closure with a captured variable and show a second closure that
  accidentally retains a large object.
- Replace undefined Pi types with a small local `SessionStartEvent` and
  `SessionContext` illustration, clearly marked as conceptual.
- Add a detached-method example with the failing call and an arrow-field or
  `.bind` correction.
- Add a checkpoint requiring the reader to label “definition,” “registration,”
  and “invocation.”

### 7. Generics and utility types

**Current gaps**

- `T`, constraint, `keyof`, indexed access `T[K]`, and `readonly T[]` appear
  without enough visual explanation.
- Utility types are presented as a list rather than transformations of one
  object.
- `Parameters<typeof connect>` introduces the `typeof` type operator without
  defining it.

**Enhancements**

- Show a type substitution table for `first<string>` and `first<number>`.
- Add a deliberately incorrect generic implementation and explain why its body
  cannot access arbitrary properties.
- Explain `keyof` as “the union of property names” and `T[K]` as “the value type
  at one of those names,” with a concrete expansion.
- Transform one `Retry` type through `Partial`, `Pick`, `Omit`, `Record`, and
  `Readonly`, showing the resulting shape after each.
- Define `typeof fn` in a type position as “the type of this value,” contrasting
  it with runtime `typeof value`.
- Add a small “when not to use generics” example.

### 8. Untrusted data: `unknown`, `any`, and casts

**Current gaps**

- “Boundary,” “untrusted,” “cast,” “assertion,” and “type predicate” need
  plainer definitions.
- The section uses `text` and `riskyOperation` without defining them.
- The manual guard and predicate examples are close enough that their difference
  may be missed.

**Enhancements**

- Add a boundary diagram: string/network input → `unknown` → checks → trusted
  application value.
- Contrast `any`, `unknown`, and a type assertion in a three-column table.
- Define `as` as “a promise to the checker, not a runtime operation.”
- Show the same JSON value handled four ways, with explicit runtime outcomes.
- Define type predicate syntax (`value is User`) and explain what changes for the
  caller after the predicate returns true.
- Make the JSON source self-contained and show one malformed payload.
- Add exercises that ask which line is the first trustworthy line.

### 9. Runtime schemas with TypeBox

**Current gaps**

- “Runtime schema,” “static type,” `Static<typeof Schema>`, `Value.Check`,
  JSON Schema, and provider-specific schema shapes are dense terminology.
- The builder table is useful but each builder lacks a beginner example.
- The tool example arrives with a full Pi API signature before the learner has
  learned tool call ID, update callback, or extension context.
- `StringEnum` is explained as a provider concern without first showing why an
  emitted schema shape matters.

**Enhancements**

- Start with a hand-written validator and show how it can drift from an
  interface; then introduce TypeBox as “one runtime object that also produces a
  type.”
- Add one tiny example for each core builder, grouped as primitive, object,
  optional, array, and union.
- Explain `Static<typeof Schema>` token by token.
- Explain `Value.Check` as a boolean runtime test and show valid/invalid input.
- Split tool authoring into two levels:
  1. schema plus a plain `execute(params)` conceptual example
  2. the real Pi `defineTool` signature with each argument labeled
- Show the emitted JSON shape for a literal union versus `StringEnum`.
- Add a migration example that distinguishes validation, defaults, and version
  conversion.

---

## Part 4: Async work and object lifecycle

### 10. Promises, async functions, and streams

**Current gaps**

- “Promise,” “resolve,” “reject,” “async function,” “async generator,”
  “yield,” and `for await` are assumed rather than defined.
- The section says the reader knows Promises at a surface level, but the chapter
  is explicitly for beginners.
- `Promise.all` and `allSettled` need a result-shape example.

**Enhancements**

- Add a promise timeline: start → pending → fulfilled/rejected.
- Show a plain Promise constructor result only briefly, then focus on `async` and
  `await`.
- Label the then-chain and async/await examples line by line.
- Show `Promise.all` output and the difference when one operation rejects.
- Define async generator, `yield`, and `for await...of` before the streaming
  example.
- Connect a stream item to the already-defined `StatusEvent` type.
- Add a checkpoint where the reader predicts whether a function returns one
  value or many over time.

### 11. Cancellation with `AbortSignal`

**Current gaps**

- `AbortController`, `AbortSignal`, parent/child cancellation, abort errors,
  and composition are introduced in a short span.
- The first example uses `url` without defining it.
- The difference between checking `signal.aborted` and actually passing the
  signal to I/O needs stronger emphasis.

**Enhancements**

- Define controller as the owner that triggers cancellation and signal as the
  read-only notification passed to work.
- Add a lifecycle diagram: create → pass → abort → catch/return.
- Use a complete self-contained example with a fake `url` and a `try/catch`.
- Contrast “flag checked after work” with “signal forwarded into work.”
- Explain `AbortSignal.timeout` and `AbortSignal.any` using a parent-plus-timeout
  scenario.
- Add a table of cancellation bugs and the observable result of each.
- Add a checkpoint with a corrected implementation immediately following it.

### 12. Classes, inheritance, and cleanup

**Current gaps**

- “Class,” “field,” “getter,” “private,” “protected,” “public API,” “inheritance,”
  “prototype,” `super`, `implements`, and “parameter property” are introduced
  too quickly.
- The bad parameter-property example uses syntax before the term is defined.
- Cleanup ownership is stated but not shown with a resource lifecycle.

**Enhancements**

- Add a plain JavaScript class first, then add TypeScript visibility and field
  annotations.
- Define `extends` as runtime inheritance and `implements` as a compile-time
  checklist in a side-by-side table.
- Add a `super()` call explanation with a parent/child diagram.
- Define parameter property before showing the forbidden example, then rewrite
  it explicitly.
- Show `protected` in a small subclass example, not only in prose.
- Show a class that owns a subscription and an `AbortController`, with `start`
  and `stop` as the complete lifecycle.
- Explain why `enum` and `namespace` are avoided in Pi only after the reader
  understands the erasable-syntax constraint.

---

## Part 5: Apply the skills to Pi

### 13. Pi runtime and developer workflow

**Current gaps**

- `tsgo`, `tsx`, `jiti`, type stripping, `erasableSyntaxOnly`, package build,
  and quality gate appear in one table without a first-principles explanation.
- `npx tsgo --noEmit` and `npm run check` are shown without saying what success
  or failure looks like.

**Enhancements**

- Introduce “install,” “run,” and “check” as three separate actions before
  naming tools.
- Show `npm install --ignore-scripts` for a normal local checkout and
  `npm ci --ignore-scripts` for a clean lockfile-based checkout.
- Explain `npm install <package>` versus `npm install --save-dev <package>`
  and when a dependency change also requires reviewing `package.json` and the
  lockfile.
- Add a beginner workflow box:
  1. install dependencies
  2. edit
  3. run a focused example
  4. type-check
  5. run the full gate
- Define each tool in one sentence and state whether it strips, transpiles,
  loads, or checks.
- Explain `erasableSyntaxOnly` with one accepted and one rejected class.
- Add a troubleshooting table: “runs but check fails,” “cannot resolve import,”
  “dependency is missing,” and “extension loads but types fail.”

### 14. Guided source tour: a custom tool

**Current gaps**

- `defineTool`, tool call ID, update callback, context, and result content are
  not defined before the full example.
- The source tour describes the reading order but does not annotate the code
  directly.

**Enhancements**

- Add numbered comments or a matching numbered table for every parameter.
- Show a conceptual five-line tool before the real Pi signature.
- Add an unsafe version that trusts raw arguments and a corrected schema-backed
  version.
- Explain why `params` is statically inferred but still originates from external
  model/tool input.
- Point to the actual `ToolDefinition` and `defineTool` signatures separately.

### 15. Guided source tour: an agent event

**Current gaps**

- `AgentEvent` and `AgentMessage` are named without a plain-language definition.
- The abbreviated union may be mistaken for the actual source type.
- “Emits,” “consumes,” and lifecycle event are not explained.

**Enhancements**

- Define event as “a record announcing that something happened.”
- Explain producer/emitter versus consumer/handler with a small callback diagram.
- Mark the snippet explicitly as a simplified excerpt.
- Annotate one actual event from `packages/agent/src/types.ts` and one switch in
  `agent-loop.ts`.
- Add a mini-exercise: add a variant and find the consumers that must change.

### 16. Guided source tour: state and lifecycle

**Current gaps**

- “Ownership,” “alias,” “lifecycle,” “dispose,” and “snapshot” are referenced
  through a table but not applied to a complete source example.
- The table asks useful questions without showing answers.

**Enhancements**

- Add a short pseudo-source walkthrough with a message array, signal, callback,
  and class field in one function/class.
- Answer each table question against that snippet.
- Show a state timeline: created → active → updated → stopped → disposed.
- Link each concept back to the exact earlier section.

### 17. Make a small safe change

**Current gaps**

- “Declaration,” “construction site,” “caller,” “contract,” “focused check,” and
  “runtime change” are not defined for a beginner.
- The practice change is conceptual but does not show a complete before/after.

**Enhancements**

- Turn the list into a worked change on `StatusEvent`:
  1. locate the union
  2. locate the exhaustive switch
  3. locate construction sites
  4. update tests
  5. run checks
- Include a small before/after diff.
- Explain why adding a union variant is a multi-file change.
- Add a final “stop and inspect” checklist for runtime boundary and cleanup
  regressions.

---

## Reference sections

### R1. Common TypeScript errors

**Current gaps**

- Several entries are one sentence rather than the promised error → meaning →
  broken example → fix format.
- Compiler wording is missing for generic inference, import misuse, and erasable
  syntax.
- Entries do not point back to the learning sections.

**Enhancements**

For every error, add:

1. representative compiler message
2. plain-English meaning
3. broken code
4. corrected code
5. “Return to Section X” link

Add entries for “possibly undefined,” excess properties, `never`, generic
inference, type-only import misuse, and parameter properties using this exact
format.

### R2. Utility type quick reference

**Current gaps**

- `T`, `K`, and `V` are not defined in the table.
- “Top-level” is not explained.
- The table does not distinguish static restrictions from runtime behavior.

**Enhancements**

- Add a “static only” note.
- Add one before/after shape for each utility.
- Link each row to Section 7.
- Add a warning that `Readonly`, `Partial`, and related utilities are shallow.

### R3. Glossary

**Current gaps**

Important terms used in the chapter are missing:

- ESM/module
- runtime
- type checker
- Promise
- callback
- closure
- `this`
- boundary
- schema
- `satisfies`
- `as const`
- `keyof`
- indexed access
- `AbortController`
- `AbortSignal`
- `tsx`, `jiti`, and `tsgo`
- parameter property
- `protected`

**Enhancements**

- Add these terms with one-sentence definitions and links to first introduction.
- Mark terms as JavaScript, TypeScript, Pi tooling, or runtime-validation terms.
- Add “common confusion” notes for `as` versus conversion, `readonly` versus
  freezing, and `extends` versus `implements`.

### R4. Cumulative self-check

**Current gaps**

- `runTool` is undefined.
- The code uses `unknown`, a discriminated union, callback, Promise, and signal
  all at once without a staged prompt.
- Answers are correct but do not explain why each line is safe.

**Enhancements**

Split the self-check into three levels:

1. **Shape:** narrow `Input` and explain the tag.
2. **Boundary:** identify and validate `args: unknown`.
3. **Lifecycle:** explain callback timing, Promise return, and signal ownership.

Define a tiny `runTool` signature or replace it with a comment showing the
expected contract. Put each answer directly below its question and link back to
Sections 4, 8, and 11.

### R5. Standalone extension type-checking

**Current gaps**

- `tsconfig`, `moduleResolution`, `strict`, `skipLibCheck`, and `noEmit` are
  shown without definitions.
- The reader does not know when this section applies.

**Enhancements**

- Start with a decision tree: monorepo extension versus standalone extension.
- Define each compiler option in a short table.
- Explain that `include` must match the reader’s directory layout.
- Add the relationship between `tsc --noEmit`, Pi runtime loading, and `jiti`.
- Show one expected success result and one common module-resolution failure.

### Closing

**Current gaps**

- The closing summarizes abilities but does not tell the beginner what to do
  next.
- “Architecture and extension chapters” are mentioned without a concrete next
  action.

**Enhancements**

- Add a three-step next action:
  1. open one source-tour file
  2. trace one union or tool
  3. make a documentation-only or test-only change
- Link each next chapter to the skill it assumes.
- Add a final reminder that the reference sections are meant to be revisited,
  not memorized.

---

## Recommended implementation order

1. Add vocabulary boxes and self-contained placeholders to Sections 1–3.
2. Strengthen Sections 4–5 with diagrams, event examples, and immediate answers.
3. Expand Sections 6–9, especially callbacks, generics, and TypeBox.
4. Expand Sections 10–12 with timelines and lifecycle examples.
5. Annotate the Pi source tours and replace abbreviated/undefined examples.
6. Standardize R1–R5 as reference material with links back to first teaching.
7. Render the PDF and inspect the table of contents and page transitions.
8. Run an independent beginner-content review section by section.

## Acceptance criteria for review

- No essential term appears before a one-sentence definition or familiar example.
- Each learning section contains at least two contrastive examples and one
  answerable checkpoint.
- Every code identifier is defined in the same block or explicitly labeled as a
  simplified Pi excerpt.
- The running status example appears in every relevant section from shapes
  through TypeBox and cancellation.
- Reference sections point back to the main teaching path.
- A beginner can read Sections 1–17 linearly without needing to jump ahead to
  decode terminology.
- Pi paths and API claims are checked against the repository before rendering.
