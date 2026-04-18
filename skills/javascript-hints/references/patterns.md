# JavaScript Patterns and Anti-Patterns

## Design Principles

### KISS — Keep It Simple
Choose the simplest solution that works. Prefer a plain object literal over a factory, a plain function over a class hierarchy, a union of string literals over an enum.

### Single Responsibility
Each module and function has one reason to change. Split transport/serialization, domain logic, and persistence into separate units.

### Composition Over Inheritance
Combine small, focused objects through constructor arguments. Use inheritance only for genuine "is-a" relationships.

```js
class NotificationService {
  constructor(email, sms, push) {
    this.email = email;
    this.sms = sms;
    this.push = push;
  }

  async notify(user, message, channels = new Set(["email"])) {
    if (channels.has("email")) await this.email.send(user.email, message);
    if (channels.has("sms") && this.sms && user.phone) await this.sms.send(user.phone, message);
  }
}
```

### Rule of Three
Wait until three instances before extracting an abstraction. Two similar blocks are often cheaper to keep duplicated than the wrong shared abstraction.

### Function Size
Extract when a function exceeds ~20–50 lines, mixes distinct purposes, or has 3+ levels of nesting.

## Modern Syntax Idioms

### Arrow Functions
- Arrow functions do **not** bind their own `this`, `arguments`, `super`, or `new.target`. They inherit them from the enclosing scope.
- Use arrow functions for callbacks and short helpers.
- Use `function` (or method shorthand) when `this` must refer to the call-site receiver or the function is used as a constructor.

```js
class Counter {
  constructor() {
    this.count = 0;
  }
  // Arrow preserves `this` across setTimeout
  incrementLater = () => { setTimeout(() => { this.count++; }, 1000); };
}
```

### Destructuring
```js
const { name, email, role = "member" } = user;
const [first, ...rest] = items;
const { nested: { city } = {} } = user; // default when `nested` is missing
```

### Spread and Rest
Shallow copies only. For deep clones use `structuredClone(value)`, not `JSON.parse(JSON.stringify(value))`.

```js
const merged = { ...defaults, ...overrides };
const extended = [...base, item];
function sum(...nums) { return nums.reduce((a, n) => a + n, 0); }
```

### Optional Chaining and Nullish Coalescing
```js
const city = user?.address?.city;
const timeout = config.timeout ?? 5000;   // only null/undefined
const flag    = config.enabled || true;   // ALSO triggers on 0, "", false — usually wrong
```

Use `??` for default values on nullable data. `||` misfires on legitimate falsy values (`0`, `""`, `false`).

### Template Literals
Prefer template literals over string concatenation. Use tagged templates when building HTML/SQL to centralize escaping.

### Array Methods
Prefer `map` / `filter` / `reduce` / `flatMap` / `some` / `every` over `for` loops when the transformation is the point. Use `for...of` when side effects dominate.

## Asynchronous Patterns

### async/await over `.then()` Chains
```js
async function loadDashboard(userId) {
  const user = await fetchUser(userId);
  const posts = await fetchPosts(user.id);
  return { user, posts };
}
```

### Parallelize Independent Work
```js
// Bad — sequential, wastes time
const user = await fetchUser(id);
const config = await fetchConfig();

// Good — parallel
const [user, config] = await Promise.all([fetchUser(id), fetchConfig()]);
```

### Tolerate Partial Failure
```js
const results = await Promise.allSettled(urls.map(fetchOne));
for (const r of results) {
  if (r.status === "fulfilled") use(r.value);
  else console.error(r.reason);
}
```

### `Promise.any` vs. `Promise.race`
- `Promise.any(...)` — resolves with the first **fulfilled** value. Rejects only if all reject.
- `Promise.race(...)` — settles with the first settled promise (resolve *or* reject). Useful for timeouts.

### Never `forEach` Over Async Work
```js
// BAD — outer call resolves before inner promises settle.
items.forEach(async (item) => await save(item));

// GOOD — sequential
for (const item of items) await save(item);

// GOOD — concurrent
await Promise.all(items.map((item) => save(item)));
```

### Cancel with `AbortController`
Long-running or cancellable work accepts an `AbortSignal`; propagate it down every I/O layer.

```js
const ac = new AbortController();
setTimeout(() => ac.abort(), 5000);
const res = await fetch(url, { signal: ac.signal });
```

### Error Handling
- `throw` an `Error` (or subclass). Never throw strings, numbers, or objects.
- Preserve the original cause when translating: `throw new Error("load failed", { cause: e });`.
- In `catch (e)`, the caught value may be anything. Guard before reading properties.

## Numbers and Equality

- **Always** use `===`/`!==`.
- `0.1 + 0.2 !== 0.3`. For financial math, work in integer "minor units" (cents) or use `BigInt`/a decimal library (library belongs in its own skill).
- `Number.MAX_SAFE_INTEGER` is `2^53 - 1`. Use `BigInt` literals (`10n`) when working with integers larger than that.
- `Number.isNaN(x)` and `Number.isFinite(x)` — not the global versions, which coerce.
- `Object.is(NaN, NaN)` is `true`; `NaN === NaN` is `false`.

## Immutability and Copying
- `...` (spread) and `Object.assign` produce **shallow** copies. Nested objects are still shared references.
- Deep clones: `structuredClone(value)`. Handles `Date`, `Map`, `Set`, typed arrays, cycles. Does **not** clone functions, DOM nodes, prototype chains.
- `Object.freeze(obj)` is shallow. Use a recursive freeze helper for deep immutability.

## Anti-Patterns Checklist

### Syntax and Scoping

**`var`.** Always `const`/`let`.

**Implicit globals.** Always declare. Enable strict mode (automatic in ESM) so assignments to undeclared identifiers throw.

**`arguments` object in new code.** Use rest parameters (`...args`).

**Relying on `==` coercion.** Use `===`.

### Async

**Floating promises.** Every promise must be `await`ed, `return`ed, or `.catch`-handled. A promise with no handler that rejects crashes Node (and logs in browsers).

**`await` inside `forEach`.** See above — `forEach` ignores returned promises.

**`new Promise((res, rej) => { ... res(await x); ... })` / the explicit-`Promise` antipattern.** If the function is already async, return the promise directly — do not re-wrap.

```js
// BAD
function loadUser(id) {
  return new Promise(async (resolve, reject) => {
    try { resolve(await fetch(`/u/${id}`)); }
    catch (e) { reject(e); }
  });
}

// GOOD
async function loadUser(id) { return fetch(`/u/${id}`); }
```

**Swallowing rejections with `.catch(() => {})`.** Log with context and re-throw, or translate.

**Mixing `await` and `.then()` in the same chain.** Pick one.

### Errors

**Throwing non-Error values.** `throw "oops"` discards stack traces.

**Catching without narrowing.** The caught value is `unknown` in spirit — check type before reading properties.

**Using exceptions for control flow.** Missing cache entries and "not found" results should be return values, not exceptions.

### Mutation

**Mutating function arguments.** Return a new value. When mutation is intentional, document it and name the function `mutatingX` / `inPlaceX`.

**Shared module-level mutable state.** Thread state through parameters or an injected state object.

**`JSON.parse(JSON.stringify(x))` for deep copy.** Use `structuredClone(x)`.

### API Design

**Boolean flag parameters.** `send(msg, true, false)` is unreadable. Use an options object: `send(msg, { retry: true, silent: false })`.

**Exposing internal structures at the boundary.** Do not return a database row from a public function — map to a plain object with a known shape.

### Numbers

**Floating-point math on money.** Work in integer minor units or `BigInt`.

**Global `isNaN`/`isFinite`.** Use the `Number.*` versions.

## Summary Table

| Anti-Pattern                                  | Fix                                                     |
|-----------------------------------------------|---------------------------------------------------------|
| `var`                                         | `const` / `let`                                         |
| `==` / `!=`                                   | `===` / `!==`                                           |
| Floating promises                             | `await`, `return`, or `.catch`                          |
| `await` inside `.forEach`                     | `for...of` or `Promise.all(items.map(...))`             |
| `new Promise(async ...)` wrapping             | Return the `async` function's promise directly          |
| `||` for defaults on nullable values          | `??`                                                    |
| Mutating arguments                            | Return a new value                                      |
| `JSON.parse(JSON.stringify(x))`               | `structuredClone(x)`                                    |
| Boolean flag parameters                       | Options object or separate functions                    |
| Throwing non-Error values                     | `throw new Error(...)` or a subclass                    |
| Rethrowing without `cause`                    | `new Error(..., { cause: e })`                          |
| `.catch(() => {})` swallowing rejection       | Log and re-throw or translate                           |
| Exposing internal types at the boundary       | Map to plain DTO objects                                |
| Global `isNaN` / `isFinite`                   | `Number.isNaN` / `Number.isFinite`                      |
| Floating-point arithmetic on money            | Integer minor units or `BigInt`                         |
