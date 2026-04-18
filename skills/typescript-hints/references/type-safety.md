# TypeScript Type Safety

TypeScript's value is catching errors at compile time. Use the type system deliberately: every `any`, `as`, and `!` is a hole in that guarantee. This file covers the language's type features — specific validation libraries (Zod, Yup, io-ts, etc.) belong in their own skills.

## Enable Strict Mode

Required in `tsconfig.json` for any new project:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

## `unknown` over `any`

`any` opts out of type checking. `unknown` is the correct "I don't know yet" type — you must narrow it before use.

```ts
function handle(payload: unknown): void {
  if (typeof payload === "string") {
    payload.toUpperCase(); // narrowed to string
  }
}
```

Reserve `any` for genuinely untyped third-party interop that has no declaration file, and isolate it behind a typed wrapper.

## Interface vs. Type Alias

- Use `interface` for object shapes that may be extended or merged.
- Use `type` for unions, intersections, tuple types, mapped types, conditional types, and function signatures.
- Do not mix (pick one per shape). Interfaces give slightly better error messages on object types.

## Discriminated Unions

Model finite states with a literal discriminator. The compiler narrows on `switch`.

```ts
type AsyncState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: Error };

function render<T>(s: AsyncState<T>): string {
  switch (s.status) {
    case "idle":    return "—";
    case "loading": return "…";
    case "success": return JSON.stringify(s.data);
    case "error":   return s.error.message;
    default:        return assertNever(s);
  }
}

function assertNever(x: never): never {
  throw new Error(`unexpected: ${JSON.stringify(x)}`);
}
```

`assertNever` forces a compile-time error if a new variant is added without a case.

## Generics

### Constraints
```ts
function logLength<T extends { length: number }>(x: T): T {
  console.log(x.length);
  return x;
}
```

### Default Type Parameters
```ts
interface Response<TBody = unknown> {
  status: number;
  body: TBody;
}
```

### Preserve Literals
When a generic should remember the literal value a caller passed, constrain with `extends` and use `as const` at the call site:
```ts
function pick<const K extends readonly string[]>(keys: K): K { return keys; }
const k = pick(["id", "name"] as const); // readonly ["id", "name"]
```

## Type Narrowing

Narrow with `typeof`, `instanceof`, property presence (`in`), discriminated unions, and user-defined type guards.

```ts
function isUser(x: unknown): x is User {
  return typeof x === "object"
      && x !== null
      && "id" in x
      && typeof (x as { id: unknown }).id === "string";
}
```

Assertion functions narrow and also abort on failure:
```ts
function assertString(x: unknown): asserts x is string {
  if (typeof x !== "string") throw new TypeError("expected string");
}
```

## Utility Types

Built-in utilities cover most transformations. Prefer them to hand-rolled equivalents.

| Utility            | Purpose                                                    |
|--------------------|------------------------------------------------------------|
| `Partial<T>`       | All properties optional                                    |
| `Required<T>`      | All properties required                                    |
| `Readonly<T>`      | All properties `readonly`                                  |
| `Pick<T, K>`       | Keep only keys in union `K`                                |
| `Omit<T, K>`       | Drop keys in union `K`                                     |
| `Record<K, V>`     | Object type with keys `K`, values `V`                      |
| `Exclude<T, U>`    | Remove members of `U` from union `T`                       |
| `Extract<T, U>`    | Keep only members of `T` assignable to `U`                 |
| `NonNullable<T>`   | Drop `null` and `undefined`                                |
| `ReturnType<F>`    | Return type of function `F`                                |
| `Parameters<F>`    | Parameter tuple of function `F`                            |
| `Awaited<T>`       | Unwrap `Promise<T>`                                        |
| `Capitalize<S>`, `Uncapitalize<S>`, `Uppercase<S>`, `Lowercase<S>` | String manipulation on literal types |

## Mapped Types

Transform the keys or values of an existing type.

```ts
type Getters<T> = { [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K] };

type Nullable<T> = { [K in keyof T]: T[K] | null };

type PickByType<T, U> = { [K in keyof T as T[K] extends U ? K : never]: T[K] };
```

The `-` modifier removes readonly/optional:
```ts
type Mutable<T>  = { -readonly [K in keyof T]: T[K] };
type Concrete<T> = { [K in keyof T]-?: T[K] };
```

## Conditional Types and `infer`

```ts
type ElementOf<T>   = T extends readonly (infer U)[]     ? U : never;
type ReturnOf<T>    = T extends (...a: any[]) => infer R ? R : never;
type PromiseValue<T> = T extends Promise<infer V>        ? V : T;
```

Distributive conditional types split unions:
```ts
type ToArray<T> = T extends unknown ? T[] : never;
type Mixed = ToArray<string | number>; // string[] | number[]
```

Wrap in a one-tuple to disable distribution:
```ts
type ToArrayNonDist<T> = [T] extends [unknown] ? T[] : never;
```

## Template Literal Types

```ts
type EventName = "click" | "focus" | "blur";
type Handler   = `on${Capitalize<EventName>}`; // "onClick" | "onFocus" | "onBlur"
```

## Branded (Nominal) Types

TypeScript is structural. When you need nominal identity (e.g., distinguishing `UserId` from `OrderId`), use a brand:

```ts
type Brand<T, B> = T & { readonly __brand: B };
type UserId  = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

function asUserId(raw: string): UserId { return raw as UserId; }
```

Confine the single `as` to a constructor helper so the rest of the codebase is cast-free.

## `noUncheckedIndexedAccess`

With this flag, `arr[i]` has type `T | undefined`. Always check or use `Array.prototype.at`:

```ts
const first = items[0];
if (first === undefined) return;
use(first);
```

## `readonly`, `as const`, and Immutability

- `readonly T[]` / `ReadonlyArray<T>` — array cannot be mutated through this reference.
- `readonly` on object fields — property cannot be reassigned.
- `as const` — literal, deep-readonly version of an object/array expression.

```ts
const ROLES = ["admin", "user"] as const;
type Role = (typeof ROLES)[number]; // "admin" | "user"
```

## Function Overloads

Use only when the return type genuinely depends on the input type and cannot be expressed with generics or conditional types. Overloads with the same return type are noise — use a single generic signature.

## `satisfies`

`as X` coerces and loses information. `satisfies X` **checks** a value against `X` without widening it:

```ts
const palette = {
  red:  "#ff0000",
  blue: "#0000ff",
} satisfies Record<string, `#${string}`>;

// palette.red is the literal "#ff0000", not just `#${string}`.
```

## `// @ts-` Escape Hatches

- Never use `// @ts-ignore`. Use `// @ts-expect-error <reason>` — it fails if the error disappears, so stale suppressions get cleaned up.
- Every suppression must have a short comment explaining why the type system is wrong here.
- Review suppressions during code review the same as any other assertion.

## Checklist

- [ ] `strict` + `noUncheckedIndexedAccess` + `exactOptionalPropertyTypes` enabled.
- [ ] No `any` outside isolated interop wrappers.
- [ ] No `as X` on values from outside the program without a runtime check.
- [ ] Finite-state data modeled as discriminated unions with exhaustive `assertNever`.
- [ ] External/untrusted inputs narrowed via type guards before use.
- [ ] `// @ts-expect-error` (with reason) instead of `// @ts-ignore`.
- [ ] `satisfies` used instead of `as` when the goal is "check, don't widen".
