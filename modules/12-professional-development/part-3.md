# Part 3 — Career Skills & Communication

## Why This Matters

Your code doesn't speak for itself. Without documentation, nobody knows what you built. Without case studies, nobody knows your impact. Without presentation skills, nobody hears your ideas.

> "You don't get promoted for writing great code. You get promoted for solving important problems and making sure the right people know about it." — Unknown tech lead

The engineers who advance are the ones who can articulate their value.

---

## What Goes Wrong Without This

### The Invisible Engineer

```
Annual review time:

Manager: "What were your key accomplishments this year?"

Developer: "I... worked on the order service... fixed some bugs...
            did some performance stuff..."

Manager: "Can you quantify the impact?"

Developer: "Not really... I didn't track anything..."

Manager: "We'll need more concrete examples for a promotion case."

Developer gets average rating.
Developer who documented their work gets promoted.
```

### The Undocumented System

```
New hire joins team:

New Hire: "How do I run this service locally?"
Team: "Oh, just... you know... set up the things."

New Hire: "What things?"
Team: "The database, Redis, the queue... ask Dave, he knows."

Dave: "I'm on vacation for 2 weeks."

New hire spends 3 days figuring out what takes 30 minutes with docs.
```

### The Failed Interview

```
Interview:

Interviewer: "Tell me about a challenging technical problem you solved."

Candidate: "Um... there was this bug... it was hard... I fixed it?"

Interviewer: "What was the approach? What trade-offs did you consider?"

Candidate: "I don't remember the details..."

Interviewer: Moves to next candidate.
```

---

## Documentation as a Superpower

### The Documentation Types

```
┌─────────────────────────────────────────────────────────────────┐
│                   DOCUMENTATION TYPES                            │
│                   (Diátaxis Framework)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    LEARNING-ORIENTED                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              TUTORIALS                                   │    │
│  │   "Learning by doing"                                   │    │
│  │   Step-by-step guides for beginners                     │    │
│  │   Example: "Your First OrderFlow API"                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              EXPLANATION                                 │    │
│  │   "Understanding"                                       │    │
│  │   Concepts, architecture, why decisions were made       │    │
│  │   Example: "Why We Use Event Sourcing"                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│                    GOAL-ORIENTED                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              HOW-TO GUIDES                               │    │
│  │   "Solving problems"                                    │    │
│  │   Specific tasks for experienced users                  │    │
│  │   Example: "How to Add a New Payment Provider"          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              REFERENCE                                   │    │
│  │   "Information"                                         │    │
│  │   API docs, configuration options, specs                │    │
│  │   Example: "API Endpoint Reference"                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### README Template

```markdown
# OrderFlow API

> Order management system for e-commerce platforms.

## Quick Start

```bash
# Clone and start
git clone https://github.com/yourorg/orderflow.git
cd orderflow
make up

# Verify it's running
curl http://localhost:5000/health
```

## Prerequisites

- Docker Desktop 4.x+
- .NET 8 SDK (for local development)
- VS Code or Rider

## Development Setup

### Using Docker (Recommended)

```bash
# Start all services
make up

# View logs
make logs

# Stop everything
make down
```

Services available:
- API: http://localhost:5000
- Swagger: http://localhost:5000/swagger
- Seq (logs): http://localhost:8081
- RabbitMQ: http://localhost:15672 (guest/guest)

### Local Development

```bash
# Install dependencies
dotnet restore

# Run database migrations
dotnet ef database update --project src/OrderFlow.Infrastructure

# Run the API
dotnet run --project src/OrderFlow.Api
```

## Project Structure

```
src/
├── OrderFlow.Api/           # HTTP API layer
├── OrderFlow.Application/   # Business logic
├── OrderFlow.Domain/        # Domain models
├── OrderFlow.Infrastructure/# Data access, external services
└── OrderFlow.Worker/        # Background processing
tests/
├── OrderFlow.UnitTests/
└── OrderFlow.IntegrationTests/
```

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `ConnectionStrings__Database` | PostgreSQL connection | Required |
| `ConnectionStrings__Redis` | Redis connection | Required |
| `RabbitMQ__Host` | RabbitMQ hostname | localhost |

See [Configuration Guide](./docs/configuration.md) for all options.

## API Examples

### Create Order

```bash
curl -X POST http://localhost:5000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cust_123",
    "items": [
      {"productId": "prod_456", "quantity": 2}
    ]
  }'
```

### Get Order

```bash
curl http://localhost:5000/api/orders/ord_789
```

## Troubleshooting

### Database connection fails

```bash
# Check if PostgreSQL is running
docker compose ps

# Check logs
docker compose logs postgres

# Reset database
docker compose down -v && docker compose up -d
```

### Ports already in use

```bash
# Find what's using the port
lsof -i :5000

# Use different ports
API_PORT=5001 make up
```

## Contributing

1. Create a branch: `git checkout -b feature/my-feature`
2. Make changes and test: `make test`
3. Submit PR with description

See [Contributing Guide](./CONTRIBUTING.md) for details.

## Architecture

See [Architecture Documentation](./docs/architecture.md).

## License

MIT - see [LICENSE](./LICENSE)
```

### Runbook Template

```markdown
# Runbook: Order Processing Service

## Service Overview

| Property | Value |
|----------|-------|
| Service Name | order-processing-service |
| Owner | Orders Team (@orders-team) |
| Criticality | High |
| SLA | 99.9% uptime |
| On-Call | #orders-oncall |

## Dashboards

- [Grafana Dashboard](https://grafana.internal/d/orders)
- [Application Insights](https://portal.azure.com/...)
- [Service Health](https://status.internal/orders)

## Common Alerts

### HIGH: OrderProcessingFailureRate > 5%

**What it means:** More than 5% of orders are failing to process.

**Impact:** Customer orders not being fulfilled.

**Investigation steps:**
1. Check [error logs in Seq](https://seq.internal/orders?error=true)
2. Look for patterns (specific customer, product, payment method)
3. Check dependent services (payment, inventory)

**Common causes:**
- Payment gateway timeout (check payment service)
- Database connection pool exhausted (check DB metrics)
- Invalid order data (check validation logs)

**Mitigation:**
```bash
# If payment gateway issues, enable fallback
kubectl set env deployment/order-service PAYMENT_FALLBACK_ENABLED=true

# If database issues, scale up connections
kubectl edit configmap order-service-config
# Increase ConnectionPool.MaxSize
```

### MEDIUM: OrderProcessingLatencyP99 > 5s

**What it means:** 99th percentile of order processing taking > 5 seconds.

**Impact:** Poor customer experience, potential timeouts.

**Investigation steps:**
1. Check [latency breakdown](https://grafana.internal/d/orders-latency)
2. Identify slow step (validation, inventory, payment)
3. Check for database slow queries

**Common causes:**
- Missing database index
- N+1 query issue
- External service slow

## Operational Tasks

### Restart Service

```bash
# Rolling restart (no downtime)
kubectl rollout restart deployment/order-service

# Verify
kubectl rollout status deployment/order-service
```

### Scale Service

```bash
# Scale up during high load
kubectl scale deployment/order-service --replicas=10

# Scale down after
kubectl scale deployment/order-service --replicas=3
```

### View Logs

```bash
# Recent logs
kubectl logs -l app=order-service --tail=100

# Stream logs
kubectl logs -l app=order-service -f

# Search for errors
kubectl logs -l app=order-service | grep -i error
```

### Database Operations

```bash
# Connect to database
kubectl exec -it $(kubectl get pod -l app=postgres -o name) -- psql -U orderflow

# Check slow queries
SELECT * FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;

# Check connection count
SELECT count(*) FROM pg_stat_activity WHERE datname = 'orderflow';
```

## Incident Response

### Severity Levels

| Severity | Response Time | Example |
|----------|---------------|---------|
| P1 | Immediately | Orders not processing at all |
| P2 | 30 minutes | 50%+ orders failing |
| P3 | 4 hours | Performance degradation |
| P4 | Next business day | Minor issue, workaround exists |

### Escalation Path

1. On-call engineer (@orders-oncall)
2. Team lead (@orders-lead)
3. Engineering manager (@eng-manager)
4. CTO (P1 only)

### Post-Incident

1. Create incident ticket
2. Write timeline of events
3. Schedule blameless postmortem
4. Document action items
5. Update runbook if needed

## Deployment

See [Deployment Guide](./deployment.md)

## Contacts

- **Team Slack:** #orders-team
- **On-Call:** #orders-oncall
- **Manager:** @orders-lead
- **Escalation:** @eng-manager
```

---

## Case Studies

### The STAR Method

```
Situation: What was the context?
Task:      What were you trying to accomplish?
Action:    What did you do?
Result:    What was the outcome? (Quantify!)
```

### Case Study Template

```markdown
# Case Study: Order Processing Performance Optimization

## Summary

Reduced order processing latency by 80% (5s → 1s P99) by implementing
asynchronous processing with RabbitMQ, enabling the system to handle
10x more orders during peak periods.

## Situation

In Q3 2024, our e-commerce platform experienced significant performance
issues during Black Friday preparation. The order processing endpoint was
taking 5+ seconds at P99, causing:

- Customer cart abandonment (12% increase)
- Payment gateway timeouts
- Load balancer 504 errors

The system could handle only 50 orders/second before degradation.

## Task

As the tech lead for the Orders team, I was responsible for:

1. Investigating the root cause
2. Proposing solutions within a 4-week timeline
3. Implementing with zero downtime
4. Meeting Black Friday traffic requirements (500 orders/second)

## Action

### Week 1: Investigation

- Profiled the endpoint using dotnet-trace
- Identified synchronous calls to payment, inventory, and email services
- Found N+1 query issue in order validation

Key finding: 80% of latency was waiting on external services.

### Week 2: Solution Design

Evaluated three approaches:

| Approach | Effort | Risk | Scalability |
|----------|--------|------|-------------|
| Database optimization | Low | Low | Limited |
| Async processing | Medium | Medium | High |
| CQRS + Event Sourcing | High | High | Very High |

Chose async processing as the best balance of effort vs. impact.

### Week 3-4: Implementation

1. Added RabbitMQ with MassTransit
2. Created OrderPlaced event
3. Moved payment, inventory, email to consumers
4. Implemented idempotency with outbox pattern
5. Added monitoring and alerting

```csharp
// Before: Synchronous
public async Task<Order> PlaceOrder(CreateOrderRequest request)
{
    var order = CreateOrder(request);        // 100ms
    await _payment.ChargeAsync(order);       // 2000ms
    await _inventory.ReserveAsync(order);    // 1500ms
    await _email.SendConfirmationAsync(order); // 500ms
    return order;
}

// After: Asynchronous
public async Task<Order> PlaceOrder(CreateOrderRequest request)
{
    var order = CreateOrder(request);        // 100ms
    await _outbox.PublishAsync(new OrderPlaced(order)); // 50ms
    return order;
}
```

### Rollout

- Deployed behind feature flag
- 1% → 10% → 50% → 100% over 5 days
- Monitored error rates and latency at each step

## Result

### Performance

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| P99 Latency | 5.2s | 1.1s | 79% reduction |
| P50 Latency | 2.1s | 0.3s | 86% reduction |
| Max Throughput | 50/s | 800/s | 16x increase |

### Business Impact

- Cart abandonment reduced by 8%
- Customer support tickets down 40%
- Successfully handled 2.5M orders on Black Friday
- Zero incidents during peak period

### Technical Debt Reduced

- Removed synchronous coupling between services
- Improved testability (can test API without external services)
- Established pattern for future async operations

## Lessons Learned

1. **Measure first** — Profiling identified the real bottleneck
2. **Start simple** — Async processing was enough; didn't need CQRS
3. **Feature flags are essential** — Safe rollout prevented incidents
4. **Monitor the transition** — Caught a race condition at 10% rollout

## Technologies Used

- .NET 8
- RabbitMQ
- MassTransit
- PostgreSQL
- Prometheus/Grafana
```

### Case Study for Interviews

```
The "elevator pitch" version (1 minute):

"At my previous role, we had a performance problem with our order
processing — 5 second response times during peak load, causing 12%
cart abandonment.

I led the investigation, identified that 80% of latency was synchronous
external calls, and implemented async processing with RabbitMQ.

The result was 80% latency reduction, 16x throughput improvement, and
we successfully handled 2.5 million orders on Black Friday with zero
incidents. The project also established patterns the team now uses for
all async operations."

Follow-up questions to prepare:

Q: What alternatives did you consider?
A: Database optimization (too limited) and CQRS (too complex for the
   timeline). Async processing hit the sweet spot.

Q: What was the hardest part?
A: Handling edge cases with eventual consistency — we had to implement
   idempotency and a compensation flow for partial failures.

Q: What would you do differently?
A: I'd implement distributed tracing earlier. We had to add it mid-project
   when debugging a race condition, and it would have saved time.
```

---

## Technical Presentations

### Lightning Talk Structure

```
5-Minute Lightning Talk Template:

0:00 - Hook (30 sec)
  "How many of you have dealt with [relatable problem]?"
  "What if I told you [surprising statement]?"

0:30 - Problem (1 min)
  - Describe the pain
  - Show real example
  - Make audience feel it

1:30 - Solution (2 min)
  - Key insight
  - How it works (simple!)
  - Demo or code example

3:30 - Results (30 sec)
  - Quantified impact
  - Before/after comparison

4:00 - Call to Action (30 sec)
  - One thing audience should do
  - Link to learn more

4:30 - Questions
```

### Demo Tips

```
Before the demo:
1. Have a backup (screenshot, video)
2. Increase font size (16pt minimum)
3. Close unnecessary apps/tabs
4. Disable notifications
5. Test with the actual projector/screen

During the demo:
1. Explain what you're about to do before doing it
2. Wait for commands to complete
3. Highlight important output
4. Have "checkpoints" you can return to

Demo script example:
"I'm going to create an order and show you the async processing.
[types command]
Notice the response comes back immediately.
[shows response]
Now if we look at the message queue...
[switches to RabbitMQ UI]
We can see the message being processed."
```

### Presentation Mistakes to Avoid

```
❌ Too much text on slides
❌ Reading from slides
❌ No narrative arc
❌ Assuming audience knows context
❌ Skipping the "why"
❌ No demo or concrete examples
❌ Running over time
❌ Apologizing for slides/content

✅ One idea per slide
✅ Use presenter notes
✅ Tell a story
✅ Start with context
✅ Explain why it matters
✅ Show, don't just tell
✅ Practice timing
✅ Confidence in your content
```

---

## Interview Preparation

### Behavioral Interview Framework

```
Common question types:

CONFLICT
- "Tell me about a time you disagreed with a teammate"
- "Describe a situation where you received critical feedback"

LEADERSHIP
- "Describe a time you led a project"
- "Tell me about mentoring someone"

FAILURE
- "Tell me about a time you failed"
- "Describe a project that didn't go as planned"

ACCOMPLISHMENT
- "What's your proudest technical achievement?"
- "Describe a complex problem you solved"

GROWTH
- "Tell me about something you learned recently"
- "Describe how you've grown as an engineer"
```

### Answer Template

```markdown
## Prepared Answer: Disagreement with Teammate

### Situation
On the Orders team, we needed to choose between REST and gRPC for
a new internal service. My teammate strongly advocated for gRPC
citing performance benefits.

### Task
As the tech lead, I needed to make a decision that the team could
support while considering our constraints and goals.

### Action
Instead of arguing, I suggested we evaluate both options objectively.
We created a decision matrix with criteria: performance, team experience,
debugging ease, and client compatibility.

We ran a proof of concept for both approaches and measured actual
performance in our use case. I also invited my teammate to lead the
gRPC evaluation to ensure fair assessment.

### Result
The data showed REST was better for our case — team experience and
debugging benefits outweighed the 20ms latency difference. My teammate
agreed with the data-driven approach.

More importantly, we established a decision framework we now use for
all technical choices, and my teammate appreciated being heard.

### Lessons
1. Data > opinions in technical debates
2. Involving skeptics in evaluation builds buy-in
3. The process matters as much as the decision
```

### System Design Interview

```
Structure your answer:

1. REQUIREMENTS (5 min)
   - Clarify functional requirements
   - Clarify non-functional requirements
   - Establish constraints and assumptions

2. HIGH-LEVEL DESIGN (10 min)
   - Draw the main components
   - Explain data flow
   - Identify APIs

3. DEEP DIVE (15 min)
   - Focus on interesting/challenging parts
   - Discuss trade-offs
   - Show depth of knowledge

4. WRAP UP (5 min)
   - Address bottlenecks
   - Discuss scaling
   - Consider edge cases

Tips:
- Think out loud
- Ask clarifying questions
- Mention trade-offs
- Reference real systems you've built
- Draw as you talk
```

---

## Building Your Brand

### Visibility Strategies

```
Ways to increase visibility (without being annoying):

1. Internal
   - Present at team meetings
   - Write internal blog posts
   - Host brown bag sessions
   - Document decisions in ADRs
   - Participate in architecture reviews

2. External (optional)
   - Conference talks
   - Blog posts
   - Open source contributions
   - Meetup presentations
   - Technical writing

3. Daily
   - Clear PR descriptions
   - Detailed design docs
   - Helpful code reviews
   - Slack answers with context
```

### Career Progression Framework

```
Junior → Mid-Level → Senior → Staff

Junior:
- Execute defined tasks
- Learn patterns and practices
- Ask questions, take feedback

Mid-Level:
- Work independently
- Make local decisions
- Mentor juniors

Senior:
- Lead projects
- Make system-level decisions
- Influence team practices
- Mentor mid-levels

Staff:
- Lead multiple teams/projects
- Set technical direction
- Solve cross-org problems
- Grow senior engineers

Key question: Are you operating at the next level?
If yes consistently, you're ready for promotion.
```

---

## Hands-On Exercise: Career Portfolio

### Step 1: Document Current Work

For each major project in the last year:
- Write a one-paragraph summary
- Quantify the impact (numbers!)
- Note technologies used
- Identify key learnings

### Step 2: Create README

Write or improve documentation for a project:
- Quick start guide
- Project structure explanation
- Troubleshooting section

### Step 3: Write Case Study

Pick your best project and write a full case study:
- Use the STAR method
- Include before/after metrics
- Prepare the 1-minute version

### Step 4: Practice Presentation

Give a 5-minute lightning talk to a friend/colleague:
- Record yourself
- Get feedback
- Iterate and improve

### Step 5: Prepare Interview Answers

Write out answers for 5 behavioral questions:
- Conflict resolution
- Leadership example
- Failure and recovery
- Technical achievement
- Learning experience

---

## Deliverables

1. **README**: Complete project documentation
2. **Runbook**: Operational documentation for one service
3. **Case Study**: Full write-up of a key project
4. **Presentation**: Recorded lightning talk
5. **Interview Prep**: Written STAR-format answers

---

## Reflection Questions

1. If you left tomorrow, could someone else run your projects?
2. How would you explain your biggest achievement to a non-technical person?
3. What's the most valuable thing you've learned this year?
4. Where do you want to be in 2 years, and what's blocking that?

---

## Resources

### Documentation
- [Diátaxis](https://diataxis.fr/) - Documentation framework
- [Write the Docs](https://www.writethedocs.org/)
- [Google Documentation Best Practices](https://google.github.io/styleguide/docguide/)

### Career Development
- [The Staff Engineer's Path](https://www.oreilly.com/library/view/the-staff-engineers/9781098118723/) - Tanya Reilly
- [StaffEng.com](https://staffeng.com/) - Will Larson
- [Refactoring Career](https://www.youtube.com/watch?v=7uYF4DdKzDI) - Sarah Mei

### Interviews
- [Cracking the Coding Interview](https://www.crackingthecodinginterview.com/)
- [System Design Interview](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)
- [Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)

### Presentations
- [Talk Like TED](https://www.amazon.com/Talk-Like-TED-Public-Speaking-Secrets/dp/1250041120)
- [The Art of Public Speaking](https://www.youtube.com/watch?v=Unzc731iCUY) - Chris Anderson

---

**Congratulations!** You've completed the Professional Development module.

Return to [Module 12 Overview](./README.md) or the [Roadmap Home](../../README.md)
