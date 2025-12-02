# Self-Assessment: Where Should You Start?

Use this checklist to identify gaps and find your starting point. Be honest — checking boxes you can't actually do wastes your time.

**How to use:** For each statement, mark it only if you can do it **right now, without looking anything up**.

---

## Module 01 — C# Foundations

### Can you confidently...

- [ ] Explain when to use `struct` vs `class` vs `record` and why
- [ ] Predict whether code allocates on the stack or heap
- [ ] Use nullable reference types (`string?`) and handle nulls properly
- [ ] Write LINQ that translates to efficient SQL (not client-side evaluation)
- [ ] Explain why `async void` is dangerous
- [ ] Avoid deadlocks with `async/await` (no `.Result` or `.Wait()`)
- [ ] Use generics with constraints (`where T : class, new()`)
- [ ] Implement SOLID principles in real code, not just explain them

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here.** Weak C# fundamentals will slow everything else. |
| 4-6 | Skim Part 1-2, focus on Parts 3-4 |
| 7-8 | Skip to Module 02 |

---

## Module 02 — REST Fundamentals

### Can you confidently...

- [ ] Design a resource-based API (nouns, not verbs in URLs)
- [ ] Choose the correct HTTP method for any operation
- [ ] Explain idempotency and why PUT/DELETE should be idempotent
- [ ] Select appropriate status codes (when 400 vs 422 vs 404)
- [ ] Design error responses using ProblemDetails format
- [ ] Implement pagination, filtering, and sorting query parameters
- [ ] Explain versioning strategies and their tradeoffs
- [ ] Use HTTP caching headers (ETag, Cache-Control, If-None-Match)

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 01 |
| 4-6 | Skim the module, focus on Parts 3-4 |
| 7-8 | Skip to Module 03 |

---

## Module 03 — Building APIs

### Can you confidently...

- [ ] Explain the ASP.NET Core middleware pipeline order
- [ ] Configure dependency injection with correct lifetimes (scoped/transient/singleton)
- [ ] Build Minimal API endpoints with proper validation
- [ ] Implement global exception handling with ProblemDetails
- [ ] Set up structured logging with Serilog
- [ ] Implement JWT authentication and authorization policies
- [ ] Write integration tests using WebApplicationFactory
- [ ] Containerize an API with Docker

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Modules 01-02 |
| 4-6 | Focus on authentication and testing parts |
| 7-8 | Skip to Module 04 |

---

## Module 04 — EF Core & SQL

### Can you confidently...

- [ ] Design a normalized database schema with proper keys and indexes
- [ ] Write complex SQL queries (joins, window functions, CTEs)
- [ ] Explain transaction isolation levels and when to use each
- [ ] Configure EF Core relationships (1:1, 1:N, N:N) with Fluent API
- [ ] Avoid N+1 queries using Include or projections
- [ ] Write EF Core migrations and handle schema changes safely
- [ ] Use Dapper for performance-critical queries
- [ ] Read and optimize execution plans

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 03 |
| 4-6 | Focus on Parts 4-5 (advanced EF, Dapper) |
| 7-8 | Skip to Module 05 |

---

## Module 05 — NoSQL

### Can you confidently...

- [ ] Explain CAP theorem and its practical implications
- [ ] Choose between SQL, Redis, and MongoDB for a given use case
- [ ] Design MongoDB documents with proper schema patterns
- [ ] Build MongoDB aggregation pipelines
- [ ] Implement Redis caching with appropriate TTLs
- [ ] Use Redis for distributed locking
- [ ] Handle eventual consistency in application code

**Score: ___ / 7**

| Score | Recommendation |
|-------|----------------|
| 0-2 | **Start here** after Module 04 |
| 3-5 | Focus on the parts relevant to your stack |
| 6-7 | Skip to Module 06 |

---

## Module 06 — Messaging

### Can you confidently...

- [ ] Explain when to use sync vs async communication
- [ ] Differentiate commands vs events vs integration events
- [ ] Implement the outbox pattern for reliable messaging
- [ ] Handle poison messages and dead-letter queues
- [ ] Design idempotent message consumers
- [ ] Use RabbitMQ or Azure Service Bus
- [ ] Trace messages across services with correlation IDs

**Score: ___ / 7**

| Score | Recommendation |
|-------|----------------|
| 0-2 | **Start here** after Module 05 |
| 3-5 | Focus on outbox pattern and reliability |
| 6-7 | Skip to Module 07 |

---

## Module 07 — Testing

### Can you confidently...

- [ ] Design a testing strategy using the test pyramid
- [ ] Write unit tests for domain logic with proper assertions
- [ ] Know when to use mocks vs fakes vs stubs
- [ ] Build integration tests with WebApplicationFactory
- [ ] Use Testcontainers for database tests
- [ ] Write contract tests with Pact
- [ ] Debug flaky tests and fix them
- [ ] Achieve meaningful coverage (not just line coverage)

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 06 |
| 4-6 | Focus on integration and contract testing |
| 7-8 | Skip to Module 08 |

---

## Module 08 — Security

### Can you confidently...

- [ ] List the OWASP Top 10 and how they apply to APIs
- [ ] Implement JWT authentication with proper validation
- [ ] Design authorization policies (RBAC, claims-based)
- [ ] Implement resource-based authorization ("user owns this resource")
- [ ] Configure rate limiting
- [ ] Manage secrets securely (Key Vault, not appsettings.json)
- [ ] Conduct basic threat modeling for a feature

**Score: ___ / 7**

| Score | Recommendation |
|-------|----------------|
| 0-2 | **Start here** after Module 07 |
| 3-5 | Focus on authorization and secrets management |
| 6-7 | Skip to Module 09 |

---

## Module 09 — Performance

### Can you confidently...

- [ ] Profile an application to find CPU/memory bottlenecks
- [ ] Use BenchmarkDotNet to measure code performance
- [ ] Interpret GC stats and reduce allocations
- [ ] Diagnose and fix async deadlocks
- [ ] Design for horizontal scaling (stateless services)
- [ ] Implement OpenTelemetry tracing
- [ ] Build observability dashboards (Grafana, etc.)
- [ ] Load test with k6 or similar tools

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 08 |
| 4-6 | Focus on profiling and observability |
| 7-8 | Skip to Module 10 |

---

## Module 10 — Architecture

### Can you confidently...

- [ ] Explain Clean Architecture and its dependency rules
- [ ] Compare modular monolith vs microservices tradeoffs
- [ ] Write Architecture Decision Records (ADRs)
- [ ] Define bounded contexts and aggregate boundaries
- [ ] Implement domain events for cross-aggregate communication
- [ ] Design API gateway routing with YARP
- [ ] Implement saga patterns (orchestration or choreography)
- [ ] Apply resilience patterns (retry, circuit breaker, bulkhead)

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 09 |
| 4-6 | Focus on DDD and resilience patterns |
| 7-8 | Skip to Module 11 |

---

## Module 11 — DevOps

### Can you confidently...

- [ ] Design a Git branching strategy (trunk-based, GitFlow)
- [ ] Build CI/CD pipelines with GitHub Actions
- [ ] Write optimized Dockerfiles (multi-stage builds)
- [ ] Deploy to Azure Container Apps or Kubernetes
- [ ] Implement infrastructure as code (Bicep/Terraform)
- [ ] Configure health checks and readiness probes
- [ ] Manage database migrations in CI/CD
- [ ] Set up secret rotation automation

**Score: ___ / 8**

| Score | Recommendation |
|-------|----------------|
| 0-3 | **Start here** after Module 10 |
| 4-6 | Focus on containers and IaC |
| 7-8 | Skip to Module 12 |

---

## Module 12 — Professional Skills

### Can you confidently...

- [ ] Write clear PR descriptions that reviewers appreciate
- [ ] Give constructive code review feedback
- [ ] Use advanced debugger features (conditional breakpoints, tracepoints)
- [ ] Write documentation that others can actually follow
- [ ] Explain technical decisions to non-technical stakeholders
- [ ] Create runbooks for operational tasks

**Score: ___ / 6**

| Score | Recommendation |
|-------|----------------|
| 0-2 | Work on this **alongside** other modules |
| 3-4 | Focus on documentation and communication |
| 5-6 | Skip or use as reference |

---

## Module 13 — Legacy Code

### Can you confidently...

- [ ] Navigate an unfamiliar codebase systematically
- [ ] Read stack traces and identify root causes quickly
- [ ] Write characterization tests for undocumented code
- [ ] Find seams to isolate dependencies for testing
- [ ] Apply the strangler fig pattern for migrations
- [ ] Refactor safely using IDE tools and small commits

**Score: ___ / 6**

| Score | Recommendation |
|-------|----------------|
| 0-2 | **Essential** — do this after Module 07 |
| 3-4 | Focus on characterization tests and refactoring |
| 5-6 | Use as reference when needed |

---

## Your Starting Point

Add up where you scored lowest. That's likely where you'll get the most value.

**Common paths:**

| If you're... | Start with... |
|--------------|---------------|
| Fresh out of tutorials | Module 01 |
| Can code but APIs feel shaky | Module 02 |
| Built APIs but they're messy | Module 03 |
| Never really understood databases | Module 04 |
| Good at code, weak on operations | Module 11 |
| Senior but gaps in fundamentals | Wherever you scored lowest |

---

## Honest Self-Assessment Tips

1. **"I've used it" ≠ "I understand it"** — Using EF Core doesn't mean you can optimize queries
2. **Tutorial knowledge fades** — If you learned it 6 months ago and haven't used it, retest yourself
3. **Interview prep ≠ practical skill** — Knowing SOLID definitions isn't the same as applying them
4. **Gaps compound** — Weak Module 01 skills make everything else harder

When in doubt, start earlier. Moving fast through material you know is better than struggling with gaps.
