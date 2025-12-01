# Part 1 — Code Reviews & Collaboration

## Why This Matters

Code review isn't about finding bugs — it's about sharing knowledge, maintaining standards, and building trust. A good code review makes everyone on the team better.

> "Code review is not about finding defects. It's about knowledge sharing and improving the design of the system." — Google Engineering Practices

Every PR is a teaching moment. Every review is a learning opportunity.

---

## What Goes Wrong Without This

### The Drive-By PR

```markdown
## PR: Update order service

Changed some stuff. LGTM?

Files changed: 47
Lines: +1,247 -892
```

**Reviewer's thoughts:**
- "What problem does this solve?"
- "Why these changes?"
- "Did they test this?"
- "I don't have time for this..."

**Result:** Rubber-stamp approval, bugs shipped to production.

### The Nitpick War

```markdown
Reviewer 1: "Use var instead of explicit type"
Reviewer 2: "I prefer explicit types for clarity"
Author: "The style guide says var"
Reviewer 1: "Style guide is outdated"

15 comments later: Still arguing about var
Real issue: Missing null check that will crash production
```

### The Knowledge Silo

```
Senior dev writes feature.
Senior dev reviews their own code.
Senior dev merges.

Nobody else understands the code.
Senior dev leaves.
Team is lost.
```

---

## The Anatomy of a Great PR

### PR Template

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->

## Summary

<!-- What does this PR do? One paragraph max. -->

## Problem

<!-- What problem does this solve? Link to issue/ticket. -->

Closes #123

## Solution

<!-- How does this solve the problem? Key decisions made. -->

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that causes existing functionality to change)
- [ ] Refactoring (no functional changes)
- [ ] Documentation update

## Testing

<!-- How was this tested? -->

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manually tested in development environment

**Test Evidence:**
<!-- Screenshots, curl commands, test output -->

## Checklist

- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing tests pass locally
- [ ] I have updated documentation as needed
- [ ] I have considered security implications

## Screenshots (if applicable)

<!-- Before/after screenshots for UI changes -->

## Deployment Notes

<!-- Any special deployment steps? Feature flags? -->

## Follow-up Tasks

<!-- What needs to happen after this merges? -->

- [ ] Update monitoring alerts
- [ ] Notify dependent teams
```

### Example: Great PR Description

```markdown
## Summary

Add date range filtering to the order search API. Customers can now
filter orders by `startDate` and `endDate` query parameters.

## Problem

Customers with high order volumes can't efficiently find orders from
specific time periods. Currently they must paginate through all orders.

Closes #234 (Customer request from Acme Corp)

## Solution

Added optional `startDate` and `endDate` query parameters to
`GET /api/orders`. Both parameters are validated as ISO 8601 dates.

**Key decisions:**
- Used inclusive date range (startDate <= orderDate <= endDate)
- Default: no date filter (returns all orders)
- Added index on `Orders.OrderDate` for performance

## Testing

- Added unit tests for DateRange validation
- Added integration tests for API endpoint
- Load tested with 1M orders, queries return in <100ms

```bash
# Test command
curl "https://localhost:5001/api/orders?startDate=2024-01-01&endDate=2024-01-31"
```

## Screenshots

![API response with date filter](./screenshots/date-filter-response.png)

## Deployment Notes

- Database migration adds index (runs async, non-blocking)
- Feature is backwards compatible, no breaking changes
```

---

## Code Review Best Practices

### The Reviewer Mindset

```
Before reviewing, ask yourself:
1. Do I understand what this PR is trying to do?
2. Do I have enough context to review it well?
3. Do I have the time to review it properly now?

If any answer is "no", ask for clarification or schedule later.
```

### Conventional Comments

Use prefixes to clarify intent:

| Prefix | Meaning | Blocking? |
|--------|---------|-----------|
| `praise:` | Something done well | No |
| `nitpick:` | Minor style preference | No |
| `suggestion:` | Improvement idea | No |
| `question:` | Need clarification | Maybe |
| `thought:` | Random observation | No |
| `issue:` | Something wrong | Yes |
| `blocking:` | Must fix before merge | Yes |

### Example Comments

```markdown
# Good - Clear and actionable
praise: Great use of the Result pattern here! Makes error handling very clear.

nitpick: Consider extracting this magic number to a constant. Not blocking.

suggestion: We could use a switch expression here for cleaner syntax:
```csharp
return status switch
{
    OrderStatus.Pending => "Awaiting payment",
    OrderStatus.Shipped => "On the way",
    _ => "Unknown"
};
```

question: Is there a reason we're not using the existing DateValidator?
I might be missing context.

issue: This will throw NullReferenceException if customer.Address is null.
We need to add a null check here.

blocking: This bypasses authorization. All users can now see all orders.
Must add authorization check before merge.
```

### Review Checklist

```markdown
## Code Review Checklist

### Correctness
- [ ] Does the code do what the PR claims?
- [ ] Are edge cases handled?
- [ ] Are error conditions handled appropriately?

### Security
- [ ] No SQL injection vulnerabilities
- [ ] No XSS vulnerabilities
- [ ] No hardcoded secrets
- [ ] Authorization checks in place

### Performance
- [ ] No N+1 queries
- [ ] No unnecessary allocations in hot paths
- [ ] Appropriate caching where needed

### Maintainability
- [ ] Code is readable and self-documenting
- [ ] No dead code or commented-out code
- [ ] Follows existing patterns in the codebase

### Testing
- [ ] Tests cover the happy path
- [ ] Tests cover error cases
- [ ] Tests are readable and maintainable
```

---

## Design Documents & ADRs

### When to Write a Design Doc

```
Write a design doc when:
- Change affects multiple services
- Change has significant performance implications
- Change introduces new patterns or technologies
- Multiple valid approaches exist
- Decision will be hard to reverse

Don't write a design doc when:
- Change is small and obvious
- It's a bug fix with clear solution
- You're procrastinating on actual work
```

### Design Document Template

```markdown
# Design: Order Processing Pipeline Refactor

**Author:** @yourname
**Date:** 2024-01-15
**Status:** Proposed | Approved | Implemented | Deprecated
**Reviewers:** @teammate1, @teammate2

## Context

<!-- What's the current situation? Why are we considering this? -->

The current order processing is synchronous, causing 2-3 second response
times during peak load. This affects customer experience and increases
timeout errors.

## Goals

<!-- What are we trying to achieve? Be specific and measurable. -->

1. Reduce order placement response time to <500ms
2. Handle 10x current order volume
3. Maintain data consistency guarantees

## Non-Goals

<!-- What are we explicitly NOT trying to solve? -->

- Changing the order data model
- Real-time inventory updates (separate initiative)
- Payment provider migration

## Options Considered

### Option 1: Async Processing with RabbitMQ

**Pros:**
- Proven pattern in our stack
- Team has experience
- Good tooling (MassTransit)

**Cons:**
- Adds operational complexity
- Need to handle eventual consistency

**Estimated effort:** 2 sprints

### Option 2: Database Optimization

**Pros:**
- Simpler architecture
- No new infrastructure

**Cons:**
- Limited scalability
- Doesn't solve underlying issue

**Estimated effort:** 1 sprint

### Option 3: CQRS with Event Sourcing

**Pros:**
- Ultimate scalability
- Full audit trail

**Cons:**
- Significant complexity
- Team has no experience
- Over-engineered for our needs

**Estimated effort:** 4+ sprints

## Recommendation

Option 1: Async Processing with RabbitMQ

This balances complexity with scalability. We can leverage existing
MassTransit patterns and the team's experience.

## Implementation Plan

1. Add RabbitMQ infrastructure (Week 1)
2. Implement async order placement (Week 2)
3. Update client to handle async responses (Week 3)
4. Migration and monitoring (Week 4)

## Metrics

- P99 response time for order placement
- Order processing error rate
- Message queue depth

## Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Message loss | Low | High | Implement outbox pattern |
| Inconsistent state | Medium | Medium | Add compensation logic |

## Open Questions

1. How long should we wait before timing out an order?
2. Should we notify customers of async processing?

## Appendix

### References
- [Our RabbitMQ Standards](./rabbitmq-standards.md)
- [MassTransit Documentation](https://masstransit-project.com/)

### Diagrams

```
[Client] --> [API] --> [RabbitMQ] --> [Order Processor]
                                           |
                                           v
                                      [Database]
```
```

### Architecture Decision Record (ADR)

```markdown
# ADR-001: Use MassTransit for Message Processing

**Date:** 2024-01-15
**Status:** Accepted
**Deciders:** @yourname, @techlead, @architect

## Context

We need a way to process orders asynchronously to improve response times.
Options include raw RabbitMQ, MassTransit, NServiceBus, or AWS SQS.

## Decision

We will use MassTransit with RabbitMQ.

## Rationale

1. **Team familiarity** - 3/5 team members have MassTransit experience
2. **Open source** - No licensing costs
3. **RabbitMQ integration** - Works well with our existing infrastructure
4. **Testing support** - In-memory transport for unit tests

We considered NServiceBus but the licensing cost ($4k/dev/year) wasn't
justified for our use case.

## Consequences

### Positive
- Faster time to production
- Easier debugging with familiar tools
- Strong community support

### Negative
- Tied to RabbitMQ (harder to migrate to SQS later)
- MassTransit learning curve for 2 team members

### Neutral
- Need to document MassTransit patterns for the team

## Related ADRs
- ADR-002: Message Serialization Format (upcoming)
```

---

## Async Collaboration

### Effective Issue Writing

```markdown
# Issue: Order search returns wrong results for date filters

## Description

When filtering orders by date range, orders from the boundary dates
are sometimes excluded incorrectly.

## Steps to Reproduce

1. Create order on 2024-01-15 at 23:59 UTC
2. Search with startDate=2024-01-15
3. Order is not returned

## Expected Behavior

Order should be included when its date matches the filter.

## Actual Behavior

Order is excluded from results.

## Environment

- API version: 2.3.1
- Database: PostgreSQL 16
- Timezone: UTC

## Investigation So Far

Looked at OrderRepository.cs:45, the comparison uses `<` instead of `<=`.

## Suggested Fix

Change line 45 from:
```csharp
.Where(o => o.OrderDate < endDate)
```
To:
```csharp
.Where(o => o.OrderDate <= endDate)
```

## Impact

Medium - Affects customers using date filters, ~15% of searches.
```

### GitHub Discussions for RFCs

```markdown
# RFC: API Versioning Strategy

## Summary

Propose a versioning strategy for our public API to allow breaking changes
without disrupting existing clients.

## Motivation

We need to make breaking changes to the order response schema, but have
external clients that depend on the current format.

## Proposed Options

### Option A: URL Path Versioning
`/api/v1/orders` → `/api/v2/orders`

### Option B: Header Versioning
`Api-Version: 2024-01-15`

### Option C: Query Parameter
`/api/orders?version=2`

## Questions for Discussion

1. Which option aligns with industry standards?
2. How long should we support old versions?
3. How do we handle versioning in documentation?

## Voting

React with emoji to vote:
- 👍 Option A (URL Path)
- ❤️ Option B (Header)
- 🎉 Option C (Query Parameter)
```

---

## Giving and Receiving Feedback

### The Feedback Sandwich (Don't Do This)

```
❌ "The code is great, but you have a critical security bug,
    but overall nice work!"

The middle (important) part gets lost.
```

### Direct, Kind Feedback

```
✅ "I found a security issue that needs fixing before we merge.
    Line 45 allows SQL injection. Here's how to fix it:
    [code example]
    Happy to pair on this if helpful!"

Clear problem, clear solution, offer of help.
```

### Receiving Feedback Gracefully

```
Reviewer: "This approach won't scale. Consider using X instead."

❌ Bad response: "That's how we've always done it."
❌ Bad response: "I don't have time to change this."
❌ Bad response: [No response, just changes code]

✅ Good response: "Good point about scalability. I wasn't aware of X.
                   Can you point me to an example? I'll update the PR."

✅ Good response: "I considered X but went with Y because [reason].
                   What do you think about that trade-off?"
```

---

## Hands-On Exercise: Review Practice

### Step 1: Create PR Template

Add `.github/PULL_REQUEST_TEMPLATE.md` to your repository using the
template provided above.

### Step 2: Review 5 Real PRs

Find 5 PRs in open-source projects or your work codebase:
- Focus on code you don't fully understand
- Use conventional comments
- Time yourself (aim for 15-30 min per PR)

Document:
- What did you learn?
- What was hard about reviewing?
- How did the author respond to feedback?

### Step 3: Write a Design Document

Pick a recent technical decision and write an ADR:
- What was the context?
- What options did you consider?
- Why did you choose this option?
- What are the consequences?

### Step 4: Reflect on Your Communication

Review your last 5 PRs:
- Would a new team member understand them?
- Did reviewers ask clarifying questions?
- What could you improve?

---

## Deliverables

1. **PR Template**: Committed to repository
2. **Review Log**: 5 reviews with comments and learnings
3. **Design Document/ADR**: For a real decision
4. **Self-Assessment**: Reflection on communication growth

---

## Reflection Questions

1. What's the difference between a blocking and non-blocking comment?
2. How do you balance thoroughness with review speed?
3. When should you write a design doc vs. just write the code?
4. How has your PR writing changed after this module?

---

## Resources

### Code Review
- [How to Do Code Reviews Like a Human](https://mtlynch.io/human-code-reviews-1/) - Michael Lynch
- [Google Engineering Practices](https://google.github.io/eng-practices/)
- [Conventional Comments](https://conventionalcomments.org/)
- [Code Review Best Practices](https://www.youtube.com/watch?v=a9_0UUUNt-Y) - Trisha Gee

### Design Documents
- [Design Docs at Google](https://www.industrialempathy.com/posts/design-docs-at-google/)
- [Architecture Decision Records](https://adr.github.io/)
- [RFC Process](https://engineering.squarespace.com/blog/2019/the-power-of-yes-if)

### Communication
- [The Staff Engineer's Path](https://www.oreilly.com/library/view/the-staff-engineers/9781098118723/) - Tanya Reilly
- [Difficult Conversations](https://www.penguinrandomhouse.com/books/331191/difficult-conversations-by-douglas-stone-bruce-patton-and-sheila-heen/)

---

**Next:** [Part 2 — Developer Productivity & Debugging](./part-2.md)
