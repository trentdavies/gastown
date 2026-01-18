# Formulas Guide

Formulas are TOML-based workflow definitions that provide structured, reusable patterns for multi-step work. This guide explains how to read, write, and use formulas in Gas Town.

## What Are Formulas?

A **formula** is a template that defines:

- **Steps** - The discrete tasks to complete
- **Dependencies** - Which steps depend on others (`needs`)
- **Variables** - Parameterizable values (`{{variable}}`)
- **Instructions** - Prose guidance for each step

Formulas enable:

- **Crash recovery** - Work resumes from the last completed step
- **Auditability** - Each step creates a bead for tracking
- **Reusability** - Same pattern applies to different work
- **Validation** - Cycle detection, required fields, unique IDs

## Formula Types

| Type | Purpose | Use Case |
|------|---------|----------|
| **workflow** | Sequential steps with dependencies | Feature implementation, patrols |
| **convoy** | Parallel legs with optional synthesis | Coordinated parallel work |
| **expansion** | Template-based with variable substitution | Reusable patterns |
| **aspect** | Cross-cutting concerns (before/after advice) | Security audits, logging |

## Basic Structure

```toml
formula = "my-formula"           # Unique identifier
type = "workflow"                # workflow | convoy | expansion | aspect
version = 1                      # Schema version
description = "What this formula does"

[[steps]]
id = "step-1"                    # Unique within formula
title = "First step"             # Human-readable title
description = "Detailed instructions for the agent"

[[steps]]
id = "step-2"
title = "Second step"
needs = ["step-1"]               # Depends on step-1
description = "This runs after step-1 completes"

[vars]
[vars.feature]
description = "The feature being implemented"
required = true                  # Must be provided at instantiation
```

## Example 1: Simple Workflow (`shiny`)

The canonical "Engineer in a Box" pattern - design before code, review before ship.

```toml
description = "Engineer in a Box - the canonical right way. Design before you code. Review before you ship."
formula = "shiny"
type = "workflow"
version = 1

[[steps]]
description = "Think carefully about architecture before writing code. Consider: How does this fit into the existing system? What are the edge cases? What could go wrong? Is there a simpler approach?"
id = "design"
title = "Design {{feature}}"

[[steps]]
description = "Write the code for {{feature}}. Follow the design. Keep it simple. Don't gold-plate."
id = "implement"
needs = ["design"]
title = "Implement {{feature}}"

[[steps]]
description = "Review the implementation. Check for: Does it match the design? Are there obvious bugs? Is it readable and maintainable? Are there security concerns?"
id = "review"
needs = ["implement"]
title = "Review implementation"

[[steps]]
description = "Write and run tests. Unit tests for new code, integration tests if needed, run the full test suite, fix any regressions."
id = "test"
needs = ["review"]
title = "Test {{feature}}"

[[steps]]
description = "Submit for merge. Final check: git status, git diff. Commit with clear message. Follow your role's git workflow for landing code."
id = "submit"
needs = ["test"]
title = "Submit for merge"

[vars]
[vars.assignee]
description = "Who is assigned to this work"
[vars.feature]
description = "The feature being implemented"
required = true
```

**Dependency Graph:**
```
design → implement → review → test → submit
```

**Usage:**
```bash
bd cook shiny
bd mol pour shiny --var feature="dark mode toggle"
```

## Example 2: Complex Workflow (`mol-polecat-work`)

A comprehensive work lifecycle for ephemeral workers, showing detailed instructions.

```toml
formula = "mol-polecat-work"
version = 4

description = """
Full polecat work lifecycle from assignment through completion.

## Polecat Contract (Self-Cleaning Model)

You are a self-cleaning worker. You:
1. Receive work via your hook (pinned molecule + issue)
2. Work through molecule steps using `bd ready` / `bd close <step>`
3. Complete and self-clean via `gt done` (submit + nuke yourself)
4. You are GONE - Refinery merges from MQ
"""

[[steps]]
id = "load-context"
title = "Load context and verify assignment"
description = """
Initialize your session and understand your assignment.

**1. Prime your environment:**
```bash
gt prime                    # Load role context
bd prime                    # Load beads context
```

**2. Check your hook:**
```bash
gt hook               # Shows your pinned molecule and hook_bead
```

**3. Understand the requirements:**
- What exactly needs to be done?
- What files are likely involved?
- Are there dependencies or blockers?
- What does "done" look like?

**Exit criteria:** You understand the work and can begin implementation."""

[[steps]]
id = "branch-setup"
title = "Set up working branch"
needs = ["load-context"]
description = """
Ensure you're on a clean feature branch ready for work.

```bash
git status
git branch --show-current
git checkout -b polecat/<name>    # If not on feature branch
git fetch origin
git rebase origin/main            # Sync with latest
```

**Exit criteria:** You're on a clean feature branch, rebased on latest main."""

[[steps]]
id = "implement"
title = "Implement the solution"
needs = ["branch-setup"]
description = """
Do the actual implementation work.

**Working principles:**
- Follow existing codebase conventions
- Make atomic, focused commits
- Keep changes scoped to the assigned issue
- Don't gold-plate or scope-creep

**Exit criteria:** Implementation complete, all changes committed."""

[[steps]]
id = "run-tests"
title = "Run tests and verify coverage"
needs = ["implement"]
description = """
Verify your changes don't break anything.

```bash
go test ./...               # Or appropriate test command
```

**ALL TESTS MUST PASS.** Do not proceed with failures.

**Exit criteria:** All tests pass, new code has appropriate test coverage."""

[[steps]]
id = "submit-and-exit"
title = "Submit work and self-clean"
needs = ["run-tests"]
description = """
Submit your work and clean up. You cease to exist after this step.

```bash
gt done
```

**Exit criteria:** Work submitted, sandbox nuked, session exited."""

[vars]
[vars.issue]
description = "The issue ID assigned to this polecat"
required = true
```

**Key Features:**

1. **Detailed instructions** - Each step has prose guidance with code examples
2. **Exit criteria** - Clear definition of "done" for each step
3. **Failure modes** - Documents what to do when things go wrong
4. **Variables** - `{{issue}}` is substituted at instantiation

## Example 3: Aspect Formula (`security-audit`)

Aspects inject before/after steps around existing workflow steps.

```toml
description = "Cross-cutting security concern. Applies security scanning before and after implementation steps."
formula = "security-audit"
type = "aspect"
version = 1

[[advice]]
target = "implement"
[advice.around]

[[advice.around.before]]
description = "Pre-implementation security check. Review for secrets/credentials in scope. Check dependencies for known vulnerabilities."
id = "{step.id}-security-prescan"
title = "Security prescan for {step.id}"

[[advice.around.after]]
description = "Post-implementation security scan. Scan new code for vulnerabilities (SAST). Check for hardcoded secrets. Review for OWASP Top 10 issues."
id = "{step.id}-security-postscan"
title = "Security postscan for {step.id}"

[[advice]]
target = "submit"
[advice.around]

[[advice.around.before]]
description = "Pre-submission security check. Final vulnerability scan before merge."
id = "{step.id}-security-prescan"
title = "Security prescan for {step.id}"

[[advice.around.after]]
description = "Post-submission security verification. Confirm no new vulnerabilities introduced."
id = "{step.id}-security-postscan"
title = "Security postscan for {step.id}"

[[pointcuts]]
glob = "implement"

[[pointcuts]]
glob = "submit"
```

**How It Works:**

When composed with `shiny`, the aspect wraps targeted steps:

```
design
    ↓
implement-security-prescan    ← Injected BEFORE
    ↓
implement                     ← Original step
    ↓
implement-security-postscan   ← Injected AFTER
    ↓
review
    ↓
test
    ↓
submit-security-prescan       ← Injected BEFORE
    ↓
submit                        ← Original step
    ↓
submit-security-postscan      ← Injected AFTER
```

**Usage:**
```toml
# In your workflow formula:
[compose]
aspects = ["security-audit"]
```

## Example 4: Patrol Workflow (`mol-witness-patrol`)

Patrol formulas are cyclical workflows for oversight agents.

```toml
formula = 'mol-witness-patrol'
version = 2

description = """
Per-rig worker monitor patrol loop.

The Witness is the Pit Boss for your rig. You watch polecats, nudge them toward
completion, verify clean git state before kills, and escalate stuck workers.

**You do NOT do implementation work.** Your job is oversight, not coding.

## Patrol Shape (Linear)

inbox-check → process-cleanups → check-refinery → survey-workers
                                                        ↓
     ┌──────────────────────────────────────────────────┘
     ↓
check-timer-gates → check-swarm → ping-deacon → patrol-cleanup → context-check → loop-or-exit
"""

[[steps]]
id = 'inbox-check'
title = 'Process witness mail'
description = """
Check inbox and handle messages.

```bash
gt mail inbox
```

For each message type:
- **POLECAT_DONE**: Auto-nuke if clean, create cleanup wisp if dirty
- **MERGED**: Archive after acknowledging
- **HELP**: Assess and help or escalate to Mayor
- **HANDOFF**: Read predecessor context, continue work
"""

[[steps]]
id = 'process-cleanups'
needs = ['inbox-check']
title = 'Process pending cleanup wisps'
description = """
Handle polecats with dirty state that couldn't be auto-nuked.

```bash
bd list --wisp --labels=cleanup --status=open
```

For each cleanup wisp, diagnose and resolve."""

[[steps]]
id = 'survey-workers'
needs = ['process-cleanups']
title = 'Inspect all active polecats'
description = """
Survey all polecats using agent beads (ZFC: trust what agents report).

```bash
bd list --type=agent --json
```

Check agent_state for each polecat and take appropriate action."""

[[steps]]
id = 'loop-or-exit'
needs = ['survey-workers']
title = 'Loop or exit for respawn'
description = """
End of patrol cycle decision.

**If context LOW**: Squash wisp, create new patrol wisp, continue
**If context HIGH**: Write handoff mail, exit for respawn
"""
```

**Key Pattern:**

Patrol workflows end with a "loop-or-exit" step that either:
1. Creates a new wisp and continues (low context)
2. Hands off to a fresh session (high context)

This enables continuous monitoring without context exhaustion.

## Formula Lifecycle

```
Formula (TOML source)
    │ bd cook
    ↓
Protomolecule (frozen template)
    │ bd mol pour
    ↓
Molecule (active workflow with step beads)
    │ bd mol squash
    ↓
Digest (archived summary)
```

### Commands

```bash
# List available formulas
gt formula list

# Show formula details
gt formula show shiny

# Create protomolecule from formula
bd cook shiny

# Instantiate molecule (persistent)
bd mol pour shiny --var feature="dark mode"

# Instantiate wisp (ephemeral, for patrols)
bd mol wisp mol-witness-patrol

# Work through steps
bd mol current                 # What step is next?
bd close <step-id> --continue  # Complete step, advance

# Archive completed molecule
bd mol squash <mol-id>
```

## Resolution Hierarchy

Formulas are resolved in order (most specific wins):

1. **Project-level**: `<project>/.beads/formulas/`
2. **Town-level**: `~/.beads/formulas/`
3. **System-level**: Compiled into `gt` binary

This allows project-specific overrides of system formulas.

## Variables

Variables use `{{name}}` syntax and are substituted at instantiation.

### Definition

```toml
[vars]
[vars.feature]
description = "The feature being implemented"
required = true      # Must be provided

[vars.assignee]
description = "Who is assigned"
default = "unassigned"   # Optional with default
```

### Usage in Steps

```toml
[[steps]]
title = "Design {{feature}}"
description = "Design the {{feature}} feature assigned to {{assignee}}"
```

### Instantiation

```bash
bd mol pour shiny --var feature="authentication" --var assignee="polecat-01"
```

## Best Practices

### Step Design

1. **Clear exit criteria** - Define what "done" means
2. **Actionable instructions** - Include actual commands
3. **Failure handling** - Document what to do when stuck
4. **Atomic scope** - Each step should be completable in one session

### Dependency Management

1. **Linear when possible** - Simpler to reason about
2. **Parallel when independent** - Use convoy type for true parallelism
3. **No cycles** - Formula validation catches these

### Variable Usage

1. **Required vs optional** - Mark appropriately
2. **Meaningful defaults** - Don't require what can be inferred
3. **Documentation** - Describe what each variable is for

### Formula Naming

| Prefix | Purpose | Example |
|--------|---------|---------|
| `mol-*` | System molecules (patrols, lifecycle) | `mol-witness-patrol` |
| `shiny*` | Implementation patterns | `shiny`, `shiny-secure` |
| `<project>-*` | Project-specific | `gastown-release` |

## Advanced Features

### Composition

```toml
# Extend a base formula
extends = ["base-formula"]

# Apply aspects
[compose]
aspects = ["security-audit", "logging"]

# Expand a step with another formula
[[compose.expand]]
target = "implement"
with = "tdd-cycle"
```

### Step Types

```toml
[[steps]]
id = "wait-for-approval"
type = "wait"              # Not a task, blocks until condition met
[steps.backoff]
initial = "5m"
max = "1h"
```

### Tier Hints

```toml
[[steps]]
id = "simple-check"
tier = "haiku"             # Suggest smaller model for simple steps

[[steps]]
id = "architecture-design"
tier = "opus"              # Suggest larger model for complex reasoning
```

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Cycle detected | Step A needs B needs A | Remove circular dependency |
| Missing variable | Required var not provided | Add `--var name=value` |
| Step not found | `needs` references unknown ID | Check step IDs match |
| Formula not found | Not in resolution path | Check `.beads/formulas/` |

## Key Files

| File | Purpose |
|------|---------|
| `internal/formula/parser.go` | Formula parsing and validation |
| `internal/formula/types.go` | Formula data structures |
| `internal/formula/formulas/` | System formulas (40+ files) |
| `.beads/formulas/` | Project/town-level formulas |
