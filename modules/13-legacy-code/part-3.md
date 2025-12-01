# 13 — Working with Legacy Code & Debugging

## Part 3 / 3 — Making Safe Changes to Legacy Code

---

## Why This Skill Matters

The scariest code to change is code without tests. You fix one thing, break three others. Teams become afraid to touch certain files. Features take 10x longer because nobody understands the implications.

This part teaches you to change legacy code safely:
- Add tests to untestable code
- Refactor without breaking things
- Migrate incrementally (strangler fig pattern)
- Build confidence in changes

---

## What Goes Wrong Without This

**Real scenario:** A developer needs to add a feature to a 2000-line class with no tests. They make the change, manually test the happy path, and deploy. A week later, an edge case breaks. The bug is in their change. They fix it, breaking something else. Three hotfixes later, the team decides to never touch that file again.

**Real scenario:** A team decides to "rewrite" a legacy module. Six months later, the rewrite is 80% done but missing critical edge cases the legacy code handled. They can't ship it. They maintain two versions. Eventually, the rewrite is abandoned. Total waste: 6 months.

---

## Core Skill #1 — Characterization Tests

### What They Are

Characterization tests document what code *actually does*, not what it *should do*. They're your safety net for changes.

### The Process

1. **Write a test that calls the code**
2. **Assert what you *expect* to happen**
3. **Run the test — it fails**
4. **Change the assertion to match actual behavior**
5. **Now you have a test that will break if behavior changes**

```csharp
// Step 1-2: Write test with expected assertion
[Fact]
public void CalculateDiscount_WithLargeOrder_ReturnsExpectedDiscount()
{
    var order = CreateOrderWithTotal(1000m);

    var discount = _calculator.CalculateDiscount(order);

    Assert.Equal(100m, discount); // 10% expected
}

// Step 3: Test fails — actual discount is 150m

// Step 4: Update assertion to match reality
[Fact]
public void CalculateDiscount_WithLargeOrder_Returns15Percent()
{
    var order = CreateOrderWithTotal(1000m);

    var discount = _calculator.CalculateDiscount(order);

    Assert.Equal(150m, discount); // Actual behavior: 15%
}
```

Now if you change the calculator and discount becomes 140m, the test fails. You'll know immediately.

### Characterization Test Strategy

```csharp
public class LegacyOrderProcessor
{
    // This class has 50 methods. Test them all? No.

    // Priority 1: Methods you need to change
    // Priority 2: Methods called by methods you change
    // Priority 3: Public API methods
    // Priority 4: Everything else (when you have time)
}

// Focus tests on the code paths your change will affect
```

---

## Core Skill #2 — Finding Seams

### What's a Seam?

A seam is a place where you can change behavior without editing code. Seams let you:
- Inject test doubles
- Add logging
- Change implementations
- Break dependencies

### Types of Seams

**Constructor Seam (Dependency Injection):**
```csharp
// Hard to test
public class OrderService
{
    private readonly EmailClient _emailer = new EmailClient();

    public void PlaceOrder(Order order)
    {
        // ... logic
        _emailer.Send(order.Customer.Email, "Order confirmed");
    }
}

// Add a seam
public class OrderService
{
    private readonly IEmailClient _emailer;

    public OrderService(IEmailClient emailer)
    {
        _emailer = emailer;
    }

    // Now you can inject a fake
}
```

**Method Seam (Protected Virtual):**
```csharp
// Can't inject dependency? Make the method virtual.
public class OrderService
{
    public void PlaceOrder(Order order)
    {
        // ... logic
        SendConfirmation(order);
    }

    protected virtual void SendConfirmation(Order order)
    {
        var emailer = new EmailClient();
        emailer.Send(order.Customer.Email, "Order confirmed");
    }
}

// In tests, subclass and override
public class TestableOrderService : OrderService
{
    public bool ConfirmationSent { get; private set; }

    protected override void SendConfirmation(Order order)
    {
        ConfirmationSent = true;
    }
}
```

**Extract and Override:**
```csharp
// Original: Direct database call
public decimal GetCustomerBalance(Guid customerId)
{
    using var conn = new SqlConnection(_connectionString);
    return conn.QuerySingle<decimal>(
        "SELECT Balance FROM Customers WHERE Id = @Id",
        new { Id = customerId });
}

// Step 1: Extract to protected method
public decimal GetCustomerBalance(Guid customerId)
{
    return QueryBalance(customerId);
}

protected virtual decimal QueryBalance(Guid customerId)
{
    using var conn = new SqlConnection(_connectionString);
    return conn.QuerySingle<decimal>(
        "SELECT Balance FROM Customers WHERE Id = @Id",
        new { Id = customerId });
}

// Step 2: Override in tests
public class TestableService : CustomerService
{
    public Dictionary<Guid, decimal> Balances { get; } = new();

    protected override decimal QueryBalance(Guid customerId)
        => Balances.GetValueOrDefault(customerId);
}
```

---

## Core Skill #3 — The Strangler Fig Pattern

### What It Is

Instead of rewriting, gradually replace pieces while keeping the system running. Like a strangler fig tree that grows around its host.

### The Process

```
Phase 1: New code lives alongside old
┌─────────────────────────────────────┐
│           Application               │
├──────────────────┬──────────────────┤
│   Legacy Code    │   New Code       │
│   (still used)   │   (growing)      │
└──────────────────┴──────────────────┘

Phase 2: Route traffic to new code
┌─────────────────────────────────────┐
│           Application               │
├──────────────────┬──────────────────┤
│   Legacy Code    │   New Code       │
│   (less used)    │   (primary)      │
└──────────────────┴──────────────────┘
           ↑               ↑
       10% traffic     90% traffic

Phase 3: Remove legacy
┌─────────────────────────────────────┐
│           Application               │
├─────────────────────────────────────┤
│            New Code                 │
│          (100% traffic)             │
└─────────────────────────────────────┘
```

### Example: Replacing a Service

```csharp
// Step 1: Create interface for existing behavior
public interface IOrderCalculator
{
    decimal CalculateTotal(Order order);
}

// Step 2: Wrap legacy code
public class LegacyOrderCalculator : IOrderCalculator
{
    public decimal CalculateTotal(Order order)
    {
        // Calls the existing ugly code
        return LegacyOrderHelpers.CalculateOrderTotal(
            order.Items.ToArray(),
            order.Customer.DiscountLevel,
            order.ShippingMethod);
    }
}

// Step 3: Create new implementation
public class NewOrderCalculator : IOrderCalculator
{
    public decimal CalculateTotal(Order order)
    {
        var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
        var discount = _discountCalculator.Calculate(order);
        var shipping = _shippingCalculator.Calculate(order);
        return subtotal - discount + shipping;
    }
}

// Step 4: Feature flag to switch
public class OrderCalculatorSelector : IOrderCalculator
{
    private readonly IFeatureFlags _flags;
    private readonly LegacyOrderCalculator _legacy;
    private readonly NewOrderCalculator _new;

    public decimal CalculateTotal(Order order)
    {
        if (_flags.IsEnabled("new-calculator"))
            return _new.CalculateTotal(order);
        return _legacy.CalculateTotal(order);
    }
}

// Step 5: Compare results (shadow testing)
public class ComparingOrderCalculator : IOrderCalculator
{
    public decimal CalculateTotal(Order order)
    {
        var legacyResult = _legacy.CalculateTotal(order);
        var newResult = _new.CalculateTotal(order);

        if (legacyResult != newResult)
        {
            _logger.LogWarning(
                "Calculator mismatch for order {OrderId}: legacy={Legacy}, new={New}",
                order.Id, legacyResult, newResult);
        }

        return legacyResult; // Still use legacy until confident
    }
}
```

---

## Core Skill #4 — Safe Refactoring Steps

### The Golden Rule

**Never refactor and change behavior in the same commit.**

- Commit 1: Refactor (tests still pass, behavior unchanged)
- Commit 2: Add feature (behavior changes)

If something breaks, you know which commit caused it.

### Safe Refactoring Moves

**Extract Method:**
```csharp
// Before: Long method
public void ProcessOrder(Order order)
{
    // 20 lines of validation
    // 30 lines of calculation
    // 15 lines of persistence
}

// After: Extracted methods
public void ProcessOrder(Order order)
{
    ValidateOrder(order);
    CalculateTotals(order);
    PersistOrder(order);
}
```

**Extract Class:**
```csharp
// Before: God class
public class OrderService
{
    public void PlaceOrder() { }
    public void CancelOrder() { }
    public void ShipOrder() { }
    public decimal CalculateDiscount() { }
    public decimal CalculateShipping() { }
    public decimal CalculateTax() { }
    // ... 50 more methods
}

// After: Focused classes
public class OrderService { /* placement only */ }
public class OrderCancellation { /* cancellation only */ }
public class OrderShipping { /* shipping only */ }
public class OrderCalculations { /* calculations only */ }
```

**Replace Conditional with Polymorphism:**
```csharp
// Before: Switch statement
public decimal CalculateShipping(Order order)
{
    switch (order.ShippingType)
    {
        case ShippingType.Standard: return 5m;
        case ShippingType.Express: return 15m;
        case ShippingType.Overnight: return 30m;
        default: throw new ArgumentException();
    }
}

// After: Strategy pattern
public interface IShippingCalculator
{
    decimal Calculate(Order order);
}

public class StandardShipping : IShippingCalculator
{
    public decimal Calculate(Order order) => 5m;
}

// ... etc
```

### IDE Refactoring Tools

Use IDE refactoring features — they're safer than manual edits:
- **Rename** (F2) — Updates all references
- **Extract Method** (Ctrl+R, M) — Creates method from selection
- **Extract Interface** — Creates interface from class
- **Move Type to File** — One class per file
- **Inline Variable** — Removes unnecessary variables

---

## Hands-On: Legacy Code Surgery

### Exercise 1: Add Tests to Untestable Code

Take this untestable class:
```csharp
public class OrderProcessor
{
    public void Process(Guid orderId)
    {
        using var conn = new SqlConnection(ConfigurationManager.ConnectionStrings["DB"].ConnectionString);
        var order = conn.QuerySingle<OrderDto>("SELECT * FROM Orders WHERE Id = @Id", new { Id = orderId });

        if (order.Status != "Pending")
            throw new InvalidOperationException("Order not pending");

        var emailer = new SmtpClient();
        emailer.Send(new MailMessage("noreply@example.com", order.CustomerEmail, "Processing", "Your order is being processed"));

        conn.Execute("UPDATE Orders SET Status = 'Processing' WHERE Id = @Id", new { Id = orderId });
    }
}
```

Tasks:
1. Identify the seams
2. Add interfaces for dependencies
3. Write characterization tests
4. Refactor to make it testable

### Exercise 2: Strangler Fig Migration

Your OrderFlow has a legacy `PriceCalculator`. Replace it:
1. Create `IOrderPriceCalculator` interface
2. Wrap legacy in `LegacyPriceCalculator`
3. Build new `CleanPriceCalculator`
4. Add comparison logging
5. Feature flag the switch
6. Monitor for mismatches
7. Remove legacy when confident

### Exercise 3: Break Up a God Class

Find (or create) a class with too many responsibilities. Split it:
1. List all the responsibilities
2. Group related methods
3. Extract new classes
4. Update callers
5. Verify tests still pass

---

## Deliverables

1. **Characterization test suite** — Tests documenting existing behavior
2. **Seam identification guide** — Analysis of a legacy class showing where seams can be added
3. **Strangler fig migration plan** — Document planning the replacement of a legacy component
4. **Refactoring commit history** — Series of commits showing safe, incremental refactoring

---

## Resources

### Must-Read
- [Working Effectively with Legacy Code](https://www.oreilly.com/library/view/working-effectively-with/0131177052/) — The entire book
- [Refactoring: Improving the Design of Existing Code](https://refactoring.com/) — Martin Fowler

### Videos
- [Sandi Metz: Therapeutic Refactoring](https://www.youtube.com/watch?v=J4dlF0kcThQ)
- [Woody Zuill: Turn Up the Good](https://www.youtube.com/watch?v=7B4YloHIyM0)

### Articles
- [Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html) — Martin Fowler
- [Working with Legacy Code](https://understandlegacycode.com/) — Nicolas Carlo

---

## Module Summary

After completing Module 13, you can:

| Skill | Evidence |
|-------|----------|
| Navigate unfamiliar code | System map created through exploration |
| Debug systematically | Issues diagnosed using logs and traces |
| Write characterization tests | Safety net for legacy code changes |
| Find and use seams | Dependencies isolated for testing |
| Apply strangler fig pattern | Incremental migration without rewrites |
| Refactor safely | Changes made without breaking behavior |

These skills make you valuable on any team, in any codebase.

---

## Reflection Questions

1. When is it worth rewriting vs refactoring?
2. How do you balance "cleaning up" with "shipping features"?
3. What's the minimum test coverage you need before changing code?
4. How do you convince a team to invest in legacy code improvements?

---

**Complete!** Return to the [module index](../00-introduction/README.md) or review any modules you want to strengthen.
