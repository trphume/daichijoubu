---
name: python-hints
description: This skill should be used when writing, reviewing, or refactoring Python code — any time files match `**/*.py` or `**/pyproject.toml`, or the user mentions Python conventions, type hints, generics, Protocols, dataclasses, `asyncio`, `unittest`, `unittest.mock`, context managers, or error handling. Covers general Python language guidance only — conventions, patterns and anti-patterns, stdlib-based testing, and type safety. Specific third-party tool, framework, or library rules (linters, formatters, type checkers, test frameworks, validation libraries, web frameworks, ORMs) belong in their own skills.
---

# python-hints

Project-wide Python language guidance. Apply these rules when authoring or modifying Python code in this repository.

## Scope

These references cover the Python language, its standard library, and the official toolchain (`python`, `unittest`, `pip`, `venv`). Guidance for specific third-party libraries and tools — linters/formatters, type checkers, test frameworks (pytest and similar), validation libraries, web frameworks, ORMs, HTTP clients, etc. — belongs in dedicated skills.

## When to apply

Apply when any of the following is true:
- The task touches a file matching `**/*.py`, `**/pyproject.toml`, `**/requirements*.txt`, or `**/setup.cfg`.
- The user is designing a Python package, service, CLI, or library.
- The user asks about Python idioms, error handling, async, dependency injection, or performance.
- The user is writing or reviewing stdlib-based tests, `unittest` tests, or `unittest.mock` usage.
- The user is adding type annotations, defining `Protocol`s, using generics, or configuring type checking.

## How to apply

1. Before writing or editing Python code, load the reference file(s) relevant to the task. Do not rely on memory — these rules may evolve.
2. Prefer the obvious, readable solution. Do not introduce abstractions, closures, or indirection that the task does not require.
3. Match the repository's existing patterns before applying a rule — if a file already follows a rule, stay consistent; if it violates one, surface the mismatch rather than silently "fixing" surrounding code.
4. If a rule touches a specific third-party tool the project uses, defer to that tool's dedicated skill and do not restate its rules here.

## Non-negotiables (quick reference)

- Target Python 3.10+; use `T | None`, `list[T]`, `dict[K, V]` — not `Optional`, `List`, `Dict`.
- Annotate every public function parameter, return type, and class attribute.
- Never use bare `except:` or `except Exception: pass`. Catch specific exceptions; log or re-raise. When translating, use `raise NewError(...) from e`.
- Never call blocking I/O (`time.sleep`, synchronous sockets, synchronous file I/O) inside `async def`. Use async-native APIs or `asyncio.to_thread`.
- Always use context managers for resources (`with open(...)`, `async with`).
- Validate external input at the boundary. Never unpack a raw `dict` into a domain model.
- Do not expose persistence models through API responses — translate to DTOs.
- No mutable default arguments. Default to `None` and build the value inside the function.
- Tests: one behavior per test, always cover error paths, mock at boundaries not internal collaborators.
- Every `# type: ignore` must have a specific error code and an explanatory comment.

## Reference Files

Load the specific file(s) for the task — do not load all four unnecessarily.

- **`references/conventions.md`** — Target version, PEP 8 naming, import grouping, Google-style docstrings, line length, packaging layout, dataclass/enum usage.
- **`references/patterns.md`** — Design principles (KISS, SRP, separation of concerns, composition over inheritance, Rule of Three, function size, DI) and an anti-patterns checklist (exposed persistence models, mixed I/O + business logic, bare except, `raise from`, unclosed resources, blocking in async, mutable defaults, scattered retry, hard-coded config).
- **`references/testing.md`** — Layout, AAA pattern, test naming, `unittest` setup/teardown, `subTest` parameterization, mocking at boundaries with `unittest.mock`, retries, time injection, async tests with `IsolatedAsyncioTestCase`, what not to test.
- **`references/type-safety.md`** — Modern annotations, union syntax, type narrowing with guards and `assert_never`, generics with `TypeVar` bounds and constraints, structural typing with `Protocol`, type aliases (PEP 695 vs. `TypeAlias`), `Callable`/`Awaitable`, `TypedDict`/`NamedTuple`, minimizing `Any`, `# type: ignore` hygiene.
