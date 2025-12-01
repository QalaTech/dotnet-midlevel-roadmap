# 02 — HTTP & REST Fundamentals

## Why This Module Exists

Every backend developer "knows" REST. Few understand it deeply.

The difference shows up in:
- APIs that break when requirements change
- Status codes that confuse frontend teams
- Endpoints that don't compose well
- Designs that require awkward workarounds

This module teaches REST as a **design discipline**, not just a set of conventions.

---

## What You'll Build

OrderFlow's **API contract** — no code yet, just documentation:
- Resource hierarchy (orders, products, customers, line items)
- Endpoint design for every operation
- Request/response shapes with examples
- Error catalog with ProblemDetails format
- Versioning strategy
- Caching headers plan

This API spec becomes the blueprint for Module 03.

---

## Module Structure

### [Part 1 — HTTP Verbs, Resources, and API Design](./part-1.md)
- Resources vs actions (REST fundamentals)
- HTTP methods and their semantics
- Idempotency and safety
- URL structure and naming conventions

### [Part 2 — Status Codes, Errors, and Filtering](./part-2.md)
- Choosing the right status code
- Error responses with ProblemDetails
- Filtering, sorting, pagination patterns
- Query parameter design

### [Part 3 — Relationships, Versioning, and Caching](./part-3.md)
- Representing relationships (nested vs linked)
- Versioning strategies and tradeoffs
- HTTP caching headers (ETag, Cache-Control)
- Conditional requests (If-None-Match, If-Modified-Since)

### [Part 4 — API Contracts, Rate Limiting, and Longevity](./part-4.md)
- OpenAPI/Swagger specification
- Contract-first design
- Rate limiting strategies
- Designing for long-term maintenance

---

## Prerequisites

Before starting this module:
- Complete Module 01 (C# Foundations)
- Basic understanding of HTTP (requests, responses, headers)
- Familiarity with JSON

---

## Key Concepts You'll Master

1. **Resource Modeling** — Think in nouns, not verbs
2. **HTTP Semantics** — What each method and status code actually means
3. **Idempotency** — Why it matters for reliability
4. **API Evolution** — Design for change from day one

---

## Tools You'll Use

- **Postman / Bruno** — Design and test API requests
- **Swagger Editor** — Write OpenAPI specifications
- **curl / HTTPie** — Command-line HTTP clients
- **Miro / diagrams.net** — Resource relationship diagrams

---

## Exit Criteria

You can:
1. Design a REST API from requirements
2. Choose appropriate HTTP methods and status codes
3. Document an API with OpenAPI
4. Explain versioning tradeoffs
5. Design for caching and performance

---

**Start:** [Part 1 — HTTP Verbs, Resources, and API Design](./part-1.md)
