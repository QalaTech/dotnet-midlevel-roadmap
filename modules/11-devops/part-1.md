# Part 1 — Git Mastery & Workflow

## Why This Matters

Git isn't just version control — it's your team's communication system. A good Git history tells the story of how your codebase evolved. A bad one is forensic archaeology.

> "If you can't explain why something was done by reading the commit history, the commit history is broken." — Unknown wise engineer

Every team member who reads `git log` should understand:
- What changed
- Why it changed
- Who can answer questions about it

---

## What Goes Wrong Without This

### The Blame Game

```bash
$ git blame api/OrderController.cs

a1b2c3d4 (Unknown 2023-01-15) public async Task<IActionResult> Get()
f5e6d7c8 (wip       2023-02-20) {
9a8b7c6d (fix       2023-03-10)     // TODO: remove this hack
e1f2g3h4 (asdf      2023-04-01)     var orders = await _service.GetOrders();

"Who wrote this? Why? What were they thinking?"
No one remembers. The context is lost forever.
```

### The Merge Conflict Nightmare

```
Developer A: Working on feature/orders for 3 weeks
Developer B: Working on feature/payments for 3 weeks
Both: Touched the same 15 files

Merge attempt:
  47 conflicts in 12 files
  3 hours to resolve
  "Did I break something?"

Result: Entire afternoon lost, bugs introduced
```

### The "What Broke Prod?" Mystery

```bash
$ git log --oneline
a1b2c3d Merge branch 'feature/stuff'
b2c3d4e Merge branch 'main' into feature/stuff
c3d4e5f Merge branch 'feature/other-stuff'
d4e5f6g fixes
e5f6g7h more fixes
f6g7h8i final fix
g7h8i9j ok really final fix

"Something broke in the last week. Which commit?"
"I have no idea. They all say 'fix'."
```

---

## Git Internals You Need to Know

### The Three Trees

```
┌────────────────────────────────────────────────────────────────┐
│                        WORKING DIRECTORY                       │
│                   (Files on your disk)                         │
│                                                                │
│   OrderController.cs (modified)                                │
│   appsettings.json (untracked)                                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                              │
                              │ git add
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                        STAGING AREA                            │
│                   (Index / "Staged Changes")                   │
│                                                                │
│   OrderController.cs (staged for commit)                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                              │
                              │ git commit
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                        REPOSITORY                              │
│                   (Commit History)                             │
│                                                                │
│   commit a1b2c3d: "Add order filtering by date"               │
│   commit b2c3d4e: "Implement order service"                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Commits Are Snapshots, Not Diffs

```
Common misconception:
  "A commit stores the changes made"

Reality:
  "A commit is a complete snapshot of the entire repo"

Each commit contains:
  - SHA: Unique identifier (a1b2c3d4e5f6...)
  - Tree: Complete state of all files
  - Parent(s): Previous commit(s)
  - Author: Who wrote it
  - Committer: Who applied it
  - Message: Why it exists

Git computes diffs on the fly by comparing snapshots.
```

### Understanding Refs (Branches, Tags, HEAD)

```
     HEAD (current position)
        │
        ▼
      main
        │
        ▼
    ┌───────┐    ┌───────┐    ┌───────┐
    │ abc123│◄───│ def456│◄───│ ghi789│
    └───────┘    └───────┘    └───────┘
        ▲
        │
   feature/orders

Branches are just pointers to commits!
Moving a branch = moving a pointer.
Merging = creating a new commit with multiple parents.
```

---

## Branching Strategies

### Trunk-Based Development

Best for: Teams that deploy frequently, have good test coverage.

```
main ─────●─────●─────●─────●─────●─────●─────●─────●
          │     │     │     │     │     │     │     │
          │     │     │     └─────┤     │     │     │
          │     │     │   feature │     │     │     │
          │     │     │   (< 2 days)    │     │     │
          │     │     │                 │     │     │
          │     └─────┤                 │     │     │
          │   feature │                 │     │     │
          │   (1 day) │                 │     │     │
          │           │                 │     │     │
          └───────────┤                 │     │     │
             feature  │                 │     │     │
             (hours)  │                 │     │     │
```

**Rules:**
- Feature branches live < 2 days
- Always merge to main
- Use feature flags for incomplete work
- Deploy main continuously

### GitHub Flow

Best for: Most teams, good balance of simplicity and safety.

```
main ─────●─────●─────────────────●─────●─────────●─────●
          │                       │     │         │     │
          └───────●───────●───────┘     │         │     │
                  feature/orders        │         │     │
                  (PR required)         │         │     │
                                        │         │     │
                                        └────●────┘     │
                                        feature/payment │
                                        (PR required)   │
                                                        │
                                               deploy ──┘
```

**Rules:**
- `main` is always deployable
- Create branches for features
- Open PR when ready for review
- Deploy after merge to main

### GitFlow

Best for: Packaged software, versioned releases.

```
main ────●──────────────────────────────●──────────────●
         │                              │              │
         │       ┌──────────────────────┤              │
         │       │                      │              │
develop ─●───●───●───●───●───●───●──────●──────●───●───●
         │   │       │       │          │      │   │
         │   │       │       └──────────│──────┤   │
         │   │       │      feature/pay │      │   │
         │   │       │                  │      │   │
         │   │       └──────────────────│──────┘   │
         │   │       feature/orders     │          │
         │   │                          │          │
         │   └──────────────────────────┤          │
         │    release/1.0               │          │
         │                              │          │
         └──────────────────────────────┤          │
           hotfix/security              │          │
                                        │          │
                                   tag: v1.0  tag: v1.1
```

**Rules:**
- `main` = production releases only
- `develop` = integration branch
- Feature branches from develop
- Release branches for stabilization
- Hotfix branches for emergencies

### Choosing a Strategy

| Factor | Trunk-Based | GitHub Flow | GitFlow |
|--------|-------------|-------------|---------|
| Deploy frequency | Multiple/day | Daily-weekly | Monthly+ |
| Team experience | High | Any | Any |
| Test coverage | High required | Good | Any |
| Feature flags | Required | Optional | Not needed |
| Release process | Simple | Simple | Formal |

---

## Commit Message Excellence

### The Anatomy of a Good Commit

```
feat(orders): add date range filtering to order search

Customers requested ability to filter orders by date range.
This adds startDate and endDate query parameters to the
GET /api/orders endpoint.

- Add DateRange value object for validation
- Update OrderService.SearchAsync to accept date parameters
- Add integration tests for date filtering

Closes #234
```

**Structure:**
```
<type>(<scope>): <subject>
                                        ← blank line
<body>
                                        ← blank line
<footer>
```

### Commit Types (Conventional Commits)

| Type | Use For |
|------|---------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, semicolons, etc. |
| `refactor` | Code change that doesn't fix or add |
| `test` | Adding or updating tests |
| `chore` | Build, tooling, dependencies |
| `perf` | Performance improvement |
| `ci` | CI/CD configuration |

### Bad vs Good Commit Messages

```
# Bad
fix bug
update
wip
asdf
Merge branch 'main' of github.com/org/repo

# Good
fix(auth): prevent token refresh during logout

fix(orders): handle null customer address in validation

refactor(api): extract common error handling to middleware

docs(readme): add local development setup instructions

chore(deps): update FluentValidation to 11.8.0
```

### Writing the Body

**Answer these questions:**
1. **Why** was this change made?
2. **What** problem does it solve?
3. **How** does it work (if not obvious)?

```
# Before
fix: fixed the thing

# After
fix(payments): prevent duplicate charge on retry

When a payment request times out, the client retries.
Previously, this could result in double-charging if the
first request actually succeeded on the server.

Added idempotency key tracking to detect duplicate
requests within a 24-hour window. Duplicate requests
now return the original transaction result.

Fixes #567
```

---

## Pull Request Excellence

### The PR Checklist

Before opening a PR:

```markdown
## Self-Review Checklist
- [ ] Code compiles without warnings
- [ ] All tests pass locally
- [ ] I've added tests for new functionality
- [ ] I've updated documentation if needed
- [ ] Commit messages are meaningful
- [ ] I've rebased on latest main
- [ ] PR title follows conventional commit format
```

### PR Size Matters

```
# Too big - reviewer fatigue
+1,247 -892 across 45 files
"I'll review this later" → Never reviewed properly

# Just right
+150 -50 across 8 files
Reviewers can understand the change fully

# Ideal for fast review
+30 -10 across 3 files
Reviewed in 10 minutes, merged same day
```

### PR Description Template

```markdown
## What

Brief description of the change.

## Why

Link to issue/ticket: #123

Business context: Customers need to filter orders by date.

## How

- Added DateRange value object
- Updated OrderService
- Added integration tests

## Testing

- [ ] Unit tests added
- [ ] Integration tests pass
- [ ] Manual testing done in dev environment

## Screenshots (if UI changes)

## Deployment Notes

Any special deployment steps or feature flags needed?
```

---

## Essential Git Commands

### Daily Workflow

```bash
# Start new work
git checkout main
git pull origin main
git checkout -b feature/order-filtering

# Make changes, commit often
git add src/Orders/DateRange.cs
git commit -m "feat(orders): add DateRange value object"

# Stay up to date with main
git fetch origin
git rebase origin/main

# Push and create PR
git push -u origin feature/order-filtering
```

### Fixing Mistakes

```bash
# Undo last commit but keep changes
git reset --soft HEAD~1

# Undo last commit and discard changes
git reset --hard HEAD~1

# Fix the last commit message
git commit --amend -m "New message"

# Add forgotten file to last commit
git add forgotten-file.cs
git commit --amend --no-edit

# Undo a pushed commit (creates new commit)
git revert <commit-sha>
```

### Interactive Rebase (Cleaning History)

```bash
# Before PR, clean up your commits
git rebase -i origin/main

# In the editor:
pick a1b2c3d feat: add order filtering
squash b2c3d4e fix typo
squash c3d4e5f fix test
pick d4e5f6g refactor: extract date validation

# Result: 2 clean commits instead of 4
```

### Stashing Work

```bash
# Save current changes without committing
git stash push -m "WIP: order filtering"

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{2}
```

### Finding Problems

```bash
# Who changed this line?
git blame src/Orders/OrderService.cs

# When was this bug introduced?
git bisect start
git bisect bad                     # Current commit is bad
git bisect good v1.5.0            # This version was good
# Git checks out middle commit
# You test, then:
git bisect good  # or  git bisect bad
# Repeat until found

# Search commit messages
git log --grep="order filtering"

# Search code changes
git log -p -S "GetOrdersByDate"
```

---

## Pre-commit Hooks

### Setting Up Husky.NET

```bash
# Install Husky
dotnet new tool-manifest
dotnet tool install Husky
dotnet husky install
```

```json
// .husky/task-runner.json
{
  "tasks": [
    {
      "name": "dotnet-format",
      "command": "dotnet",
      "args": ["format", "--verify-no-changes"],
      "include": ["**/*.cs"]
    },
    {
      "name": "dotnet-build",
      "command": "dotnet",
      "args": ["build", "--no-restore"]
    },
    {
      "name": "dotnet-test",
      "command": "dotnet",
      "args": ["test", "--no-build", "--filter", "Category!=Integration"]
    }
  ]
}
```

### Pre-push Hook

```bash
#!/bin/sh
# .husky/pre-push

echo "Running pre-push checks..."

# Run unit tests
dotnet test --filter "Category!=Integration" --no-restore
if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Push aborted."
    exit 1
fi

echo "✅ All checks passed!"
```

### Commit Message Validation

```bash
#!/bin/sh
# .husky/commit-msg

commit_msg=$(cat "$1")

# Conventional commits regex
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci)(\(.+\))?: .{1,50}"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
    echo "❌ Commit message doesn't follow Conventional Commits format"
    echo "   Expected: type(scope): description"
    echo "   Example: feat(orders): add date filtering"
    exit 1
fi

echo "✅ Commit message format valid"
```

---

## Hands-On Exercise: Clean Up OrderFlow History

### Step 1: Review Current History

```bash
git log --oneline --graph -20
```

Look for:
- Vague commit messages
- "WIP" commits
- Multiple commits that should be one

### Step 2: Create a Feature Branch Properly

```bash
git checkout main
git pull
git checkout -b feature/improve-order-search
```

### Step 3: Make Atomic Commits

Instead of one big commit:
```bash
# Commit 1: The model change
git add src/Orders/DateRange.cs
git commit -m "feat(orders): add DateRange value object for date filtering"

# Commit 2: The service change
git add src/Orders/OrderService.cs
git commit -m "feat(orders): implement date range filtering in OrderService"

# Commit 3: The tests
git add tests/Orders/DateRangeTests.cs tests/Orders/OrderServiceTests.cs
git commit -m "test(orders): add unit tests for date range filtering"
```

### Step 4: Practice Interactive Rebase

```bash
# Simulate a messy history
echo "// test" >> src/Orders/OrderService.cs
git add . && git commit -m "wip"
echo "// more test" >> src/Orders/OrderService.cs
git add . && git commit -m "wip2"

# Clean it up
git rebase -i HEAD~2
# Change 'pick' to 'squash' for the second commit
# Edit the final commit message
```

### Step 5: Set Up Hooks

Configure Husky.NET and verify:
```bash
# This should run checks before committing
echo "// bad format" >> src/Orders/OrderService.cs
git add . && git commit -m "test hooks"
# Should fail due to formatting
```

---

## Deliverables

1. **Branching Strategy Document**: Define your team's approach
2. **Git Hooks**: Configured pre-commit and commit-msg hooks
3. **Clean Feature Branch**: Demonstrate proper commit history
4. **PR Template**: `.github/pull_request_template.md`

---

## Reflection Questions

1. Why does commit message quality matter for debugging?
2. When would you use `rebase` vs `merge`?
3. What's the risk of rewriting history that's been pushed?
4. How do you balance commit frequency with meaningful commits?

---

## Resources

### Git Fundamentals
- [Pro Git Book](https://git-scm.com/book/en/v2) (free)
- [Git Internals](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)

### Branching Strategies
- [Trunk-Based Development](https://trunkbaseddevelopment.com/)
- [GitHub Flow](https://docs.github.com/en/get-started/quickstart/github-flow)
- [GitFlow](https://nvie.com/posts/a-successful-git-branching-model/)

### Commit Conventions
- [Conventional Commits](https://www.conventionalcommits.org/)
- [How to Write a Git Commit Message](https://cbea.ms/git-commit/)

### Tools
- [Husky.NET](https://alirezanet.github.io/Husky.Net/)
- [Commitlint](https://commitlint.js.org/)
- [GitKraken](https://www.gitkraken.com/) - Visual Git client

---

**Next:** [Part 2 — CI/CD Pipelines](./part-2.md)
