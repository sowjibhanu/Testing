# Known Issues

This page documents limitations, caveats, and areas for improvement in Baton. Check here before reporting bugs or contributing.

## Limitations

### 1. Statistics Unavailable in Timed Mode

**Issue**: When using `-t` (duration mode), per-request statistics (min/max/avg response time) are not collected.

**Location**: `timed_worker.go:30` — `sendRequest()` calls `performRequest()` instead of `performRequestWithStats()`

**Impact**: Users cannot see response time distribution in timed mode, only total throughput.

**Workaround**: Use `-r` (request count) mode if you need statistics. Run multiple tests with different concurrency levels to estimate performance.

**Why**: Timed mode doesn't know request count in advance, so can't pre-allocate timing buffers efficiently.

### 2. First Request Timing Excluded

**Issue**: The first request from each worker is excluded from timing statistics.

**Location**: `worker.go:73` — `collectStatistics()` skips first timing

**Impact**: Slightly fewer data points (N-1 instead of N per worker).

**Rationale**: First request includes client setup overhead, skewing averages.

**Workaround**: None needed; this is intentional. Use large request counts for more accurate statistics.

### 3. No Dynamic Request Generation

**Issue**: Cannot generate request bodies or URLs dynamically (e.g., incrementing IDs).

**Location**: `baton.go:125` — Single request template used for all requests

**Impact**: All requests are identical (except when using CSV file with `-z`).

**Workaround**: Pre-generate CSV file with all variations and use `-z` flag.

**Future**: Planned feature (see README.md "Features which are on the horizon").

### 4. Fragile CSV Header Parsing

**Issue**: Header parsing splits on `:` only, doesn't handle edge cases.

**Location**: `csv_parsing.go:10` — `extractHeaders()` uses simple string split

**Impact**: Headers with `:` in the value will be parsed incorrectly.

**Example**: `Authorization: Bearer: token` would split incorrectly.

**Workaround**: Avoid `:` in header values, or use URL encoding.

**Better approach**: Use proper CSV parsing with quoted fields.

### 5. No Request Validation

**Issue**: Invalid URLs or methods are not validated before sending.

**Location**: `baton.go:125` — No validation of URL format or HTTP method

**Impact**: Invalid requests fail silently, counted as connection errors.

**Workaround**: Test your URL and method manually before running load test.

### 6. Response Time Bucketing Loses Precision

**Issue**: Response times are divided into 10 fixed brackets, losing granularity.

**Location**: `baton.go:175` — `rtCounts` array with 10 brackets

**Impact**: Cannot see exact percentiles (e.g., p95, p99).

**Example**: If min=10ms and max=1000ms, brackets are 99ms wide.

**Workaround**: Use external tools to post-process raw response times if needed.

**Better approach**: Implement proper percentile calculation (p50, p95, p99).

### 7. No Request Timeout Configuration

**Issue**: Cannot set timeout for individual requests.

**Location**: `worker.go:48` — `fasthttp.Client` created with default timeout

**Impact**: Slow servers can hang workers indefinitely.

**Workaround**: Set OS-level timeout or use `timeout` command wrapper.

**Better approach**: Add `-timeout` flag to configure per-request timeout.

### 8. Hardcoded Test Server Port

**Issue**: Integration tests hardcode port 8888.

**Location**: `baton_test.go:54` — `var port = "8888"`

**Impact**: Tests fail if port 8888 is already in use.

**Workaround**: Kill process using port 8888 before running tests.

**Better approach**: Use OS-assigned port (port 0) and pass to tests.

### 9. No Graceful Shutdown

**Issue**: No way to stop a running load test cleanly (except Ctrl+C).

**Location**: `baton.go:115` — Workers run until requests channel closes

**Impact**: Cannot pause or resume a test.

**Workaround**: Use Ctrl+C to stop (may lose final results).

**Better approach**: Add signal handling for SIGINT/SIGTERM.

### 10. Memory Overhead for Large Request Counts

**Issue**: All response times stored in memory for statistics.

**Location**: `worker.go:82` — `responseTimes` slice grows unbounded

**Impact**: High memory usage for very large request counts (e.g., 10M requests).

**Example**: 1M requests × 4 bytes per timing = 4MB per worker.

**Workaround**: Use timed mode (`-t`) instead of count mode (`-r`).

**Better approach**: Stream statistics or use fixed-size ring buffer.

## Test Coverage Gaps

### Missing Tests

- **TLS/SSL verification**: `-i` flag not tested
- **Large request bodies**: No test for multi-MB bodies
- **Concurrent CSV loading**: Multiple workers with `-z` flag
- **Error recovery**: Behavior when server returns 5xx errors
- **Timeout scenarios**: Slow/unresponsive servers
- **Edge cases**: Empty body, missing URL, invalid method

### How to Add Tests

1. Add test function to `baton_test.go`
2. Use `setupAndListen()` helper to start test server
3. Create `Configuration` with test parameters
4. Assert on `HTTPTestHandler` state

Example:
```go
func TestIgnoreTLSFlag(t *testing.T) {
    config := defaultConfig()
    config.ignoreTLS = true
    // ... test implementation
}
```

## Documentation Gaps

- No architecture diagram (see Architecture-Decisions for text version)
- No performance tuning guide (e.g., optimal concurrency)
- No troubleshooting guide (e.g., "why is throughput low?")
- No examples for complex CSV files
- No Docker usage examples

## Future Improvements

### High Priority

1. **Proper percentile calculation** — Replace bucketing with p50/p95/p99
2. **Request timeout configuration** — Add `-timeout` flag
3. **Graceful shutdown** — Handle SIGINT/SIGTERM
4. **Statistics in timed mode** — Collect timings even in `-t` mode

### Medium Priority

5. **Dynamic request generation** — Template-based URL/body generation
6. **Better CSV parsing** — Handle edge cases and quoted fields
7. **Request validation** — Validate URLs and methods before sending
8. **Performance tuning** — Reduce memory overhead for large counts

### Low Priority

9. **Metrics export** — JSON/Prometheus output format
10. **Distributed testing** — Coordinate multiple Baton instances
11. **Request recording** — Record and replay HTTP traffic
12. **GUI dashboard** — Real-time metrics visualization

## Reporting Issues

When reporting a bug:

1. Check this page first — may be a known limitation
2. Include Baton version: `baton -version` (if available)
3. Include Go version: `go version`
4. Provide minimal reproduction case
5. Include full error output and logs

## Contributing Fixes

Before fixing an issue:

1. Read [Coding Standards](Coding-Standards)
2. Check if there are existing tests for the area
3. Add tests for your fix
4. Run `go test -v` to ensure all tests pass
5. Run `gofmt -w` on modified files
6. See [CONTRIBUTING.md](../CONTRIBUTING.md) for PR process
