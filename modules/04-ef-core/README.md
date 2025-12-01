# **04 — EF Core, SQL, Dapper & Database Design**

> **Goal:** Own relational data design end-to-end — from schema modeling to EF Core patterns to raw SQL/Dapper optimizations. Every part ends with code, diagrams, and metrics captured in your learning repo.

## 🗺️ Module Overview
- **Cadence:** 5 parts (~6–7 weeks). Expect two focused weeks on modeling/SQL, then one week each for EF Core fundamentals, advanced topics, and Dapper.
- **Success looks like:** You can model a domain, tune indexes, ship EF Core code confidently, and decide when to drop down to SQL or Dapper backed by benchmarks.
- **Artifacts to produce:** ERDs, migration scripts, benchmark markdown, EF sample projects, Dapper repositories, decision records.

## 📂 Part Files
1. [Part 1 — Database Foundations: Modeling & Indexing](./part-1.md) — Convert business requirements into tables, constraints, migrations, and indexing experiments with evidence.
2. [Part 2 — SQL Mastery: Querying & Concurrency](./part-2.md) — Level-up your SQL fluency, window functions, and concurrency troubleshooting.
3. [Part 3 — EF Core Fundamentals](./part-3.md) — Configure DbContexts, relationships, LINQ translation, and change tracker behavior with tests.
4. [Part 4 — EF Core Advanced Topics](./part-4.md) — Tackle migrations strategy, projections, interceptors, and value converters.
5. [Part 5 — Dapper & Raw SQL](./part-5.md) — Benchmark EF vs Dapper, implement multi-mapping, and mix raw SQL with EF safely.

## ✅ Module Exit Criteria
- You can explain, with code and diagrams, the full data life cycle from schema to API DTO.
- You can review teammates’ migrations/queries confidently.
- You have benchmarks proving when EF is enough and when Dapper/SQL is required.
