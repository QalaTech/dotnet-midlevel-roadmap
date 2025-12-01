# Contributing to the .NET Backend Roadmap

## Content Structure

Each module follows a consistent structure:

```
modules/XX-module-name/
├── README.md      # Module overview, structure, prerequisites, exit criteria
├── part-1.md      # First part of the module
├── part-2.md      # Second part
├── part-3.md      # Third part
└── part-4.md      # Fourth part (if applicable)
```

## Module README Format

Every module README should include:

1. **Why This Module Exists** — What problem it solves
2. **What You'll Build** — Concrete deliverables for OrderFlow
3. **Module Structure** — Links to all parts with descriptions
4. **Prerequisites** — Which modules must come first
5. **Key Concepts** — 4-5 main takeaways
6. **Tools You'll Use** — Technologies covered
7. **Exit Criteria** — How to know you've mastered it

## Part File Format

Each part file should include:

1. **Header** — Module name and part number
2. **What You'll Learn** — Brief overview
3. **Core Concepts** — 3-5 concepts with:
   - Why it matters
   - What goes wrong without it
   - Code examples (good and bad)
4. **Hands-On** — OrderFlow exercises
5. **Deliverables** — What to produce
6. **Resources** — Links to books, videos, docs
7. **Reflection Questions** — Self-assessment

## Style Guidelines

- Use active voice
- Include "What Goes Wrong" examples for every concept
- Show both good and bad code patterns
- Link concepts to OrderFlow whenever possible
- Keep code examples under 30 lines

## Adding New Content

1. Follow the existing module format
2. Update the root README.md module table
3. Update the master-roadmap.md if adding new modules
4. Ensure all internal links work

## Fixing Issues

- Broken links: Use relative paths (e.g., `./part-1.md`, `../00-introduction/README.md`)
- Typos: Submit a PR with the fix
- Outdated content: Note the .NET version being discussed
