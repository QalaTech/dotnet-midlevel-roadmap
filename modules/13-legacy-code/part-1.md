# 13 — Working with Legacy Code & Debugging

## Part 1 / 3 — Reading and Understanding Unfamiliar Code

---

## Why This Skill Matters

You join a new team. There's a bug in production. Your manager says "it's somewhere in the order processing flow." You've never seen this codebase. You have 4 hours.

This happens constantly. The developers who thrive are the ones who can navigate unfamiliar code quickly.

---

## What Goes Wrong Without This

**Real scenario:** A developer is asked to add a feature to an existing system. Instead of understanding the current code, they build the feature "their way." It works but doesn't integrate properly. Data flows break. The codebase now has two different patterns for the same thing. Technical debt compounds.

**Real scenario:** A bug report comes in. A developer can't find the relevant code. They spend 6 hours searching randomly. A senior developer finds it in 20 minutes because they know how to navigate.

---

## Core Skill #1 — Strategic Code Reading

### Don't Read Top to Bottom

Code isn't a novel. Reading every file sequentially wastes time. Instead:

1. **Start from the entry point** — Find where the feature begins (API endpoint, event handler, scheduled job)
2. **Follow the happy path** — Trace the main flow before edge cases
3. **Note dependencies** — What services/repositories does this code use?
4. **Identify the core logic** — Where are the actual business rules?

### The 3-Layer Approach

```
Layer 1: Bird's Eye View (10 minutes)
├── What does this system do? (README, project names)
├── What's the tech stack? (csproj files, packages)
└── What's the structure? (folder organization)

Layer 2: Feature Flow (30 minutes)
├── Entry point (controller, handler)
├── Business logic (services, domain)
├── Data access (repositories, queries)
└── External integrations (APIs, queues)

Layer 3: Deep Dive (as needed)
├── Specific algorithms
├── Edge case handling
└── Error paths
```

### Tools for Navigation

**IDE Features:**
- Go to Definition (F12)
- Find All References (Shift+F12)
- Navigate to Symbol (Ctrl+T)
- Call Hierarchy
- File Structure view

**Command Line:**
```bash
# Find files containing a term
grep -r "OrderService" --include="*.cs"

# Find class definitions
grep -r "class.*Order" --include="*.cs"

# Find usages of a method
grep -r "ProcessOrder" --include="*.cs"
```

---

## Core Skill #2 — Understanding Without Documentation

### Read Tests First

Tests are executable documentation. They show:
- How to call the code
- What inputs are expected
- What outputs are produced
- What edge cases exist

```csharp
// This test tells you a lot about OrderService
[Fact]
public async Task PlaceOrder_WithInsufficientStock_ThrowsException()
{
    // Arrange
    var product = new Product("SKU-001", stock: 5);
    var order = new Order();
    order.AddItem(product, quantity: 10);

    // Act & Assert
    await Assert.ThrowsAsync<InsufficientStockException>(
        () => _service.PlaceOrderAsync(order));
}
```

From this test, you learn:
- `OrderService` has a `PlaceOrderAsync` method
- Orders have items with products and quantities
- Stock validation happens during order placement
- There's an `InsufficientStockException`

### Read Exception Handlers

Exception handlers reveal what can go wrong:

```csharp
try
{
    await ProcessPaymentAsync(order);
}
catch (PaymentDeclinedException)
{
    // Tells you: payments can be declined
    await RevertInventoryReservationAsync(order);
}
catch (PaymentTimeoutException)
{
    // Tells you: payments can timeout
    await EnqueueRetryAsync(order);
}
```

### Read Database Migrations

Migrations show how data evolved:

```csharp
// Migration from 6 months ago
migrationBuilder.AddColumn<bool>(
    name: "IsArchived",
    table: "Orders",
    defaultValue: false);

// This tells you: "IsArchived" was added later
// Some code might not account for it
// Old records have false by default
```

---

## Core Skill #3 — Creating Mental Models

### Draw as You Read

Don't keep everything in your head. Sketch:
- Sequence diagrams for flows
- Class diagrams for relationships
- Data flow diagrams for transformations

```
Request → Controller → Service → Repository → Database
                ↓
           Validator
                ↓
         EventPublisher → Queue → Worker
```

### Ask "Why" Questions

As you read, note questions:
- Why is this check here? (there was probably a bug)
- Why is this duplicated? (maybe a refactor was abandoned)
- Why is this commented out? (maybe it's still needed)

Don't delete "weird" code until you understand why it exists.

### Find the Seams

Seams are places where you can change behavior without modifying code:
- Interfaces (swap implementations)
- Configuration (change behavior via settings)
- Feature flags (enable/disable functionality)
- Event handlers (add new behavior)

These are your safest places to make changes.

---

## Core Skill #4 — Documenting What You Learn

### The "New Developer" Test

Write documentation that would help a new developer. This forces you to:
- Identify what's not obvious
- Explain the "why" not just the "what"
- Note the gotchas and surprises

### Architecture Decision Records (ADRs)

When you discover why something was done a certain way, document it:

```markdown
# ADR-003: Why Orders Use Soft Deletes

## Context
Orders are never physically deleted from the database.

## Decision
We use an `IsDeleted` flag instead of DELETE statements.

## Reasons
1. Audit requirements — we must keep order history for 7 years
2. Cascading issues — deleting orders would orphan related records
3. Recovery — users sometimes "delete" by mistake

## Consequences
- Queries must always filter by `IsDeleted = false`
- Storage grows over time (addressed by archiving job)
- Some reports need to include deleted orders
```

### Code Comments for Future You

When you figure out something tricky:

```csharp
// IMPORTANT: This must run BEFORE payment processing
// because the payment gateway validates stock availability
// Bug #1234: Orders were being charged for out-of-stock items
await _inventoryService.ReserveStockAsync(order);
await _paymentService.ProcessAsync(order);
```

---

## Hands-On: OrderFlow Archaeology

### Exercise 1: Blind Navigation

Have someone else (or yourself in 2 weeks) add a feature to OrderFlow without any explanation. Document:
- How long it took to find the relevant code
- What helped most (tests, names, structure)
- What hindered most

### Exercise 2: Create a System Map

Without looking at existing documentation, create:
- A component diagram showing all services
- A sequence diagram for order placement
- A data flow diagram for inventory updates

Compare with reality. Note what surprised you.

### Exercise 3: Document the Undocumented

Find three things in OrderFlow that aren't documented but should be:
- A non-obvious business rule
- A workaround for a known issue
- A dependency that isn't obvious

Write the missing documentation.

---

## Deliverables

1. **Codebase navigation guide** — Your personal checklist for exploring new code
2. **OrderFlow system map** — Diagrams created through exploration
3. **ADRs for discovered decisions** — At least 3 decisions you reverse-engineered
4. **Reading session recording** — Screen capture of you exploring unfamiliar code, narrating your thought process

---

## Resources

### Must-Read
- [Working Effectively with Legacy Code, Chapter 16-17](https://www.oreilly.com/library/view/working-effectively-with/0131177052/) — Understanding code
- [The Art of Readable Code](https://www.oreilly.com/library/view/the-art-of/9781449318482/) — What makes code understandable

### Videos
- [Sandi Metz: Go Ahead, Make a Mess](https://www.youtube.com/watch?v=mpA2F1In41w) — Understanding messy code
- [Kevlin Henney: Seven Ineffective Coding Habits](https://www.youtube.com/watch?v=ZsHMHukIlJY)

### Tools
- [Understand](https://scitools.com/) — Code visualization
- [NDepend](https://www.ndepend.com/) — .NET code analysis
- [Mermaid](https://mermaid.js.org/) — Diagrams as code

---

## Reflection Questions

1. What's the first thing you look for in a new codebase?
2. How do you know when you understand code "enough" to change it?
3. What documentation would have helped you most when joining your last project?
4. How do you balance time spent understanding vs time spent changing?

---

**Next:** [Part 2 — Debugging Systematically](./part-2.md)
