# Go Patterns

## Error Handling
- Always handle errors explicitly — never use `_` to discard errors
- Use `fmt.Errorf("context: %w", err)` to wrap errors with context
- Define sentinel errors with `errors.New` for expected failure cases
- Use `errors.Is()` and `errors.As()` for error checking, not string comparison
- Return errors to the caller; only log at the top-level handler
- Avoid `os.Exit(1)` and `log.Fatal`/`log.Fatalf` — they terminate immediately and skip all deferred functions (cleanup, flushes, unlocks). Return errors up the call stack instead; only use `os.Exit` in `main()` after deferred functions have run

## Interfaces
- Prefer defining interfaces at the consumer, not the producer
- For domain contracts shared across multiple packages, a shared `domain/` or `port/` package is acceptable
- Keep interfaces small — prefer 1-2 methods
- Accept interfaces, return concrete types
- Use interface composition to build larger contracts from small ones
- Use type assertions to detect optional behavior (e.g., check if `io.Writer` also implements `Flusher`)

```go
func WriteAndFlush(w io.Writer, data []byte) error {
    if _, err := w.Write(data); err != nil {
        return err
    }
    if f, ok := w.(Flusher); ok {
        return f.Flush()
    }
    return nil
}
```

## Concurrency
- Prefer channels for coordination, mutexes for state protection.
- Always pass `context.Context` as the first parameter.
- Use `errgroup` for parallel tasks that can fail — it propagates the first error and cancels siblings.
- Never start goroutines without a way to stop them (context cancellation or a closed channel).
- Use `sync.Once` for lazy initialization, not double-checked locking.
- Worker pools: fan out with `chan Job`, collect with `chan Result`, close results after `wg.Wait()`.
- Graceful shutdown: trap `SIGINT`/`SIGTERM`, call `server.Shutdown(ctx)` with a timeout.
- Avoid goroutine leaks: use buffered channels or `select` with `ctx.Done()` so goroutines never block forever.

```go
// Worker pool
func WorkerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    wg.Wait()
    close(results)
}

// Graceful shutdown
func GracefulShutdown(server *http.Server) {
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Printf("server forced to shutdown: %v", err)
    }
}

// Avoiding goroutine leaks — use buffered channel + select
func safeFetch(ctx context.Context, url string) <-chan []byte {
    ch := make(chan []byte, 1)
    go func() {
        data, err := fetch(url)
        if err != nil {
            return
        }
        select {
        case ch <- data:
        case <-ctx.Done():
        }
    }()
    return ch
}
```

## Dependency Injection
- Wire dependencies in `main.go` or a dedicated `wire.go`
- Constructor functions: `NewService(deps...) *Service`
- Accept interfaces as constructor parameters for testability
- No global mutable state — pass dependencies explicitly

## HTTP Handlers
- Middleware pattern for cross-cutting concerns (logging, auth, recovery)
- Always set timeouts on `http.Server` and `http.Client`
- Decode request → validate → call domain logic → encode response

## Performance
- Preallocate slices when size is known: `make([]T, 0, len(items))`
- Use `sync.Pool` to reuse frequently allocated objects (e.g., `bytes.Buffer`) and reduce GC pressure
- Use `strings.Builder` for string concatenation in loops — or `strings.Join` when you already have a slice

```go
// Preallocate slices
results := make([]Result, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}

// sync.Pool for reusable buffers
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func ProcessRequest(data []byte) []byte {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    buf.Write(data)
    return buf.Bytes()
}

// strings.Builder for concatenation in loops
var sb strings.Builder
for i, p := range parts {
    if i > 0 {
        sb.WriteString(",")
    }
    sb.WriteString(p)
}
return sb.String()
```

## Functional Options

```go
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```
