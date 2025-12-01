# Curated Resources for .NET Backend Engineers

This is the definitive resource list for the roadmap. Every link has been chosen because it teaches the "why" — not just the "how."

---

## Books (Must-Read)

### Fundamentals
| Book | Author | Why Read It |
|------|--------|-------------|
| [C# in Depth, 4th Edition](https://csharpindepth.com/) | Jon Skeet | The authoritative deep-dive into C# language features. Read chapters relevant to each concept, not cover-to-cover. |
| [Pro .NET Memory Management](https://prodotnetmemory.com/) | Konrad Kokosa | Understand what actually happens when you allocate memory. Essential for performance work. |
| [Designing Data-Intensive Applications](https://dataintensive.net/) | Martin Kleppmann | The best explanation of distributed systems tradeoffs ever written. Read before Module 05-06. |
| [Release It! 2nd Edition](https://pragprog.com/titles/mnee2/release-it-second-edition/) | Michael Nygard | Production failure patterns and how to prevent them. Full of war stories. |

### Architecture & Design
| Book | Author | Why Read It |
|------|--------|-------------|
| [Domain-Driven Design](https://www.domainlanguage.com/ddd/) | Eric Evans | The original. Dense but essential for understanding bounded contexts. |
| [Implementing Domain-Driven Design](https://www.informit.com/store/implementing-domain-driven-design-9780321834577) | Vaughn Vernon | More practical than Evans. Read this first if Evans feels too abstract. |
| [Clean Architecture](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/) | Robert Martin | Controversial but influential. Understand it so you can decide what to keep. |
| [Building Microservices, 2nd Edition](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/) | Sam Newman | When and how to split services. Includes strong warnings about premature decomposition. |

### Professional Skills
| Book | Author | Why Read It |
|------|--------|-------------|
| [The Pragmatic Programmer, 20th Anniversary](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) | Hunt & Thomas | Career-defining advice on software craftsmanship. |
| [Working Effectively with Legacy Code](https://www.oreilly.com/library/view/working-effectively-with/0131177052/) | Michael Feathers | How to safely change code you don't understand. You'll need this. |
| [A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php) | John Ousterhout | Short, opinionated, practical. Great counterpoint to over-engineering. |

---

## YouTube Channels & Playlists

### .NET Deep Dives
| Channel | Best For | Key Videos |
|---------|----------|------------|
| [Nick Chapsas](https://www.youtube.com/@nickchapsas) | Modern .NET practices, performance | [The Fastest Way to Parse JSON](https://www.youtube.com/watch?v=8EY-lWKJcYA), [Stop Using Async Void](https://www.youtube.com/watch?v=zhCRX3B7qwY), [EF Core Best Practices](https://www.youtube.com/watch?v=qkJ9keBmQWo) |
| [Milan Jovanovic](https://www.youtube.com/@MilanJovanovicTech) | Clean Architecture, DDD | [Clean Architecture Playlist](https://www.youtube.com/playlist?list=PLYpjLpq5ZDGstQ5afRz-34o_0dexr1RGa), [Outbox Pattern](https://www.youtube.com/watch?v=XALvnX7MPeo) |
| [Raw Coding](https://www.youtube.com/@RawCoding) | ASP.NET Core internals | [Authentication Deep Dive](https://www.youtube.com/playlist?list=PLOeFnOV9YBa7dnrjpOG6lMpcyd7Wn7E8V), [Middleware Series](https://www.youtube.com/playlist?list=PLOeFnOV9YBa4LsMB9Xf7TXvkr7snEHvWR) |
| [Zoran Horvat](https://www.youtube.com/@zaboravansen) | OOP, design patterns, refactoring | [Domain Modeling Playlist](https://www.youtube.com/playlist?list=PLfGYIpXujUqKIYGNcLoJYz6b-oAWvBBIc) |

### System Design & Architecture
| Channel | Best For | Key Videos |
|---------|----------|------------|
| [Hussein Nasser](https://www.youtube.com/@haborHousemand) | Database internals, networking | [Database Indexing Explained](https://www.youtube.com/watch?v=lYh6LrSIDvY), [PostgreSQL vs MySQL](https://www.youtube.com/watch?v=btjBNKP49xE) |
| [CodeOpinion](https://www.youtube.com/@CodeOpinion) | Architecture patterns, messaging | [Event-Driven Architecture](https://www.youtube.com/watch?v=STKCRSUsyP0), [Microservices Boundaries](https://www.youtube.com/watch?v=MrV0DqTqpFU) |
| [Mark Richards](https://www.youtube.com/@markrichards5014) | Software architecture fundamentals | [Lesson Series](https://www.youtube.com/playlist?list=PLdsOZAx8I5umhnn5LLTNJbFgwA3xbycar) |

### DevOps & Infrastructure
| Channel | Best For | Key Videos |
|---------|----------|------------|
| [TechWorld with Nana](https://www.youtube.com/@TechWorldwithNana) | Docker, Kubernetes, CI/CD | [Docker Tutorial](https://www.youtube.com/watch?v=3c-iBn73dDE), [GitHub Actions](https://www.youtube.com/watch?v=R8_veQiYBjI) |
| [DevOps Toolkit](https://www.youtube.com/@DevOpsToolkit) | GitOps, cloud native | [ArgoCD Tutorial](https://www.youtube.com/watch?v=MeU5_k9ssrs) |

---

## Official Documentation (Actually Worth Reading)

### Microsoft
| Resource | Why It's Good |
|----------|--------------|
| [.NET Fundamentals](https://learn.microsoft.com/en-us/dotnet/fundamentals/) | Memory management, GC, async patterns explained properly |
| [ASP.NET Core Documentation](https://learn.microsoft.com/en-us/aspnet/core/) | Comprehensive and well-maintained |
| [EF Core Documentation](https://learn.microsoft.com/en-us/ef/core/) | Especially the [Performance](https://learn.microsoft.com/en-us/ef/core/performance/) section |
| [.NET Architecture Guides](https://dotnet.microsoft.com/en-us/learn/dotnet/architecture-guides) | Free e-books on microservices, containers, DevOps |
| [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) | How Microsoft designs APIs. Opinionated and useful. |

### External
| Resource | Why It's Good |
|----------|--------------|
| [RFC 7231 (HTTP Semantics)](https://datatracker.ietf.org/doc/html/rfc7231) | The actual HTTP spec. Read sections on methods and status codes. |
| [RFC 7807 (Problem Details)](https://datatracker.ietf.org/doc/html/rfc7807) | Standard error response format. Short and essential. |
| [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) | Security best practices with code examples |
| [12 Factor App](https://12factor.net/) | Foundational principles for cloud-native apps |
| [OpenTelemetry Documentation](https://opentelemetry.io/docs/) | The observability standard. Start with .NET docs. |

---

## Blogs (Individual Posts Worth Bookmarking)

### C# & .NET
- [Stephen Cleary on Async](https://blog.stephencleary.com/2012/02/async-and-await.html) — The definitive async/await explainer
- [Stephen Toub on ValueTask](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/) — When to use ValueTask vs Task
- [Andrew Lock's Blog](https://andrewlock.net/) — Deep ASP.NET Core tutorials
- [Steve Gordon's Blog](https://www.stevejgordon.co.uk/) — HttpClient, performance, internals

### Architecture & Design
- [Martin Fowler's Bliki](https://martinfowler.com/bliki/) — Architecture patterns and refactoring
- [Jimmy Bogard's Blog](https://www.jimmybogard.com/) — MediatR creator, practical architecture
- [Kamil Grzybek's Blog](https://www.kamilgrzybek.com/) — Modular monolith, DDD in .NET
- [Oskar Dudycz's Blog](https://event-driven.io/) — Event sourcing and messaging patterns

### Databases
- [Brent Ozar's Blog](https://www.brentozar.com/blog/) — SQL Server performance (many concepts apply to any RDBMS)
- [Use The Index, Luke](https://use-the-index-luke.com/) — Free online book on database indexing
- [Alex Xu's ByteByteGo](https://blog.bytebytego.com/) — System design with database focus

### Security
- [Troy Hunt's Blog](https://www.troyhunt.com/) — Practical web security
- [Scott Brady's Blog](https://www.scottbrady91.com/) — OAuth, OIDC, identity in .NET

---

## GitHub Repositories (Study These)

### Reference Architectures
| Repository | What You'll Learn |
|------------|-------------------|
| [dotnet/eShop](https://github.com/dotnet/eShop) | Microsoft's microservices reference. Complex but comprehensive. |
| [jasontaylordev/CleanArchitecture](https://github.com/jasontaylordev/CleanArchitecture) | Popular project structure template |
| [ardalis/CleanArchitecture](https://github.com/ardalis/CleanArchitecture) | Alternative take with different opinions |
| [kgrzybek/modular-monolith-with-ddd](https://github.com/kgrzybek/modular-monolith-with-ddd) | Full DDD implementation in .NET |
| [oskardudycz/EventSourcing.NetCore](https://github.com/oskardudycz/EventSourcing.NetCore) | Event sourcing patterns and samples |

### Tools & Libraries
| Repository | What It Does |
|------------|--------------|
| [App-vNext/Polly](https://github.com/App-vNext/Polly) | Resilience and transient-fault-handling |
| [jbogard/MediatR](https://github.com/jbogard/MediatR) | In-process messaging/CQRS |
| [FluentValidation/FluentValidation](https://github.com/FluentValidation/FluentValidation) | Validation library |
| [serilog/serilog](https://github.com/serilog/serilog) | Structured logging |
| [DapperLib/Dapper](https://github.com/DapperLib/Dapper) | Micro-ORM for performance-critical queries |

---

## Online Courses (Paid, But Worth It)

| Course | Platform | Why |
|--------|----------|-----|
| [C# Advanced Topics](https://www.pluralsight.com/paths/c-development-best-practices) | Pluralsight | Fills gaps from tutorials |
| [Entity Framework Core Deep Dive](https://www.pluralsight.com/courses/ef-core-6-fundamentals) | Pluralsight | Julie Lerman's comprehensive course |
| [MongoDB for .NET Developers](https://learn.mongodb.com/learning-paths/mongodb-dotnet-developer-path) | MongoDB University | Free, official, good |
| [Redis University](https://university.redis.com/) | Redis | Free courses on caching patterns |
| [Kubernetes Fundamentals](https://training.linuxfoundation.org/training/kubernetes-fundamentals/) | Linux Foundation | Worth it if you'll manage clusters |

---

## Tools You Should Know

### Development
| Tool | Purpose | Link |
|------|---------|------|
| Rider / VS Code | IDE | [Rider](https://www.jetbrains.com/rider/), [VS Code](https://code.visualstudio.com/) |
| Docker Desktop | Local containers | [Docker](https://www.docker.com/products/docker-desktop/) |
| Postman / Bruno | API testing | [Postman](https://www.postman.com/), [Bruno](https://www.usebruno.com/) (open source) |
| Azure Data Studio | Database management | [Azure Data Studio](https://azure.microsoft.com/en-us/products/data-studio/) |

### Debugging & Profiling
| Tool | Purpose | Link |
|------|---------|------|
| dotnet-trace | Performance tracing | [Docs](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace) |
| dotnet-dump | Memory dump analysis | [Docs](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump) |
| PerfView | Deep performance analysis | [GitHub](https://github.com/microsoft/perfview) |
| BenchmarkDotNet | Micro-benchmarking | [Docs](https://benchmarkdotnet.org/) |

### Observability
| Tool | Purpose | Link |
|------|---------|------|
| Seq | Log aggregation (free for dev) | [Seq](https://datalust.co/seq) |
| Jaeger | Distributed tracing | [Jaeger](https://www.jaegertracing.io/) |
| Grafana | Dashboards | [Grafana](https://grafana.com/) |
| k6 | Load testing | [k6](https://k6.io/) |

---

## Podcasts (For Commutes)

| Podcast | Why Listen |
|---------|------------|
| [.NET Rocks](https://www.dotnetrocks.com/) | Long-running, interviews with .NET experts |
| [The Hanselminutes Podcast](https://www.hanselminutes.com/) | Broader tech topics from Scott Hanselman |
| [Software Engineering Daily](https://softwareengineeringdaily.com/) | Deep dives into infrastructure and architecture |
| [CoRecursive](https://corecursive.com/) | Programming stories and history |

---

## Communities

| Community | What It's For |
|-----------|---------------|
| [r/dotnet](https://www.reddit.com/r/dotnet/) | News, questions, discussions |
| [.NET Discord](https://discord.gg/dotnet) | Real-time help |
| [Stack Overflow](https://stackoverflow.com/questions/tagged/.net) | Specific technical questions |
| [Dev.to #dotnet](https://dev.to/t/dotnet) | Blog posts and tutorials |

---

## How to Use This List

1. **Don't read everything** — Pick resources relevant to your current module
2. **Books are references** — Read chapters as needed, not cover-to-cover
3. **Watch videos at 1.5x** — Most tutorials are too slow
4. **Code along** — Reading without typing teaches nothing
5. **Revisit resources** — The same content hits differently after you've struggled with a problem

The best resource is always the one that answers your current question. Bookmark this page and return when stuck.
