# Architecture Decisions

This page explains how Baton is structured and the key design decisions behind it.

## System Overview

Baton is a load testing orchestrator that coordinates multiple concurrent workers to send HTTP requests and aggregate results.

```
┌─────────────────────────────────────────────────────────────┐
│ main() - CLI Entry Point (baton.go:68)                      │
│ ├─ Parse flags                                              │
│ ├─ Create Configuration                                     │
│ └─ Create Baton instance and call run()                     │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Baton.run() (baton.go:82)                                   │
│ ├─ Validate configuration                                   │
│ ├─ Prepare run (parse CSV, setup channels)                  │
│ ├─ Spawn N workers as goroutines                            │
│ ├─ Wait for all workers to finish                           │
│ └─ Process and print results                                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Worker Goroutines (worker.go, count_worker.go, etc.)        │
│ ├─ Receive requests from channel                            │
│ ├─ Send HTTP requests                                       │
│ ├─ Record response times and status codes                   │
│ └─ Send results back via channel                            │
└─────────────────────────────────────────────────────────────┘
```

## Main Components

### 1. Baton Orchestrator (`baton.go`)

**Responsibility**: Coordinate the entire load test execution.

**Key methods**:
- `main()` (line 68) — Parse CLI flags and start execution
- `run()` (line 82) — Main orchestration loop
- `processResults()` (line 145) — Aggregate worker results

**Data structures**:
- `Baton` (line 47) — Holds configuration and result
- `runConfiguration` (line 57) — Runtime state (channels, client, requests)

### 2. Worker System (`worker.go`, `count_worker.go`, `timed_worker.go`)

**Responsibility**: Execute HTTP requests and collect metrics.

**Base worker** (`worker.go:13`):
```go
type worker struct {
    httpResult  HTTPResult
    client      *fasthttp.Client
    requests    <-chan bool
    httpResults chan<- HTTPResult
    done        chan<- bool
}
```

**Polymorphism via interface** (`worker.go:18`):
```go
type workable interface {
    sendRequests(requests []preLoadedRequest)
    sendRequest(request preLoadedRequest)
    setCustomClient(client *fasthttp.Client)
}
```

**Two implementations**:

1. **CountWorker** (`count_worker.go:22`) — Sends fixed number of requests
   - Collects timing statistics for each request
   - Used when `-r` flag specifies request count
   - Calls `collectStatistics()` to compute min/max/avg

2. **TimedWorker** (`timed_worker.go:22`) — Sends requests for a duration
   - Runs until time expires
   - Used when `-t` flag specifies duration
   - Does NOT collect per-request statistics (see Known Issues)

### 3. Configuration (`configuration.go`)

**Responsibility**: Hold and validate CLI parameters.

**Fields** (configuration.go:7):
```go
type Configuration struct {
    body             string
    concurrency      int
    dataFilePath     string
    duration         int
    ignoreTLS        bool
    method           string
    numberOfRequests int
    requestsFromFile string
    suppressOutput   bool
    url              string
    wait             int
}
```

**Validation** (configuration.go:33):
- Concurrency must be ≥ 1
- Number of requests must be > 0

### 4. CSV Parsing (`csv_parsing.go`)

**Responsibility**: Load requests from CSV file.

**Format** (RFC-4180):
```
<method>,<url>,[<body>],[<header-key>:<header-value>, ...]
```

**Function** (csv_parsing.go:32):
```go
func preLoadRequestsFromFile(filename string) ([]preLoadedRequest, error)
```

**Header parsing** (csv_parsing.go:10):
- Splits on `:` to extract key-value pairs
- Handles multiple headers per request

### 5. Result Aggregation (`result.go`, `http_result.go`)

**HTTPResult** (http_result.go:7) — Counters for a single worker:
```go
type HTTPResult struct {
    connectionErrorCount int
    status1xxCount       int
    status2xxCount       int
    status3xxCount       int
    status4xxCount       int
    status5xxCount       int
    maxTime              int
    minTime              int
    timeSum              int64
    totalSuccess         int
    responseTimes        []int
    responseTimesPercent [][3]int
}
```

**Result** (result.go:7) — Aggregated results:
```go
type Result struct {
    httpResult        HTTPResult
    totalRequests     int
    timeTaken         time.Duration
    requestsPerSecond int
    hasStats          bool
    averageTime       float32
    minTime           int
    maxTime           int
}
```

**Aggregation** (baton.go:145):
- Collects results from all workers
- Sums counters
- Computes min/max/avg response times
- Calculates percentile buckets (10 brackets)

### 6. Logging (`log_writer.go`)

**Responsibility**: Suppress or enable output.

**Custom writer** (log_writer.go:7):
```go
type logWriter struct {
    enabled bool
}
```

**Usage** (baton.go:161):
- Configured via `-o` flag
- Allows silent operation for scripting

## Data Flow

### Request Count Mode (`-r 1000`)

```
1. Parse flags → Configuration
2. prepareRun() creates:
   - requests channel (buffered with 1000 items)
   - results channel (buffered with concurrency size)
   - done channel (buffered with concurrency size)
3. Spawn N workers (N = concurrency)
4. Each worker:
   - Reads from requests channel
   - Sends HTTP request
   - Records timing in timings channel
   - Repeats until requests channel closes
5. Worker calls collectStatistics():
   - Reads all timings
   - Computes min/max/avg
   - Skips first request (overhead)
6. Worker sends HTTPResult to results channel
7. Main thread collects all results
8. Aggregates and prints
```

### Timed Mode (`-t 10`)

```
1. Parse flags → Configuration
2. prepareRun() creates channels (same as above)
3. Spawn N workers
4. Each worker:
   - Records start time
   - Sends HTTP requests in loop
   - Checks elapsed time
   - Stops when duration expires
   - Does NOT collect per-request statistics
5. Worker sends HTTPResult to results channel
6. Main thread collects results
7. Aggregates and prints (no per-request stats)
```

## Key Design Decisions

### 1. Two Execution Modes (Count vs. Timed)

**Decision**: Support both `-r` (fixed count) and `-t` (duration) modes.

**Rationale** (baton.go:103):
- Count mode: Useful for benchmarking (consistent load)
- Timed mode: Useful for stress testing (sustained load)

**Trade-off**: Timed mode doesn't collect per-request statistics (see Known Issues).

### 2. Worker Polymorphism via Interface

**Decision**: Use `workable` interface for different worker types.

**Rationale** (baton.go:115):
```go
var worker workable
if preparedRunConfiguration.timedMode {
    worker = newTimedWorker(...)
} else {
    worker = newCountWorker(...)
}
```

**Benefit**: Easy to add new worker types without changing orchestrator.

### 3. Channel-Based Coordination

**Decision**: Use Go channels for worker coordination.

**Rationale** (baton.go:130):
- `requests` channel: Distributes work to workers
- `results` channel: Collects results from workers
- `done` channel: Signals completion

**Benefit**: Idiomatic Go, safe concurrent access, no locks needed.

### 4. FastHTTP Choice

**Decision**: Use `github.com/valyala/fasthttp` instead of standard `net/http`.

**Rationale** (Gopkg.toml:27):
- Higher throughput for load testing
- Lower memory allocation
- Better for high-concurrency scenarios

**Trade-off**: Less familiar API, fewer features than standard library.

### 5. Response Time Bucketing

**Decision**: Divide response times into 10 fixed brackets for percentile reporting.

**Implementation** (baton.go:175):
```go
var numOfBrackets = 10
rtCounts := make([][3]int, numOfBrackets)
bs := (max - min) / numOfBrackets
```

**Benefit**: Simple, fast percentile calculation.

**Trade-off**: Loses precision; cannot compute exact p50/p95/p99 (see Known Issues).

### 6. First Request Timing Exclusion

**Decision**: Skip first request's timing statistics to exclude client setup overhead.

**Implementation** (worker.go:73):
```go
if first {
    first = false
    continue
}
```

**Rationale**: First request includes TLS handshake, connection setup, etc.

**Trade-off**: Slightly fewer data points (N-1 instead of N per worker).

### 7. Atomic Operations for Thread Safety

**Decision**: Use `sync/atomic` for counter updates in tests.

**Rationale** (baton_test.go:42):
```go
atomic.AddUint32(&h.noRequestsReceived, 1)
```

**Benefit**: Lock-free, high-performance counter updates.

## Concurrency Model

**Goroutines per worker**: Each worker runs in its own goroutine.

**Channel communication**:
- Main thread sends work via `requests` channel
- Workers send results via `results` channel
- Workers signal completion via `done` channel

**Synchronization**:
- Main thread waits for all workers to finish before aggregating results
- No shared mutable state between workers (each has its own HTTPResult)

## Request Execution Flow

1. **CLI Parsing** (baton.go:68-75): Flags converted to Configuration struct
2. **Validation** (configuration.go:33): Concurrency and request count checked
3. **Preparation** (baton.go:103): Channels created, CSV loaded if needed
4. **Worker Spawn** (baton.go:115): N goroutines created, each running worker.sendRequest(s)
5. **Request Distribution** (baton.go:130): Requests channel filled with N items
6. **Execution** (worker.go:60): Each worker reads from channel, sends HTTP request
7. **Timing** (worker.go:48): Response time recorded (count mode only)
8. **Status Recording** (worker.go:54): HTTP status code categorized
9. **Aggregation** (baton.go:145): Results collected from all workers
10. **Output** (result.go:20): Formatted results printed to stdout

## Error Handling Strategy

**Configuration errors**: Caught early, fail fast with `log.Fatalf()`

**File I/O errors**: Returned from `prepareRun()`, propagated to main

**HTTP errors**: Counted as connection errors, not fatal

**Channel operations**: Rely on Go's panic for programming errors (e.g., send on closed channel)
