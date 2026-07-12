# Content Plan: TypeScript chapter (`01-typescript-for-pi.qmd`)

## Goal and audience

**Audience.** A developer who knows basic JavaScript (variables, functions, objects, arrays, `if`/`for`, promises at a surface level) but has little or no TypeScript experience.

**Success criteria.**

1. After reading only this chapter, the reader can understand most TypeScript code they will encounter in the Pi monorepo.
2. Every important concept has **multiple code examples that compare and contrast** (good vs bad; wrong vs right; alternative styles; compile-time vs runtime; JavaScript vs TypeScript).
3. Independent human-content review must pass the plan, then each revised section, then the final chapter.
4. Rendered PDF page count for this chapter alone must be **≥ 50 pages**.

Chapter currently renders at roughly **9 pages** (TOC pages 14–22). Target length is about **5–6×** current content, primarily by deepening beginner explanations, adding contrastive example stacks, mini-exercises, and more Pi-attached walkthroughs—not by padding.

## Writing conventions for every section

1. **From JS first.** Open with a short plain-JavaScript version of the idea when useful, then show what TypeScript adds.
2. **Compare and contrast blocks.** Each important concept gets at least two of:
   - wrong / compiles-but-wrong / correct
   - `interface` vs `type`, `any` vs `unknown`, `as` cast vs validation, mutable vs `readonly`, etc.
   - Pi-idiomatic vs not allowed (erasable syntax)
3. **Side-by-side tables or consecutive code fences** labelled clearly (`// bad`, `// better`, `// pi style`).
4. **Common mistakes** callouts with the actual compiler message when practical.
5. **"In Pi" bridge** at the end of each concept section with concrete package paths.
6. **Mini checklist** ("Can you now…?") after major sections.
7. **No fluff.** Direct technical prose. No emojis. Accurate Pi names and paths.
8. Keep `#sec:typescript` and existing Part A–C skeleton so cross-references stay stable; expand *inside* sections and add new lettered subsections as needed.

## Target outline (expanded)

### Opening (~1–1.5 pages)

- What TypeScript is for readers who only know JS
- What this chapter does *not* try to teach (full advanced toolkit)
- How the chapter is organized: Part A concepts → Part B Pi mapping → Part C optional extension setup
- "How to read Pi TypeScript in 30 seconds" mental model (runtime value vs type-only; tags narrow unions; validate external data)

### Part A: TypeScript concepts

Each A.x section targets roughly 3–6 rendered pages so the bulk of the 50-page requirement lives here.

#### A.0 Primer for JS-only readers (NEW, ~3 pages)

JavaScript → TypeScript bridge before any Pi concepts. Keep this compressed and orientational; deeper runtime-vs-compile-time treatment lives in A.1.

1. Types describe values; they do not change runtime behavior (brief)
2. Annotating vs inferring
3. The two worlds: editor/type-checker world vs Node runtime world (orientation only)
4. A tiny annotated script written three ways:
   - plain JS
   - TS with inference
   - TS with explicit annotations
5. What the type checker can and cannot see (teaser; A.1 expands)
6. Table: JS construct → TS equivalent you will see in Pi

#### A.1 Why types matter in a JS runtime (expand, ~3–4 pages)

Current content is a good seed. Expand with:

1. Compile-time safety vs runtime reality (multiple examples)
2. Compare: typed internal code vs untrusted boundaries
3. `as` casts: compare three versions
   -idiotmatically wrong cast of JSON
   - cast after partial checks
   - proper validation / type guard
4. Table: places types protect you vs places they do not
5. Mini analysis of a faked Pi config object arriving as JSON
6. Checklist + common mistakes

#### A.2 Interfaces, type aliases, and literal types (expand, ~4 pages)

1. JS object shape assumptions vs declared interfaces
2. Interface vs type alias: three paired examples
3. Optional (`?`), required, and `| undefined` — contrast all three
4. Literal unions for statuses (`"idle" | "streaming"`)
5. `readonly` properties at the type level
6. Index signatures when needed (and why Pi mostly prefers explicit fields)
7. Intersection (`&`) vs extension (`extends`) side-by-side
8. When Pi uses interface vs type (public events vs utility unions)
9. Common mistakes: optional confusion; empty object type; structural typing surprise
10. Checklist

#### A.3 Discriminated unions and narrowing (expand, ~5 pages)

Most important pattern for Pi. Heavy example density.

1. JS "these objects kind of look similar" → TS discriminant
2. Simple unions with `typeof` / `instanceof`
3. Discriminant on `type` / `role` (multiple examples)
4. `switch` + `never` exhaustiveness (compare incomplete vs complete)
5. Type predicates (`value is Foo`) with contrast for plain booleans
6. Control-flow narrowing with truthiness, equality, `in`
7. `as const` for literal preservation: widen vs not widen
8. Nested unions (content blocks inside messages) with progressive complexity
9. Failure modes: colliding field names, missing discriminant, excessive `any`
10. Pi map: AgentEvent, session entries, content blocks
11. Checklist: pick the right narrowing tool

#### A.4 Generics (expand, ~4 pages)

1. Motivation without generics (loss of type info / duplication)
2. Identity, `first`, map-style helpers with inference shown step by step
3. Generic constraints (`T extends …`) with allow/deny examples
4. Multiple type parameters; default type parameters (light)
5. Generic interfaces / types used as reusable shapes
6. `readonly T[]` vs `T[]` contrast
7. How schema helpers use type parameters (preview of TypeBox section)
8. Common mistakes: treating `T` as a runtime value; over-genericizing
9. Pi: `defineTool` inference sketch
10. Checklist

#### A.5 `unknown` is the honest type for untrusted data (expand, ~3–4 pages)

1. `any` disables safety (multiple bug demos)
2. `unknown` forces narrowing
3. Nested object narrowing walkthrough
4. Errors in `catch` (`unknown` default)
5. Narrowing toolbox table
6. Boundary map: network JSON, config files, tool args, extension state
7. Compare four validation styles: casts, manual narrow, type guard, schema
8. Pi defensive error helpers and boundaries
9. Checklist

#### A.6 Async/await, Promises, and AbortSignal (expand, ~4 pages)

Audience already knows promises at a surface level. Compress basic Promise review; spend most of the section on typing, streaming, and cancellation.

1. Brief sync vs async reminder (short)
2. Promise then-chains vs async/await (same function rewritten, one pair)
3. Typing Promises: `Promise<T>`, `async function` return types
4. Error handling: then-reject vs try/catch await (contrast)
5. Parallel work: `Promise.all` / `Promise.allSettled` (brief)
6. Async iterables / `for await…of` (streaming mental model — important for Pi)
7. AbortController / AbortSignal lifecycle: start, pass, abort, handle (primary weight)
8. Common cancellation bugs (not forwarding signal; swallowing AbortError)
9. Pi: forward agent-supplied signal into tools and fetches
10. Checklist

#### A.7 Objects, spreads, mutability, and copies (expand, ~3 pages)

Do not reteach basic JS spreads. Emphasize type-level and Pi-state implications.

1. Brief reference-semantics reminder only as it interacts with types
2. Shallow update patterns with type-preserving spreads
3. Nested shared references: how they break immutability assumptions under types
4. `readonly` arrays/objects and generics interaction
5. `as const` + nested objects and literal widening
6. Imm-ish update patterns used in agent state snapshots
7. Mutable vs intentionally frozen contracts
8. Pi: message/tool array snapshots; which extension payloads mutate
9. Checklist

#### A.8 Functions as values, callbacks, and closures (expand, ~3–4 pages)

1. First-class functions: store, pass, return
2. Function type syntax forms (compare 2–3 notations)
3. Callbacks registered now, called later (event-handler pattern)
4. Closures capturing outer state (good use + leak risk)
5. Arrow vs `function` vs methods: `this` contrast table
6. Optional and default parameters at type level
7. Returning functions / small factory patterns
8. Unused params (`_event`) convention
9. Pi: extension hooks, tool execute functions
10. Checklist

#### A.9 Classes, fields, and visibility (expand, ~3–4 pages)

1. When a class is warranted vs a plain object/module
2. Fields, constructor assignment, methods, getters
3. `private` / `protected` / public as compile-time tools
4. **Erasable syntax constraint**: parameter properties banned; allowed rewrite
5. Why Pi avoids `enum` / namespaces (brief, with correct rewrite)
6. Interface implemented by a class (when it appears)
7. Dispose / cleanup patterns at API surface level
8. Pi: Agent, TUI classes; prefer small public API
9. Checklist

#### A.10 Runtime validation with TypeBox (expand, ~4–5 pages)

1. Problem restatement: erased types cannot validate network input
2. Hand-written validator vs TypeBox (same schema, contrast drift)
3. Core TypeBox builders beginners will see: Object, String, Number, Boolean, Array, Optional, Union
4. `Static<typeof Schema>` derivation explained two ways
5. `Value.Check` narrowing flow
6. Tool-parameter schema example with multiple compare cases
7. `StringEnum` vs `Type.Union([Type.Literal(...)])` (Google-compat)
8. Where versioned config still needs migration after schema check
9. Pi re-exports of `Type` / `StringEnum`
10. Checklist

#### A.11 Modules, imports, and type-only imports (NEW, ~3 pages)

Moved up conceptually so beginners understand files as units before Part B, while keeping Part B.2 as Pi-specific rules.

1. ESM overview for JS-only readers
2. Named vs default exports (contrast)
3. `import type` vs value imports (from the same module in pairs)
4. Circular dependency smell (without deep package digression)
5. Top-level imports only (no inline dynamic import for types)
6. Extension vs package distinction preview
7. Checklist

#### A.12 Utility types you will actually see (NEW, ~3 pages)

Quality over coverage. Drop rare utilities (`Extract`/`Exclude`). Give remaining ones real worked examples.

1. `Partial`, `Required`, `Readonly` with object demos and contrast
2. `Pick` / `Omit` worked example pair
3. `Record<K,V>` vs index signatures
4. `ReturnType` / `Parameters` light usage if seen in advanced helpers
5. `NonNullable` on optional chains / unions
6. When *not* to reach for a utility type
7. Checklist

#### A.13 Reading dense TypeScript (NEW, ~2–3 pages)

Meta-skill section.

1. A deliberately dense signature, annotated line by line
2. Decoding generic constraints in English
3. Ignore noise first: start from nominals and discriminants
4. The 6-question reading checklist (promoted with worked examples)
5. Mini practice: three short Pi-flavored snippets with walkthrough answers

### Part B: TypeScript as used in Pi

#### B.1 How Pi runs TypeScript (expand, ~3 pages)

1. Runtime matrix table: Node strip-types, `tsx`, `jiti`, published `dist/`
2. Node version floor and stripping flags (accurate per engines)
3. What type stripping does *not* do (no typecheck; no enum emit)
4. `tsgo` / `npm run check` / package builds
5. Compare developer loop: edit → check → run interactive
6. Common failure: code works under jiti/tsx but fails `erasableSyntaxOnly`

#### B.2 Modules and imports in Pi (expand, ~2–3 pages)

1. Relative `.ts` imports within packages
2. Cross-package `@earendil-works/*` imports
3. Source mapping in root tsconfig vs published dist mapping
4. `import type` examples from real extension patterns
5. Forbidden: inline dynamic type imports
6. Side-by-side: extension file imports vs internal package imports

#### B.3 Pi's type patterns by example (expand, ~6–8 pages)

This is the "can I understand Pi now?" section. Longer, with multi-snippet walkthroughs.

For each pattern, structure: **What it looks like → why → minimal self-contained clone → point to real path → common misread**.

1. Discriminated unions (`AgentEvent`, session entries, content blocks)
2. Exhaustiveness (`never`)
3. Generics + TypeBox (`defineTool`) full walkthrough
4. StringEnum for Google-shaped schemas
5. Classes (Agent / TUI) ownership of state
6. AbortSignal forwarding
7. Snapshotting arrays/objects before mutation-sensitive work
8. Extension callback registration shape
9. Optional: a "read this 20-line sample end-to-end" synthetic file combining several patterns

#### B.4 Worked mini-tours of real shapes (NEW, ~3–4 pages)

Short guided readings (not architecture replacements).

1. A tool definition end-to-end
2. A tiny event switch on `type`
3. A config-ish object validated then used
4. A function that receives `signal: AbortSignal`

Include "wrong reading" vs "correct reading" for each.

#### B.5 Compact reading checklist (renumber existing B.4, expand, ~1–2 pages)

Expand checklist into a one-page laminated style with worked application to a single tough snippet.

### Part C: Optional / annexes

#### C.1 Type-checking an extension outside this monorepo (keep + polish, ~2 pages)

Keep current content; add a "you are inside the monorepo if…" decision tree and common package-name pitfalls.

#### C.2 Quick error catalog (NEW, ~2–3 pages)

10 common TS errors a Pi beginner will hit, each with:

- error text (paraphrased or realistic sample)
- meaning in plain English
- minimal broken example
- fixed example
- Pi-specific note if relevant

Candidates: type not assignable; property does not exist; object possibly undefined; cannot find name / module; excess property check; never assignment fail; generic inference fail; import type misuse; private member access; erasable syntax / parameter property errors.

#### C.3 Glossary of TypeScript terms used in this chapter (NEW, ~1–2 pages)

Short definitions only (cross-ref main guide glossary when appropriate): discriminant, narrow, exhaustiveness, erasable syntax, structural typing, type-only import, AbortSignal, TypeBox schema, Static type, etc.

#### C.4 Self-check: can you read this now? (NEW, ~2 pages)

A graded progression of 4–6 synthetic but Pi-shaped snippets. Each offers questions; answers immediately follow so the PDF remains self-contained for independent study.

### Closing (~0.5–1 page)

- What the next architecture chapter assumes you now know
- Pointer to extension chapters for TypeBox-in-tools deep dives already covered here at a pedagogical level

## Estimated page budget

| Section | Estimate |
|---|---|
| Opening | 1.5 |
| A.0 Primer | 3 |
| A.1 | 3.5 |
| A.2 | 4 |
| A.3 | 5 |
| A.4 | 4 |
| A.5 | 3.5 |
| A.6 | 4 |
| A.7 | 3 |
| A.8 | 3.5 |
| A.9 | 3.5 |
| A.10 | 4.5 |
| A.11 Modules | 3 |
| A.12 Utilities | 3 |
| A.13 Dense reading | 2.5 |
| B.1 | 3 |
| B.2 | 2.5 |
| B.3 | 7 |
| B.4 Mini-tours | 3.5 |
| B.5 Checklist | 1.5 |
| C.1 | 2 |
| C.2 Error catalog | 2.5 |
| C.3 Glossary | 1.5 |
| C.4 Self-check | 2 |
| Closing | 0.5 |
| **Total** | **~78 pages (design target 55–80; hard floor 50)** |

Page density in Quarto/PDF may compress examples; the plan deliberately overshoots to keep a safe margin above 50 after render.

## Implementation approach (to satisfy goal process)

1. **Plan validation first (this document).** Independent human-content-reviewer must PASS the plan against the guidelines before any section is rewritten.
2. **Section iteration order.** Write/expand one section cluster at a time, render when useful, then send that section (not the whole book) to an independent human-content-reviewer. Only after PASS, lock the section and continue.
   Suggested iteration units:

   1. Opening + A.0
   2. A.1 + A.2
   3. A.3
   4. A.4 + A.5
   5. A.6 + A.7
   6. A.8 + A.9
   7. A.10
   8. A.11 + A.12 + A.13
   9. B.1 + B.2
   10. B.3 + B.4 + B.5
   11. C.1–C.4 + closing
   12. Full-chapter consistency + PDF page-count verification (≥50)

3. **Quality gates per section unit.**
   - Beginner readability for JS-only readers
   - Multiple compare/contrast examples for important concepts
   - Technical accuracy for Pi (paths, package names,Node strip-types, erasable syntax)
   - No fluff; natural technical voice
   - Independent human-content-reviewer **PASS** required

4. **Finishing.**
   - Every rewritten section has a recorded independent reviewer PASS
   - Quarto PDF rebuild; measure chapter page count from PDF with objective extraction
   - If < 50 pages, expand weakest/shortest sections with more contrastive examples and worked snippets (not filler) until ≥ 50

## Out of scope

- Full TypeScript handbook (conditional types deep dives, advanced mapped types, decorators, namespaces, declaration merging, variance theory)
- Replacing later architecture/extension chapters
- Teaching Python/Rust type systems beyond light analogies
- Changing Pi source code; this is a documentation chapter only

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| PDF denser than estimates → under 50 pages | Overshoot outline; re-measure after each big batch; add self-checks and variant examples if short |
| Content becomes an abstract TS tutorial decoupled from Pi | Every A section ends with "In Pi"; Part B thick with paths |
| Overwhelms beginners | A.0 primer; progressive examples; checklists; keep optional annexes marked optional |
| Fabricated API or wrong paths | Spot-check against repo for every "In Pi" claim that names files |
| Reviewer fails for tone (robotic / tutorial LMS style) | Prefer concrete direct prose; show, don't moralize |

## Reviewer validation checklist (for plan approval)

The independent reviewer should PASS the plan only if:

1. A JS-basic reader path is complete enough to unlock "most Pi TypeScript" after reading.
2. Every important concept is scheduled for multiple compare/contrast examples.
3. Coverage matches Pi-relevant TypeScript (not a random TS course).
4. Process requires per-section independent review before proceeding.
5. Page budget is credible for ≥ 50 PDF pages without pure padding.
6. Scope truncation (advanced TS excluded) is sensible for the goal.

---

**Status:** PASS (independent human-content-reviewer). Non-blocking refinements from review integrated into A.0, A.6, A.7, A.12. Section-by-section writing may start.
