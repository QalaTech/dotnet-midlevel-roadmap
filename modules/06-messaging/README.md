# 06 — Messaging, Queues & Event-Driven Systems

## Why This Module Exists

Most junior developers only know HTTP request/response. But real systems need:
- Background processing (don't block user waiting for email)
- System decoupling (orders don't need to know about shipping)
- Retry guarantees (payment must eventually process)
- Event broadcasting (notify multiple services of a change)

Messaging enables all of this. But it also introduces:
- Message loss
- Duplicate processing
- Ordering problems
- Dead letter queue explosions

This module teaches you to build reliable async systems.

---

## What You'll Build

OrderFlow's messaging infrastructure:
- **Order Processing Pipeline** — Order created → Validate → Charge → Fulfill
- **Notification System** — Email, SMS, push notifications
- **Outbox Pattern** — Reliable event publishing
- **Dead Letter Handling** — Poison message recovery

---

## Module Structure

### [Part 1 — Messaging Foundations](./part-1.md)
- Commands vs Events vs Queries
- Message contracts and versioning
- Idempotent handlers
- When to use messaging vs HTTP

### [Part 2 — Queues and Message Brokers](./part-2.md)
- Azure Service Bus / RabbitMQ setup
- Retry policies and backoff
- Dead letter queues
- Competing consumers

### [Part 3 — The Outbox Pattern](./part-3.md)
- The dual-write problem
- Implementing transactional outbox
- Event dispatching
- Consumer deduplication

### [Part 4 — Operating Message Systems](./part-4.md)
- Observability and tracing
- Scaling workers
- Incident response
- Chaos engineering

---

## Prerequisites

Before starting this module:
- Complete Module 04 (EF Core) — understand transactions
- Complete Module 05 (NoSQL/Redis) — understand distributed systems
- Have Docker installed (for message brokers)

---

## Key Concepts You'll Master

1. **Eventual Consistency** — Accept that systems aren't always in sync
2. **At-Least-Once Delivery** — Handle duplicates gracefully
3. **Transactional Outbox** — Never lose events
4. **Dead Letter Queues** — Deal with poison messages

---

## Tools You'll Use

- **Azure Service Bus** or **RabbitMQ** — Message broker
- **MassTransit** — .NET messaging abstraction
- **Polly** — Retry policies
- **OpenTelemetry** — Distributed tracing

---

## Exit Criteria

You can:
1. Design an event-driven feature from scratch
2. Implement reliable message publishing with outbox
3. Build idempotent consumers
4. Debug message flow issues using traces

---

**Start:** [Part 1 — Messaging Foundations](./part-1.md)
