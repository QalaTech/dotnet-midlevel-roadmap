# Full .NET Backend Engineer Roadmap (Master Outline)

This outline drives a motivated junior .NET backend engineer to solid mid-level competence over twelve months. Each module targets a real competency area, is broken into digestible parts, and ends with deliverables that prove the skill exists. Expect to spend 2–3 focused weeks per part; ship code, document lessons, and request feedback relentlessly.

## How to Use This Roadmap
- Move sequentially; every module builds upon the prior foundations.
- Treat each part like a sprint: set learning goals on day one, deliver artifacts by day ten, retro your gaps on day fourteen.
- Maintain a learning repository with notes, sample services, benchmarks, ADRs, and tests so mentors can review evidence.
- Pair theory with production-grade practice—test, benchmark, deploy, and observe every artifact you produce.

---

## Module 01 — Deep C# & .NET Foundations (4 parts)
**Outcome:** Write idiomatic, performant, testable C# that teammates can trust.  
**Recommended duration:** 5–6 weeks.

### Part 1 — Types, Memory, Collections
**Focus:** Value vs reference semantics, string behavior, memory layout, core collections, nullable reference types, generics.
**Key competencies**
- Explain stack vs heap allocations and justify when structs or classes fit.
- Control allocations with `Span<T>`, pools, and `StringBuilder`.
- Select the right collection (`List`, `Dictionary`, `ImmutableArray`, `ConcurrentDictionary`) for throughput and safety.
- Configure nullable reference types and generics constraints to encode intent.
**Hands-on practice**
- Instrument sample code with BenchmarkDotNet to compare struct/class, boxing, and string scenarios.
- Build a tiny in-memory repository that surfaces nullability warnings and generics constraints.
- Produce a “collections cookbook” that maps scenarios to optimal data structures.
**Checkpoint**
- Deliver a two-page memo plus benchmark results showing how a poor type choice harmed performance and how you fixed it.

### Part 2 — Language Expressiveness
**Focus:** Records, immutability, exception design, expression trees, delegates, lambdas.
**Key competencies**
- Model domain concepts with records and immutable types, enforcing invariants through constructors and factory methods.
- Design exception hierarchies, favor guard clauses, and surface domain faults via typed responses.
- Compose delegates (`Func`, `Action`) and expression trees for dynamic filtering or rule engines.
- Build reusable extension methods that remain discoverable and testable.
**Hands-on practice**
- Implement a rule-based discount engine that mixes expression trees and delegates; unit-test every rule.
- Harden an API’s exception strategy by surfacing domain exceptions through ProblemDetails middleware.
**Checkpoint**
- Record a short demo of the discount engine showing expression tree inspection and custom exceptions in action.

### Part 3 — Abstractions, SOLID, Patterns
**Focus:** Interfaces, dependency inversion, SOLID, Strategy/Decorator/Factory patterns, reflection and metadata.
**Key competencies**
- Refactor feature code to obey SRP/ISP, choosing abstraction boundaries intentionally.
- Implement Strategy, Decorator, and Factory patterns to remove conditional complexity.
- Use reflection responsibly for plug-in loading, metadata-driven validation, or generic pipelines.
- Explain SOLID tradeoffs in code reviews and ADRs.
**Hands-on practice**
- Convert a monolithic service into composable command handlers using Strategy/Decorator for validation, caching, logging.
- Build a reflection-driven configuration loader that maps attributes to behaviors.
**Checkpoint**
- Submit a PR (against your learning repo) showing before/after diagrams plus tests covering the new abstractions.

### Part 4 — Advanced Runtime Mastery
**Focus:** Spans, async internals, streams, coding standards, mastery exercises.
**Key competencies**
- Use `Span<T>`, pooled buffers, and channels to process data without allocations nor blocking.
- Diagnose async deadlocks, configure `ValueTask`, and avoid sync-over-async.
- Follow `.editorconfig` and analyzers to keep codebases coherent.
- Profile end-to-end to prove performance claims.
**Hands-on practice**
- Build a file-ingestion worker that streams data, parses records via spans, and honors cancellation tokens.
- Create a “C# kata suite” mixing async tests, analyzers, and benchmarking.
**Checkpoint**
- Publish flame graphs comparing naive vs optimized ingestion in your README plus the supporting code.

**Suggested resources:** _C# in Depth_, _Pro .NET Memory Management_, Microsoft Learn C# DDD path, BenchmarkDotNet docs.

---

## Module 02 — HTTP & REST Fundamentals (4 parts)
**Outcome:** Design HTTP APIs with clear semantics, correct status codes, and predictable contracts.  
**Recommended duration:** 4 weeks.

### Part 1 — REST Fundamentals: HTTP, Verbs, Idempotency
**Focus:** HTTP request lifecycle, verb semantics, resource modeling, idempotency.
**Key competencies**
- Trace raw HTTP requests/responses, explaining what each header influences.
- Map operations into resource-based endpoints rather than RPC verbs.
- Distinguish safety, idempotency, and mutability for every verb you expose.
- Document canonical URIs, naming conventions, and relationships.
**Hands-on practice**
- Use curl/Fiddler to craft raw requests, verifying headers, verbs, and retry safety.
- Model a sample domain (e.g., commerce) into a resource tree, highlighting idempotent operations.
- Build Postman/Newman tests that replay requests to prove idempotent behavior.
**Checkpoint**
- Publish an API design doc (Markdown + diagrams) describing resources, verbs, and idempotency justification.

### Part 2 — Status Codes, Errors, Filtering
**Focus:** HTTP status codes, error semantics, filtering, sorting, pagination patterns.
**Key competencies**
- Return accurate status codes and differentiate 4xx vs 5xx boundaries.
- Provide consistent error envelopes (ProblemDetails) with correlation IDs.
- Implement filtering/pagination query objects with validation and default ordering.
- Document sorting/filter shapes so clients can reason about discoverability.
**Hands-on practice**
- Build middleware that translates exceptions into ProblemDetails responses.
- Add filtering/sorting/pagination to a sample endpoint using strongly typed value objects.
- Create integration tests covering success, validation, and error scenarios.
**Checkpoint**
- Share a Newman test run plus logs showing coverage for every status code and filter combination.

### Part 3 — Relationships, Versioning, Caching
**Focus:** Representing relationships, versioning strategies, caching, conditional requests.
**Key competencies**
- Choose between nested routes, linking, and embedding when expressing relationships.
- Evaluate URL, query, header, and media-type versioning with backward-compatibility plans.
- Implement HTTP caching headers (Cache-Control, ETag, Last-Modified) and conditional requests.
- Measure cache hit ratios and stale data tolerances.
**Hands-on practice**
- Add `ETag`/`If-None-Match` and `If-Unmodified-Since` handling to GET/PUT endpoints.
- Document and implement a versioning strategy (e.g., header-based) in the sample API.
- Provide discovery links (HATEOAS or explicit URIs) to related resources.
**Checkpoint**
- Demonstrate reduced DB calls through caching metrics and document the versioning decision record.

### Part 4 — API Contracts, Rate Limiting, Longevity
**Focus:** API contracts, rate limiting, idempotency keys, long-term design and governance.
**Key competencies**
- Treat OpenAPI/JSON Schema as the single source of truth, linting for consistency.
- Automate contract tests to detect breaking changes.
- Apply rate limiting, quotas, and backoff strategies aligned with client SLAs.
- Store and validate idempotency keys for operations that must not double-execute.
**Hands-on practice**
- Generate and hand-curate OpenAPI definitions; enforce linting in CI.
- Implement ASP.NET Core rate limiting middleware or integrate API Management throttling.
- Build an `IdempotencyKeys` table with TTL plus middleware enforcing uniqueness.
**Checkpoint**
- Run contract tests (Newman, Pact, or Dredd) in CI and attach the report plus rate-limiter metrics.

**Suggested resources:** Microsoft REST API Guidelines, RFC 7231/7232 digest, Stoplight or Spectral OpenAPI linters, API University blog.

---

## Module 03 — Minimal APIs & Web API Architecture (4 parts)
**Outcome:** Build production-grade ASP.NET Core Minimal APIs with structure, validation, security, and deployments.  
**Recommended duration:** 5 weeks.

### Part 1 — Pipeline, DI, Validation, Structure
**Focus:** ASP.NET Core hosting pipeline, dependency injection lifetimes, validation strategies, project organization.
**Key competencies**
- Describe the order of middleware, endpoint filters, and endpoint execution.
- Configure scoped/transient/singleton services safely; avoid captured DbContexts.
- Apply validation via FluentValidation or endpoint filters, returning actionable errors.
- Structure features using endpoint groups, extension methods, and dedicated folders per vertical slice.
**Hands-on practice**
- Build a `Products` API leveraging minimal APIs, grouping endpoints by feature.
- Extract DI registrations into extension methods and integrate configuration binding.
- Implement validation filters that short-circuit invalid requests.
**Checkpoint**
- Share architecture diagrams plus automated tests verifying validation and DI misconfiguration detection.

### Part 2 — Middleware, Errors, Logging, Auth
**Focus:** Custom middleware, error handling, structured logging, authentication.
**Key competencies**
- Build reusable middleware for correlation IDs, exception handling, and metrics.
- Apply Serilog or Seq for structured logging, enriching context along the pipeline.
- Implement JWT or cookie authentication; understand token validation lifecycle.
- Leverage authorization handlers/policies even in minimal APIs.
**Hands-on practice**
- Create an exception-handling middleware emitting ProblemDetails plus correlation IDs.
- Configure Serilog sinks (console/file/Seq) and demonstrate log events during integration tests.
- Add JWT Bearer authentication and protect specific endpoints with policies.
**Checkpoint**
- Provide log samples showing correlation IDs from request to response plus tests covering secured endpoints.

### Part 3 — Data Access, Background Jobs, EF Integration
**Focus:** Querying, pagination, filtering, background processing, EF Core integration.
**Key competencies**
- Compose EF Core queries using `IQueryable`, projections, and asynchronous execution.
- Accept query parameters, convert to specification/predicate builder, and apply pagination defaults.
- Build background jobs using `IHostedService`, `BackgroundService`, or Quartz/Hangfire.
- Manage DbContext lifetime across HTTP and worker boundaries.
**Hands-on practice**
- Implement advanced GET endpoints supporting filtering, search, and pagination with query DTOs.
- Write a background job that syncs data or sends notifications, persisting job state.
- Profile SQL generated by queries; refactor to projections when necessary.
**Checkpoint**
- Capture SQL logs demonstrating efficient pagination and include worker tests proving safe DbContext usage.

### Part 4 — Architecture, Testing, Observability, Deployment
**Focus:** Clean architecture alignment, vertical slices, automated testing, telemetry, and deployment.
**Key competencies**
- Apply Clean Architecture/vertical slice design to keep minimal APIs maintainable.
- Use `WebApplicationFactory`/`TestServer` for integration tests and request pipelines.
- Emit metrics/traces using OpenTelemetry, integrate health checks, design dashboards.
- Containerize the API and deploy to Azure App Service or Container Apps.
**Hands-on practice**
- Refactor endpoints into vertical slices using MediatR or lightweight handlers.
- Add `WebApplicationFactory` tests verifying happy paths and failure cases.
- Instrument the app with OpenTelemetry exporting to Jaeger or Azure Monitor.
- Create a Dockerfile, compose file, and GitHub Actions workflow to deploy.
**Checkpoint**
- Provide deployment notes, health-check URLs, and test results gating your CI pipeline.

**Suggested resources:** ASP.NET Core docs, Nick Chapsas Minimal API playlist, Steve Gordon’s articles on middleware/hosted services, .NET Aspire samples.

---

## Module 04 — EF Core, SQL, Dapper & Database Design (5 parts)
**Outcome:** Own relational modeling, query tuning, ORM usage, and raw SQL optimizations.  
**Recommended duration:** 6–7 weeks.
**Detailed files:** [Module overview](../modules/04-ef-core/README.md) | [Part 1](../modules/04-ef-core/part-1.md) | [Part 2](../modules/04-ef-core/part-2.md) | [Part 3](../modules/04-ef-core/part-3.md) | [Part 4](../modules/04-ef-core/part-4.md) | [Part 5](../modules/04-ef-core/part-5.md)

### Part 1 — Database Foundations: Modeling & Indexing
**Focus:** Relational modeling, keys, constraints, indexing strategies, normalization vs denormalization.
**Key competencies**
- Design ER diagrams with primary/foreign keys and enforce referential integrity.
- Choose clustered vs nonclustered indexes, covering indexes, and filtered indexes.
- Balance normalization with denormalization for read models and reporting tables.
- Document naming conventions, schemas, and migration processes.
**Hands-on practice**
- Model a transactional domain (orders/inventory) in a SQL database with migrations.
- Create indexes, examine execution plans, and measure impact on query performance.
- Simulate denormalized read models to support dashboards.
**Checkpoint**
- Deliver ER diagrams plus SQL scripts showcasing indexes and before/after execution plans.

### Part 2 — SQL Mastery: Querying & Concurrency
**Focus:** SELECT mastery, joins, window functions, aggregates, transactions, locking.
**Key competencies**
- Write complex joins (inner/left/right/cross) and explain their cardinality.
- Use window functions (`ROW_NUMBER`, `LEAD`, `LAG`) for analytics scenarios.
- Manage transactions with appropriate isolation levels and detect lock contention.
- Use CTEs, temp tables, and query hints judiciously.
**Hands-on practice**
- Solve 10+ SQL challenges including ranking, running totals, and gap detection.
- Benchmark different isolation levels under concurrent load using a console app.
- Capture deadlock graphs and propose fixes.
**Checkpoint**
- Publish a notebook or markdown file containing each query, execution plan, and learned tradeoffs.

### Part 3 — EF Core Fundamentals
**Focus:** DbContext lifecycle, LINQ translation, change tracking, relationships (1:1, 1:N, N:N).
**Key competencies**
- Configure DbContext lifetimes, connection resiliency, and logging.
- Predict how LINQ translates to SQL; avoid client-side evaluation.
- Model relationships with fluent API, navigations, owned entities.
- Track entity states, attach/detach, and handle concurrency tokens.
**Hands-on practice**
- Build a repository-free service using DbContext directly and demonstrate proper scopes.
- Log generated SQL, identify N+1 queries, and fix them with Includes or projections.
- Model all relationship types in sample domain with tests covering cascade behavior.
**Checkpoint**
- Ship integration tests using `SqliteInMemory` or Testcontainers verifying relationship mapping and change tracker behavior.

### Part 4 — EF Core Advanced Topics
**Focus:** Migrations, projections vs entities, no-tracking queries, interceptors, owned types, value conversions.
**Key competencies**
- Manage migrations across environments, handle destructive changes safely.
- Use `Select` projections to return DTOs without tracking when writes are unnecessary.
- Implement interceptors for auditing, soft deletes, or multi-tenancy.
- Map owned types and value conversions to harmonize domain models and persistence.
**Hands-on practice**
- Add projection-only queries for read models and measure performance vs tracked entities.
- Create an interceptor that logs slow queries or enforces soft-delete filters.
- Handle large schema change (e.g., splitting table) through staged migrations.
**Checkpoint**
- Document migration strategy plus demonstrate interceptor logs capturing targeted behavior.

### Part 5 — Dapper & Raw SQL
**Focus:** When to use Dapper, parameterized queries, multi-mapping, handling massive datasets, hybrid EF + Dapper patterns.
**Key competencies**
- Choose Dapper for hot paths requiring raw SQL control or stored procedure access.
- Secure parameterized queries to avoid SQL injection.
- Use multi-mapping to hydrate object graphs and paginated streaming for large datasets.
- Blend EF Core for CRUD with Dapper for reporting without transactional drift.
**Hands-on practice**
- Replace a high-volume read endpoint with Dapper, measuring latency improvements.
- Implement multi-mapping for aggregates (e.g., order + line items) using buffered/unbuffered queries.
- Stream CSV exports via Dapper + `IDataReader`.
**Checkpoint**
- Provide benchmark results comparing EF vs Dapper for the same query plus architecture notes on when you switch.

**Suggested resources:** Microsoft Learn SQL path, `SQL Performance Explained`, EF Core docs, Dapper repo samples, Brent Ozar blog.

---

## Module 05 — NoSQL & Distributed Data (3 parts)
**Outcome:** Understand when polyglot persistence is appropriate and operate MongoDB/Redis effectively.  
**Recommended duration:** 3–4 weeks.
**Detailed files:** [Module overview](../modules/05-nosql/README.md) | [Part 1](../modules/05-nosql/part-1.md) | [Part 2](../modules/05-nosql/part-2.md) | [Part 3](../modules/05-nosql/part-3.md)

### Part 1 — NoSQL Foundations
**Focus:** Key-value/document stores, CAP theorem, consistency vs availability.
**Key competencies**
- Describe CAP tradeoffs and choose datastores based on consistency, latency, and schema needs.
- Differentiate document, column, key-value, and graph stores.
- Model data with eventual consistency in mind (conflict resolution, idempotent updates).
**Hands-on practice**
- Design a read-heavy feature and decide whether relational or NoSQL fits best; justify metrics.
- Simulate eventual consistency by writing conflict resolution tests.
**Checkpoint**
- Produce an ADR comparing SQL vs NoSQL for a feature, including CAP rationale and data modeling sketches.

### Part 2 — Document DB (MongoDB)
**Focus:** Collections, flexible schema design, indexing, aggregation pipelines.
**Key competencies**
- Model documents with nested arrays, references, or bucketing strategies.
- Create compound/TTL/text indexes and inspect explain plans.
- Build aggregation pipelines for reporting.
- Use MongoDB transactions when necessary.
**Hands-on practice**
- Create a Mongo-backed audit log for your API, including retention policies.
- Build aggregation queries for metrics dashboards; expose them via minimal API.
**Checkpoint**
- Share pipeline samples plus explain-plan screenshots proving indexes are used.

### Part 3 — Caching & Redis
**Focus:** Redis basics, TTL, distributed locking, pub/sub, when to cache vs store.
**Key competencies**
- Choose caching strategies (cache-aside, write-through, write-behind).
- Configure Redis data structures (strings, hashes, sets, sorted sets).
- Implement distributed locks and TTL policies; avoid dog-piling.
- Decide when Redis should augment, not replace, the primary database.
**Hands-on practice**
- Add Redis caching to hot GET endpoints with instrumentation showing hit rates.
- Implement distributed locking for a critical section (e.g., inventory hold).
- Use pub/sub or streams to broadcast domain events.
**Checkpoint**
- Provide Grafana/console metrics showing cache hit rate, eviction trends, and lock health.

**Suggested resources:** MongoDB University Developer path, Redis University RU101, Azure Cache for Redis samples.

---

## Module 06 — Messaging, Queues & Event-Driven Systems (4 parts)
**Outcome:** Build resilient asynchronous workflows mixing queues, events, and workers.  
**Recommended duration:** 4–5 weeks.
**Detailed files:** [Module overview](../modules/06-messaging/README.md) | [Part 1](../modules/06-messaging/part-1.md) | [Part 2](../modules/06-messaging/part-2.md) | [Part 3](../modules/06-messaging/part-3.md) | [Part 4](../modules/06-messaging/part-4.md)

### Part 1 — Messaging Foundations
**Focus:** Event-driven architecture, commands vs events, ordering, idempotency essentials.
**Key competencies**
- Distinguish synchronous request/response vs async messaging.
- Define commands, events, integration events, and when each applies.
- Explain ordering guarantees, partitions, and deduplication strategies.
- Design idempotent consumers that handle retries gracefully.
**Hands-on practice**
- Draw sequence diagrams for command vs event flows in your sample system.
- Implement an in-memory message bus abstraction and prove idempotency via tests.
**Checkpoint**
- Submit diagrams plus tests that simulate duplicate message delivery without side effects.

### Part 2 — Queues & Delivery Guarantees
**Focus:** Azure Queue/Service Bus/RabbitMQ basics, dead-letter queues, retry strategies, poison message handling.
**Key competencies**
- Configure queue entities, message TTLs, and peek-lock semantics.
- Implement exponential backoff and poison message handling.
- Monitor DLQ counts and surface alerts.
- Secure queue access with shared keys or managed identities.
**Hands-on practice**
- Use Azure Storage Queue or RabbitMQ locally to enqueue work from the API and process via worker service.
- Configure DLQ routing and dashboards.
**Checkpoint**
- Demonstrate, via logs/tests, that poison messages land in DLQ and healthy messages eventually succeed.

### Part 3 — Integration Events & Outbox
**Focus:** Outbox pattern, at-least-once delivery, deduplication.
**Key competencies**
- Persist integration events in the same transaction as the aggregate.
- Build an outbox worker that publishes to the queue/bus, marking completion atomically.
- Apply deduplication keys to ensure consumers handle at-least-once semantics.
**Hands-on practice**
- Add an outbox table to your sample app and implement a worker draining it to Service Bus or RabbitMQ.
- Write consumer logic that ignores duplicates based on message IDs.
**Checkpoint**
- Provide test logs showing event persistence, retry, and deduplication working end-to-end.

### Part 4 — Real Event Systems
**Focus:** Building consumer workers, scaling, distributed tracing for events.
**Key competencies**
- Use worker services or Functions to scale out consumers horizontally.
- Trace message processing with OpenTelemetry spans.
- Monitor lag, throughput, and failure rates.
- Apply partitioning/sharding strategies for scale.
**Hands-on practice**
- Deploy a worker service (container or Azure Function) that handles real queue load.
- Instrument processing time metrics and distributed traces across API → outbox → bus → worker.
**Checkpoint**
- Publish dashboards showing lag, throughput, and traces linking HTTP requests to downstream event processing.

**Suggested resources:** Microsoft Messaging patterns e-book, Jimmy Bogard’s posts on outbox/inbox, Azure Service Bus docs, RabbitMQ tutorials.

---

## Module 07 — Testing Mastery (4 parts)
**Outcome:** Confidently test from unit to contract level with reliable automation.  
**Recommended duration:** 4 weeks.
**Detailed files:** [Module overview](../modules/07-testing/README.md) | [Part 1](../modules/07-testing/part-1.md) | [Part 2](../modules/07-testing/part-2.md) | [Part 3](../modules/07-testing/part-3.md) | [Part 4](../modules/07-testing/part-4.md)

### Part 1 — Testing Foundations
**Focus:** What to test, test pyramid, mocks vs fakes vs stubs, TDD mindset.
**Key competencies**
- Describe the test pyramid and choose the right level for each behavior.
- Use mocks/fakes/stubs appropriately; avoid brittle tests.
- Structure projects (test naming, fixtures, data builders).
- Track coverage metrics and know when coverage isn’t the goal.
**Hands-on practice**
- Audit an existing project’s tests, classify by level, and plan improvements.
- Build helper libraries (AutoFixture builders, Bogus data) to speed test writing.
**Checkpoint**
- Publish a testing strategy doc covering pyramid distribution, tooling, and conventions.

### Part 2 — Unit Testing Domain & Application
**Focus:** Commands/queries, domain events, value objects.
**Key competencies**
- Test domain invariants without hitting the database.
- Use CQRS command/query handlers to isolate business rules.
- Inspect domain events and verify side effects.
**Hands-on practice**
- Write unit tests for aggregates/value objects with guard clauses.
- Use MediatR testable handlers or pipeline behaviors with mocks/fakes.
**Checkpoint**
- Provide coverage reports and highlight at least three business rules codified via tests.

### Part 3 — Integration Testing Minimal APIs
**Focus:** `WebApplicationFactory`, in-memory DBs, Testcontainers.
**Key competencies**
- Spin up the API in memory and issue HTTP requests from tests.
- Swap infrastructure (DB, queues) with Dockerized dependencies or Sqlite.
- Seed data per test and tear down cleanly.
**Hands-on practice**
- Create integration test suite hitting endpoints via `HttpClient`.
- Use Testcontainers to boot SQL/Redis locally during tests.
**Checkpoint**
- Provide CI logs showing integration tests running in headless mode with container lifecycle.

### Part 4 — End-to-End & Contract Testing
**Focus:** API contract testing, Postman + Newman, Pact for consumer-driven contracts.
**Key competencies**
- Automate smoke/E2E flows using REST clients or Playwright + API.
- Generate and validate contracts against OpenAPI.
- Build Pact tests for cross-team integration.
**Hands-on practice**
- Script Postman/Newman runs for happy-path flows plus failure cases.
- Implement Pact tests between your API and a fake consumer or provider.
**Checkpoint**
- Archive contract test artifacts and integrate them into CI gating rules.

**Suggested resources:** xUnit.net docs, `Testing .NET Core applications` (Pluralsight), Testcontainers-dotnet examples, Pact docs.

---

## Module 08 — Security, Auth & Identity (4 parts)
**Outcome:** Ship APIs with secure defaults, hardened authentication/authorization, and secret hygiene.  
**Recommended duration:** 4 weeks.
**Detailed files:** [Module overview](../modules/08-security/README.md) | [Part 1](../modules/08-security/part-1.md) | [Part 2](../modules/08-security/part-2.md) | [Part 3](../modules/08-security/part-3.md) | [Part 4](../modules/08-security/part-4.md)

### Part 1 — Security Foundations
**Focus:** OWASP Top 10, input sanitization, secure defaults.
**Key competencies**
- Map OWASP Top 10 to concrete API threats.
- Sanitize/validate input, encode output, and avoid overexposing data.
- Configure HTTPS everywhere, HSTS, and TLS settings.
- Threat model features before coding.
**Hands-on practice**
- Run OWASP ZAP or Burp against your sample API and patch findings.
- Add centralized validation and escaping helpers for user-generated content.
**Checkpoint**
- Produce a threat model plus remediation checklist for your API.

### Part 2 — Authentication
**Focus:** JWT, cookie-based auth, OAuth2/OpenID Connect.
**Key competencies**
- Implement ASP.NET Core Identity or external IdP integration.
- Configure JWT validation (issuer, audience, signing keys) and rotation.
- Understand OAuth2 flows (auth code, client credentials) and when to apply each.
**Hands-on practice**
- Use Azure AD B2C or Auth0 to secure the API with OAuth2/OIDC.
- Build a simple SPA or Postman collection to demonstrate auth flows.
**Checkpoint**
- Capture sequence diagrams for login/token refresh and include logs proving token validation.

### Part 3 — Authorization
**Focus:** Claims, policies, roles, resource-based authorization.
**Key competencies**
- Author policies using requirements/handlers.
- Enforce RBAC and ABAC models, factoring tenant or ownership rules.
- Build resource-based checks for row-level security.
**Hands-on practice**
- Create policies for admin vs contributor vs reader plus custom requirement for “owns resource”.
- Write tests hitting endpoints with different claims/roles ensuring enforcement.
**Checkpoint**
- Document policy definitions and include failing tests for unauthorized users.

### Part 4 — API Security Operations
**Focus:** Rate limits, API keys, RBAC beyond code, secrets management.
**Key competencies**
- Configure rate limits (per IP, per user) with graceful responses.
- Issue and rotate API keys securely; store secrets in Key Vault or similar.
- Automate secret scanning and rotation.
**Hands-on practice**
- Add ASP.NET rate-limiting middleware plus caching to store counts.
- Integrate Azure Key Vault or AWS Secrets Manager for config.
- Set up GitHub Advanced Security/TruffleHog scanning.
**Checkpoint**
- Provide documentation on secret rotation cadence and rate-limiter dashboards.

**Suggested resources:** OWASP Cheat Sheets, Microsoft Identity platform docs, Auth0 blog, Azure Key Vault samples.

---

## Module 09 — Performance, Scaling & Observability (4 parts)
**Outcome:** Diagnose bottlenecks, scale APIs, and run them with instrumentation.  
**Recommended duration:** 4 weeks.
**Detailed files:** [Module overview](../modules/09-performance/README.md) | [Part 1](../modules/09-performance/part-1.md) | [Part 2](../modules/09-performance/part-2.md) | [Part 3](../modules/09-performance/part-3.md) | [Part 4](../modules/09-performance/part-4.md)

### Part 1 — Performance Foundations
**Focus:** CPU vs memory bottlenecks, latency vs throughput, async correctness.
**Key competencies**
- Use PerfView/dotnet-trace/BenchmarkDotNet for profiling.
- Interpret GC stats, thread pool behavior, async state machines.
- Set latency budgets and throughput targets for endpoints.
**Hands-on practice**
- Profile a slow endpoint, identify CPU/memory hotspots, apply fixes.
- Build dashboards tracking latency percentiles and throughput.
**Checkpoint**
- Share before/after metrics plus profiling artifacts.

### Part 2 — Scaling APIs
**Focus:** Horizontal scale, load balancing, statelessness considerations.
**Key competencies**
- Design services to run behind load balancers with sticky vs non-sticky sessions.
- Externalize state (distributed cache, DB) to keep instances stateless.
- Apply autoscaling rules (CPU, queue length) and load testing.
**Hands-on practice**
- Deploy multiple API instances behind Azure Front Door or Nginx.
- Load-test with k6 or Locust, capturing scaling metrics.
**Checkpoint**
- Document scaling plan plus k6 reports capturing throughput and error rates.

### Part 3 — Observability Deep Dive
**Focus:** OpenTelemetry, trace IDs, metrics, Prometheus/Grafana.
**Key competencies**
- Instrument traces, logs, and metrics with correlation IDs.
- Export telemetry to Jaeger, Zipkin, Application Insights, or Prometheus.
- Build Grafana dashboards visualizing key SLIs/SLOs.
**Hands-on practice**
- Wire OpenTelemetry to emit traces across API, worker, database.
- Capture business metrics (orders processed) in addition to technical metrics.
**Checkpoint**
- Provide dashboard screenshots plus trace samples that link HTTP requests to downstream dependencies.

### Part 4 — Troubleshooting & Tuning
**Focus:** Deadlocks, DB bottlenecks, index tuning, production debugging.
**Key competencies**
- Recognize thread pool starvation and async deadlocks.
- Diagnose SQL locking/blocking and tune indexes accordingly.
- Use dotnet-dump, SOS, or production profiler captures to debug.
**Hands-on practice**
- Intentionally cause a deadlock and resolve it by adjusting transaction scopes.
- Capture `sp_WhoIsActive` or Azure SQL insights to analyze blocking.
- Build runbooks for high CPU, high latency, or DB saturation incidents.
**Checkpoint**
- Publish a troubleshooting playbook with reproduction steps and mitigations for at least three real issues.

**Suggested resources:** .NET Performance Fundamentals (Channel 9), Azure Monitor docs, k6 load-testing guide, SQLSkills indexing courses.

---

## Module 10 — Architecture Patterns & Distributed Systems (4 parts)
**Outcome:** Design systems across architectural styles and reason about tradeoffs.  
**Recommended duration:** 5 weeks.
**Detailed files:** [Module overview](../modules/10-architecture/README.md) | [Part 1](../modules/10-architecture/part-1.md) | [Part 2](../modules/10-architecture/part-2.md) | [Part 3](../modules/10-architecture/part-3.md) | [Part 4](../modules/10-architecture/part-4.md)

### Part 1 — Architecture Patterns
**Focus:** Clean Architecture, Hexagonal, Vertical Slice, Modular Monolith.
**Key competencies**
- Identify coupling seams and enforce dependency rules.
- Decide when modular monolith beats microservices for a given context.
- Document architecture decisions with ADRs.
**Hands-on practice**
- Refactor the sample solution into a modular monolith with feature modules and bounded contexts.
- Present comparisons between MVC controllers vs vertical slice endpoints.
**Checkpoint**
- Produce diagrams and ADRs defending the chosen architecture.

### Part 2 — Domain-Driven Design
**Focus:** Aggregates, entities, value objects, repositories, domain services.
**Key competencies**
- Define bounded contexts and shared kernels.
- Model aggregates with clear invariants and transactional boundaries.
- Implement repositories that respect aggregate rules.
- Use domain services for cross-aggregate logic.
**Hands-on practice**
- Model a complex domain (e.g., lending) with aggregates and publish domain events.
- Conduct an event-storming workshop (even solo) and document outcomes.
**Checkpoint**
- Publish domain model diagrams plus sample code showing aggregates enforcing invariants.

### Part 3 — Microservices Strategy
**Focus:** When to split, API gateways, service boundaries, saga patterns.
**Key competencies**
- Recognize the cost of distributed systems and only split with justification.
- Design contract boundaries and handshake patterns between services.
- Implement API gateway routing, composition, and aggregation.
- Describe saga choreography vs orchestration for long-running workflows.
**Hands-on practice**
- Split part of the monolith into a separate service, exposing APIs through YARP or Azure API Management.
- Prototype a saga orchestrator for order fulfillment using messages.
**Checkpoint**
- Document service boundaries, gateway configuration, and saga sequence diagrams.

### Part 4 — Resilience Patterns
**Focus:** Retry, circuit breakers, bulkheads, timeouts.
**Key competencies**
- Apply Polly or Resilience Pipelines for retries/circuit breakers.
- Set timeouts appropriate to dependencies.
- Partition resources (bulkheads) to prevent cascading failures.
- Monitor resilience metrics to detect thrashing.
**Hands-on practice**
- Wrap outbound HTTP/DB calls with Polly policies and expose metrics.
- Simulate downstream outages and observe circuit breaker behavior.
**Checkpoint**
- Share logs/metrics proving retry/circuit breaker/bulkhead effectiveness plus documented settings.

**Suggested resources:** _Implementing Domain-Driven Design_, Microsoft Architecture e-books, Polly documentation, YARP samples.

---

## Module 11 — DevOps, CI/CD & Cloud Fundamentals (4 parts)
**Outcome:** Deliver features through reliable pipelines, containerize services, and manage infrastructure as code.  
**Recommended duration:** 5 weeks.
**Detailed files:** [Module overview](../modules/11-devops/README.md) | [Part 1](../modules/11-devops/part-1.md) | [Part 2](../modules/11-devops/part-2.md) | [Part 3](../modules/11-devops/part-3.md) | [Part 4](../modules/11-devops/part-4.md)

### Part 1 — DevOps Foundations
**Focus:** Source control discipline, Git branching, environments.
**Key competencies**
- Follow trunk-based or GitFlow workflows and keep branches short-lived.
- Automate commit hooks, formatting, and analyzers.
- Define environment promotion strategies (dev, test, staging, prod).
**Hands-on practice**
- Configure branch protection rules and PR templates in GitHub.
- Automate formatting/analyzers via `dotnet format` or Roslyn analyzers in CI.
**Checkpoint**
- Document branching strategy plus evidence of protected branches and automated checks.

### Part 2 — CI/CD Pipelines
**Focus:** GitHub Actions/Azure DevOps, automated testing, approvals.
**Key competencies**
- Build multi-stage pipelines that compile, test, scan, and publish artifacts.
- Cache dependencies, parallelize jobs, and surface logs/artifacts.
- Integrate security scans (SCA, SAST) into the pipeline.
**Hands-on practice**
- Create GitHub Actions workflow with build, unit/integration tests, and container publish.
- Add quality gates (coverage, linting) and manual approval for production deploy.
**Checkpoint**
- Share pipeline runs showing green builds, test artifacts, and gating rationale.

### Part 3 — Containers & Cloud
**Focus:** Docker, Kubernetes basics, Azure Container Apps/App Service.
**Key competencies**
- Write optimized Dockerfiles (multi-stage, smaller base images).
- Configure container health probes, environment variables, and secrets.
- Understand Kubernetes basics (pods, services, deployments) or Azure Container Apps concepts.
- Deploy to Azure App Service/Container Apps with zero-downtime strategies.
**Hands-on practice**
- Containerize API and worker, run locally via docker-compose.
- Deploy to Azure Container Apps or App Service, capturing logs and metrics.
**Checkpoint**
- Provide deployment pipeline snippet plus screenshot of running container in Azure.

### Part 4 — Infrastructure as Code
**Focus:** Bicep/Terraform basics, deploying DBs, secret rotation.
**Key competencies**
- Model infrastructure (App Service, DB, Key Vault) in code with parameters and modules.
- Handle schema migrations during deployment.
- Automate key rotation and configuration updates.
**Hands-on practice**
- Author Bicep or Terraform modules for API + DB + cache, apply to dev/test/prod.
- Integrate EF Core migrations or Flyway into release pipelines.
**Checkpoint**
- Store IaC code in repo and show pipeline logs applying it plus documentation on secret rotation cadence.

**Suggested resources:** Microsoft DevOps Labs, GitHub Actions starter workflows, Azure Bicep docs, HashiCorp Learn Terraform.

---

## Module 12 — Professional Development Skills (3 parts)
**Outcome:** Demonstrate developer maturity through communication, productivity, and career readiness.  
**Recommended duration:** 3 weeks (parallel with other modules).
**Detailed files:** [Module overview](../modules/12-professional-development/README.md) | [Part 1](../modules/12-professional-development/part-1.md) | [Part 2](../modules/12-professional-development/part-2.md) | [Part 3](../modules/12-professional-development/part-3.md)

### Part 1 — Code Reviews & Collaboration
**Focus:** Good PR etiquette, writing readable code, reviewing others’ work.
**Key competencies**
- Craft PR descriptions with context, testing evidence, and screenshots.
- Review others’ code constructively, citing principles and offering alternatives.
- Facilitate async collaboration via GitHub Discussions, issues, and RFCs.
**Hands-on practice**
- Pair with a peer or mentor to exchange PRs weekly, practicing actionable feedback.
- Write ADRs or design docs for roadmap features.
**Checkpoint**
- Maintain a log of PRs reviewed/submitted plus reviewer feedback demonstrating clarity.

### Part 2 — Developer Productivity
**Focus:** Tooling, debugging mastery, IDE shortcuts.
**Key competencies**
- Master debugger features (conditional breakpoints, tracepoints, watch windows).
- Automate repetitive tasks with snippets, templates, editor macros.
- Measure and reduce context switching during the workday.
**Hands-on practice**
- Record a debugging session solving a tricky bug using advanced debugger tooling.
- Customize IDE profiles, keymaps, and snippets; document them.
**Checkpoint**
- Publish a productivity playbook with tooling setup and before/after efficiency metrics.

### Part 3 — Career Skills
**Focus:** Writing documentation, explaining technical decisions, preparing for interviews.
**Key competencies**
- Produce documentation (README, runbooks, onboarding guides) that others can follow.
- Communicate technical decisions to non-engineers.
- Prepare for behavioral and system design interviews by explaining past outcomes.
**Hands-on practice**
- Write a case study describing a feature from problem statement to deployment including lessons learned.
- Conduct mock interviews or lightning talks summarizing modules completed.
**Checkpoint**
- Share published documentation plus mock interview feedback from peers or mentors.

**Suggested resources:** _The Pragmatic Programmer_, Basecamp’s Shape Up essays, GitHub Docs guide, “Cracking the Coding Interview” (behavior prep).

---

## Module 13 — Working with Legacy Code & Debugging (3 parts)
**Outcome:** Navigate, debug, and safely change code you didn't write.
**Recommended duration:** 3 weeks.
**Detailed files:** [Module overview](../modules/13-legacy-code/README.md) | [Part 1](../modules/13-legacy-code/part-1.md) | [Part 2](../modules/13-legacy-code/part-2.md) | [Part 3](../modules/13-legacy-code/part-3.md)

### Part 1 — Reading Unfamiliar Code
**Focus:** Navigation strategies, understanding without documentation, creating mental models.
**Key competencies**
- Explore codebases systematically using IDE tools and search.
- Read tests and exception handlers to understand behavior.
- Document discoveries with ADRs and system maps.
**Hands-on practice**
- Navigate OrderFlow as if you've never seen it.
- Create system diagrams through exploration only.
- Document three undocumented decisions you discover.
**Checkpoint**
- System map plus ADRs created through code archaeology.

### Part 2 — Debugging Systematically
**Focus:** Stack traces, logs, breakpoints, production debugging.
**Key competencies**
- Read stack traces and identify root causes.
- Use debugger features beyond F5/F10 (conditional breakpoints, tracepoints).
- Analyze structured logs with correlation IDs.
- Debug production issues with dotnet-trace and dotnet-dump.
**Hands-on practice**
- Debug intentionally introduced bugs in OrderFlow.
- Create a debugging playbook for common issue types.
- Profile and fix a memory leak.
**Checkpoint**
- Debugging session recordings with narrated thought process.

### Part 3 — Making Safe Changes
**Focus:** Characterization tests, seams, strangler fig pattern, safe refactoring.
**Key competencies**
- Write characterization tests for legacy code.
- Find seams to isolate dependencies for testing.
- Apply strangler fig pattern for incremental migration.
- Refactor safely with IDE tools and small commits.
**Hands-on practice**
- Add tests to untestable OrderFlow code.
- Migrate a legacy component using strangler fig.
- Break up a god class incrementally.
**Checkpoint**
- Test suite for legacy code plus refactoring commit history.

**Suggested resources:** _Working Effectively with Legacy Code_ (Feathers), _Refactoring_ (Fowler), Julia Evans debugging zines.

---

## Putting It All Together
- Build **OrderFlow** across all modules — one project that grows in complexity as you learn.
- Each part feeds a learning repo containing code, notes, architecture diagrams, benchmarks, and ADRs.
- Revisit modules quarterly to close gaps; teaching a lunch-and-learn on each module cements mastery.
- Mid-level readiness is proven when you can build a feature end-to-end, defend the architectural tradeoffs, automate its delivery, and explain its impact to stakeholders.

See [SAMPLE-PROJECT.md](../modules/00-introduction/SAMPLE-PROJECT.md) for the complete OrderFlow specification.
See [RESOURCES.md](../modules/00-introduction/RESOURCES.md) for curated learning materials.

---

## Mentor Alignment Checklist
Before executing the curriculum, run a working session with mentors or team leads and capture action items for each module below. Update the roadmap afterward so expectations stay documented.

- **Module 04 — EF Core, SQL, Dapper & Database Design:** Confirm the domains, schemas, and hot spots your team cares about. Align on how migrations are managed, what queries routinely hurt performance, and when engineers should choose EF vs Dapper so every checkpoint exercises company realities.
- **Module 05 — NoSQL & Distributed Data:** Identify the NoSQL services in production (Cosmos DB, Mongo Atlas, Redis Enterprise, etc.). Map CAP tradeoffs, retention policies, and cache eviction strategies to those offerings, and document which data sets actually justify non-relational storage.
- **Module 06 — Messaging, Queues & Event-Driven Systems:** Review the messaging stack currently in use (Azure Service Bus, RabbitMQ, Kafka). Capture required retry policies, DLQ monitoring practices, and outbox/inbox conventions so the exercises mimic live workloads.
- **Module 07 — Testing Mastery:** Compare the roadmap’s test pyramid with your team’s expectations. Note the preferred frameworks, mocking libraries, container tooling, minimum coverage, and CI quality gates to avoid practicing the wrong stack.
- **Module 08 — Security, Auth & Identity:** Document the identity providers, token lifetimes, secrets platforms, and compliance constraints already enforced in production. Integrate the recurring OWASP or audit findings mentors mention directly into your exercises.
- **Module 09 — Performance, Scaling & Observability:** Align on the observability platform (Application Insights, Grafana, Datadog, etc.) plus existing SLOs. Target performance labs toward the incident types the team actually firefights (DB locking, CPU spikes, async misuse).
- **Module 10 — Architecture Patterns & Distributed Systems:** Ask which architectural styles the org defaults to today (modular monolith vs microservices) and where upcoming projects demand new bounded contexts, gateways, or sagas. Use those answers to anchor ADRs and design artifacts.
- **Module 11 — DevOps, CI/CD & Cloud Fundamentals:** Match the CI/CD tooling, branching rules, release cadence, and IaC languages to what production uses today. Capture non-negotiable approval steps or security scans so your pipelines look identical to real ones.
- **Module 12 — Professional Development Skills:** Clarify the communication habits, PR etiquette, documentation level, and mentoring expectations that leaders reward. Bake those specifics into the deliverables (PR templates, runbooks, brown-bag presentations).

During the review, collect links to live systems (dashboards, repos, runbooks) and record them in your learning repository so future cohorts benefit as well.
