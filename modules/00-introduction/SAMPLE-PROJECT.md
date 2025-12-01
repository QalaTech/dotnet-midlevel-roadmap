# The Unified Learning Project: OrderFlow

Throughout this roadmap, you'll build **OrderFlow** — a realistic order management system that grows in complexity as you progress through the modules. This isn't a toy project; it mirrors the systems you'll work on in real companies.

## Why One Project?

Most learning paths have you build disconnected sample apps. You finish a tutorial, throw away the code, and start fresh. That's not how real engineering works.

In production, you:
- Inherit existing code and extend it
- Live with decisions made months ago
- Watch complexity grow and learn to manage it
- See how "small" choices compound over time

OrderFlow teaches you this. By Module 11, you'll have a system with real history, real tradeoffs, and real technical debt you created yourself.

---

## The Domain: OrderFlow

**OrderFlow** is an order management system for a B2B wholesale distributor. It handles:

- **Customers** — businesses that place orders
- **Products** — items with inventory, pricing tiers, and categories
- **Orders** — line items, status tracking, fulfillment states
- **Inventory** — stock levels, reservations, backorders
- **Notifications** — order confirmations, shipping updates, low-stock alerts
- **Reporting** — sales analytics, inventory turnover, customer insights

This domain is complex enough to exercise every concept in the roadmap but simple enough to understand quickly.

---

## How OrderFlow Grows by Module

### Module 01 — C# Foundations
Build the **domain models**: `Order`, `LineItem`, `Product`, `Customer`, `Money`, `Address`. Practice value types vs reference types, records, nullable reference types. Write a simple in-memory repository.

**Deliverable:** Domain model library with unit tests proving invariants.

### Module 02 — REST Fundamentals
Design the **API contract** on paper. Map resources, choose verbs, define status codes, plan error responses. No code yet — just documentation.

**Deliverable:** API design doc with resource tree, verb mapping, and error catalog.

### Module 03 — Building APIs
Implement the **Orders API** using ASP.NET Core Minimal APIs. CRUD for orders, validation, error handling, structured logging, basic auth.

**Deliverable:** Running API with Swagger docs, integration tests, and Postman collection.

### Module 04 — EF Core & SQL
Add **SQL Server persistence**. Design the schema, write migrations, implement the repository pattern, optimize queries, add Dapper for reporting endpoints.

**Deliverable:** Database with migrations, EF Core repositories, Dapper-powered dashboard queries.

### Module 05 — NoSQL
Add **Redis caching** for product catalog and **MongoDB** for audit logs. Learn when each datastore fits.

**Deliverable:** Hybrid data layer with SQL, Redis, and Mongo working together.

### Module 06 — Messaging
Implement **async order processing**. When an order is placed, publish an event. A worker service handles inventory reservation, sends notifications, and updates fulfillment status.

**Deliverable:** RabbitMQ/Azure Service Bus integration with outbox pattern and dead-letter handling.

### Module 07 — Testing
Build a **comprehensive test suite**. Unit tests for domain logic, integration tests for API endpoints, contract tests for external consumers.

**Deliverable:** Test pyramid implemented with 80%+ coverage on business logic.

### Module 08 — Security
Harden the API. Add **JWT authentication**, role-based authorization (admin vs customer), rate limiting, secrets management, and threat modeling.

**Deliverable:** Secured API with auth flows documented and penetration test checklist completed.

### Module 09 — Performance
Profile and optimize. Add **OpenTelemetry tracing**, identify slow queries, implement caching strategies, load test with k6.

**Deliverable:** Performance baseline, optimization report, and monitoring dashboard.

### Module 10 — Architecture
Refactor toward **Clean Architecture**. Extract bounded contexts, implement domain events properly, evaluate whether to split into microservices (spoiler: probably not yet).

**Deliverable:** Architecture Decision Records (ADRs) documenting every significant choice.

### Module 11 — DevOps
Build the **CI/CD pipeline**. Dockerize the app, write GitHub Actions workflows, deploy to Azure Container Apps, implement infrastructure as code.

**Deliverable:** Fully automated deployment pipeline with staging and production environments.

### Module 12 — Professional Development
Document everything. Write the **README**, runbooks, onboarding guide. Present the system to a mentor or peer as if you're onboarding a new team member.

**Deliverable:** Production-ready documentation and recorded walkthrough.

### Module 13 — Legacy Code & Debugging
Intentionally **break things** and fix them. Introduce bugs, diagnose them with proper tooling, practice reading stack traces and logs, refactor without breaking functionality.

**Deliverable:** Troubleshooting playbook and recorded debugging sessions.

---

## Repository Structure

```
orderflow/
├── src/
│   ├── OrderFlow.Domain/           # Domain models, value objects, domain events
│   ├── OrderFlow.Application/      # Use cases, handlers, interfaces
│   ├── OrderFlow.Infrastructure/   # EF Core, Dapper, Redis, Mongo, messaging
│   ├── OrderFlow.Api/              # Minimal API endpoints
│   └── OrderFlow.Worker/           # Background job processor
├── tests/
│   ├── OrderFlow.Domain.Tests/     # Unit tests
│   ├── OrderFlow.Application.Tests/# Handler tests
│   ├── OrderFlow.Api.Tests/        # Integration tests
│   └── OrderFlow.Contracts.Tests/  # Contract tests
├── docs/
│   ├── api-design.md               # Module 02 deliverable
│   ├── architecture/               # ADRs
│   ├── runbooks/                   # Operational docs
│   └── postmortems/                # Incident learnings
├── infra/
│   ├── docker-compose.yml          # Local development
│   ├── Dockerfile                  # Container build
│   └── bicep/                      # Infrastructure as code
├── .github/
│   └── workflows/                  # CI/CD pipelines
└── README.md
```

---

## Getting Started

### Prerequisites
- .NET 8 SDK
- Docker Desktop
- VS Code or Rider
- Azure CLI (for later modules)

### Initial Setup (Module 01)

```bash
# Create solution
dotnet new sln -n OrderFlow

# Create projects
dotnet new classlib -n OrderFlow.Domain -o src/OrderFlow.Domain
dotnet new xunit -n OrderFlow.Domain.Tests -o tests/OrderFlow.Domain.Tests

# Add to solution
dotnet sln add src/OrderFlow.Domain
dotnet sln add tests/OrderFlow.Domain.Tests

# Add test reference
dotnet add tests/OrderFlow.Domain.Tests reference src/OrderFlow.Domain
```

You'll add more projects as you progress through modules.

---

## Real-World Parallels

| OrderFlow Concept | Real-World Equivalent |
|-------------------|----------------------|
| Order placement | E-commerce checkout, B2B procurement |
| Inventory reservation | Warehouse management, ticketing systems |
| Async notifications | Email/SMS services, webhook dispatchers |
| Multi-tenant considerations | SaaS platforms, enterprise software |
| Audit logging | Compliance systems, financial services |
| Reporting dashboards | Business intelligence, analytics |

---

## What You'll Learn That Tutorials Don't Teach

1. **Decisions compound** — That quick fix in Module 03 becomes technical debt by Module 08
2. **Consistency is hard** — Keeping naming, patterns, and styles consistent across months of work
3. **Refactoring is normal** — Your Module 04 code will look embarrassing by Module 10, and that's fine
4. **Documentation rots** — You'll learn to keep docs updated or suffer the consequences
5. **Production is different** — Local dev is easy; real environments have network issues, timeouts, and race conditions

---

## Milestone Checkpoints

At each milestone, you should be able to:

**After Module 03:** Demo a working API to someone non-technical and explain what it does.

**After Module 06:** Explain the difference between synchronous and asynchronous processing and why OrderFlow uses both.

**After Module 09:** Show metrics proving your optimizations actually improved performance.

**After Module 11:** Deploy a change to production in under 10 minutes with confidence it won't break anything.

**After Module 13:** Diagnose and fix a bug in code you haven't looked at in weeks.

---

## Reference Implementations

These open-source projects show similar patterns at production scale:

- **[eShop](https://github.com/dotnet/eShop)** — Microsoft's reference architecture for .NET microservices
- **[Clean Architecture Template](https://github.com/jasontaylordev/CleanArchitecture)** — Jason Taylor's widely-used project structure
- **[Ardalis Clean Architecture](https://github.com/ardalis/CleanArchitecture)** — Steve Smith's take on the pattern
- **[Full Modular Monolith](https://github.com/kgrzybek/modular-monolith-with-ddd)** — Kamil Grzybek's comprehensive DDD example
- **[EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore)** — Oskar Dudycz's event sourcing samples

Study these, but build OrderFlow yourself. Reading code teaches patterns; writing code builds intuition.
