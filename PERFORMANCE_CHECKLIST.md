# Performance Checklist for Functions Framework Implementers

This quick-reference checklist helps Functions Framework implementers ensure they've addressed key performance considerations.

## 🚀 Startup Performance

- [ ] Framework initialization completes **before** listening on HTTP port
- [ ] Target function is loaded and validated during startup (not on first request)
- [ ] Heavy dependencies are loaded lazily (only when needed)
- [ ] Framework startup overhead is **< 100ms** on standard hardware
- [ ] Configuration is parsed once at startup (not per-request)
- [ ] For compiled languages: code is pre-compiled during build phase

**Test**: Measure cold start time from process start to ready-to-serve

## ⚡ Request Handling

- [ ] Request bodies are **not** fully buffered in memory by default
- [ ] Response streaming is supported for large payloads (>1MB)
- [ ] CloudEvent parsing is lazy (attributes parsed before data payload)
- [ ] HTTP headers are parsed efficiently (case-insensitive lookup without iteration)
- [ ] No unnecessary serialization/deserialization steps
- [ ] Request body parsing is on-demand (for HTTP functions)

**Test**: Profile request handling with 1MB, 10MB, and 100MB payloads

## 🔄 Concurrency

- [ ] Framework can handle **at least 10 concurrent requests** without degradation
- [ ] Worker pool or event loop is used (not thread-per-request)
- [ ] Thread/worker pool is pre-allocated and reused
- [ ] Non-blocking I/O is used for I/O-bound operations
- [ ] Function invocations are properly isolated (no state leakage)
- [ ] Concurrent requests don't cause race conditions

**Test**: Load test with 10, 50, and 100 concurrent requests

## 💾 Memory Management

- [ ] Memory buffering is bounded and configurable
- [ ] Request/response size limits are enforced
- [ ] Resources are cleaned up after each request
- [ ] No memory leaks (verify with long-running tests)
- [ ] Large objects are released promptly after use
- [ ] File handles and connections are closed properly

**Test**: Run extended load test and monitor memory usage over time

## 🔌 Resource Pooling

- [ ] Connection pools are used for database clients
- [ ] HTTP client connections are reused
- [ ] Connection pool sizes are configurable
- [ ] Idle connections are cleaned up
- [ ] Pool exhaustion is handled gracefully
- [ ] Connections are validated before reuse

**Test**: Monitor connection creation rate under load

## ⚠️ Error Handling

- [ ] Error paths are as fast or faster than success paths
- [ ] Timeouts are implemented for:
  - [ ] Function execution (default: 60s)
  - [ ] Request body read (default: 30s)
  - [ ] External service calls
- [ ] Full request/response bodies are **not** logged on every error
- [ ] Error handling doesn't allocate excessive memory
- [ ] Framework returns appropriate HTTP status codes (4xx/5xx)

**Test**: Inject errors and measure error response time

## 📦 Caching

- [ ] Configuration is cached (not re-parsed per request)
- [ ] Parsed function code is cached
- [ ] Connection pools serve as connection cache
- [ ] No `ETag` header is sent by default
- [ ] Framework **doesn't** cache request-specific data
- [ ] Caches are cleared when appropriate

**Test**: Verify configuration is parsed only once

## 📊 Observability

- [ ] Basic metrics (request count, latency) add **< 1% overhead**
- [ ] Expensive tracing/profiling is opt-in
- [ ] Logging is asynchronous (doesn't block requests)
- [ ] Sampling is supported for high-volume scenarios
- [ ] `traceparent` header parsing is efficient
- [ ] Observability features can be disabled

**Test**: Measure performance with/without observability enabled

## 🎯 CloudEvent Processing

- [ ] Both binary and structured content modes are supported
- [ ] Binary mode is optimized (lower overhead)
- [ ] CloudEvent attributes are parsed before data payload
- [ ] Large data payloads (>1MB) are streamed
- [ ] Schema validation is opt-in (due to overhead)
- [ ] Parsing errors return appropriate 400 status

**Test**: Benchmark both content modes with various payload sizes

## 🛣️ Routing

- [ ] Static paths (`/robots.txt`, `/favicon.ico`) are handled without function invocation
- [ ] Path matching is efficient (hash lookup or trie)
- [ ] Health check endpoint (if provided) responds in **< 10ms**
- [ ] Routing adds minimal overhead
- [ ] All other paths route to the function

**Test**: Measure routing overhead (request to function invocation)

## 🧪 Testing

- [ ] Performance benchmarks exist and are tracked
- [ ] Load testing is performed regularly
- [ ] Metrics tracked:
  - [ ] Requests per second (throughput)
  - [ ] Latency (p50, p95, p99)
  - [ ] Cold start time
  - [ ] Memory usage
- [ ] Performance regressions are detected in CI
- [ ] Concurrent load testing is performed
- [ ] Resource limits are tested (memory, file descriptors)

## 🏗️ Language-Specific Optimizations

### Python
- [ ] Use async WSGI/ASGI servers (`uvicorn`, `gunicorn` with async workers)
- [ ] Avoid global imports in function code
- [ ] Use `asyncio` for I/O operations

### Node.js
- [ ] Event loop is never blocked
- [ ] Clustering is used for CPU-bound operations
- [ ] Dependencies are minimized (small `node_modules`)

### Go
- [ ] Goroutines are used for concurrency
- [ ] `GOMAXPROCS` is set appropriately
- [ ] `sync.Pool` is used for buffer reuse

### Java
- [ ] Virtual threads are used (Java 21+) or thread pool is configured
- [ ] JVM heap size is configured appropriately
- [ ] Non-blocking I/O is used (Netty, etc.)

### .NET
- [ ] `async`/`await` is used throughout
- [ ] Thread pool is configured
- [ ] `Span<T>` and `Memory<T>` are used to reduce allocations

### C++
- [ ] RAII is used for resource management
- [ ] Move semantics are used to avoid copies
- [ ] Memory pools are used for allocations

## 📈 Performance Targets

Based on standard hardware (2 CPU cores, 2GB RAM):

| Metric | Target |
|--------|--------|
| Cold start (framework overhead) | < 100ms |
| Request latency (empty function, p50) | < 5ms |
| Request latency (empty function, p99) | < 50ms |
| Throughput (empty function) | > 1000 req/s |
| Concurrent requests | ≥ 10 without degradation |
| Memory overhead (idle) | < 50MB |

## 🔍 Profiling

- [ ] CPU profiling is performed under load
- [ ] Memory profiling identifies leaks and allocations
- [ ] I/O profiling reveals blocking operations
- [ ] Flame graphs are generated for bottleneck identification
- [ ] Profiling is done for both cold start and steady state

## 📚 Documentation

- [ ] Performance characteristics are documented
- [ ] Configuration options affecting performance are documented
- [ ] Known performance limitations are documented
- [ ] Benchmarking methodology is documented
- [ ] Performance tuning guide exists for users

## ✅ Final Validation

Before releasing:

1. **Benchmark** against previous version
2. **Profile** under realistic load
3. **Test** resource limits (what happens at max memory/connections?)
4. **Verify** no performance regressions
5. **Document** any performance changes

---

## How to Use This Checklist

1. **During Development**: Check items as you implement features
2. **During Review**: Use as code review checklist
3. **Before Release**: Ensure all items are addressed
4. **Regular Audits**: Revisit periodically to ensure compliance

## References

- [Identifying and Fixing Slow Code](IDENTIFYING_SLOW_CODE.md) - Guide to identifying and fixing slow or inefficient code patterns
- [Performance Optimization Guide](PERFORMANCE.md) - Detailed best practices
- [Specification Improvements](SPECIFICATION_IMPROVEMENTS.md) - Suggested spec enhancements
- [Functions Framework Specification](README.md#functions-framework-contract) - Core requirements

---

**Note**: This checklist represents best practices. Some items may not be applicable to all implementation languages or deployment environments. Use judgment based on your specific context.
