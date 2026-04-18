# TypeScript / JavaScript Patterns and Anti-Patterns

## Design Principles

### KISS — Keep It Simple
Choose the simplest solution that works. Prefer a map literal over a registry/factory, a plain function over a class, a union type over an inheritance hierarchy.

### Single Responsibility
Each module, class, or function has one reason to change. Split transport/serialization, domain logic, and persistence into separate units.

### Composition Over Inheritance
Combine small, focused objects through constructor injection. Use inheritance only for genuine "is-a" relationships.

```ts
class NotificationService {
  constructor(
    private readonly email: EmailSender,
    private readonly sms?: SmsSender,
    private readonly push?: PushSender,
  ) {}

  async notify(user: User, message: string, channels = new Set(["email"])): Promise<void> {
    if (channels.has("email")) await this.email.send(user.email, message);
    if (channels.has("sms") && this.sms && user.phone) await this.sms.send(user.phone, message);
  }
}
```

### Rule of Three
Wait until three instances before extracting an abstraction. Two similar blocks are often cheaper to keep duplicated than the wrong shared abstraction.

### Function Size
Extract when a function exceeds ~20–50 lines, mixes distinct purposes, or has 3+ levels of nesting.

## Modern JavaScript Idioms

These are ES2015+ features TypeScript inherits from JavaScript. Use them consistently.

### Arrow Functions
- Prefer arrow functions for callbacks and short helpers.
- Use classic `function` declarations only when `this` binding or hoisting is needed.
- Return object literals with parentheses: `const make = (id: string) => ({ id });`.

### Destructuring
```ts
const { name, email, role = "member" } = user;
const [first, ...rest] = items;

function greet({ name, age = 18 }: { name: string; age?: number }): string { ... }
```

### Spread and Rest
```ts
const merged = { ...defaults, ...overrides };
const extended = [...base, newItem];

function sum(...nums: number[]): number {
  return nums.reduce((acc, n) => acc + n, 0);
}
```
Object/array spread produces a **shallow** copy. For deep clones use `structuredClone`.

### Optional Chaining and Nullish Coalescing
```ts
const city = user?.address?.city;
const result = obj.method?.();
const value = config.timeout ?? 5000;    // only null/undefined
const flag  = config.enabled || true;    // also triggers on 0, "", false — usually NOT what you want
```
Use `??` for default values, not `||`. `||` treats `0`, `""`, and `false` as "missing".

### Template Literals
Use template literals instead of string concatenation. Use tagged templates for escaping/formatting when HTML/SQL would otherwise be built by concatenation.

### Array Methods
Prefer `map` / `filter` / `reduce` / `flatMap` / `some` / `every` over `for` loops when the transformation is the point. Use `for...of` when side effects dominate. Avoid `forEach` for async work — it does not await.

## Asynchronous Patterns

### async/await over `.then()` Chains
```ts
// Preferred
async function loadDashboard(userId: string): Promise<Dashboard> {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(user.id);
  return { user, posts };
}
```

### Run Independent Work in Parallel
```ts
// Bad — sequential, wastes time
const user = await fetchUser(id);
const config = await fetchConfig();

// Good — parallel
const [user, config] = await Promise.all([fetchUser(id), fetchConfig()]);
```

### Tolerate Partial Failure
```ts
const results = await Promise.allSettled(urls.map(fetchOne));
for (const r of results) {
  if (r.status === "fulfilled") ...;
  else console.error(r.reason);
}
```

### Never `forEach` Over Async Work
```ts
// BAD — the outer call resolves before the inner promises settle.
items.forEach(async (item) => await save(item));

// GOOD — sequential
for (const item of items) await save(item);

// GOOD — concurrent
await Promise.all(items.map((item) => save(item)));
```

### Typed Errors in `catch`
With `useUnknownInCatchVariables`, `catch (e)` types `e` as `unknown`. Narrow before use:
```ts
try {
  await work();
} catch (e) {
  if (e instanceof NotFoundError) return null;
  throw e;
}
```

### Cancel with `AbortController`
Long-running or cancellable work accepts an `AbortSignal` so the caller can abort cleanly. Propagate the signal down every layer that performs I/O.

## Anti-Patterns Checklist

### Types

**`any` used to silence the type checker.** Use `unknown` and narrow, or introduce a proper type.

**Type assertions (`as X`) without a runtime check.** If the value comes from outside the program (JSON, form, `localStorage`), validate it first — do not cast.

```ts
// BAD
const user = JSON.parse(raw) as User;

// GOOD
const parsed = JSON.parse(raw);
if (!isUser(parsed)) throw new Error("malformed user");
const user = parsed; // narrowed to User
```

**Non-null assertion (`!`) on values that are legitimately nullable.** Allowed only when a previous statement has already proved non-null.

**`Function`, `Object`, or `{}` as a type.** These accept almost anything. Use a specific signature (`(x: number) => string`), `Record<string, unknown>`, or `object` depending on intent.

### Mutation and Immutability

**Mutating function arguments.** Return a new value instead. When mutation is intentional, mark the parameter explicitly and document it.

**Shared mutable module-level state.** Module-scope `let` bindings that multiple consumers write to create implicit coupling. Thread state through parameters or inject a state object.

### Async

**Floating promises.** Every promise must be awaited, returned, or explicitly handled with `.catch(...)`. Enable `no-floating-promises` in the project's linter.

**`await` inside a `.forEach` callback.** See above — use `for...of` or `Promise.all`.

**Swallowing rejections.** `promise.catch(() => {})` hides bugs. Log with context and re-throw, or translate to a domain error.

**Mixing `await` and `.then()` in the same chain.** Pick one.

### Errors

**Throwing non-Error values.** `throw "oops"` discards stack traces. Always throw an `Error` (or subclass).

**Catching and rethrowing without context.** When translating, preserve the cause: `throw new DomainError("load failed", { cause: e });`.

**Using exceptions for control flow.** A missing cache entry is not an exception; return `null` or a result type.

### API Design

**Exposing internal structures at the boundary.** Do not return a database row or an ORM entity from a public function. Map to a DTO type defined in the module.

**Boolean flag parameters.** A function taking `(id: string, force: boolean, silent: boolean)` invites call-site confusion. Prefer an options object with named keys, or split into multiple functions.

### Performance

**`JSON.parse(JSON.stringify(x))` for cloning.** Use `structuredClone(x)`.

**Building arrays with repeated `concat` or spread in loops.** Use `push` on a mutable local, or `flatMap`.

## Summary Table

| Anti-Pattern                                  | Fix                                                     |
|-----------------------------------------------|---------------------------------------------------------|
| `any` to silence the checker                  | `unknown` + narrowing, or a real type                   |
| `as X` without a guard                        | Validate at the boundary, then narrow                   |
| Non-null `!` on nullable values               | Check and handle, or restructure to prove non-null      |
| Floating promises                             | `await`, `return`, or attach `.catch`                   |
| `await` inside `.forEach`                     | `for...of` or `Promise.all(items.map(...))`             |
| `||` for defaults on nullable values          | `??`                                                    |
| Mutating arguments                            | Return a new value                                      |
| `JSON.parse(JSON.stringify(x))`               | `structuredClone(x)`                                    |
| Boolean flag parameters                       | Options object or separate functions                    |
| Throwing non-Error values                     | `throw new Error(...)` or a subclass                    |
| Rethrowing without `cause`                    | `new Error(..., { cause: e })`                          |
| Exposing internal types at the boundary       | Map to DTOs                                             |
