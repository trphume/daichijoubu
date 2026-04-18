---
name: go-hints
description: This skill should be used when writing, reviewing, or refactoring Go code — any time files match `**/*.go`, `**/go.mod`, or `**/go.sum`, or the user mentions Go/Golang conventions, error handling, interfaces, goroutines/channels, context, `testing.T`, table-driven tests, golden files, `t.Cleanup`, integration tests, or `gofmt`/`goimports`/`go vet`. Covers general Go language guidance only — conventions, patterns (errors, concurrency, DI, HTTP, performance, functional options), and testing rules (helpers, table tests, golden files, integration lifecycle, parallel safety). Third-party library or linter rules belong in their own skills.
---

# go-hints

Project-wide Go guidance. Apply these rules when authoring or modifying Go code in this repository.

## When to apply

Apply when any of the following is true:
- The task touches a file matching `**/*.go`, `**/go.mod`, or `**/go.sum`
- The user is designing a Go package, service, or CLI
- The user asks about Go idioms, error handling, concurrency, HTTP handlers, dependency injection, or performance tuning
- The user is writing or reviewing Go tests, test helpers, table-driven tests, golden files, or integration tests

## How to apply

1. Before writing or editing Go code, load the reference file(s) relevant to the task. Do not rely on memory — these rules may evolve.
2. Prefer the obvious, readable solution. Do not introduce abstractions, closures, or indirection that the task does not require.
3. Match the repository's existing patterns before applying a rule from the references — if a file already follows a rule, stay consistent; if it violates one, surface the mismatch rather than silently "fixing" surrounding code.
4. For test changes, confirm which kind of test (unit vs. integration) and which isolation strategy the project uses before adding new tests.

## Scope

These references cover the Go language and its standard library + official toolchain (`go`, `gofmt`, `goimports`, `go vet`, `go test`, `go mod`). Guidance for specific third-party libraries (assertion helpers, mocking frameworks, external linters, web frameworks, ORMs, etc.) belongs in dedicated skills.

## Non-negotiables (quick reference)

- `gofmt` and `goimports` — code must be formatted; imports grouped stdlib / external / internal.
- Errors are values: handle them explicitly; wrap with `fmt.Errorf("context: %w", err)`; never discard with `_`.
- Never use `log.Fatal*` or `os.Exit` outside `main()` after deferred cleanup has run.
- `context.Context` is the first parameter of any function that performs I/O, blocks, or spawns goroutines.
- No goroutine without a stop signal (context cancellation or closed channel).
- Accept interfaces, return concrete types. Define interfaces at the consumer.
- Tests use `_test` package suffix (black-box) by default. Helpers take `testing.TB`, call `t.Helper()`, and register teardown with `t.Cleanup()`.
- Assert errors with `errors.Is` / `errors.As`, never string matching.
- CI runs `go test -race -count=1 ./...`.

## Reference Files

Load the specific file for the task — do not load all three unnecessarily.

- **`references/conventions.md`** — Core principles, naming, project layout (`cmd/`, `internal/`, `pkg/`), formatting, and dependency hygiene (`go mod tidy`, `go vet`, `tools.go`).
- **`references/patterns.md`** — Error handling, interfaces, concurrency (worker pools, graceful shutdown, goroutine-leak avoidance), dependency injection, HTTP handlers, performance (`sync.Pool`, `strings.Builder`, preallocation), and functional options.
- **`references/testing.md`** — Test placement, unit vs. integration separation via env var, `testutil` helpers, `t.Cleanup` vs. `defer`, table-driven tests, golden files with `-update` flag, error assertions, integration lifecycle and isolation strategies, `t.Parallel()` safety, context/timeouts, and CI expectations.
