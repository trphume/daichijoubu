# Go Conventions

## Core Principles
- Prefer obvious, readable solutions over clever implementations.
- If a simpler approach works, use it — avoid unnecessary abstractions, closures, or indirection.

```go
// Good — obvious and readable
func GetUser(id string) (*User, error) {
    user, err := db.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("get user %s: %w", id, err)
    }
    return user, nil
}

// Bad — unnecessary closure and indirection
func GetUser(id string) (*User, error) {
    return func() (*User, error) {
        if u, e := db.FindUser(id); e == nil {
            return u, nil
        } else {
            return nil, e
        }
    }()
}
```

## Naming
- Short variable names in small scopes: `r` for reader, `ctx` for context.
- Descriptive names for exported functions and types.
- Package names: short, lowercase, singular (`auth`, not `authentication`).
- No stuttering: `auth.Client`, not `auth.AuthClient`.
- Interfaces: behavioral names — `Reader`, `Validator`, `Publisher` (not `IReader`).

## Project Layout
- `cmd/` for application entry points.
- `internal/` for private application code.
- `pkg/` only for truly reusable library code (prefer `internal/`).
- Keep `main.go` thin — wire dependencies, start server, that's it.

## Formatting
- `gofmt` is non-negotiable — code must be formatted.
- Group imports into blocks: stdlib, external, internal. Use `goimports` if available.

## Tooling and Dependencies
- Run `go mod tidy` before committing.
- Run `go vet ./...` as part of CI.
- Use `//go:build tools`-style pins in a `tools.go` file for any dev tools the project depends on, so they are tracked in `go.mod`.
- Configuration for any additional linter belongs in its own dedicated skill, not here.
