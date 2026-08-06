# Architecture Decisions

This page explains how Baton is structured and the key design decisions behind it.

## System Overview

Baton is a load testing orchestrator that coordinates multiple concurrent workers to send HTTP requests and collect statistics. The system has two execution modes: **count-based** (fixed number of requests) and **timed** (requests for a duration).

```
┌─────────────────────────────────────────────────────────────┐
│                    Baton (Orchestrator)                     │
│                      baton.go:line 1                        │
└─────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                │             │             │
         ┌──────▼──────┐ ┌───▼────────┐ ┌─▼──────────────┐
         │ Configuration│ │ CSV Parser │ │ Result Aggreg. │
         │ config.go:1  │ │ csv_pa.go:1│ │ result.go:1    │
         └──────────────┘ └────────────┘ └────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │                           │
         ┌──────▼──────────┐        ┌──────▼──────────┐
         │  CountWorker    │        │  TimedWorker    │
         │ count_wor.go:1  │        │ timed_wor.go:1  │
         └─────────────────┘        └─────────────────┘
                │                           │
                └─────────────┬─────────────┘
                              │
                        ┌─────▼──────┐
                        │ Base Worker │
                        │ worker.go:1 │
                        └─────┬──────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
            ┌───────▼────────┐  ┌──────▼──────┐
            │  FastHTTP      │  │ HTTP Result │
            │  Client        │  │ http_res.go │
            └────────────────┘  └─────────────┘
```

## Main Components

### 1. Baton Orchestrator (`baton.go:line 1`)

**Responsibility**: Parse CLI flags, coordinate workers, aggregate results, and output statistics.

**Key Methods**:
- `main()` — Entry point, parses flags and creates Baton instance
- `(baton *Baton) run()` — Orchestrates the entire test execution
- `processResults()` — Aggregates worker results and computes statistics

**Data Flow**:
1. Parse CLI flags into `Configuration` struct (see `baton.go:line 23`)
2. Validate configuration (see `baton.go:line 105`)
3. Prepare run (create channels, load CSV if needed) (see `baton.go:line 109`)
4. Spawn workers as goroutines (see `baton.go:line 118`)
5. Wait for all workers to complete (see `baton.go:line 135`)
6. Aggregate results from worker channel (see `baton.go:line 145`)
7. Print formatted results (see `result.go:line 48`)

### 2. Configuration (`configuration.go:line 1`)

**Responsibility**: Hold and validate test parameters.

**Fields** (see `configuration.go:line 10`):
- `body` — Request body string
- `concurrency` — Number of concurrent workers
- `dataFilePath` — Path to file containing request body
- `duration` — Time-based test duration in seconds
- `ignoreTLS` — Skip TLS certificate validation
- `method` — HTTP method (GET, POST, PUT, DELETE)
- `numberOfRequests` — Fixed number of requests to send
- `requestsFromFile` — Path to CSV file with requests
- `suppressOutput` — Disable logging
- `url` — Target URL
- `wait` — Seconds to wait before starting test

**Validation** (see `configuration.go:line 32`):
- Concurrency must be >= 1
- Number of requests must be > 0

### 3. Worker System

**Base Worker** (`worker.go:line 1`):
- Holds HTTP client, channels, and result statistics
- Implements `performRequest()` and `performRequestWithStats()`
- Implements `recordCount()` to categorize responses by status code
- Implements `collectStatistics()` to compute min/max/avg response times

**Worker Interface** (`worker.go:line 30`):
```go
type workable interface {
    sendRequests(requests []preLoadedRequest)
    sendRequest(request preLoadedRequest)
    setCustomClient(client *fasthttp.Client)
}
```

**CountWorker** (`count_worker.go:line 1`):
- Sends a fixed number of requests
- Collects response time statistics
- Used when `-r` flag is specified

**TimedWorker** (`timed_worker.go:line 1`):
- Sends requests for a specified duration
- Does NOT collect response time statistics (see Known Issues)
- Used when `-t` flag is specified

### 4. Result Aggregation (`result.go:line 1`)

**Responsibility**: Collect and format test results.

**Result Struct** (see `result.go:line 10`):
- `httpResult` — HTTP response statistics
- `totalRequests` — Total requests sent
- `timeTaken` — Total test duration
- `requestsPerSecond` — Throughput metric
- `hasStats` — Whether response time stats are available
- `averageTime` — Average response time in ms
- `minTime` / `maxTime` — Response time bounds

**Output Format** (see `result.go:line 48`):
- Formatted table with aligned columns
- Response time percentile buckets (10 brackets)
- Status code breakdowns (1xx, 2xx, 3xx, 4xx, 5xx, errors)

### 5. HTTP Result Statistics (`http_result.go:line 1`)

**Responsibility**: Track HTTP response statistics.

**Fields** (see `http_result.go:line 10`):
- Status code counters: `status1xxCount`, `status2xxCount`, etc.
- Response time tracking: `responseTimes` (array), `responseTimesPercent` (brackets)
- Error tracking: `connectionErrorCount`
- Aggregation: `timeSum`, `totalSuccess`

### 6. CSV Parsing (`csv_parsing.go:line 1`)

**Responsibility**: Parse CSV request files.

**Format** (see `csv_parsing.go:line 32`):
```
<method>,<url>,[<body>],[<header-key>:<header-value>, ...]
```

**Example**:
```
POST,http://localhost:8888,body,Accept: application/xml,Content-type: Secret
GET,http://localhost:8888,,,
```

**Parsing** (see `csv_parsing.go:line 32`):
- Reads CSV with standard library `encoding/csv`
- Extracts method, URL, body, and headers
- Headers are split on `:` character
- Returns slice of `preLoadedRequest` structs

## Data Flow

### Count-Based Test Flow

```
1. CLI flags → Configuration
2. Configuration.validate()
3. prepareRun() → creates channels, loads CSV if needed
4. Spawn N workers (goroutines)
5. Each worker:
   a. Reads from requests channel
   b. Executes HTTP request
   c. Records response time and status
   d. Repeats until requests channel closes
   e. Sends HTTPResult to results channel
6. Main goroutine waits for all workers on done channel
7. Aggregates results from results channel
8. Computes statistics (min, max, avg, percentiles)
9. Prints formatted output
```

### Timed Test Flow

```
1. CLI flags → Configuration
2. Configuration.validate()
3. prepareRun() → creates channels, sets timedMode=true
4. Spawn N workers (goroutines)
5. Each worker:
   a. Records start time
   b. Executes HTTP requests in loop
   c. Checks elapsed time, breaks when duration exceeded
   d. Sends HTTPResult to results channel (no timing stats)
6. Main goroutine waits for all workers on done channel
7. Aggregates results from results channel
8. Prints output (no response time statistics)
```

## Key Design Decisions

### 1. Two Execution Modes (Count vs. Timed)

**Decision**: Support both fixed request counts and time-based testing.

**Rationale**: Different testing scenarios require different approaches:
- Count-based: Measure throughput and response times for a known workload
- Timed: Measure sustained throughput over a time period

**Implementation** (see `baton.go:line 113`):
```go
if preparedRunConfiguration.timedMode {
    worker = newTimedWorker(...)
} else {
    worker = newCountWorker(...)
}
```

**Trade-off**: Timed mode sacrifices response time statistics for simplicity (see Known Issues).

### 2. Worker Polymorphism via Interface

**Decision**: Use `workable` interface for different worker types.

**Rationale**: Allows clean separation between count and timed workers while sharing base functionality.

**Implementation** (see `worker.go:line 30`):
```go
type workable interface {
    sendRequests(requests []preLoadedRequest)
    sendRequest(request preLoadedRequest)
    setCustomClient(client *fasthttp.Client)
}
```

**Trade-off**: Slight overhead of interface dispatch, but cleaner code organization.

### 3. Channel-Based Coordination

**Decision**: Use Go channels for worker coordination instead of locks/mutexes.

**Rationale**: Channels are idiomatic Go and prevent race conditions naturally.

**Implementation** (see `baton.go:line 127`):
- `requests` channel: Distributes work
- `results` channel: Collects statistics
- `done` channel: Signals completion

**Trade-off**: Channels have overhead, but correctness and readability are prioritized.

### 4. FastHTTP for HTTP Client

**Decision**: Use `github.com/valyala/fasthttp` instead of standard library `net/http`.

**Rationale**: FastHTTP is optimized for high-throughput scenarios and reduces allocations.

**Implementation** (see `worker.go:line 48`):
```go
if err := worker.client.Do(req, resp); err != nil {
    worker.httpResult.connectionErrorCount++
}
```

**Trade-off**: FastHTTP has a different API than `net/http`, but performance is critical for load testing.

### 5. Response Time Bucketing

**Decision**: Divide response times into 10 brackets for percentile reporting.

**Rationale**: Provides distribution insight without storing all individual response times.

**Implementation** (see `baton.go:line 165`):
```go
var numOfBrackets = 10
rtCounts := make([][3]int, numOfBrackets)
bs := (max - min) / numOfBrackets
```

**Trade-off**: Loses precision compared to full percentile calculation, but reduces memory usage.

### 6. First Request Timing Exclusion

**Decision**: Exclude the first request's timing from statistics.

**Rationale**: First request includes client initialization overhead.

**Implementation** (see `worker.go:line 130`):
```go
if first {
    first = false
    continue
}
```

**Trade-off**: Slightly inaccurate for very small request counts, but more representative for typical loads.

### 7. Atomic Operations for Test Counters

**Decision**: Use `sync/atomic` for thread-safe counter updates in tests.

**Rationale**: Avoids mutex overhead for simple counter increments.

**Implementation** (see `baton_test.go:line 35`):
```go
atomic.AddUint32(&h.noRequestsReceived, 1)
```

**Trade-off**: Atomic operations are slightly slower than regular assignments, but guarantee correctness.

## Concurrency Model

**Goroutine per worker**: Each worker runs in its own goroutine (see `baton.go:line 118`):
```go
go worker.sendRequest(request)
```

**Channel coordination**:
- Main goroutine spawns N workers
- Main goroutine waits on `done` channel N times (see `baton.go:line 135`)
- Workers send results to `results` channel (see `worker.go:line 145`)
- Main goroutine collects N results (see `baton.go:line 145`)

**No shared mutable state**: Each worker has its own `HTTPResult` struct, eliminating race conditions.

## Error Handling Strategy

**Validation-first**: Configuration is validated before any work begins (see `baton.go:line 105`).

**Graceful degradation**: Connection errors are counted but don't stop the test (see `worker.go:line 48`).

**Fatal errors**: Unrecoverable errors (invalid config, file I/O) call `log.Fatalf()` (see `baton.go:line 106`).

## Performance Considerations

1. **Request pooling**: FastHTTP reuses request/response objects (see `worker.go:line 48`)
2. **Channel buffering**: Channels are buffered to avoid blocking (see `baton.go:line 193`)
3. **Atomic counters**: Used instead of mutexes for test verification
4. **Static binary**: Docker build produces minimal image with no runtime dependencies
