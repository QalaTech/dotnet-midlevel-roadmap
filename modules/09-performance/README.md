# 09 — Performance, Scaling & Observability

## Why This Module Exists

Performance isn't about making things fast — it's about making things **reliably fast under load**. The difference matters:

- **Fast in development**: 50ms response time with 1 user
- **Fast in production**: 50ms response time with 10,000 concurrent users

Most performance problems aren't discovered until production. By then:
- Users are frustrated
- Revenue is lost
- Emergency fixes are expensive
- Root cause is buried under layers of panic

This module teaches you to measure, scale, observe, and troubleshoot **before** the incident.

---

## The Performance Paradox

> "Premature optimization is the root of all evil" — Donald Knuth

But also:

> "We can't optimize what we don't measure" — Peter Drucker

The resolution: **Measure first, optimize strategically, verify with data.**

---

## What Goes Wrong Without This

### The Gradual Slowdown
```
Month 1: 80ms average response time
Month 3: 200ms (nobody notices)
Month 6: 800ms (users complain)
Month 9: 2000ms (users leave)
Month 12: "We need to rewrite everything"
```

### The Black Friday Incident
```
Traffic: 10x normal
Database: Connection pool exhausted
Cache: Memory pressure causes evictions
API: 503 errors everywhere
Revenue lost: $2.3 million in 4 hours
Post-mortem: "We never load tested"
```

### The Memory Leak That Took 3 Weeks to Find
```
Day 1: Deploy new feature
Day 7: Occasional slow responses (ignored)
Day 14: Daily restarts needed (workaround)
Day 21: Finally profile, find String concatenation in a loop
Fix: 5 minutes to implement StringBuilder
```

---

## What You'll Build

OrderFlow's performance infrastructure:

- **Performance Baselines** — Know what "normal" looks like
- **Load Testing Suite** — Prove you can handle expected traffic
- **Observability Stack** — OpenTelemetry, metrics, traces, dashboards
- **Troubleshooting Playbooks** — When things break at 3 AM

---

## Module Structure

### [Part 1 — Performance Foundations](./part-1.md)
- Profiling with real tools (BenchmarkDotNet, dotnet-trace, PerfView)
- Memory analysis and GC tuning
- Defining SLOs that matter
- The optimization loop

### [Part 2 — Scaling APIs Horizontally](./part-2.md)
- Stateless design patterns
- Load testing with k6
- Autoscaling strategies
- Database connection pooling

### [Part 3 — Observability Deep Dive](./part-3.md)
- OpenTelemetry setup
- Distributed tracing
- Metrics and alerting
- Dashboard design

### [Part 4 — Troubleshooting & Tuning Playbook](./part-4.md)
- Memory dump analysis
- Thread pool starvation
- SQL query tuning
- Incident response patterns

---

## Prerequisites

Before starting this module:
- Complete Module 03 (Building APIs) — have a working API to measure
- Complete Module 04 (EF Core) — understand database access patterns
- Complete Module 07 (Testing) — integration tests provide load test foundations

---

## Key Concepts You'll Master

1. **Measure Before Optimizing** — Data-driven decisions, not guesses
2. **SLOs/SLIs/SLAs** — Define "good enough" with numbers
3. **The Four Golden Signals** — Latency, traffic, errors, saturation
4. **Observability vs Monitoring** — Understanding vs alerting

---

## Tools You'll Use

| Category | Tools |
|----------|-------|
| **Profiling** | BenchmarkDotNet, dotnet-trace, PerfView, JetBrains dotMemory |
| **Load Testing** | k6, Locust, Azure Load Testing |
| **Observability** | OpenTelemetry, Prometheus, Grafana, Jaeger |
| **APM** | Application Insights, Datadog, New Relic |
| **Diagnostics** | dotnet-dump, dotnet-gcdump, SOS debugger |

---

## Exit Criteria

You can:
1. Profile an application and identify the actual bottleneck
2. Run load tests that simulate production traffic patterns
3. Implement distributed tracing across services
4. Build dashboards that answer "is the system healthy?"
5. Diagnose common production issues from dumps and logs
6. Write runbooks that others can follow during incidents

---

## Resources

### Essential Reading
- [.NET Performance Best Practices](https://docs.microsoft.com/en-us/dotnet/framework/performance/performance-tips)
- [Writing High-Performance .NET Code](https://www.writinghighperf.net/) by Ben Watson
- [Site Reliability Engineering](https://sre.google/sre-book/table-of-contents/) (free online)

### Video Resources
- [Adam Sitnik - Performance in .NET](https://www.youtube.com/watch?v=4yALYEINbyI)
- [Stephen Toub - Performance Improvements in .NET](https://www.youtube.com/watch?v=8w4NkMwF9Qc)
- [Grafana k6 - Getting Started](https://www.youtube.com/watch?v=Hu1K63IOb1o)

### Tools Documentation
- [BenchmarkDotNet](https://benchmarkdotnet.org/articles/overview.html)
- [OpenTelemetry .NET](https://opentelemetry.io/docs/languages/net/)
- [k6 Documentation](https://k6.io/docs/)

---

**Start:** [Part 1 — Performance Foundations](./part-1.md)
