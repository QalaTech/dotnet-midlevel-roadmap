# 10 — Architecture Patterns & Distributed Systems

## Why This Module Exists

Architecture isn't about drawing boxes and arrows. It's about **making decisions that are expensive to change later**.

> "Architecture is about the important stuff. Whatever that is." — Ralph Johnson

The difference between a mid-level and senior engineer often comes down to:
- Mid-level: "I implemented the feature"
- Senior: "I implemented the feature in a way that won't haunt us in 2 years"

---

## What Goes Wrong Without This

### The Big Ball of Mud

```
Year 1: "Let's just ship it"
Year 2: "We need to add this feature somewhere... anywhere"
Year 3: "Don't touch that file, it does everything"
Year 4: "We need to rewrite the whole thing"

Cost of rewrite: 18 months, 3 engineers
Cost of doing it right: 2 extra weeks upfront
```

### The Distributed Monolith

```
Team: "Let's split into microservices for scalability!"

Result:
- All services still deploy together
- One service change breaks three others
- Network calls where function calls used to work
- Same coupling, now with latency

"We took a bad monolith and made it a worse distributed system"
```

### The Accidental Complexity Spiral

```
New feature request arrives
├── "Let me check which service handles this"
├── "Actually it spans 4 services"
├── "Service A needs data from B"
├── "B needs to call C first"
├── "C is owned by another team, they're busy"
├── "Let me just duplicate the data..."
├── "Now data is inconsistent"
└── "Why did we split these services again?"
```

---

## What You'll Build

OrderFlow's evolution through architectural maturity:

1. **Well-Structured Monolith** — Clean boundaries inside a single deployment
2. **Domain Model** — Aggregates, entities, value objects, domain events
3. **Service Extraction** — When and how to split (and when not to)
4. **Resilient Integration** — Circuit breakers, retries, sagas

---

## Module Structure

### [Part 1 — Architecture Patterns](./part-1.md)
- Clean Architecture vs Vertical Slices vs Modular Monolith
- Dependency rules and boundary enforcement
- Architecture Decision Records (ADRs)
- When to use what

### [Part 2 — Domain-Driven Design](./part-2.md)
- Event Storming workshop format
- Bounded Contexts and Context Maps
- Aggregates, Entities, and Value Objects
- Domain Events and eventual consistency

### [Part 3 — Service Boundaries](./part-3.md)
- When to split (and when not to)
- API Gateway patterns
- Saga orchestration and choreography
- Data ownership and consistency

### [Part 4 — Resilience Patterns](./part-4.md)
- Circuit breakers and retries with Polly
- Timeouts and bulkheads
- Chaos engineering basics
- Graceful degradation

---

## Prerequisites

Before starting this module:
- Complete Module 06 (Messaging) — understand async communication
- Complete Module 07 (Testing) — test architectural boundaries
- Complete Module 09 (Performance) — understand distributed system challenges

---

## Key Concepts You'll Master

1. **Boundaries** — Where one thing ends and another begins
2. **Coupling vs Cohesion** — What changes together, stays together
3. **Trade-offs** — Every architectural decision has costs
4. **Evolution** — Good architecture enables change

---

## The Architecture Decision Framework

Before making any architectural change, answer:

1. **What problem are we solving?** (Not "what's trendy")
2. **What are the options?** (Always at least 3)
3. **What are the trade-offs?** (Nothing is free)
4. **How will we know if it worked?** (Measurable outcomes)
5. **How hard is it to reverse?** (Reversibility matters)

---

## Tools You'll Use

| Category | Tools |
|----------|-------|
| **Diagramming** | Miro, PlantUML, C4 Model, Structurizr |
| **Architecture Testing** | NetArchTest, ArchUnit |
| **API Gateway** | YARP, Azure API Management, Kong |
| **Resilience** | Polly, Microsoft.Extensions.Resilience |
| **Chaos Testing** | Toxiproxy, Chaos Monkey |

---

## Exit Criteria

You can:
1. Explain the trade-offs between architectural patterns
2. Write ADRs that future engineers will thank you for
3. Design service boundaries based on domain, not technology
4. Implement resilience patterns that handle real failures
5. Know when to refactor architecture and when to leave it alone

---

## The Architecture Maturity Journey

```
Level 1: "It works"
         └── Code solves the immediate problem

Level 2: "It's organized"
         └── Clear folder structure, naming conventions

Level 3: "Dependencies flow correctly"
         └── Domain doesn't depend on infrastructure

Level 4: "Boundaries are enforced"
         └── Modules can't accidentally couple

Level 5: "Changes are localized"
         └── Most changes touch one module

Level 6: "Evolution is planned"
         └── ADRs document why, not just what
```

---

## Resources

### Essential Reading
- [Clean Architecture](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/) by Robert C. Martin
- [Domain-Driven Design Distilled](https://www.oreilly.com/library/view/domain-driven-design-distilled/9780134434964/) by Vaughn Vernon
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/) by Sam Newman

### Video Resources
- [Jimmy Bogard - Vertical Slice Architecture](https://www.youtube.com/watch?v=SUiWfhAhgQw)
- [Udi Dahan - Finding Service Boundaries](https://www.youtube.com/watch?v=dnhshUdRW70)
- [Martin Fowler - Microservices](https://www.youtube.com/watch?v=wgdBVIX9ifA)

### Tools Documentation
- [NetArchTest](https://github.com/BenMorris/NetArchTest)
- [YARP Documentation](https://microsoft.github.io/reverse-proxy/)
- [Polly Documentation](https://github.com/App-vNext/Polly)

---

**Start:** [Part 1 — Architecture Patterns](./part-1.md)
