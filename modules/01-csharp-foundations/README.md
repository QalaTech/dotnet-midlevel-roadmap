# 01 — Deep C# & .NET Foundations

## Why This Module Exists

C# is your primary tool. Everything else — APIs, EF Core, messaging, Clean Architecture — sits on top of your ability to express ideas in C# accurately and efficiently.

Most junior plateaus come from weak C# fundamentals, not framework issues. You can follow tutorials forever and still write code that:
- Allocates memory unnecessarily
- Deadlocks under load
- Confuses value and reference semantics
- Breaks in production but works locally

This module fixes that.

---

## What You'll Build

OrderFlow's **domain models** and foundational code:
- Value objects: `Money`, `Address`
- Entities: `Product`, `Customer`, `LineItem`
- Aggregate root: `Order`
- Custom exceptions with context
- In-memory repository with proper LINQ usage

---

## Module Structure

### [Part 1 — Types, Memory, and LINQ Mastery](./part-1.md)
- Value types vs reference types
- Nullable reference types (NRT)
- Records and immutability
- Deep LINQ understanding
- Async/await fundamentals

### [Part 2 — Exceptions, Expression Trees, Memory, and Delegates](./part-2.md)
- Exception design principles
- Expression trees and why EF Core uses them
- Memory management and GC awareness
- Delegates, Func, Action, and closures

### [Part 3 — Generics, SOLID, Design Patterns, and Reflection](./part-3.md)
- Generics and constraints
- SOLID principles (real understanding, not interview answers)
- Strategy, Decorator, Factory patterns
- Reflection and when to use it

### [Part 4 — Async Streams, DI Deep Dive, and Mastery Exercises](./part-4.md)
- Span<T> and Memory<T> for high-performance scenarios
- Async streams for processing large datasets
- Dependency injection internals
- Practical mastery exercises

---

## Prerequisites

Before starting this module:
- Basic C# syntax (variables, loops, classes, methods)
- Git basics (clone, commit, push)
- .NET 8 SDK installed
- VS Code or Rider set up

---

## Key Concepts You'll Master

1. **Memory Model** — Stack vs heap, when allocations happen
2. **Type System** — Value vs reference, records, nullability
3. **LINQ** — IEnumerable vs IQueryable, deferred execution
4. **Async** — When to use it, common mistakes
5. **Design** — SOLID, patterns, clean abstractions

---

## Tools You'll Use

- **BenchmarkDotNet** — Prove performance claims with numbers
- **LINQPad** — Experiment with LINQ interactively
- **.NET CLI** — Create projects, run tests
- **dotnet-trace** — Basic profiling

---

## Exit Criteria

You can:
1. Explain when to use struct vs class vs record
2. Write LINQ that translates to efficient SQL
3. Design exceptions that help rather than hide problems
4. Understand async/await well enough to avoid common deadlocks
5. Build domain models with proper invariants

---

**Start:** [Part 1 — Types, Memory, and LINQ Mastery](./part-1.md)
