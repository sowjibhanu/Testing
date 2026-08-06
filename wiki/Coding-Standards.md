# Coding Standards

This page documents the conventions and best practices followed in the Baton codebase.

## Language & Build

- **Language**: Go (golang)
- **Minimum Version**: Go 1.x (compatible with Alpine Linux)
- **Dependency Manager**: [Dep](https://github.com/golang/dep) (see `Gopkg.toml:line 1`)
- **Build Tool**: Standard `go build` with CGO disabled for static binaries
- **Docker**: Multi-stage builds for minimal image size (see `Dockerfile:line 1`)

## Code Formatting

**Mandatory**: All Go files must be formatted with `gofmt -w` before committing.

Reference: `CONTRIBUTING.md` — "Ensure that you run `gofmt -w file.go` against any file which you have modified"

All files follow standard Go formatting conventions with:
- Tab indentation (Go standard)
- Line length: no hard limit, but keep readable
- Imports: organized by standard library, then third-party

## Naming Conventions

### Types & Interfaces
- **PascalCase** for exported types: `Baton`, `Configuration`, `HTTPResult`, `Result`
- **camelCase** for unexported types: `worker`, `countWorker`, `timedWorker`
- **Interface names** end with `-able`: `workable` (see `worker.go:line 30`)

### Functions & Methods
- **PascalCase** for exported functions: `newResult()`, `newWorker()`, `newCountWorker()`
- **camelCase** for unexported functions: `prepareRun()`, `processResults()`, `configureLogging()`
- **Receiver methods** use pointer receivers for mutation: `(baton *Baton) run()`

### Variables & Constants
- **camelCase** for local variables: `configuration`, `preparedRunConfiguration`, `timeSum`
- **UPPER_CASE** for package-level flags: `body`, `concurrency`, `duration` (see `baton.go:line 23`)
- **Descriptive names**: `connectionErrorCount`, `status2xxCount`, `responseTimesPercent`

## File Organization

**One concept per file** — Each file focuses on a single responsibility:

- `baton.go` — Main orchestrator, CLI parsing, result processing (see `baton.go:line 1`)
- `worker.go` — Base worker type, HTTP execution, statistics collection (see `worker.go:line 1`)
- `count_worker.go` — Fixed-count request worker (see `count_worker.go:line 1`)
- `timed_worker.go` — Time-based request worker (see `timed_worker.go:line 1`)
- `configuration.go` — Configuration struct and validation (see `configuration.go:line 1`)
- `result.go` — Result aggregation and formatted output (see `result.go:line 1`)
- `http_result.go` — HTTP response statistics (see `http_result.go:line 1`)
- `csv_parsing.go` — CSV request file parsing (see `csv_parsing.go:line 1`)
- `log_writer.go` — Custom logging control (see `log_writer.go:line 1`)

## Error Handling

**Early validation**: Configuration is validated immediately after parsing (see `baton.go:line 105`)

```go
err := baton.configuration.validate()
if err != nil {
    log.Fatalf("Invalid configuration: %v", err)
}
```

**Error returns**: Functions return errors as the last return value (see `csv_parsing.go:line 32`)

**Fatal errors**: Use `log.Fatalf()` for unrecoverable errors during initialization

## Testing

**Integration tests** with local HTTP server (see `baton_test.go:line 1`):
- Tests use a real HTTP server on port 8888
- Each test sets up its own configuration
- Tests verify request counts, methods, bodies, URIs, and headers
- Tests use hex encoding for body verification

Example test structure (see `baton_test.go:line 70`):
```go
func TestRequestCount(t *testing.T) {
    config := defaultConfig()
    config.numberOfRequests = 10000
    testHandler := setupAndListen(config)
    // assertions...
}
```

## Concurrency

**Goroutines with channels** for worker coordination:
- Workers are spawned as goroutines (see `baton.go:line 118`)
- Channels coordinate work distribution and result collection
- `requests` channel: distributes work to workers
- `results` channel: collects HTTP statistics from workers
- `done` channel: signals worker completion

**Atomic operations** for thread-safe counters (see `baton_test.go:line 35`):
```go
atomic.AddUint32(&h.noRequestsReceived, 1)
atomic.StoreInt64(&h.lastTimestamp, time.Now().Unix())
```

## Comments

- **Package-level comments**: Describe the purpose of exported types
- **Function comments**: Exported functions have doc comments (e.g., `// Baton implements the load tester`)
- **Inline comments**: Used sparingly for non-obvious logic
- **TODO/FIXME**: Not currently used in the codebase

## Dependencies

**Minimal external dependencies**:
- `github.com/valyala/fasthttp` — High-performance HTTP client (see `Gopkg.toml:line 26`)
- Standard library: `crypto/tls`, `encoding/csv`, `flag`, `fmt`, `io`, `log`, `math`, `os`, `strings`, `sync/atomic`, `time`

## Docker & Deployment

**Multi-stage builds** (see `Dockerfile:line 1`):
1. Builder stage: Compiles with full Go toolchain
2. Runtime stage: Uses `scratch` image with only the binary and CA certificates
3. Static binary: `CGO_ENABLED=0` for portability
4. Non-root user: `batonuser` for security

## License & Copyright

All source files include Apache License 2.0 header (see `baton.go:line 1`):
```
/*
 * Copyright 2026 WWT
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...
 */
```
