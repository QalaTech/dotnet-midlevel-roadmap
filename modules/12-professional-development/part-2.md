# Part 2 — Developer Productivity & Debugging

## Why This Matters

The difference between a 1x and 10x developer isn't intelligence — it's systems. The best engineers automate repetitive work, master their tools, and debug systematically.

> "Hours of debugging can save you minutes of reading documentation." — Unknown

Every minute you spend fighting your tools is a minute not spent solving problems.

---

## What Goes Wrong Without This

### The Slow Developer

```
Task: Add new field to API endpoint

Junior approach:
1. Open solution (2 min)
2. Navigate to controller... where is it? (3 min)
3. Add field, build (30 sec)
4. Build fails, wrong project selected (1 min)
5. Rebuild, test manually (2 min)
6. Realize forgot to update DTO (1 min)
7. Rebuild, test again (2 min)
8. Push, PR (1 min)

Total: 12+ minutes for a 30-second change

Senior approach:
1. Ctrl+T, type "OrderController" (5 sec)
2. Add field, Ctrl+Shift+B (30 sec)
3. Run test with Ctrl+R,T (20 sec)
4. Git commit with keybinding (10 sec)

Total: ~1 minute
```

### The Print Statement Debugger

```csharp
// Real production code found in the wild
public async Task<Order> GetOrderAsync(int id)
{
    Console.WriteLine("ENTERING GetOrderAsync");  // Debug
    Console.WriteLine($"ID: {id}");                // Debug
    var order = await _db.Orders.FindAsync(id);
    Console.WriteLine($"Order: {order}");          // Debug
    Console.WriteLine("EXITING GetOrderAsync");    // Debug
    return order;
}

// Left in production, filling logs with noise
// Meanwhile: debugger sits unused
```

### The Context-Switching Chaos

```
Monday morning:
  9:00 - Start feature work
  9:15 - Slack message, context switch
  9:30 - Back to feature, where was I?
  9:45 - Email notification, check it
  10:00 - Meeting (30 min)
  10:30 - Back to feature, forgot context again
  10:45 - Another Slack message
  11:00 - Still haven't made real progress

Result: 2 hours of "work", 20 minutes of coding
```

---

## IDE Mastery

### Essential Keyboard Shortcuts (Rider/Visual Studio)

| Action | Rider | Visual Studio |
|--------|-------|---------------|
| Go to anything | Ctrl+T | Ctrl+, |
| Go to file | Ctrl+Shift+T | Ctrl+Shift+T |
| Go to symbol | Ctrl+Alt+Shift+T | Ctrl+T |
| Find usages | Shift+F12 | Shift+F12 |
| Rename | Ctrl+R,R | Ctrl+R,R |
| Extract method | Ctrl+R,M | Ctrl+R,M |
| Quick actions | Alt+Enter | Ctrl+. |
| Navigate back | Ctrl+- | Ctrl+- |
| Navigate forward | Ctrl+Shift+- | Ctrl+Shift+- |
| Run tests | Ctrl+R,T | Ctrl+R,T |
| Debug tests | Ctrl+R,D | Ctrl+R,Ctrl+T |
| Toggle breakpoint | F9 | F9 |
| Step over | F10 | F10 |
| Step into | F11 | F11 |
| Run to cursor | Ctrl+F10 | Ctrl+F10 |

### Must-Know Features

```
Features that save hours:

1. Multiple Cursors
   - Ctrl+Alt+Click: Add cursor
   - Ctrl+D: Select next occurrence
   - Use case: Rename variable in multiple places

2. Structural Search and Replace
   - Search for patterns, not just text
   - Example: Find all `if (x == null) throw new ArgumentNullException()`
   - Replace with: `ArgumentNullException.ThrowIfNull(x)`

3. Code Generation
   - Constructor from fields
   - Implement interface
   - Generate equals/hashcode

4. Live Templates
   - `prop` → full property
   - `ctor` → constructor
   - Create custom templates for patterns you use often

5. Recent Files
   - Ctrl+E: See recently opened files
   - Faster than navigating file tree
```

### Rider-Specific Productivity

```
Rider features worth learning:

1. Solution-Wide Analysis
   - Shows errors across all projects
   - Catches issues before build

2. Database Tools
   - Query databases directly from IDE
   - See schema, data, execute queries

3. HTTP Client
   - .http files for API testing
   - Version controlled API calls

4. Docker Integration
   - Run containers from IDE
   - Attach debugger to containers

5. Code Cleanup on Save
   - Automatically format code
   - Apply code style rules
   - Remove unused imports
```

---

## Debugging Mastery

### The Debugging Mindset

```
Before touching the debugger:

1. Reproduce the bug
   - Can you trigger it reliably?
   - What's the simplest reproduction case?

2. Formulate a hypothesis
   - "I think X is happening because of Y"
   - This gives you something to test

3. Gather evidence
   - Logs, metrics, stack traces
   - What do we know for sure?

4. Test the hypothesis
   - Now use the debugger
   - Prove or disprove your theory

5. Fix and verify
   - Make the smallest change possible
   - Prove the fix works
```

### Advanced Debugger Features

```csharp
// Conditional breakpoints
// Only break when condition is true
// Right-click breakpoint → Conditions → Expression
// Example: orderId == 42 && status == "Failed"

// Tracepoints (logging without code changes)
// Right-click breakpoint → Actions → Log a message
// Example: "Order {orderId} processing, Status: {status}"
// Check "Continue execution" to not stop

// Data breakpoints (break on value change)
// Right-click variable → Break When Value Changes
// Useful for finding where data gets corrupted

// Exception settings
// Debug → Windows → Exception Settings
// Break on first-chance exceptions
// Useful for finding swallowed exceptions

// Immediate window
// Execute expressions during debugging
// > order.Items.Sum(i => i.Price)
// > _service.GetOrderStatus(orderId)
```

### Debugging Distributed Systems

```
The challenge:
- Request flows through multiple services
- Problem could be anywhere
- Traditional debugger doesn't help

The approach:

1. Correlation ID
   - Add to all requests
   - Filter logs by correlation ID
   - See the full request flow

2. Distributed tracing
   - Jaeger, Zipkin, Application Insights
   - Visualize request path
   - See timing of each step

3. Strategic logging
   - Log at service boundaries
   - Include request/response data
   - Log at decision points

4. Local reproduction
   - Docker Compose to run dependencies
   - Attach debugger to your service
   - Replay the problematic request
```

### Debugging Case Study

```markdown
## Bug: Orders occasionally saved with wrong total

### Symptoms
- ~2% of orders have incorrect totals
- Happens randomly, can't reproduce reliably
- Started after recent deployment

### Investigation

**Step 1: Gather data**
- Pulled 100 affected orders from logs
- Correlation: All occurred during high traffic
- Pattern: Wrong totals are always from previous orders

**Step 2: Hypothesis**
"Race condition in order calculation - concurrent requests
are sharing state somehow"

**Step 3: Code review**
Found in OrderCalculator.cs:
```csharp
public class OrderCalculator
{
    private decimal _runningTotal; // SHARED STATE!

    public decimal Calculate(Order order)
    {
        _runningTotal = 0;
        foreach (var item in order.Items)
        {
            _runningTotal += item.Price * item.Quantity;
        }
        return _runningTotal;
    }
}
```

Problem: OrderCalculator registered as Singleton, but has
instance state. Concurrent requests corrupt each other.

**Step 4: Verify**
- Wrote load test with concurrent order creation
- Reproduced issue in 10 seconds
- Confirmed hypothesis

**Step 5: Fix**
```csharp
public class OrderCalculator
{
    public decimal Calculate(Order order)
    {
        return order.Items.Sum(i => i.Price * i.Quantity);
    }
}
```

**Step 6: Prevention**
- Added analyzer rule to flag Singleton with mutable state
- Added concurrent test to catch similar issues
```

---

## Workflow Automation

### Dotfiles Repository

```bash
# Create a dotfiles repo for your setup
# ~/.dotfiles/

.dotfiles/
├── .bashrc              # Shell configuration
├── .zshrc               # Zsh configuration
├── .gitconfig           # Git settings
├── .vimrc               # Vim settings
├── install.sh           # Installation script
├── Brewfile             # macOS packages
├── winget.json          # Windows packages
└── vscode/
    ├── settings.json    # VS Code settings
    └── extensions.txt   # Extension list
```

### Shell Aliases

```bash
# ~/.bashrc or ~/.zshrc

# Git shortcuts
alias gs='git status'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline -20'
alias gd='git diff'
alias gco='git checkout'
alias gb='git branch'

# .NET shortcuts
alias dn='dotnet'
alias dnr='dotnet run'
alias dnb='dotnet build'
alias dnt='dotnet test'
alias dnw='dotnet watch run'

# Docker shortcuts
alias dc='docker compose'
alias dcu='docker compose up -d'
alias dcd='docker compose down'
alias dcl='docker compose logs -f'

# Navigation
alias ..='cd ..'
alias ...='cd ../..'
alias ll='ls -la'

# Project-specific
alias orderflow='cd ~/dev/orderflow && code .'
alias logs='docker compose logs -f api'
alias rebuild='docker compose build --no-cache && docker compose up -d'

# Functions
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Find in code
fic() {
    grep -r "$1" --include="*.cs" .
}
```

### Git Aliases

```ini
# ~/.gitconfig

[alias]
    # Shortcuts
    s = status
    co = checkout
    br = branch
    ci = commit
    st = status

    # Useful commands
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = !gitk

    # Pretty log
    lg = log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit

    # Find branches containing commit
    fb = "!f() { git branch -a --contains $1; }; f"

    # Show what you've done today
    today = log --since=midnight --author='Your Name' --oneline

    # Amend without editing message
    amend = commit --amend --no-edit

    # Undo last commit but keep changes
    undo = reset HEAD~1 --mixed

    # Clean up merged branches
    cleanup = "!git branch --merged | grep -v '\\*\\|main\\|master' | xargs -n 1 git branch -d"
```

### VS Code Settings

```json
// settings.json
{
    // Editor
    "editor.fontSize": 14,
    "editor.tabSize": 4,
    "editor.formatOnSave": true,
    "editor.minimap.enabled": false,
    "editor.wordWrap": "on",
    "editor.bracketPairColorization.enabled": true,

    // Files
    "files.autoSave": "onFocusChange",
    "files.exclude": {
        "**/bin": true,
        "**/obj": true,
        "**/node_modules": true
    },

    // Terminal
    "terminal.integrated.defaultProfile.windows": "PowerShell",
    "terminal.integrated.fontSize": 13,

    // C# / .NET
    "omnisharp.enableRoslynAnalyzers": true,
    "omnisharp.enableEditorConfigSupport": true,

    // Git
    "git.autofetch": true,
    "git.confirmSync": false,

    // Keybindings for productivity
    "keybindings": [
        {
            "key": "ctrl+shift+t",
            "command": "workbench.action.terminal.toggleTerminal"
        }
    ]
}
```

### Development Environment Script

```powershell
# scripts/setup-dev.ps1

# Check prerequisites
function Test-Command($command) {
    try {
        Get-Command $command -ErrorAction Stop | Out-Null
        return $true
    } catch {
        return $false
    }
}

Write-Host "Setting up development environment..." -ForegroundColor Cyan

# Install winget packages
$packages = @(
    "Microsoft.DotNet.SDK.8",
    "Docker.DockerDesktop",
    "Git.Git",
    "Microsoft.VisualStudioCode",
    "JetBrains.Rider",
    "GitHub.cli",
    "Postman.Postman"
)

foreach ($package in $packages) {
    if (Test-Command winget) {
        Write-Host "Installing $package..."
        winget install --id $package --silent
    }
}

# Clone repositories
$repoPath = "$env:USERPROFILE\dev"
if (!(Test-Path $repoPath)) {
    New-Item -ItemType Directory -Path $repoPath | Out-Null
}

# Setup Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.autocrlf true
git config --global pull.rebase true

# Install VS Code extensions
$extensions = @(
    "ms-dotnettools.csharp",
    "ms-azuretools.vscode-docker",
    "eamodio.gitlens",
    "esbenp.prettier-vscode",
    "streetsidesoftware.code-spell-checker"
)

foreach ($ext in $extensions) {
    code --install-extension $ext
}

Write-Host "Setup complete!" -ForegroundColor Green
```

---

## Personal Metrics

### What to Measure

```
Useful metrics for self-improvement:

1. Cycle Time
   - Time from starting work to merging PR
   - Goal: Reduce this over time

2. PR Size
   - Lines changed per PR
   - Smaller PRs = faster reviews

3. Bug Fix Time
   - Time from bug report to fix deployed
   - Tracks debugging efficiency

4. Context Switches
   - How often you switch tasks
   - Fewer switches = more deep work

5. Meeting Load
   - Hours in meetings per week
   - Protect your focus time
```

### Time Tracking Experiment

```markdown
## Week 1: Baseline

Track your time for one week using Toggl, RescueTime, or manual log.

Categories:
- Coding
- Code review
- Meetings
- Slack/Email
- Debugging
- Documentation
- Other

### Results Example:

| Category | Hours | Percentage |
|----------|-------|------------|
| Coding | 12 | 30% |
| Meetings | 10 | 25% |
| Slack/Email | 8 | 20% |
| Code review | 4 | 10% |
| Debugging | 4 | 10% |
| Other | 2 | 5% |

### Analysis:
- Only 30% of time on actual coding
- 25% in meetings - can I reduce?
- 20% on communication - batching opportunity?

## Week 2: Experiment

Changes made:
1. Batch Slack to 3 times per day
2. Decline optional meetings
3. Block 2-hour focus time each morning

Results:
- Coding increased to 45%
- Meetings reduced to 15%
- Completed 40% more PRs
```

### Building Better Habits

```
The habit loop:

Cue → Routine → Reward

Example: Improving code review habit

Cue: Morning coffee
Routine: Review 2 PRs before own work
Reward: Check off on todo list, feeling of helping team

Implementation:
1. Set calendar reminder: "Code review with coffee"
2. Keep PR list bookmarked
3. Track streak in notion/spreadsheet

After 2 weeks: Automatic habit
```

---

## Hands-On Exercise: Productivity Audit

### Step 1: Time Tracking

Track your time for 5 work days:
- Use Toggl, RescueTime, or spreadsheet
- Categorize every 15-minute block
- Be honest about context switches

### Step 2: Identify Waste

Look for patterns:
- When are you most productive?
- What causes most context switches?
- What repetitive tasks could be automated?

### Step 3: Implement Changes

Pick 3 improvements:
1. One automation (script, alias, template)
2. One schedule change (focus blocks, meeting batching)
3. One tool upgrade (learn 5 new shortcuts)

### Step 4: Measure Impact

After 2 weeks:
- Re-run time tracking
- Compare to baseline
- Document what worked

### Step 5: Create Productivity Playbook

Document your setup:
- IDE configuration
- Shell aliases and scripts
- Debugging approach
- Daily routine

---

## Deliverables

1. **Dotfiles Repository**: Version-controlled development setup
2. **Time Tracking Analysis**: Before/after comparison
3. **Debugging Case Study**: Documented investigation
4. **Productivity Playbook**: Personal workflow guide

---

## Reflection Questions

1. What's your most time-consuming repetitive task?
2. How often do you use the debugger vs. print statements?
3. When are you most productive, and how can you protect that time?
4. What would a 10x productivity improvement look like for you?

---

## Resources

### IDE Productivity
- [Rider Productivity Guide](https://www.jetbrains.com/help/rider/Productivity_Guide.html)
- [Visual Studio Productivity Tips](https://www.youtube.com/watch?v=IA1WpTVklLQ) - Scott Hanselman
- [VS Code Tips and Tricks](https://code.visualstudio.com/docs/getstarted/tips-and-tricks)

### Debugging
- [Julia Evans' Debugging Zine](https://jvns.ca/blog/2019/06/23/a-few-debugging-resources/)
- [The Debugging Mindset](https://www.youtube.com/watch?v=W9ghXs9DpR8) - Sasha Goldshtein
- [Debugging with Visual Studio](https://learn.microsoft.com/en-us/visualstudio/debugger/)

### Productivity
- [Deep Work](https://www.calnewport.com/books/deep-work/) - Cal Newport
- [The Pragmatic Programmer](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/)
- [Hammock Driven Development](https://www.youtube.com/watch?v=f84n5oFoZBc) - Rich Hickey

### Dotfiles
- [Awesome Dotfiles](https://github.com/webpro/awesome-dotfiles)
- [GitHub Dotfiles](https://dotfiles.github.io/)
- [Holman Dotfiles](https://github.com/holman/dotfiles)

---

**Next:** [Part 3 — Career Skills & Communication](./part-3.md)
