# Baton Wiki

Welcome to **Baton**, a high-performance load testing tool written in Go.

## What is Baton?

Baton is a command-line load testing tool that sends HTTP requests (GET, POST, PUT, DELETE) to a target server and measures performance metrics. It supports:

- **Concurrent requests** — Multiple workers sending requests in parallel
- **Two execution modes** — Fixed request count (`-r`) or time-based duration (`-t`)
- **Custom request bodies** — Inline (`-b`) or from file (`-f`)
- **Batch requests** — Load complex requests from CSV files (`-z`)
- **Custom headers** — Per-request headers when using CSV mode
- **TLS/SSL bypass** — Ignore certificate validation (`-i`)

## Quick Start

```bash
# Install
go get -u github.com/americanexpress/baton

# Run 200,000 requests with 10 concurrent workers
baton -u http://localhost:8080/test -c 10 -r 200000

# Run for 30 seconds with 5 workers
baton -u http://localhost:8080/test -c 5 -t 30
```

See [README.md](../README.md) for full usage options.

## Project Layout

```
baton/
├── baton.go              # Main orchestrator and CLI entry point
├── worker.go             # Base worker implementation
├── count_worker.go       # Worker for fixed request counts
├── timed_worker.go       # Worker for time-based testing
├── configuration.go      # Configuration validation
├── csv_parsing.go        # CSV request file parsing
├── result.go             # Result aggregation and output
├── http_result.go        # HTTP response counters
├── log_writer.go         # Custom logging with suppress capability
├── baton_test.go         # Integration tests
├── Dockerfile            # Multi-stage Docker build
├── Gopkg.toml            # Dependency management (Go Dep)
└── wiki/                 # This documentation
```

## Key Concepts

- **Concurrency** — Number of parallel workers (`-c` flag)
- **Request Count Mode** — Send fixed number of requests (`-r` flag), collect per-request statistics
- **Timed Mode** — Send requests for a duration (`-t` flag), no per-request statistics
- **CSV Requests** — Load complex requests from file (`-z` flag)
- **Statistics** — Response times, status codes, throughput, and percentile distribution

## Documentation

- **[Coding Standards](Coding-Standards)** — Development conventions, formatting, naming, testing
- **[Architecture Decisions](Architecture-Decisions)** — System design, components, data flow, trade-offs
- **[Known Issues](Known-Issues)** — Limitations, caveats, test coverage gaps, future improvements

## Contributing

Before contributing:

1. Read [CONTRIBUTING.md](../CONTRIBUTING.md) for commit formatting
2. Review [Coding Standards](Coding-Standards) for conventions
3. Check [Known Issues](Known-Issues) to avoid duplicating known problems
4. Run `gofmt -w` on modified files
5. Ensure tests pass: `go test -v`

## License

Licensed under [Apache License 2.0](../LICENSE.md). See [CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md) for community guidelines.
