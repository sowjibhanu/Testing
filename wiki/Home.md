# Baton Wiki

Welcome to the Baton project wiki. Baton is a high-performance load testing tool written in Go that supports GET, POST, PUT, and DELETE HTTP requests.

## Quick Links

- **[Coding Standards](Coding-Standards)** — Development conventions, formatting rules, and best practices
- **[Architecture Decisions](Architecture-Decisions)** — System design, components, and key trade-offs
- **[Known Issues](Known-Issues)** — Limitations, rough edges, and areas for improvement

## Project Overview

Baton is designed to efficiently load test HTTP endpoints with support for:
- Multiple concurrent workers
- Fixed request counts or time-based testing
- CSV-based request files with custom headers
- Detailed response time statistics and breakdowns
- TLS/SSL certificate validation control

## Getting Started

For new contributors, start with [Coding Standards](Coding-Standards) to understand the project's conventions, then read [Architecture Decisions](Architecture-Decisions) to learn how the system is structured.

Before making changes, check [Known Issues](Known-Issues) to understand existing limitations and avoid duplicating known problems.

## Repository Structure

```
.
├── baton.go              # Main orchestrator and CLI entry point
├── worker.go             # Base worker implementation
├── count_worker.go       # Worker for fixed request counts
├── timed_worker.go       # Worker for time-based testing
├── configuration.go      # Configuration validation
├── result.go             # Result aggregation and output
├── http_result.go        # HTTP response statistics
├── csv_parsing.go        # CSV request file parsing
├── log_writer.go         # Custom logging control
├── baton_test.go         # Integration tests
├── Dockerfile            # Multi-stage Docker build
├── Gopkg.toml            # Dependency management
└── test-resources/       # Test fixtures
```

## Key Files

- **baton.go:line 1** — Main entry point and orchestration logic
- **worker.go:line 1** — Base worker type and HTTP request execution
- **baton_test.go:line 1** — Integration tests with local HTTP server
