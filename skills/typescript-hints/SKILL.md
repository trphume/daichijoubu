---
name: typescript-hints
description: This skill should be used when writing, reviewing, or refactoring TypeScript or JavaScript code — any time files match `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mts`, `**/*.cts`, `**/tsconfig*.json`, or `**/package.json`, or the user mentions TypeScript/JavaScript conventions, strict mode, generics, conditional/mapped/template-literal types, utility types, discriminated unions, narrowing, async/await, Promises, `AbortController`, or `structuredClone`. Covers general TS/JS language guidance only — conventions, patterns and anti-patterns, and type safety. Specific third-party tool, framework, or library rules (linters, formatters, bundlers, test runners, validation libraries, web frameworks, ORMs, UI frameworks) belong in their own skills.
---

# typescript-hints

Project-wide TypeScript (and JavaScript) language guidance. Apply these rules when authoring or modifying TS/JS code in this repository.

## Scope

These references cover the TypeScript language, the JavaScript features it inherits (ES2015+), and platform-neutral runtime built-ins (`Promise`, `AbortController`, `structuredClone`, `URL`). They do **not** cover:

- Linters or formatters (ESLint, Biome, Prettier, …)
- Bundlers / build tools (tsc-as-a-build-tool specifics, esbuild, webpack, vite, rollup, …)
- Test frameworks (Jest, Vitest, Mocha, `node:test`, …)
- Validation libraries (Zod, Yup, io-ts, Valibot, …)
- Runtime-specific APIs (Node-only, Deno-only, Bun-only, browser-only)
- Web frameworks, UI libraries, ORMs, state managers

Those belong in dedicated skills.

## When to apply

Apply when any of the following is true:
- The task touches a file matching `**/*.ts`, `**/*.tsx`, `**/*.js`, `**/*.jsx`, `**/*.mts`, `**/*.cts`, `**/tsconfig*.json`, or `**/package.json`.
- The user is designing a TypeScript package, service, CLI, or library.
- The user asks about TS/JS idioms, async/await, error handling, or dependency injection.
- The user is adding type annotations, defining generics/conditional/mapped types, or modeling state with discriminated unions.
- The user is configuring TypeScript compiler options.

## How to apply

1. Before writing or editing TS/JS code, load the reference file(s) relevant to the task. Do not rely on memory — these rules may evolve.
2. Prefer the obvious, readable solution. Do not introduce abstractions, generics, or indirection that the task does not require.
3. Match the repository's existing patterns before applying a rule — if a file already follows a rule, stay consistent; if it violates one, surface the mismatch rather than silently "fixing" surrounding code.
4. If a rule touches a specific third-party tool the project uses, defer to that tool's dedicated skill and do not restate its rules here.

## Non-negotiables (quick reference)

- `strict: true` is required. Add `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and `noImplicitOverride`.
- No `any` outside isolated interop wrappers. Use `unknown` and narrow.
- No `as X` on values from outside the program without a runtime type guard.
- No `// @ts-ignore` — use `// @ts-expect-error <reason>`.
- Every promise must be awaited, returned, or `.catch`-handled. No floating promises.
- Never `await` inside `Array.prototype.forEach` — use `for...of` or `Promise.all(items.map(...))`.
- Use `??` for default values on nullable types, not `||`.
- `const` by default; `let` only when reassigning; never `var`. Prefer `readonly` and `as const`.
- Throw `Error` (or a subclass). Preserve causes with `new Error("...", { cause: e })`.
- In `catch (e)`, `e` is `unknown` — narrow before use.
- Use `structuredClone(x)` for deep copies, not `JSON.parse(JSON.stringify(x))`.
- Finite states are discriminated unions with an exhaustive `assertNever` default.
- Return DTOs at API boundaries, not internal persistence types.

## Reference Files

Load the specific file(s) for the task — do not load all three unnecessarily.

- **`references/conventions.md`** — Compiler options, naming, `const`/`let`/`var`, imports/exports (named vs. default, type-only imports), function and class style, TSDoc, project layout.
- **`references/patterns.md`** — Design principles (KISS, SRP, composition over inheritance, Rule of Three, function size), modern JS idioms (destructuring, spread, optional chaining, nullish coalescing, array methods), async patterns (`async/await`, `Promise.all`, `Promise.allSettled`, `AbortController`, typed errors in catch), and an anti-patterns checklist with fix table.
- **`references/type-safety.md`** — Strict mode, `unknown` vs. `any`, `interface` vs. `type`, discriminated unions with `assertNever`, generics with constraints/defaults/literal preservation, narrowing and assertion functions, utility types table, mapped/conditional/template-literal types, branded types, `readonly`/`as const`, `satisfies`, `@ts-expect-error` hygiene.
