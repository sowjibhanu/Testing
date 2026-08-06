# Copyright 2026 World wide technology

# Project Wiki Documentation

This project maintains comprehensive wiki documentation to help developers understand the codebase, conventions, and architecture.

## Wiki Pages

### [Coding Standards](https://github.com/sowjibhanu/Testing/wiki/Coding-Standards)
Covers the project's development conventions and best practices:
- **Language & Build**: Go with Dep dependency management
- **Code Formatting**: Mandatory `gofmt -w` on all modified files
- **Naming Conventions**: PascalCase for types, camelCase for variables
- **File Organization**: One concept per file (baton.go, worker.go, configuration.go, etc.)
- **Error Handling**: Early validation, error returns, fatal error handling
- **Testing**: Integration tests with local HTTP server
- **Concurrency**: Goroutines with channel-based coordination
- **Docker & Deployment**: Multi-stage build with static binary

### [Architecture Decisions](https://github.com/sowjibhanu/Testing/wiki/Architecture-Decisions)
Explains the system design and key architectural components:
- **Main Components**: Baton orchestrator, Worker system, Configuration, Result aggregation, CSV parsing, Logging
- **Data Flow**: Complete flow from CLI flags through request execution to result output
- **Design Decisions**: 7 key decisions including two execution modes, worker polymorphism, channel coordination, FastHTTP choice, response time bucketing, first-request timing exclusion, and atomic operations
- **Trade-offs**: Analysis of benefits and costs for each design decision
- **Concurrency Model**: Explanation of goroutine coordination and channel usage

### [Known Issues](https://github.com/sowjibhanu/Testing/wiki/Known-Issues)
Documents limitations, caveats, and areas for improvement:
- **10 Specific Limitations**: Each with code location, impact, and workaround
  - Statistics unavailable in timed mode
  - First request timing excluded
  - No dynamic request generation
  - Fragile CSV header parsing
  - No request validation
  - Response time bucketing loses precision
  - No request timeout configuration
  - Hardcoded test server port
  - No graceful shutdown
  - Memory overhead for large request counts
- **Test Coverage Gaps**: Areas not covered by tests
- **Documentation Gaps**: Missing documentation sections
- **Future Improvements**: Acknowledged features not yet implemented

## How to Use

1. **For New Contributors**: Start with [Coding Standards](https://github.com/sowjibhanu/Testing/wiki/Coding-Standards) to understand project conventions
2. **For Understanding Design**: Read [Architecture Decisions](https://github.com/sowjibhanu/Testing/wiki/Architecture-Decisions) to learn why components are structured as they are
3. **Before Contributing**: Check [Known Issues](https://github.com/sowjibhanu/Testing/wiki/Known-Issues) to understand limitations and avoid duplicating known problems

## Accessing the Wiki

The wiki is accessible at: https://github.com/sowjibhanu/Testing/wiki

All pages are maintained in the project's GitHub wiki system and are automatically indexed and searchable.
