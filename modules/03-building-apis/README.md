# 03 — Building Web APIs with ASP.NET Core

## Why This Module Exists

Knowing REST theory (Module 02) is useless if you can't implement it correctly.

This module bridges design and implementation. You'll build the OrderFlow API, learning:
- How requests flow through ASP.NET Core
- Why middleware order matters (and when it breaks things)
- How dependency injection actually works
- When Minimal APIs vs Controllers make sense

By the end, you'll have a production-structured API, not tutorial code.

---

## What You'll Build

The OrderFlow API becomes real code:
- Orders, Products, Customers endpoints using Minimal APIs
- Proper validation with FluentValidation
- Global error handling with ProblemDetails
- Structured logging with Serilog
- JWT authentication and policy-based authorization
- Background job processing with IHostedService

---

## Module Structure

### [Part 1 — The Request Pipeline, Endpoints, and DI](./part-1.md)
- ASP.NET Core request pipeline
- Middleware order and why it matters
- Minimal API endpoint registration
- Dependency injection lifetimes (scoped, transient, singleton)

### [Part 2 — Validation, Errors, and Logging](./part-2.md)
- FluentValidation integration
- Global exception handling
- ProblemDetails responses
- Structured logging with Serilog

### [Part 3 — Authentication, Authorization, and Security](./part-3.md)
- JWT Bearer authentication
- Claims and policies
- Authorization handlers
- Security headers and HTTPS

### [Part 4 — Background Jobs and Deployment](./part-4.md)
- IHostedService and BackgroundService
- Job scheduling patterns
- Docker containerization
- Health checks and readiness probes

---

## Prerequisites

Before starting this module:
- Complete Module 01 (C# Foundations) — understand async/await
- Complete Module 02 (REST Fundamentals) — know what you're building
- Have the OrderFlow API design doc from Module 02

---

## Key Concepts You'll Master

1. **Request Pipeline** — Middleware execution order
2. **Dependency Injection** — Service lifetimes and scopes
3. **Validation** — Input validation at the boundary
4. **Error Handling** — Consistent error responses
5. **Security** — Authentication and authorization patterns

---

## Tools You'll Use

- **ASP.NET Core 8** — Minimal APIs
- **FluentValidation** — Request validation
- **Serilog** — Structured logging
- **Swagger/OpenAPI** — API documentation
- **Docker** — Containerization
- **Postman / Bruno** — API testing

---

## Exit Criteria

You can:
1. Build a well-structured Minimal API
2. Configure middleware in the correct order
3. Implement proper validation and error handling
4. Secure endpoints with JWT authentication
5. Containerize and deploy the API

---

**Start:** [Part 1 — The Request Pipeline, Endpoints, and DI](./part-1.md)
