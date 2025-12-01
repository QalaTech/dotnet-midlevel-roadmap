# 13 — Working with Legacy Code & Debugging

## Why This Module Exists

Here's a secret nobody tells junior developers: **you'll spend 70% of your career working with code you didn't write.** Most of that code won't be clean, won't follow patterns you learned, and won't have tests.

This module teaches the skills tutorials ignore:
- Reading and understanding unfamiliar code
- Debugging production issues systematically
- Changing code safely when there are no tests
- Refactoring without breaking things

These skills separate developers who can only build greenfield projects from developers who can ship value in any codebase.

---

## What You'll Learn

| Part | Focus | Skills |
|------|-------|--------|
| [Part 1](./part-1.md) | Reading Unfamiliar Code | Navigation, understanding, documentation |
| [Part 2](./part-2.md) | Debugging Systematically | Stack traces, logs, breakpoints, profiling |
| [Part 3](./part-3.md) | Safe Changes | Characterization tests, strangler fig, refactoring |

---

## Prerequisites

- Complete Modules 01-07 (you need foundation before you can debug effectively)
- A real codebase to practice on (OrderFlow or a work project)

---

## The Reality Check

**What tutorials teach:**
- Build from scratch
- Everything is clean
- Tests exist
- Documentation is current

**What production looks like:**
- 5-year-old codebase with 10 previous authors
- Mixed patterns (some Clean Architecture, some spaghetti)
- Tests cover 20% of the code (the easy 20%)
- Documentation describes the system from 3 years ago

This module prepares you for reality.

---

## Resources

### Books (Essential)
- [Working Effectively with Legacy Code](https://www.oreilly.com/library/view/working-effectively-with/0131177052/) — Michael Feathers (the definitive guide)
- [Refactoring](https://refactoring.com/) — Martin Fowler (patterns for safe changes)

### Videos
- [Julia Evans: Debugging Zines](https://wizardzines.com/) — Visual guides to debugging
- [Scott Hanselman: Debugging Tips](https://www.youtube.com/watch?v=2V0K_xL0OYo)

### Tools
- [dotnet-trace](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
- [dotnet-dump](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump)
- [PerfView](https://github.com/microsoft/perfview)

---

## OrderFlow Connection

In this module, you'll:
1. Intentionally "age" your OrderFlow codebase (remove tests, mix patterns)
2. Practice debugging issues you introduce
3. Refactor safely back to clean code
4. Document the process as if onboarding a new team member

This simulates joining a real team with a real codebase.
