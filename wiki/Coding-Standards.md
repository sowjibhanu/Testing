# Coding Standards

This page documents the conventions and best practices followed in the Baton codebase.

## Language & Build

- **Language**: Go (golang)
- **Minimum Version**: Go 1.x (see `Dockerfile` for build environment)
- **Dependency Manager**: [Go Dep](https://golang.github.io/dep/) (see `Gopkg.toml`)
- **Build**: Multi-stage Docker build with static binary output (see `Dockerfile`)

## Code Formatting

**Mandatory**: All Go files must be formatted with `gofmt -w` before committing.

From `CONTRIBUTING.md`:
> Ensure that you run `gofmt -w file.go` against any file which you have modified to ensure consistent formatting across the project.

This is enforced in pull requests.

## Naming Conventions

- **Types**: PascalCase (e.g., `Baton`, `Configuration`, `HTTPResult`)
- **Unexported types**: camelCase (e.g., `worker`, `countWorker`, `timedWorker`)
- **Functions**: PascalCase for exported, camelCase for unexported (e.g., `newCountWorker()`, `performRequest()`)
- **Variables**: camelCase (e.g., `numberOfRequests`, `preLoadedRequests`, `connectionErrorCount`)
- **Constants**: camelCase (e.g., `port = "8888"`)

Examples from the codebase:
- `baton.go:47` — `type Baton struct` (exported type)
- `worker.go:13` — `type worker struct` (unexported type)
- `worker.go:18` — `type workable interface` (exported interface)
- `count_worker.go:22` — `func newCountWorker()` (unexported constructor)

## File Organization

**One concept per file**: Each file focuses on a single responsibility.

- `baton.go` — Main orchestrator, CLI parsing, request coordination
- `worker.go` — Base worker implementation and common methods
- `count_worker.go` — Worker for fixed request counts
- `timed_worker.go` — Worker for time-based testing
- `configuration.go` — Configuration struct and validation
- `csv_parsing.go` — CSV file parsing for bulk requests
- `result.go` — Result aggregation and formatted output
- `http_result.go` — HTTP response counters and totals
- `log_writer.go` — Custom logging with suppress capability
- `baton_test.go` — Integration tests

## Error Handling

**Early validation**: Configuration is validated before execution.

From `configuration.go:33`:
```go
func (configuration *Configuration) validate() error {
    if configuration.concurrency < 1 || configuration.numberOfRequests == 0 {
        return errors.New("invalid concurrency level or number of requests")
    }
    return nil
}
```

**Error returns**: Functions return errors as the last return value.

From `csv_parsing.go:32`:
```go
func preLoadRequestsFromFile(filename string) ([]preLoadedRequest, error)
```

**Fatal errors**: Critical errors during execution call `log.Fatalf()`.

From `baton.go:88`:
```go
if err != nil {
    log.Fatalf("Invalid configuration: %v", err)
}
```

## Testing

**Integration tests**: Tests use a local HTTP server to verify end-to-end behavior.

From `baton_test.go:60`:
```go
func startServer() *HTTPTestHandler {
    // Starts a fasthttp server on port 8888
    // Tests verify requests are received correctly
}
```

**Test coverage**: Tests verify:
- Request count accuracy (`TestRequestCount`, `TestRequestCountWithMoreWorkers`)
- HTTP method correctness (`TestThatTheCorrectHTTPMethodIsUsed`)
- Request body transmission (`TestThatBodyHasCorrectValue`)
- URI handling (`TestThatServerReceivesCorrectURI`)
- File-based requests (`TestPostRequestLoadedFromFile`)
- Custom headers (`TestThatHeadersAreSetWhenSendingFromFile`)
- Timing accuracy (`TestThatTimeOptionRunsForCorrectAmountOfTime`)
- File body loading (`TestLoadPostFromTextFile`)

Run tests with: `go test -v`

## Concurrency

**Goroutines**: Workers run as goroutines, one per concurrent request.

From `baton.go:115`:
```go
for w := 1; w <= baton.configuration.concurrency; w++ {
    var worker workable
    // ...
    go worker.sendRequests(preparedRunConfiguration.preLoadedRequests)
}
```

**Channels**: Coordination between workers and main thread uses channels.

From `baton.go:130`:
```go
requests := make(chan bool, configuration.numberOfRequests)
results := make(chan HTTPResult, configuration.concurrency)
done := make(chan bool, configuration.concurrency)
```

**Atomic operations**: Response counting uses atomic operations for thread safety.

From `baton_test.go:42`:
```go
atomic.AddUint32(&h.noRequestsReceived, 1)
```

## HTTP Client

**FastHTTP**: Uses `github.com/valyala/fasthttp` for high-performance HTTP.

From `Gopkg.toml:27`:
```toml
[[constraint]]
  name = "github.com/valyala/fasthttp"
  revision = "e5f51c11919d4f66400334047b897ef0a94c6f3c"
```

**Request reuse**: FastHTTP requests and responses are acquired and released for efficiency.

From `worker.go:60`:
```go
req := fasthttp.AcquireRequest()
resp := fasthttp.AcquireResponse()
// ... use req and resp ...
```

## Docker & Deployment

**Multi-stage build**: Reduces final image size by building in one stage and copying only the binary.

From `Dockerfile:1-30`:
```dockerfile
FROM golang:alpine as builder
# ... build stage ...
RUN go build -a -installsuffix cgo -o /go/bin/baton

FROM scratch
# ... copy only binary and certs ...
COPY --from=builder /go/bin/baton /go/bin/baton
```

**Static binary**: Built with `CGO_ENABLED=0` for portability.

From `Dockerfile:20`:
```dockerfile
ENV CGO_ENABLED 0
```

## Comments

- **Exported types**: Documented with comment above the type definition
- **Unexported helpers**: May have inline comments explaining logic
- **Complex logic**: Inline comments explain non-obvious behavior

Examples:
- `baton.go:47` — `// Baton implements the load tester`
- `worker.go:73` — `// The first request is associated with overhead...`
