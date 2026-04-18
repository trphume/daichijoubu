# JavaScript Conventions

## Target Baseline
- Target ES2022+ for new code (top-level `await`, private class fields, `Error.cause`, `Object.hasOwn`, `Array.prototype.at`, `structuredClone`).
- Use ES modules (`import`/`export`). Do not mix ESM and CommonJS in the same module.
- Add `"type": "module"` in `package.json` for ESM packages. Use the `.cjs` extension when a single file must stay CommonJS.
- Strict mode is automatic in ES modules and class bodies. In legacy scripts, put `"use strict";` at the top.

Tool-specific configuration (linters, formatters, bundlers, test runners, type checkers) belongs in its own skill.

## Naming
- Classes, constructors: `PascalCase` (`UserRepository`).
- Functions, methods, variables: `camelCase` (`getUser`, `orderCount`).
- Module-level constants naming fixed values: `SCREAMING_SNAKE_CASE` (`MAX_RETRIES`). Regular `const` for non-literal values stays `camelCase`.
- Private fields: `#field`. Do not rely on `_` prefix for privacy — it is a convention, not enforcement.
- File names: follow the project's convention (`kebab-case.js` or `camelCase.js`). One primary export per file named to match the file.

## `const` / `let` / `var`
- `const` by default. `let` only when reassignment is required.
- Never use `var`. It hoists and has function-scope semantics that cause subtle bugs.
- Use `Object.freeze` or deep-freeze helpers for constants that must not be mutated.

## Imports and Exports
- Prefer named exports over default exports. Default exports hinder refactoring (renames do not propagate) and make re-exports noisier.
- Group imports: Node built-ins (`node:fs`, `node:path`) → third-party → first-party. One blank line between groups.
- Use explicit file extensions in relative imports under pure ESM: `import { x } from "./util.js";`.
- Avoid wildcard imports (`import * as utils from ...`) in application code — they defeat tree-shaking and hide what is used.

```js
import { readFile } from "node:fs/promises";

import { createLogger } from "./logger.js";
```

## Functions
- Prefer arrow functions for callbacks and single-expression helpers.
- Use `function` declarations for top-level named functions when you want hoisting or the function uses `this` (methods, constructors, generators).
- Prefer one options object over many positional parameters past 2–3 arguments. Destructure at the signature with defaults.

```js
export function createUser({ email, name, role = "member" }) {
  // ...
}
```

- Avoid mutating parameters. Return a new value instead.

## Classes
- Use `#private` fields for true privacy.
- Mark fields `static` only when they are genuinely per-class, not per-instance.
- Avoid inheritance except for genuine "is-a" relationships. Prefer composition.
- Use `Object.freeze(this)` at the end of the constructor for value classes that should be immutable.

## Equality
- Always use `===` and `!==`. `==` has coercion rules that cause subtle bugs.
- Use `Number.isNaN(x)` — not the global `isNaN`, which coerces.
- Use `Number.isFinite(x)` — not the global `isFinite`.
- Use `Object.is(a, b)` when you need to distinguish `-0` from `0` or treat `NaN` as equal to itself.

## Documentation
- Use JSDoc for all exported symbols. Even without a type checker, JSDoc serves as readable contract documentation and feeds IDE tooltips.

```js
/**
 * Process items concurrently using a worker pool.
 *
 * @param {Item[]} items  Items to process. Must not be empty.
 * @param {number} [maxWorkers=4]  Maximum concurrent workers.
 * @returns {Promise<BatchResult>}
 * @throws {RangeError} if `items` is empty.
 */
export async function processBatch(items, maxWorkers = 4) { /* ... */ }
```

## Formatting
- Line length: 100–120 characters (follow repository setting).
- One statement per line. Semicolons terminate statements (unless the whole repository is semicolon-free).
- Break long argument lists one-per-line with trailing comma.

## Module Hygiene
- No side effects at import time. A bare `import "./mod.js"` must be cheap and pure — no network calls, no global mutation, no logging.
- No top-level mutable state shared across the module graph. Thread state through parameters or a dedicated state object.
- Keep barrel files (`index.js` re-exports) shallow; deep barrels cause circular-dependency and tree-shaking issues.
