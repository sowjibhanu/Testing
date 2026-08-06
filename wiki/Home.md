# Baton Wiki

Welcome to the Baton project! This is a high-performance load testing tool written in Go.

## What is Baton?

Baton is a command-line load testing tool that allows you to send HTTP requests (GET, POST, PUT, DELETE) to a target server and measure performance metrics. It supports concurrent requests, custom request bodies, headers, and can load requests from CSV files.

## Quick Links

- **[Coding Standards](Coding-Standards)** — Development conventions, formatting rules, and testing practices
- **[Architecture Decisions](Architecture-Decisions)** — System design, main components, and key trade-offs
- **[Known Issues](Known-Issues)** — Limitations, caveats, and areas for improvement

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
├── baton_test.go         # Integration tests
├── Dockerfile            # Multi-stage Docker build
├── Gopkg.toml            # Dependency management (Go Dep)
└── wiki/                 # This documentation
```

## Getting Started

1. **Install**: `go get -u github.com/americanexpress/baton`
2. **Run**: `baton -u http://localhost:8080/test -c 10 -r 200000`
3. **Read**: See [README.md](../README.md) for full usage options

## Key Concepts

- **Concurrency**: Multiple workers send requests in parallel (`-c` flag)
- **Two Execution Modes**: Fixed request count (`-r`) or time-based (`-t`)
- **CSV Requests**: Load complex requests from a file (`-z` flag)
- **Statistics**: Response times, status codes, and throughput metrics

## Contributing

Before contributing, please:
1. Read [CONTRIBUTING.md](../CONTRIBUTING.md)
2. Review [Coding Standards](Coding-Standards)
3. Check [Known Issues](Known-Issues) to avoid duplicating known problems
4. Run `gofmt -w` on modified files
5. Ensure tests pass: `go test -v`
