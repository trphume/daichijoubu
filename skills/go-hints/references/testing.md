# Go Testing

## Test File Placement and Naming

- Place test files in the same directory as the package under test.
- Use the `_test` package suffix (e.g. `package user_test`) to force black-box testing through public APIs only. This produces tests that survive refactors and verify the real contract.
- Exception: white-box tests (`package user`) are acceptable only for testing unexported helpers that have complex logic worth verifying in isolation. Annotate these with a comment explaining why internal access is needed.

### Separating Unit and Integration Tests

- Do not use build tags to separate unit and integration tests. Build tags hide compilation errors, break IDE support, and are non-discoverable.
- Use an environment variable guard with a shared test helper:

```go
func IntegrationTest(t *testing.T) {
    t.Helper()
    if os.Getenv("INTEGRATION") == "" {
        t.Skip("set INTEGRATION=1 to run integration tests")
    }
}
```

- Every integration test must call this helper as its first line. Unit tests require no annotation — they are the default.
- CI pipelines run `go test ./...` for unit tests and `INTEGRATION=1 go test ./...` for integration tests as separate steps.

## Test Helpers and Shared Setup

- Keep shared test helpers in a `testutil` package. This includes: container lifecycle helpers, database connection factories, fixture builders, and common assertion helpers.
- Container setup (database, message broker, cache) lives in `testutil` and returns a connection string or client. The calling test owns the lifecycle via `t.Cleanup()`.
- Every test helper function must accept `testing.TB` as its first argument and call `t.Helper()` immediately. Using `testing.TB` (the interface) rather than `*testing.T` means helpers work with tests, benchmarks, and fuzz tests.
- Test helpers must never return errors. Call `t.Fatal` on failure inside the helper instead. This eliminates `if err != nil` boilerplate from test bodies and keeps tests focused on the scenario, not on setup plumbing.
- Use `t.Cleanup()` to register teardown logic inside the helper that creates the resource. Do not return cleanup functions for the caller to `defer`. Do not use `defer` in the parent test for resource teardown. `t.Cleanup` is framework-aware — it waits for parallel subtests to complete before running, whereas `defer` executes when the parent goroutine returns, which may be before parallel subtests finish.

```go
// Good — creation and teardown are co-located, caller gets a clean return value.
func newTestDB(tb testing.TB) *sql.DB {
    tb.Helper()
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        tb.Fatal(err)
    }
    tb.Cleanup(func() { db.Close() })
    return db
}

// Bad — caller must remember to defer, and defer runs too early with t.Parallel().
func newTestDB(tb testing.TB) (*sql.DB, func()) {
    // ...
    return db, func() { db.Close() }
}
```

## Table-Driven Tests

- Structure table tests with a `tests` slice of anonymous structs. Each entry has a `name string` field and uses `t.Run(tt.name, ...)` for subtest isolation.
- Include a `setup` function field in the table struct when test cases need different preparation. Avoid complex `if/else` chains inside the test loop.
- For tests that verify error conditions, include both `wantErr bool` and `wantErrIs error` (or `wantErrMsg string`) fields. Check the error type, not just its presence.

```go
tests := []struct {
    name      string
    input     CreateUserRequest
    setup     func(s *Service)
    wantErr   bool
    wantErrIs error
}{
    {
        name:    "succeeds with valid input",
        input:   CreateUserRequest{Name: "alice", Email: "alice@example.com"},
        setup:   func(s *Service) { /* seed state */ },
        wantErr: false,
    },
    {
        name:      "returns error when email already exists",
        input:     CreateUserRequest{Name: "bob", Email: "taken@example.com"},
        setup:     func(s *Service) { s.Seed(User{Email: "taken@example.com"}) },
        wantErr:   true,
        wantErrIs: domain.ErrDuplicateEmail,
    },
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        svc := NewService()
        if tt.setup != nil {
            tt.setup(svc)
        }

        err := svc.CreateUser(context.Background(), tt.input)

        if tt.wantErr {
            if err == nil {
                t.Fatalf("expected error, got nil")
            }
            if tt.wantErrIs != nil && !errors.Is(err, tt.wantErrIs) {
                t.Fatalf("expected error %v, got %v", tt.wantErrIs, err)
            }
            return
        }
        if err != nil {
            t.Fatalf("unexpected error: %v", err)
        }
    })
}
```

## Golden Files

- Use golden files for asserting large or complex outputs (JSON response bodies, HTML, formatted text, CLI output). If expected values fit comfortably inline in a table struct, prefer inline expectations — golden files add indirection and are only worthwhile when the output is too large or too noisy to embed in code.
- Store golden files in the `testdata/` directory, which `go build` ignores by convention. Name them after the test using `t.Name()` (e.g. `testdata/TestToJSON.golden`). For subtests, use subdirectories: `testdata/TestToJSON/valid_input.golden`.
- Provide an `-update` flag to regenerate golden files from actual output. Register a package-level flag: `var update = flag.Bool("update", false, "update golden files")`. When set, the test writes the actual output to the golden file instead of comparing. Run `go test ./... -update` to regenerate.
- Always review golden file diffs before committing. Golden files are the expected output — they are test assertions stored as files and must be reviewed during code review the same as any other test expectation.
- Normalize non-deterministic fields (timestamps, UUIDs, random IDs) before writing or comparing golden files. Either strip them, replace them with stable placeholders, or use a custom comparison function that ignores them.

### Error Assertions

- Prefer semantic error assertions over string matching. Never use string comparison or substring matching on error messages — this creates change-detector tests that break when error wording is improved. Use `errors.Is(err, ErrExpected)` for sentinel errors and `errors.As(err, &target)` for typed errors.
- Only assert on errors that your code explicitly defines and that callers depend on. If the function returns a sentinel like `domain.ErrNotFound`, assert on it. If it wraps an internal error with no defined type, asserting `err != nil` is sufficient.
- When a dependency's errors lack sufficient semantic information for your tests, treat that as a design signal — consider wrapping them with your own sentinel or typed error at the boundary rather than parsing upstream messages.

## Integration Test Lifecycle

- Start containers once per package using `TestMain`, not per test. Container startup is expensive; amortize it.
- Run database migrations programmatically during setup using the same migration tool and files used in production deployments.
- Isolate individual tests using one of these strategies (choose one per project and be consistent):
  - **Per-test transaction rollback**: wrap each test in a transaction, rollback on cleanup. Fastest, but breaks if application code manages its own transactions.
  - **Per-test schema**: create a uniquely-named schema per test, drop on cleanup. Good isolation with moderate overhead.
  - **Per-test database (template cloning)**: clone from a migrated template database. Strongest isolation, best for parallel tests, slightly higher overhead.
  - **UUID-scoped assertions**: share the database, seed with unique identifiers, assert only against your data. Simplest, requires discipline.

## Parallel Test Safety

- Call `t.Parallel()` explicitly in tests designed for concurrent execution. Never assume tests run in parallel by default.
- Tests using shared database state without an isolation strategy must not call `t.Parallel()`.

### Context and Timeouts

- Always pass `context.Background()` (or a test-scoped context with timeout) to functions under test. Never pass `nil` as a context.
- For integration tests that talk to real infrastructure, set a test-level timeout to prevent hung tests from blocking CI:

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
t.Cleanup(cancel)
```

## What Not to Test

- Do not write tests for auto-generated code. Test the code that uses them.
- Do not test the standard library. Test your usage of it.
- Do not test unexported struct field assignments or internal state. Test observable behavior through public methods.

## CI Expectations

- All tests (unit and integration) must pass with the race detector enabled: `go test -race ./...`.
- Use `-count=1` in CI to disable test caching. Cached results can mask flaky tests.
- Integration tests in CI must produce the same results as local runs. Achieve this by using programmatic container management rather than CI-specific service declarations.
