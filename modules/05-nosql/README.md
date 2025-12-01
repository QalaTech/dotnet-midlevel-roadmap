# 05 — NoSQL & Distributed Data

## Why This Module Exists

NoSQL databases solve problems that relational databases struggle with:
- Massive scale (billions of documents)
- Flexible schemas that evolve rapidly
- Geographic distribution across regions
- Specific access patterns (key-value, document, graph)

But NoSQL is not a silver bullet. Using it without understanding trade-offs leads to:
- Data inconsistency nightmares
- Performance worse than SQL
- Operational complexity
- Technical debt

This module teaches you when to use NoSQL, how to model data correctly, and how to avoid common disasters.

---

## What You'll Build

OrderFlow's NoSQL components:
- **Product Catalog** — MongoDB for flexible product schemas and search
- **Session & Cart Cache** — Redis for fast, ephemeral data
- **Activity Feed** — Event stream with Redis or MongoDB
- **Distributed Locks** — Inventory holds during checkout

---

## Module Structure

### [Part 1 — NoSQL Foundations & CAP Trade-offs](./part-1.md)
- When to use SQL vs NoSQL (decision framework)
- CAP theorem — what it really means for your app
- Eventual consistency patterns
- Real-world disasters from wrong database choices

### [Part 2 — Document Databases (MongoDB)](./part-2.md)
- Document modeling (embedding vs referencing)
- Indexes and query optimization
- Aggregation pipelines
- Transactions and consistency

### [Part 3 — Redis Caching & Distributed Primitives](./part-3.md)
- Caching patterns (cache-aside, write-through)
- Redis data structures for real use cases
- Distributed locks and rate limiting
- Pub/Sub and event streaming

---

## Prerequisites

Before starting this module:
- Complete Module 04 (EF Core) — understand relational data modeling
- Understand basic distributed systems concepts
- Have Docker installed (for MongoDB and Redis)

---

## Key Decisions You'll Learn to Make

1. **SQL or NoSQL?** — Based on data shape, access patterns, and scale
2. **Embed or Reference?** — Trade-offs in document design
3. **Cache or Not?** — When caching helps vs hurts
4. **Strong or Eventual Consistency?** — Based on business requirements

---

## Tools You'll Use

- **MongoDB** — Document database with .NET driver
- **Redis** — In-memory data store with StackExchange.Redis
- **Docker Compose** — Local development setup
- **MongoDB Compass** — Visual query tool
- **RedisInsight** — Redis debugging

---

## Exit Criteria

You can:
1. Articulate when NoSQL is the right choice with specific trade-offs
2. Design a document schema that performs well at scale
3. Implement caching that improves performance without causing stale data bugs
4. Use Redis for coordination (locks, rate limiting) safely

---

**Start:** [Part 1 — NoSQL Foundations & CAP Trade-offs](./part-1.md)
