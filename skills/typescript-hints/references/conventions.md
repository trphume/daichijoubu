# TypeScript Conventions

## Target and Compiler Options
- Target modern TypeScript (5.x) with `strict: true`. Enable additionally:
  - `noUncheckedIndexedAccess` — array/object access returns `T | undefined`.
  - `noImplicitOverride` — subclass overrides must be explicit.
  - `exactOptionalPropertyTypes` — `{ x?: number }` does not accept `{ x: undefined }`.
  - `noFallthroughCasesInSwitch`.
  - `useUnknownInCatchVariables` (implied by `strict` in 4.4+) — `catch (e)` types `e` as `unknown`.
- Use ES module output (`module: "NodeNext"` or `"ESNext"`). Do not mix CommonJS and ESM in the same package.

Tool-specific configuration (linters, formatters, bundlers, test runners) belongs in its own skill.

## Naming
- Types, interfaces, classes, enums: `PascalCase` (`UserRepository`, `OrderStatus`).
- Functions, variables, methods: `camelCase` (`getUser`, `orderCount`).
- Module-level constants that name fixed values: `SCREAMING_SNAKE_CASE` (`MAX_RETRIES`).
- Type parameters: single uppercase letter or descriptive `PascalCase` (`T`, `TValue`, `TKey`). Prefix optional with `T` in multi-param generics to distinguish from type names.
- Do not prefix interfaces with `I` (`User`, not `IUser`).
- Do not suffix types with `Type` (`User`, not `UserType`).
- File names: follow the project's convention (`kebab-case.ts` or `camelCase.ts`). One primary export per file named to match the file.

## `const` / `let` / `var`
- `const` by default. `let` only when reassignment is required. Never `var`.
- Prefer immutable data: `readonly` fields, `readonly T[]`, `ReadonlyArray<T>`, `ReadonlyMap<K, V>`.
- Use `as const` to preserve literal types in constants: `const ROLES = ["admin", "user"] as const;`.

## Imports and Exports
- Prefer named exports over default exports. Default exports hinder refactoring (renames do not propagate) and make re-exports noisier.
- Group imports: stdlib/runtime (`node:fs`, `node:path`) → third-party → first-party. One blank line between groups.
- Include the `.js` extension in relative imports when the project uses `NodeNext`/pure ESM resolution — TypeScript expects the emitted path.
- Use `import type { ... }` for type-only imports to keep runtime imports minimal and avoid accidental side effects.

```ts
import { readFile } from "node:fs/promises";

import type { Logger } from "./logger.js";
import { createLogger } from "./logger.js";
```

## Functions
- Annotate the return type on all exported functions. Let inference handle local helpers.
- Prefer arrow functions for callbacks and single-expression helpers; use `function` declarations for top-level named functions when hoisting or `this` binding matters.
- Prefer one object parameter over many positional parameters past 2–3 arguments. Destructure with defaults at the call boundary.

```ts
export function createUser({
  email,
  name,
  role = "member",
}: {
  email: string;
  name: string;
  role?: Role;
}): User {
  /* ... */
}
```

## Classes
- Use `readonly` for fields that are set once in the constructor.
- Use TypeScript parameter properties (`constructor(private readonly repo: UserRepo)`) for simple DI.
- Use `#private` fields for true privacy when the runtime needs it; otherwise `private` is sufficient for the type system.
- Avoid inheritance except for genuine "is-a" relationships. Prefer composition.

## Documentation
- Use TSDoc/JSDoc for all exported symbols. The first sentence is a terse description; add `@param`, `@returns`, `@throws`, and `@example` when useful.

```ts
/**
 * Process items concurrently using a worker pool.
 *
 * @param items  Items to process. Must not be empty.
 * @param maxWorkers  Maximum concurrent workers. Defaults to 4.
 * @throws RangeError if `items` is empty.
 */
export async function processBatch(items: Item[], maxWorkers = 4): Promise<BatchResult> { ... }
```

## Formatting
- Line length: 100–120 characters (follow repository setting).
- One statement per line; no semicolon-less style unless the whole repository uses it.
- Break long call/argument lists one-per-line with trailing comma.

## Project Layout
- `src/` for source, `dist/`/`build/` (git-ignored) for output.
- Mirror the source tree in tests: `src/user/service.ts` ↔ `src/user/service.test.ts` (or a parallel `tests/` tree).
- Keep barrel files (`index.ts` re-exports) shallow; deep barrels cause circular-dependency and tree-shaking issues.
