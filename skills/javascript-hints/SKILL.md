---
name: javascript-hints
description: This skill should be used when writing, reviewing, or refactoring vanilla JavaScript (not TypeScript) — any time files match `**/*.js`, `**/*.mjs`, `**/*.cjs`, or `**/package.json`, or the user mentions JavaScript/ES2022+ conventions, ESM vs. CommonJS, `const`/`let`, arrow functions, destructuring, spread, optional chaining, nullish coalescing, async/await, Promises, `AbortController`, `structuredClone`, `BigInt`, `Number.isNaN`, or JSDoc. Covers general JavaScript language guidance only — conventions and patterns/anti-patterns. Specific third-party tool, framework, or library rules (linters, formatters, bundlers, test runners, validation libraries, web frameworks, ORMs, UI frameworks) belong in their own skills. If the project uses TypeScript, use the `typescript-hints` skill instead.
---

# javascript-hints

Project-wide JavaScript language guidance. Apply these rules when authoring or modifying vanilla JavaScript (`.js`/`.mjs`/`.cjs`) in this repository.

## Scope

These references cover the JavaScript language (ES2022+) and platform-neutral runtime built-ins (`Promise`, `AbortController`, `structuredClone`, `URL`, `Intl`). They do **not** cover:

- TypeScript-specific features — use `typescript-hints` instead.
- Linters or formatters (ESLint, Biome, Prettier, …)
- Bundlers / build tools (esbuild, webpack, vite, rollup, …)
- Test frameworks (Jest, Vitest, Mocha, `node:test`, …)
- Validation libraries (Zod, Yup, io-ts, Valibot, …)
- Runtime-specific APIs (Node-only, Deno-only, Bun-only, browser-only)
- Web frameworks, UI libraries, ORMs, state managers

Those belong in dedicated skills.

## When to apply

Apply when any of the following is true:
- The task touches a file matching `**/*.js`, `**/*.mjs`, `**/*.cjs`, or `**/package.json` in a non-TypeScript project.
- The user is designing a JavaScript package, service, CLI, or library.
- The user asks about JS idioms, ESM vs. CommonJS, async/await, error handling, or modern syntax.
- The user is writing or reviewing JSDoc annotations.

If the file is `.ts`/`.tsx` or the project has a `tsconfig.json`, prefer `typescript-hints`.

## How to apply

1. Before writing or editing JS code, load the reference file(s) relevant to the task. Do not rely on memory — these rules may evolve.
2. Prefer the obvious, readable solution. Do not introduce abstractions, closures, or indirection that the task does not require.
3. Match the repository's existing patterns before applying a rule — if a file already follows a rule, stay consistent; if it violates one, surface the mismatch rather than silently "fixing" surrounding code.
4. If a rule touches a specific third-party tool the project uses, defer to that tool's dedicated skill and do not restate its rules here.

## Non-negotiables (quick reference)

- Target ES2022+. Use ES modules (`import`/`export`); do not mix with CommonJS in the same module.
- `const` by default; `let` only when reassigning; never `var`.
- Always `===` / `!==`, never `==` / `!=`.
- Every promise must be `await`ed, `return`ed, or `.catch`-handled. No floating promises.
- Never `await` inside `Array.prototype.forEach` — use `for...of` or `Promise.all(items.map(...))`.
- Never wrap an async function's work in `new Promise(async ...)` — return the promise directly.
- Use `??` for default values on nullable data, not `||`.
- Throw `Error` (or a subclass). Preserve causes with `new Error("...", { cause: e })`.
- Use `structuredClone(x)` for deep copies, not `JSON.parse(JSON.stringify(x))`.
- Use `Number.isNaN` / `Number.isFinite`, not the global versions.
- Prefer named exports over default exports.
- No module-level side effects at import time.
- Document exported symbols with JSDoc.

## Reference Files

Load the specific file(s) for the task — do not load both unnecessarily.

- **`references/conventions.md`** — Target baseline (ES2022+, ESM), strict mode, naming, `const`/`let`/no-`var`, imports/exports (named vs. default), function and class style, `#private` fields, equality rules, JSDoc, module hygiene.
- **`references/patterns.md`** — Design principles (KISS, SRP, composition over inheritance, Rule of Three, function size), modern syntax idioms (arrow `this`, destructuring, spread/`structuredClone`, optional chaining, nullish coalescing, array methods), async patterns (`async/await`, `Promise.all`/`allSettled`/`any`/`race`, `AbortController`, `Error` cause), numbers and equality (float precision, `BigInt`, `Number.isNaN`, `Object.is`), and an anti-patterns checklist with fix table.
