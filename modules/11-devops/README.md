# 11 — DevOps, CI/CD & Cloud Fundamentals

## Why This Module Exists

"It works on my machine" is not a deployment strategy.

The gap between writing code and running code in production is where careers are made or broken. The best feature in the world is worthless if:
- It can't be deployed safely
- It breaks when scaling
- Nobody can reproduce the environment
- Rollback takes hours instead of minutes

DevOps isn't a role — it's a mindset. **You build it, you run it.**

---

## What Goes Wrong Without This

### The Deploy-Day Disaster

```
Friday 4 PM: "Just a quick deploy"
4:15 PM: Deploy starts
4:30 PM: "Something's wrong, checking logs"
4:45 PM: "Can't reproduce locally"
5:00 PM: "Different .NET version in prod"
5:30 PM: "Let's just rollback"
5:45 PM: "Where's the previous build artifact?"
6:30 PM: "Manual rollback complete... I think"
7:00 PM: "Why is the database schema wrong?"

Monday 9 AM: "So about Friday's deploy..."
```

### The Snowflake Server

```
Developer: "The API is slow in staging"
Ops: "Works fine in prod"
Developer: "Why is staging different?"
Ops: "I don't remember what I configured"
Developer: "Can we check the scripts?"
Ops: "I made those changes in the portal"

Result: Undocumented, unreproducible infrastructure
```

### The Git Crime Scene

```
main branch:
├── Merge branch 'fix'
├── Merge branch 'fix2'
├── wip
├── asdfasdf
├── fixed the thing
├── really fixed it this time
├── Merge branch 'main' of github.com/...
└── undo last commit

"I don't know who wrote this or why"
```

---

## What You'll Build

OrderFlow's complete delivery pipeline:

1. **Git Discipline** — Clean history, meaningful commits, safe merges
2. **CI/CD Pipeline** — Automated build, test, scan, deploy
3. **Container Strategy** — Reproducible environments everywhere
4. **Infrastructure as Code** — Declarative, version-controlled cloud resources

---

## Module Structure

### [Part 1 — Git Mastery & Workflow](./part-1.md)
- Git internals you actually need
- Branching strategies that work
- Commit message discipline
- Pull request excellence

### [Part 2 — CI/CD Pipelines](./part-2.md)
- GitHub Actions deep dive
- Quality gates and security scanning
- Deployment strategies
- Pipeline as code

### [Part 3 — Containers & Docker](./part-3.md)
- Multi-stage Dockerfiles
- Compose for local development
- Container security
- Registry management

### [Part 4 — Infrastructure as Code](./part-4.md)
- Bicep/Terraform fundamentals
- Cloud resource management
- Database migrations in CI/CD
- Secrets and configuration

---

## Prerequisites

Before starting this module:
- Complete Module 07 (Testing) — tests to run in pipeline
- Complete Module 09 (Performance) — understand what to monitor
- Have access to a cloud platform (Azure, AWS, or GCP)
- Install Docker Desktop

---

## Key Concepts You'll Master

1. **Automation Over Documentation** — If a human has to remember it, automate it
2. **Everything in Source Control** — Code, config, infrastructure, scripts
3. **Immutable Artifacts** — Build once, deploy everywhere
4. **Fail Fast, Recover Fast** — Catch problems early, rollback quickly

---

## The DevOps Maturity Journey

```
Level 0: "We deploy manually"
         └── FTP files to server, hope for the best

Level 1: "We have a build server"
         └── CI builds on merge, manual deploy

Level 2: "We deploy to environments"
         └── CD to staging, manual prod approval

Level 3: "We have quality gates"
         └── Tests, scans, coverage requirements

Level 4: "Infrastructure is code"
         └── Environments are reproducible

Level 5: "We deploy continuously"
         └── Multiple deploys per day, automatic rollback
```

---

## Tools You'll Use

| Category | Tools |
|----------|-------|
| **Version Control** | Git, GitHub/Azure DevOps |
| **CI/CD** | GitHub Actions, Azure Pipelines |
| **Containers** | Docker, Docker Compose |
| **Container Registry** | Docker Hub, Azure Container Registry, GitHub Container Registry |
| **Infrastructure** | Bicep, Terraform |
| **Cloud** | Azure (App Service, Container Apps, AKS) |
| **Security** | Dependabot, CodeQL, Trivy |

---

## Exit Criteria

You can:
1. Write meaningful commit messages and maintain clean Git history
2. Create CI/CD pipelines that catch problems before production
3. Containerize applications with proper security practices
4. Define infrastructure as code for reproducible environments
5. Implement safe deployment strategies with rollback capability
6. Manage secrets without exposing them in source control

---

## The "Golden Path"

A production-ready deployment pipeline:

```
┌─────────────────────────────────────────────────────────────────┐
│                     DEVELOPER WORKFLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Create branch ──────────────────────────────────────────┐   │
│  2. Make changes                                            │   │
│  3. Run pre-commit hooks (format, lint, test)               │   │
│  4. Commit with meaningful message                          │   │
│  5. Push & create Pull Request                              │   │
│                                                             │   │
└─────────────────────────────────────────────────────────────│───┘
                                                              │
┌─────────────────────────────────────────────────────────────│───┐
│                     CI PIPELINE                             │   │
├─────────────────────────────────────────────────────────────│───┤
│                                                             │   │
│  6. Build ◄─────────────────────────────────────────────────┘   │
│  7. Unit tests                                                  │
│  8. Integration tests                                           │
│  9. Security scan                                               │
│  10. Build container image                                      │
│  11. Push to registry                                           │
│                                                                 │
└───────────────────────────────────────────────┬─────────────────┘
                                                │
┌───────────────────────────────────────────────│─────────────────┐
│                     CD PIPELINE               │                 │
├───────────────────────────────────────────────│─────────────────┤
│                                               │                 │
│  12. Deploy to staging ◄──────────────────────┘                 │
│  13. Run smoke tests                                            │
│  14. Approval gate (if required)                                │
│  15. Deploy to production                                       │
│  16. Health check                                               │
│  17. Monitor for anomalies                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Resources

### Essential Reading
- [The DevOps Handbook](https://www.oreilly.com/library/view/the-devops-handbook/9781457191381/)
- [Continuous Delivery](https://www.oreilly.com/library/view/continuous-delivery/9780321670250/) by Jez Humble
- [Pro Git](https://git-scm.com/book/en/v2) (free online)

### Video Resources
- [GitHub Actions Tutorial](https://www.youtube.com/watch?v=R8_veQiYBjI)
- [Docker for .NET Developers](https://www.youtube.com/watch?v=vmnvOITMoIg)
- [Azure Bicep Deep Dive](https://www.youtube.com/watch?v=yFw03RGdxFc)

### Documentation
- [GitHub Actions](https://docs.github.com/en/actions)
- [Docker Documentation](https://docs.docker.com/)
- [Azure Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Terraform](https://developer.hashicorp.com/terraform/docs)

---

**Start:** [Part 1 — Git Mastery & Workflow](./part-1.md)
