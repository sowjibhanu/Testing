# Known Issues

This page documents limitations, rough edges, and areas for improvement in Baton.

## Limitations

### 1. Statistics Unavailable in Timed Mode

**Location**: `timed_worker.go:line 1`, `baton.go:line 145`

**Impact**: When using `-t` (time-based testing), response time statistics (min, max, avg, percentiles) are not collected.

**Why**: Timed workers don't call `performRequestWithStats()`, only `performRequest()` (see `timed_worker.go:line 35`).

**Workaround**: Use `-r` (request count) instead of `-t` if you need response time statistics. For time-based testing, use a large request count with a timeout wrapper.

**Example**:
```bash
# This won't show response time stats
baton -u http://localhost:8080 -t 10

# This will show response time stats
baton -u http://localhost:8080 -r 100000 -c 10
```

### 2. First Request Timing Excluded from Statistics

**Location**: `worker.go:line 130`

**Impact**: The first request's response time is not included in min/max/avg calculations.

**Why**: First request includes client initialization overhead (see `worker.go:line 130` comment: "The first request is associated with overhead in setting up the client").

**Workaround**: For very small request counts (< 10), results may be skewed. Use larger request counts for accurate statistics.

**Example**:
```bash
# First request timing excluded, may skew results
baton -u http://localhost:8080 -r 5

# Better: larger request count
baton -u http://localhost:8080 -r 10000
```

### 3. No Dynamic Request Generation

**Location**: `README.md` (listed as "Features which are on the horizon")

**Impact**: Cannot generate requests with dynamic data (e.g., incrementing IDs, random values).

**Why**: CSV parsing is static; no template engine is implemented.

**Workaround**: Pre-generate CSV file with all variations, or use external tools to generate requests.

### 4. Fragile CSV Header Parsing

**Location**: `csv_parsing.go:line 10`

**Impact**: Headers with colons in values will be incorrectly parsed.

**Why**: Header parsing uses simple string split on `:` character (see `csv_parsing.go:line 10`):
```go
func extractHeaders(rawHeaders string) []string {
    headerParts := strings.Split(rawHeaders, ":")
    if len(headerParts) == 2 {
        return []string{headerParts[0], headerParts[1]}
    }
    return nil
}
```

**Workaround**: Avoid colons in header values. If needed, use URL encoding or other escaping.

**Example**:
```bash
# This will fail (colon in value)
GET,http://localhost:8080,,,Authorization: Bearer token:with:colons

# This works
GET,http://localhost:8080,,,Authorization: Bearer-token-with-dashes
```

### 5. No Request Validation

**Location**: `baton.go:line 109`

**Impact**: Invalid URLs or malformed requests are only caught at execution time.

**Why**: No pre-flight validation of requests before workers start.

**Workaround**: Test your configuration with a small request count first (`-r 1`).

### 6. Response Time Bucketing Loses Precision

**Location**: `baton.go:line 165`

**Impact**: Percentile reporting shows only 10 brackets, not true percentiles.

**Why**: Response times are divided into 10 equal-width brackets (see `baton.go:line 165`):
```go
var numOfBrackets = 10
bs := (max - min) / numOfBrackets
```

**Workaround**: For precise percentile analysis, collect raw response times externally or modify the code.

**Example Output**:
```
========= Percentage of responses received within a certain time (ms)======

       100% : 440 ms
```

This shows "100% of responses within 440ms", not true percentiles like "p50", "p95", "p99".

### 7. No Request Timeout Configuration

**Location**: `worker.go:line 48`, `baton.go:line 183`

**Impact**: Requests can hang indefinitely if server doesn't respond.

**Why**: FastHTTP client is created without timeout settings (see `baton.go:line 183`):
```go
client := &fasthttp.Client{}
```

**Workaround**: Use OS-level timeouts or network timeouts at the infrastructure level.

### 8. Hardcoded Test Server Port

**Location**: `baton_test.go:line 50`

**Impact**: Tests will fail if port 8888 is already in use.

**Why**: Test server port is hardcoded as `"8888"` (see `baton_test.go:line 50`):
```go
var port = "8888"
```

**Workaround**: Ensure port 8888 is available before running tests. Use `lsof -i :8888` to check.

### 9. No Graceful Shutdown

**Location**: `baton.go:line 135`

**Impact**: If a worker hangs, the entire test hangs indefinitely.

**Why**: Main goroutine waits indefinitely on `done` channel with no timeout (see `baton.go:line 135`):
```go
for a := 1; a <= baton.configuration.concurrency; a++ {
    <-preparedRunConfiguration.done
}
```

**Workaround**: Use OS-level timeouts (`timeout` command on Unix, `timeout` on Windows).

**Example**:
```bash
# Unix: timeout after 60 seconds
timeout 60 baton -u http://localhost:8080 -r 1000000

# Windows: timeout after 60 seconds
timeout /t 60 baton -u http://localhost:8080 -r 1000000
```

### 10. Memory Overhead for Large Request Counts

**Location**: `worker.go:line 125`, `baton.go:line 145`

**Impact**: Very large request counts (millions) consume significant memory for response time arrays.

**Why**: All response times are stored in memory (see `worker.go:line 125`):
```go
worker.httpResult.responseTimes = append(worker.httpResult.responseTimes, timing)
```

**Workaround**: Use timed mode (`-t`) instead of count mode for very large workloads, or split into multiple runs.

**Example**:
```bash
# Memory-intensive: stores 10M response times
baton -u http://localhost:8080 -r 10000000

# Better: use timed mode
baton -u http://localhost:8080 -t 60 -c 100
```

## Test Coverage Gaps

- **No tests for CSV parsing with edge cases** (empty fields, special characters, very long lines)
- **No tests for TLS/SSL certificate validation** (`-i` flag)
- **No tests for file I/O errors** (missing body file, unreadable CSV)
- **No tests for very large request counts** (memory/performance)
- **No tests for concurrent worker race conditions** (though design should prevent them)
- **No tests for malformed HTTP responses**

## Documentation Gaps

- **No API documentation** for the `workable` interface
- **No performance tuning guide** (concurrency levels, request sizes)
- **No troubleshooting guide** for common errors
- **No examples** for advanced CSV request files
- **No Docker usage examples** beyond basic build

## Future Improvements

- [ ] Add request timeout configuration (`-timeout` flag)
- [ ] Implement true percentile reporting (p50, p95, p99)
- [ ] Add dynamic request generation with templates
- [ ] Improve CSV header parsing (handle colons in values)
- [ ] Add graceful shutdown with timeout
- [ ] Implement streaming response time output (for large request counts)
- [ ] Add support for HTTP/2
- [ ] Add support for WebSocket load testing
- [ ] Add metrics export (Prometheus, JSON)
- [ ] Add request rate limiting (requests per second)
- [ ] Add support for custom authentication schemes
- [ ] Add support for request/response validation
- [ ] Implement connection pooling configuration
- [ ] Add support for distributed load testing (multiple instances)
