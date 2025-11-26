# Identifying and Fixing Slow or Inefficient Code

This guide helps developers and framework implementers identify slow or inefficient code patterns in Functions Framework implementations and provides actionable improvements.

## Table of Contents
- [Profiling and Detection Tools](#profiling-and-detection-tools)
- [Common Inefficiency Patterns](#common-inefficiency-patterns)
- [Startup Performance Issues](#startup-performance-issues)
- [Request Handling Inefficiencies](#request-handling-inefficiencies)
- [Memory and Resource Leaks](#memory-and-resource-leaks)
- [Language-Specific Issues](#language-specific-issues)
- [Measurement and Monitoring](#measurement-and-monitoring)

## Profiling and Detection Tools

Before fixing slow code, you need to identify where the bottlenecks are. Use these tools and techniques:

### General Profiling Approach

1. **Establish Baselines**: Measure current performance before making changes
2. **Profile Under Load**: Use realistic load patterns, not just single requests
3. **Measure Cold Start Separately**: Cold start and warm request performance have different bottlenecks
4. **Use Multiple Metrics**: Track latency (p50, p95, p99), throughput, and memory usage

### Language-Specific Profilers

| Language | Profiling Tools |
|----------|-----------------|
| **Python** | `cProfile`, `py-spy`, `memory_profiler`, `yappi` |
| **Node.js** | `--prof` flag, `clinic.js`, `0x`, Chrome DevTools |
| **Go** | `pprof`, `go test -bench`, `go tool trace` |
| **Java** | VisualVM, JProfiler, `async-profiler`, JFR |
| **.NET** | dotnet-trace, PerfView, BenchmarkDotNet |
| **Ruby** | `ruby-prof`, `stackprof`, `memory_profiler` |
| **C++** | Valgrind, perf, gperftools |

### Key Metrics to Collect

```
┌─────────────────────────────────────────────────────────────┐
│                    Performance Metrics                       │
├─────────────────────────────────────────────────────────────┤
│ Cold Start Time     │ Time from process start to ready      │
│ Request Latency     │ Time to handle a single request       │
│ Throughput          │ Requests per second under load        │
│ Memory Usage        │ Peak and steady-state memory          │
│ CPU Utilization     │ Processing time per request           │
│ GC Pause Time       │ Garbage collection impact             │
│ Connection Count    │ Open connections to external services │
└─────────────────────────────────────────────────────────────┘
```

## Common Inefficiency Patterns

### Pattern 1: Synchronous Blocking in Event-Driven Runtimes

**Symptoms:**
- Low throughput despite low CPU usage
- Latency increases dramatically under concurrent load
- Event loop lag (Node.js) or thread starvation (Java)

**Detection:**
```javascript
// Node.js: Detect event loop lag
const lag = require('event-loop-lag')(1000);
setInterval(() => {
  if (lag() > 100) {
    console.warn(`Event loop lag: ${lag()}ms`);
  }
}, 5000);
```

**Slow Code:**
```python
# Python - Blocking call in async context
async def handle_request(request):
    data = requests.get('http://api.example.com/data')  # BLOCKS!
    return process(data)
```

**Fixed Code:**
```python
# Python - Non-blocking async call
import aiohttp

async def handle_request(request):
    async with aiohttp.ClientSession() as session:
        async with session.get('http://api.example.com/data') as response:
            data = await response.json()
    return process(data)
```

### Pattern 2: Resource Creation Per Request

**Symptoms:**
- High latency on every request (not just first)
- Memory grows then drops (frequent GC)
- Connection errors under load (connection pool exhaustion)

**Detection:**
Look for `new` keyword or constructor calls inside request handlers.

**Slow Code:**
```java
// Java - Creating client per request
public void handleRequest(HttpRequest request, HttpResponse response) {
    HttpClient client = HttpClient.newHttpClient();  // EXPENSIVE!
    // ... use client
}
```

**Fixed Code:**
```java
// Java - Reuse client across requests
private static final HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(10))
    .build();

public void handleRequest(HttpRequest request, HttpResponse response) {
    // ... use shared client
}
```

### Pattern 3: Repeated Parsing and Serialization

**Symptoms:**
- High CPU usage per request
- Latency scales with payload size
- Memory spikes during request processing

**Detection:**
Profile CPU time and look for JSON/XML parsing taking significant percentage.

**Slow Code:**
```go
// Go - Parse, serialize, then parse again
func handleRequest(w http.ResponseWriter, r *http.Request) {
    data, _ := io.ReadAll(r.Body)
    var obj MyStruct
    json.Unmarshal(data, &obj)
    
    // Later in the code...
    jsonBytes, _ := json.Marshal(obj)  // Why serialize?
    var obj2 MyStruct
    json.Unmarshal(jsonBytes, &obj2)   // Just to parse again?
}
```

**Fixed Code:**
```go
// Go - Work with parsed data directly
func handleRequest(w http.ResponseWriter, r *http.Request) {
    var obj MyStruct
    json.NewDecoder(r.Body).Decode(&obj)
    // Work with obj directly, no re-serialization
}
```

### Pattern 4: Unbounded Memory Buffering

**Symptoms:**
- Out of memory errors with large payloads
- Memory usage grows with request size
- Process killed by OOM killer

**Detection:**
Check for patterns like `readAll()`, `getBytes()`, or collecting streams into lists.

**Slow Code:**
```javascript
// Node.js - Buffer entire request body
app.post('/upload', (req, res) => {
  let body = '';
  req.on('data', chunk => {
    body += chunk;  // Accumulates in memory
  });
  req.on('end', () => {
    processLargeFile(body);  // Memory spike
  });
});
```

**Fixed Code:**
```javascript
// Node.js - Stream processing
const { pipeline } = require('stream/promises');

app.post('/upload', async (req, res) => {
  await pipeline(
    req,
    createProcessingTransform(),
    createOutputStream()
  );
  res.send('OK');
});
```

### Pattern 5: N+1 Query Pattern

**Symptoms:**
- Latency increases linearly with data size
- Many small database queries in logs
- Network I/O dominates request time

**Detection:**
Count database/API calls per request. If it scales with data items, you have N+1.

**Slow Code:**
```ruby
# Ruby - N+1 queries
def get_orders_with_items
  orders = Order.all
  orders.map do |order|
    {
      order: order,
      items: OrderItem.where(order_id: order.id)  # Query per order!
    }
  end
end
```

**Fixed Code:**
```ruby
# Ruby - Batch loading
def get_orders_with_items
  orders = Order.includes(:order_items).all
  orders.map do |order|
    {
      order: order,
      items: order.order_items  # No additional queries
    }
  end
end
```

## Startup Performance Issues

### Issue 1: Eager Loading of Heavy Dependencies

**Symptoms:**
- Cold start time > 1 second
- CPU spike at startup
- Most startup time spent in imports/requires

**Detection:**
```bash
# Python - Profile import time
python -X importtime -c "import your_module" 2>&1 | sort -t: -k2 -n

# Node.js - Profile require time
node --require-timing your_script.js
```

**Slow Code:**
```python
# Python - Eager import of heavy library
import pandas  # 500ms+ import time
import tensorflow  # 2+ seconds import time

def handle_request(request):
    if request.type == 'ml':
        return tensorflow.predict(...)
    return simple_response()
```

**Fixed Code:**
```python
# Python - Lazy import
def handle_request(request):
    if request.type == 'ml':
        import tensorflow  # Only import when needed
        return tensorflow.predict(...)
    return simple_response()
```

### Issue 2: Synchronous Initialization

**Symptoms:**
- Cold start blocked on I/O operations
- Connection to external services during startup
- Configuration fetched at startup

**Detection:**
Profile startup time and identify I/O waits.

**Slow Code:**
```javascript
// Node.js - Blocking startup
const config = await fetch('https://config.example.com/settings').json();
const dbConnection = await createConnection(config.database);
// Only now start listening for requests
```

**Fixed Code:**
```javascript
// Node.js - Non-blocking startup with lazy initialization
let dbConnection = null;

async function getDbConnection() {
  if (!dbConnection) {
    const config = await getConfig();
    dbConnection = await createConnection(config.database);
  }
  return dbConnection;
}

// Start listening immediately, initialize on first request
```

### Issue 3: Unoptimized Container Images

**Symptoms:**
- Long image pull time
- Large memory footprint
- Slow container startup

**Detection:**
```bash
# Check image size
docker images | grep your-image

# Analyze image layers
docker history your-image
```

**Improvement Strategies:**
```dockerfile
# Bad: Large base image with unnecessary tools
FROM python:3.9

# Better: Slim base image
FROM python:3.9-slim

# Best: Distroless or Alpine for production
FROM gcr.io/distroless/python3
```

## Request Handling Inefficiencies

### Issue 1: Inefficient CloudEvent Parsing

**Symptoms:**
- Higher latency for CloudEvent functions vs HTTP
- CPU usage dominated by JSON parsing
- Memory spikes with large event data

**Slow Code:**
```java
// Java - Parse entire CloudEvent upfront
public void handleCloudEvent(CloudEvent event) {
    // getData() parses entire data payload
    MyData data = event.getData().toObject(MyData.class);
    
    // But we only need the type for routing
    if (event.getType().equals("type.we.dont.handle")) {
        return;  // Wasted parsing!
    }
}
```

**Fixed Code:**
```java
// Java - Check attributes before parsing data
public void handleCloudEvent(CloudEvent event) {
    // Check type first (already parsed from headers)
    if (event.getType().equals("type.we.dont.handle")) {
        return;  // No data parsing needed
    }
    
    // Only parse data if we need it
    MyData data = event.getData().toObject(MyData.class);
}
```

### Issue 2: Inefficient Header Processing

**Symptoms:**
- Latency increases with number of headers
- O(n) header lookups

**Slow Code:**
```python
# Python - Linear search through headers
def get_content_type(headers):
    for name, value in headers.items():
        if name.lower() == 'content-type':  # O(n) per lookup
            return value
    return None
```

**Fixed Code:**
```python
# Python - Use framework's efficient header access
def get_content_type(request):
    return request.content_type  # Optimized lookup
```

### Issue 3: Blocking Response Writes

**Symptoms:**
- High latency for large responses
- Memory proportional to response size
- Backpressure not handled

**Slow Code:**
```go
// Go - Buffer entire response before sending
func handleRequest(w http.ResponseWriter, r *http.Request) {
    result := generateLargeResult()  // 100MB in memory
    json.NewEncoder(w).Encode(result)  // All at once
}
```

**Fixed Code:**
```go
// Go - Stream response
func handleRequest(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    encoder := json.NewEncoder(w)
    
    for item := range generateResultStream() {
        encoder.Encode(item)
        if f, ok := w.(http.Flusher); ok {
            f.Flush()  // Send immediately
        }
    }
}
```

## Memory and Resource Leaks

### Leak 1: Unclosed Connections

**Symptoms:**
- File descriptor exhaustion
- "Too many open files" errors
- Connection pool exhaustion

**Detection:**
```bash
# Monitor open file descriptors
lsof -p <pid> | wc -l

# Check for connection leaks
netstat -an | grep ESTABLISHED | wc -l
```

**Slow Code:**
```csharp
// C# - Connection not properly closed
public async Task HandleRequest()
{
    var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();
    // ... use connection
    // Connection never closed if exception thrown!
}
```

**Fixed Code:**
```csharp
// C# - Proper resource cleanup
public async Task HandleRequest()
{
    await using var connection = new SqlConnection(connectionString);
    await connection.OpenAsync();
    // ... use connection
}  // Automatically disposed
```

### Leak 2: Event Listener Accumulation

**Symptoms:**
- Memory grows over time
- Increasing number of callbacks
- MaxListenersExceededWarning (Node.js)

**Slow Code:**
```javascript
// Node.js - Event listeners never removed
function handleRequest(emitter, callback) {
  emitter.on('data', callback);  // Adds listener every request
  // Listener never removed
}
```

**Fixed Code:**
```javascript
// Node.js - Proper listener management
function handleRequest(emitter, callback) {
  emitter.once('data', callback);  // Auto-removed after first emit
  // Or manually manage:
  emitter.on('data', callback);
  return () => emitter.off('data', callback);
}
```

### Leak 3: Cache Without Bounds

**Symptoms:**
- Memory grows continuously
- OOM after extended operation
- No eviction under pressure

**Slow Code:**
```python
# Python - Unbounded cache
cache = {}

def get_data(key):
    if key not in cache:
        cache[key] = fetch_data(key)  # Grows forever!
    return cache[key]
```

**Fixed Code:**
```python
# Python - Bounded LRU cache
from functools import lru_cache

@lru_cache(maxsize=1000)  # Bounded size
def get_data(key):
    return fetch_data(key)
```

## Language-Specific Issues

### Python

| Issue | Detection | Solution |
|-------|-----------|----------|
| Global Interpreter Lock (GIL) | High CPU, low parallelism | Use multiprocessing or async I/O |
| Large list comprehensions | Memory spikes | Use generators |
| String concatenation in loops | O(n²) behavior | Use `''.join()` or `io.StringIO` |

### Node.js

| Issue | Detection | Solution |
|-------|-----------|----------|
| Blocking event loop | High event loop lag | Use worker threads for CPU work |
| Deep Promise chains | Stack traces unclear | Use async/await |
| require() in hot path | CPU in require | Move to top-level |

### Go

| Issue | Detection | Solution |
|-------|-----------|----------|
| Excessive allocations | High GC pressure | Use sync.Pool |
| Goroutine leaks | Goroutine count grows | Use context cancellation |
| Interface boxing | Escape analysis shows heap | Use concrete types |

### Java

| Issue | Detection | Solution |
|-------|-----------|----------|
| Frequent young GC | GC logs | Tune heap sizing |
| Lock contention | Thread dumps | Use concurrent collections |
| Class loading at runtime | Profiler | Warmup or AOT |

### .NET

| Issue | Detection | Solution |
|-------|-----------|----------|
| LOH allocations | Memory profiler | Use ArrayPool |
| Excessive boxing | IL inspection | Use generics |
| Sync over async | Thread pool exhaustion | Async all the way |

## Measurement and Monitoring

### Establishing Baselines

Before optimizing, establish performance baselines:

```bash
# Example: Load test with wrk
wrk -t4 -c100 -d30s http://localhost:8080/

# Example: Measure cold start
time (docker run --rm your-function-image)

# Example: Memory profiling
valgrind --tool=massif ./your-binary
```

### Continuous Performance Monitoring

Implement performance regression detection:

1. **Track metrics over time**
   - Request latency percentiles (p50, p95, p99)
   - Cold start time
   - Memory usage (peak and average)
   - Throughput under load

2. **Set performance budgets**
   ```yaml
   performance_budgets:
     cold_start_ms: 100
     p99_latency_ms: 50
     memory_mb: 256
     throughput_rps: 1000
   ```

3. **Fail CI on regressions**
   ```bash
   # Example: Assert latency in CI
   if [ $P99_LATENCY -gt 50 ]; then
     echo "Performance regression: p99 latency exceeded 50ms"
     exit 1
   fi
   ```

### Performance Audit Checklist

Before deploying, verify:

- [ ] Cold start time measured and within budget
- [ ] No blocking operations in async contexts
- [ ] Resources (connections, handles) properly pooled
- [ ] Memory bounded (no unbounded caches or buffers)
- [ ] No N+1 query patterns
- [ ] Profiling shows no obvious hotspots
- [ ] Load testing completed with realistic concurrency

## Summary

The most impactful performance improvements typically come from:

1. **Connection reuse** - Pool and reuse HTTP clients, database connections
2. **Async I/O** - Never block the event loop or thread pool with I/O
3. **Lazy loading** - Defer expensive initialization until needed
4. **Streaming** - Avoid buffering large payloads in memory
5. **Caching** - Cache expensive computations with bounded caches

Use profiling tools to identify the actual bottlenecks in your specific implementation before optimizing.

## Related Documentation

- **[Performance Optimization Guide](PERFORMANCE.md)** - Comprehensive best practices
- **[Performance Checklist](PERFORMANCE_CHECKLIST.md)** - Quick-reference checklist
- **[Specification Improvements](SPECIFICATION_IMPROVEMENTS.md)** - Spec enhancement suggestions
