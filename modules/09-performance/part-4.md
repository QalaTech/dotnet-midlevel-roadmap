# Part 4 — Troubleshooting & Tuning Playbook

## Why This Matters

It's 3 AM. Your phone buzzes. The alert says "API latency critical."

Two types of engineers exist in this moment:

**Engineer A**: Panics, randomly restarts services, makes things worse, spends 4 hours guessing.

**Engineer B**: Opens the runbook, follows the decision tree, identifies the issue in 15 minutes, applies the fix, goes back to sleep.

This section makes you Engineer B.

---

## What Goes Wrong Without This

### The 4-Hour Mystery

```
3:00 AM - Alert: High latency
3:05 AM - "Is the database slow?" Checks DB, looks fine
3:20 AM - "Is it memory?" Checks memory, looks fine
3:35 AM - "Is it the cache?" Restarts Redis (makes it worse)
3:50 AM - "Is it the network?" Pings everything
4:30 AM - "Is it the new deployment?" Rolls back (no change)
5:15 AM - "Let me restart everything"
6:00 AM - Accidentally finds the issue in logs
6:05 AM - Fix: A single missing index
7:00 AM - Finally goes back to sleep

With a runbook: 15 minutes to identify, 5 minutes to fix.
```

### The "It's Always DNS" Meme (But Sometimes It Is)

```
Actual incident breakdown:
- 40% Database issues (queries, connections, locks)
- 25% Memory problems (leaks, GC pressure)
- 15% External dependencies (APIs, queues)
- 10% Configuration errors
- 5% DNS (but when it is, it's brutal)
- 5% "This shouldn't be possible"
```

---

## The Troubleshooting Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│                    INCIDENT DETECTED                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Is it affecting users NOW?                     │
├────────────────────┬────────────────────────────────────────┤
│        YES         │                  NO                    │
│   (Continue)       │    (Document, investigate later)       │
└────────────────────┴────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              What's the symptom?                            │
├─────────────┬─────────────┬─────────────┬───────────────────┤
│ High        │ Errors      │ Timeouts    │ Service           │
│ Latency     │ (5xx)       │             │ Unresponsive      │
│   │         │    │        │     │       │       │           │
│   ▼         │    ▼        │     ▼       │       ▼           │
│ [Part A]    │ [Part B]    │ [Part C]    │  [Part D]         │
└─────────────┴─────────────┴─────────────┴───────────────────┘
```

---

## Part A: High Latency Troubleshooting

### Step 1: Identify What's Slow

```bash
# Check recent latency by endpoint
curl -s http://localhost:9090/api/v1/query \
  --data-urlencode 'query=histogram_quantile(0.95, sum(rate(http_duration_bucket[5m])) by (le, endpoint))' \
  | jq '.data.result[] | {endpoint: .metric.endpoint, p95: .value[1]}'

# Expected: Identify which endpoint(s) are slow
```

### Step 2: Trace a Slow Request

```bash
# Get trace IDs for slow requests (from logs)
grep "duration_ms" /var/log/orderflow/api.log | \
  awk '$NF > 1000 {print $3}' | head -5

# Look up trace in Jaeger/Zipkin
# Find which span is slow
```

### Step 3: Common Causes and Fixes

| Symptom | Likely Cause | Quick Check | Fix |
|---------|--------------|-------------|-----|
| All endpoints slow | Database | Connection pool stats | Scale connections/add replicas |
| Single endpoint slow | Query/algorithm | Trace spans | Optimize query/add index |
| Intermittent slowness | GC pressure | GC metrics | Reduce allocations |
| Slow at specific times | External dependency | Trace external calls | Add timeout/circuit breaker |

---

## Part B: Error Rate Troubleshooting

### Step 1: Categorize Errors

```sql
-- Query your log aggregator (Seq, Elasticsearch, Loki)
SELECT
    status_code,
    exception_type,
    COUNT(*) as count
FROM logs
WHERE timestamp > NOW() - INTERVAL '1 hour'
  AND status_code >= 500
GROUP BY status_code, exception_type
ORDER BY count DESC
LIMIT 10;
```

### Step 2: Error-Specific Playbooks

#### Connection Errors

```csharp
// Symptom: SqlException - Connection pool exhausted
// Check: Npgsql statistics

public class DiagnosticController : ControllerBase
{
    [HttpGet("pool-stats")]
    public IActionResult GetPoolStats()
    {
        // Requires Npgsql 7.0+
        var stats = NpgsqlConnection.GlobalStatistics;
        return Ok(new
        {
            stats.Total,
            stats.Idle,
            stats.Busy,
            stats.Waiting,
            stats.Max
        });
    }
}
```

**Fix checklist**:
1. [ ] Check for connection leaks (missing `using` statements)
2. [ ] Check for long-running transactions
3. [ ] Increase pool size if legitimately needed
4. [ ] Add connection timeout alerts

#### Timeout Errors

```csharp
// Symptom: TaskCanceledException or TimeoutException
// Common causes:
// 1. External API slow
// 2. Database query slow
// 3. Lock contention

// Diagnostic: Add timing to external calls
public async Task<T> TimeExternalCall<T>(string name, Func<Task<T>> call)
{
    var sw = Stopwatch.StartNew();
    try
    {
        return await call();
    }
    finally
    {
        sw.Stop();
        _logger.LogInformation(
            "External call {Name} completed in {Duration}ms",
            name, sw.ElapsedMilliseconds);
    }
}
```

---

## Part C: Thread Pool Starvation

### Symptoms

- Requests queuing but CPU low
- Timeouts across multiple services
- `ThreadPool.QueueUserWorkItem` taking seconds

### Diagnosis

```csharp
// Add this diagnostic endpoint
[HttpGet("threadpool")]
public IActionResult GetThreadPoolStats()
{
    ThreadPool.GetAvailableThreads(out int workerAvailable, out int ioAvailable);
    ThreadPool.GetMaxThreads(out int workerMax, out int ioMax);
    ThreadPool.GetMinThreads(out int workerMin, out int ioMin);

    return Ok(new
    {
        Worker = new
        {
            Available = workerAvailable,
            Max = workerMax,
            Min = workerMin,
            InUse = workerMax - workerAvailable
        },
        IO = new
        {
            Available = ioAvailable,
            Max = ioMax,
            Min = ioMin,
            InUse = ioMax - ioAvailable
        }
    });
}

// Warning signs:
// - Worker.Available < 10
// - Worker.InUse close to Worker.Max
```

### Common Causes

```csharp
// Cause 1: Sync-over-async
public Order GetOrder(int id)
{
    // BLOCKS a thread pool thread waiting for I/O
    return _context.Orders.FindAsync(id).Result;
}

// Cause 2: Unbounded parallelism
public async Task ProcessAll(List<int> ids)
{
    // Creates ALL tasks at once, may exhaust thread pool
    var tasks = ids.Select(id => ProcessAsync(id));
    await Task.WhenAll(tasks);
}

// Fix: Bounded parallelism
public async Task ProcessAllBounded(List<int> ids)
{
    var options = new ParallelOptions { MaxDegreeOfParallelism = 10 };
    await Parallel.ForEachAsync(ids, options, async (id, ct) =>
    {
        await ProcessAsync(id);
    });
}

// Cause 3: Long-running synchronous work
public void ProcessData(byte[] data)
{
    // CPU-bound work blocking thread pool threads
    for (int i = 0; i < data.Length; i++)
    {
        data[i] = Transform(data[i]);  // No await, just CPU
    }
}

// Fix: Use Task.Run for CPU-bound work
public async Task ProcessDataAsync(byte[] data)
{
    await Task.Run(() =>
    {
        for (int i = 0; i < data.Length; i++)
        {
            data[i] = Transform(data[i]);
        }
    });
}
```

### Immediate Mitigation

```csharp
// Increase minimum threads (buy time while fixing root cause)
// In Program.cs or startup
ThreadPool.SetMinThreads(100, 100);

// Note: This is a WORKAROUND, not a fix!
// Find and fix the sync-over-async code
```

---

## Part D: Memory Dump Analysis

### Capturing Dumps

```bash
# Install diagnostic tools
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-gcdump

# Find the process
dotnet-dump ps

# Capture full dump (complete heap)
dotnet-dump collect -p <PID> -o /tmp/full.dmp

# Capture GC dump (smaller, just managed heap)
dotnet-gcdump collect -p <PID> -o /tmp/gc.gcdump
```

### Analyzing with dotnet-dump

```bash
# Open the dump
dotnet-dump analyze /tmp/full.dmp

# Common commands:
> dumpheap -stat
# Shows all objects sorted by total size

> dumpheap -type OrderFlow.Order -stat
# Shows specific type

> dumpheap -type System.String -stat
# Strings are often a memory hog

> gcroot <address>
# Find what's keeping an object alive

> threadpool
# Thread pool statistics

> threads
# List all threads

> clrstack
# Stack trace of current thread
```

### Common Memory Issues

#### Memory Leak: Event Handler Not Unsubscribed

```csharp
// Anti-pattern: Event subscription without cleanup
public class OrderNotifier
{
    public OrderNotifier(IOrderService service)
    {
        service.OrderCreated += OnOrderCreated;
        // Never unsubscribed = memory leak
    }

    private void OnOrderCreated(object sender, OrderEventArgs e)
    {
        // Handle event
    }
}

// Fix: Implement IDisposable
public class OrderNotifier : IDisposable
{
    private readonly IOrderService _service;

    public OrderNotifier(IOrderService service)
    {
        _service = service;
        _service.OrderCreated += OnOrderCreated;
    }

    public void Dispose()
    {
        _service.OrderCreated -= OnOrderCreated;
    }
}
```

#### Memory Leak: Static Collections Growing

```csharp
// Anti-pattern: Unbounded static cache
public static class OrderCache
{
    private static readonly Dictionary<int, Order> _cache = new();

    public static void Add(Order order)
    {
        _cache[order.Id] = order;  // Never removed!
    }
}

// Fix: Use memory cache with expiration
public class OrderCache
{
    private readonly IMemoryCache _cache;

    public void Add(Order order)
    {
        _cache.Set(order.Id, order, TimeSpan.FromMinutes(10));
    }
}
```

---

## SQL Query Tuning

### Finding Slow Queries

```sql
-- PostgreSQL: Find slow queries
SELECT
    query,
    calls,
    mean_time,
    total_time,
    rows
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- SQL Server: Find slow queries
SELECT TOP 20
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2)+1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_elapsed_time DESC;
```

### Finding Missing Indexes

```sql
-- PostgreSQL: Find sequential scans on large tables
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;

-- SQL Server: Missing index suggestions
SELECT
    mig.index_group_handle,
    mid.index_handle,
    migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,
    'CREATE INDEX IX_' + REPLACE(REPLACE(REPLACE(mid.statement, '[', ''), ']', ''), '.', '_') +
    ' ON ' + mid.statement +
    ' (' + ISNULL(mid.equality_columns, '') +
    CASE WHEN mid.equality_columns IS NOT NULL AND mid.inequality_columns IS NOT NULL THEN ',' ELSE '' END +
    ISNULL(mid.inequality_columns, '') + ')' +
    ISNULL(' INCLUDE (' + mid.included_columns + ')', '') AS create_index_statement
FROM sys.dm_db_missing_index_groups mig
INNER JOIN sys.dm_db_missing_index_group_stats migs ON migs.group_handle = mig.index_group_handle
INNER JOIN sys.dm_db_missing_index_details mid ON mig.index_handle = mid.index_handle
ORDER BY improvement_measure DESC;
```

### Analyzing Query Plans

```sql
-- PostgreSQL: Explain analyze
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2024-01-01'
  AND o.status = 'pending';

-- Look for:
-- - Seq Scan on large tables (needs index)
-- - Nested Loop with high row estimates (consider different join)
-- - Sort with high cost (add index for ORDER BY)
-- - Buffers: shared read (data not in cache)
```

### Index Strategy

```sql
-- Covering index for common query
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status)
INCLUDE (created_at, total);

-- Partial index for status queries
CREATE INDEX idx_orders_pending ON orders (created_at)
WHERE status = 'pending';

-- Index for range queries
CREATE INDEX idx_orders_created ON orders (created_at DESC);
```

---

## Database Lock Investigation

### Finding Blocking Queries

```sql
-- PostgreSQL: Find blocking queries
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_locks.pid = blocked_activity.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
    AND blocked_locks.page IS NOT DISTINCT FROM blocking_locks.page
    AND blocked_locks.tuple IS NOT DISTINCT FROM blocking_locks.tuple
    AND blocked_locks.virtualxid IS NOT DISTINCT FROM blocking_locks.virtualxid
    AND blocked_locks.transactionid IS NOT DISTINCT FROM blocking_locks.transactionid
    AND blocked_locks.classid IS NOT DISTINCT FROM blocking_locks.classid
    AND blocked_locks.objid IS NOT DISTINCT FROM blocking_locks.objid
    AND blocked_locks.objsubid IS NOT DISTINCT FROM blocking_locks.objsubid
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted;

-- SQL Server: sp_WhoIsActive (install from http://whoisactive.com)
EXEC sp_WhoIsActive @get_locks = 1;
```

### Terminating Problem Queries

```sql
-- PostgreSQL: Cancel query (graceful)
SELECT pg_cancel_backend(<pid>);

-- PostgreSQL: Terminate connection (forceful)
SELECT pg_terminate_backend(<pid>);

-- SQL Server: Kill session
KILL <spid>;
```

---

## Incident Response Runbook Template

```markdown
# Runbook: High API Latency

## Severity: P1 (User-Facing)

## Symptoms
- API p95 latency > 2 seconds
- Alert: OrderFlowHighLatency

## Quick Wins (Try First)
1. [ ] Check recent deployments (rollback if < 30 min ago)
2. [ ] Check external dependencies (Payment API, Inventory API)
3. [ ] Check database connection pool utilization

## Diagnostic Steps

### Step 1: Identify Slow Endpoint
```bash
kubectl exec -it orderflow-api-xxx -- curl localhost:8080/metrics | grep http_duration
```

### Step 2: Check Database
```bash
kubectl exec -it postgres-0 -- psql -U app -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"
```

### Step 3: Check Memory
```bash
kubectl top pods -n orderflow
```

## Mitigation Options

### Option A: Scale Out
```bash
kubectl scale deployment orderflow-api --replicas=10
```

### Option B: Enable Circuit Breaker
```bash
kubectl set env deployment/orderflow-api FEATURE_CIRCUIT_BREAKER=true
```

### Option C: Rollback
```bash
kubectl rollout undo deployment/orderflow-api
```

## Post-Incident
- [ ] Create timeline
- [ ] Identify root cause
- [ ] File follow-up tickets
- [ ] Update this runbook
```

---

## Postmortem Template

```markdown
# Postmortem: OrderFlow Latency Incident - 2024-01-15

## Summary
On January 15, 2024, OrderFlow API experienced elevated latency for 45 minutes,
affecting approximately 12,000 requests with p95 latency exceeding 5 seconds.

## Impact
- Duration: 45 minutes (14:23 - 15:08 UTC)
- Affected users: ~3,200
- Failed orders: 147
- Revenue impact: $12,450 estimated

## Timeline (All times UTC)
- 14:15 - Deployment of v2.3.4 completed
- 14:23 - First alert: p95 latency > 2s
- 14:25 - On-call acknowledged, began investigation
- 14:30 - Identified database connection pool exhaustion
- 14:35 - Attempted to increase pool size (no improvement)
- 14:45 - Found new query pattern in deployment causing lock contention
- 14:52 - Initiated rollback to v2.3.3
- 15:05 - Rollback complete
- 15:08 - Latency returned to normal

## Root Cause
A new feature in v2.3.4 introduced a query that acquired row-level locks
on the `orders` table for an extended duration. Under load, this caused
lock contention, blocking other queries and exhausting the connection pool.

## Contributing Factors
1. Query was not load-tested before deployment
2. No lock monitoring in place
3. Feature flag was not used for gradual rollout

## Action Items
| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| Add lock duration monitoring | @alice | 2024-01-22 | Open |
| Implement load testing for new queries | @bob | 2024-01-29 | Open |
| Require feature flags for DB changes | @carol | 2024-02-05 | Open |
| Fix query to use optimistic locking | @dave | 2024-01-19 | In Progress |

## Lessons Learned
### What Went Well
- Alert fired within 8 minutes of issue starting
- On-call responded quickly
- Rollback process worked smoothly

### What Could Be Improved
- Load testing should catch lock contention
- Need better visibility into database locks
- Feature flags for safer deployments

## Detection Improvement
Adding new alert:
```yaml
- alert: OrderFlowLockContention
  expr: pg_locks_count{mode="ExclusiveLock"} > 50
  for: 2m
  labels:
    severity: warning
```
```

---

## Hands-On Exercise: Incident Drill

### Setup: Create a Problem

```csharp
// Add this deliberately problematic code to OrderFlow
[HttpPost("stress-test-leak")]
public async Task<IActionResult> CreateMemoryLeak()
{
    // This endpoint creates a memory leak for training purposes
    var leakyList = new List<byte[]>();
    for (int i = 0; i < 100; i++)
    {
        leakyList.Add(new byte[1024 * 1024]);  // 1MB each
        _staticLeakList.Add(leakyList);  // Held by static = never collected
    }
    return Ok("Leaked 100MB");
}

private static readonly List<object> _staticLeakList = new();
```

### Exercise Steps

1. **Create the incident**: Call the endpoint multiple times
2. **Detect**: Watch memory metrics climb
3. **Diagnose**: Take a GC dump, analyze with `dotnet-gcdump`
4. **Mitigate**: Restart the pod
5. **Fix**: Find and remove the leak
6. **Document**: Write a mini-postmortem

---

## Deliverables

1. **Runbook Collection**: At least 3 runbooks for common issues
2. **Postmortem**: Write a postmortem for a simulated incident
3. **Query Optimization**: Document before/after for a slow query
4. **Dump Analysis**: Capture and analyze a memory dump

---

## Reflection Questions

1. Why is having runbooks important, even for experienced engineers?
2. What's the difference between mitigation and resolution?
3. How do you balance thoroughness with speed during an incident?
4. What makes a good postmortem culture?

---

## Resources

### Diagnostics
- [dotnet-dump Documentation](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump)
- [dotnet-trace Documentation](https://docs.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
- [SOS Debugging Commands](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension)

### SQL Performance
- [PostgreSQL EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [sp_WhoIsActive](http://whoisactive.com/)

### Incident Management
- [Google SRE - Being On-Call](https://sre.google/sre-book/being-on-call/)
- [PagerDuty Incident Response](https://response.pagerduty.com/)
- [Atlassian Incident Management](https://www.atlassian.com/incident-management)

### Video Resources
- [Adam Sitnik - Debugging .NET Memory Issues](https://www.youtube.com/watch?v=8cH4E9L0lrA)
- [Scott Hanselman - Debugging .NET](https://www.youtube.com/watch?v=n8pJWNdQKb0)

---

## Module Summary

You've completed the Performance, Scaling & Observability module. You can now:

- Profile and optimize .NET applications with data-driven decisions
- Design stateless systems that scale horizontally
- Implement comprehensive observability with OpenTelemetry
- Diagnose and resolve production issues systematically
- Write runbooks that enable faster incident response

**Remember**: The goal isn't to prevent all incidents — it's to detect them quickly, resolve them faster, and learn from every one.

---

**Next Module:** [Module 10 — Software Architecture Patterns](../10-architecture/README.md)
