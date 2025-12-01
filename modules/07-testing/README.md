# 07 — Testing Mastery: Unit, Integration, Contract

## Why This Module Exists

Most developers know they "should write tests." Few understand:
- What to test at each layer (and what NOT to test)
- Why some test suites become maintenance nightmares
- How to write tests that actually catch bugs
- When mocking helps vs when it hides problems

Bad testing practices cause:
- False confidence (100% coverage, tests pass, production breaks)
- Slow pipelines (30-minute integration suites)
- Flaky tests (same code, random failures)
- Maintenance burden (change one line, fix 50 tests)

This module teaches you to build test suites that are fast, reliable, and actually useful.

---

## What You'll Build

OrderFlow's comprehensive test suite:
- **Domain Unit Tests** — Validate business rules and invariants
- **Handler Tests** — Test application logic with minimal mocking
- **Integration Tests** — Verify the full stack with real dependencies
- **Contract Tests** — Ensure API compatibility across services

---

## Module Structure

### [Part 1 — Testing Foundations](./part-1.md)
- The testing pyramid (and why it's often wrong)
- What goes wrong without good testing
- Test doubles: mocks vs stubs vs fakes
- Setting up testing infrastructure

### [Part 2 — Unit Testing Domain & Application](./part-2.md)
- Testing domain invariants and value objects
- Handler/service testing patterns
- When to mock and when not to
- Testing async and event-driven code

### [Part 3 — Integration Testing](./part-3.md)
- WebApplicationFactory deep dive
- Testcontainers for real dependencies
- Database testing strategies
- Testing message handlers

### [Part 4 — Contract & E2E Testing](./part-4.md)
- Consumer-driven contracts with Pact
- API schema validation
- E2E testing strategies
- CI/CD integration

---

## Prerequisites

Before starting this module:
- Complete Module 03 (Building APIs) — understand what you're testing
- Complete Module 04 (EF Core) — understand data layer testing
- Complete Module 06 (Messaging) — understand async testing

---

## Key Concepts You'll Master

1. **Test Pyramid** — Fast unit tests at base, slow integration tests at top
2. **Test Isolation** — Each test independent, no shared state
3. **Arrange-Act-Assert** — Clear test structure
4. **Test Doubles** — Right tool for each scenario

---

## Tools You'll Use

- **xUnit** — Test framework
- **FluentAssertions** — Readable assertions
- **NSubstitute** — Mocking framework
- **Bogus / AutoFixture** — Test data generation
- **WebApplicationFactory** — Integration test harness
- **Testcontainers** — Real dependencies in tests
- **Pact** — Consumer-driven contracts

---

## Exit Criteria

You can:
1. Design a testing strategy for any feature
2. Write unit tests that validate business rules
3. Build integration tests with real dependencies
4. Implement contract tests for API compatibility
5. Debug and fix flaky tests

---

**Start:** [Part 1 — Testing Foundations](./part-1.md)
