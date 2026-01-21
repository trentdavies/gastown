# Beads Practical Guide

## Viewing Active Work

### See ready work (no blockers)
```bash
bd ready                         # Open/in_progress with no blockers
bd ready --assignee <agent>      # Filter by who's assigned
bd ready --type bug              # Filter by type
bd ready --priority 0            # Filter by priority (P0)
```

### See all work
```bash
bd list                          # All issues
bd list --status open            # Open only
bd list --status in_progress     # Currently being worked
bd list --status blocked         # Waiting on dependencies
bd list --long                   # Verbose output with assignee on own line
```

### See specific work
```bash
bd show <id>                     # Full details including dependencies
bd show gt-abc gt-def            # Multiple issues
```

### See convoys (batched work)
```bash
gt convoy list                   # All convoys
gt convoy status <convoy-id>     # Progress on specific batch
```

---

## Seeing Assignments

### Who's assigned to what
```bash
bd list --assignee toast         # Work assigned to "toast"
bd list --assignee "gastown/polecats/Toast"  # Full agent path
bd list --no-assignee            # Unassigned work
bd show <id>                     # Shows "Assignee:" in metadata
```

### What's on your hook (current agent)
```bash
gt hook                          # What YOU are actively running
gt prime                         # Full context: role, inbox, hooked work
gt prime --state                 # Session state
```

### Hook vs Assignee

| Concept | Meaning | Check With |
|---------|---------|------------|
| Assignee | Who's responsible (metadata) | `bd show <id>` |
| Hook | Who's actively running it now | `gt hook` |

### Why `bd ready` shows nothing
Work only appears in `bd ready` if:
- Status = `open` (not `in_progress`, `blocked`, or `closed`)
- Has no blockers (all dependencies closed)

Diagnose with:
```bash
bd list --status in_progress     # Already hooked
bd list --status blocked         # Waiting on deps
bd show <id>                     # Check DEPENDS ON section
```

---

## Handoffs

### Hand off your session
```bash
gt handoff                       # Spawn fresh session in your role
gt handoff -c                    # Collect state and pass to new session
gt handoff -s "context message"  # Pass message to new session
gt handoff <bead-id>             # Hook work before handing off
```

### Dispatch work to another agent
```bash
gt sling <id> <rig>              # Auto-spawn polecat, hook work
gt sling <id> crew               # Send to crew member
gt sling <id> mayor              # Send to mayor
gt sling <id> --args "instructions"  # With natural language args
```

### Signal completion
```bash
gt done                          # Complete, clear hook, terminate
gt done --status ESCALATED       # Blocker, needs human
gt done --status DEFERRED        # Pause, keep open
```

---

## Creating Beads

### Simple task
```bash
bd create "Fix login button"
# Returns: gt-a1b2c3d

bd create "Fix login button" \
  --type bug \
  --priority 1 \
  --description "Details here" \
  --assignee toast
```

Types: task, bug, feature, epic, chore, merge-request, molecule, gate, agent, role, rig, convoy, event

### Epic with children
```bash
# Create parent
bd create "Redesign auth" --type epic
# Returns: gt-xyz789

# Create children (get hierarchical IDs)
bd create "Design OAuth flow" --parent gt-xyz789
# Returns: gt-xyz789.1

bd create "Implement tokens" --parent gt-xyz789
# Returns: gt-xyz789.2
```

Check epic status:
```bash
bd epic status gt-xyz789         # Children completion summary
bd epic close-eligible           # Auto-close completed epics
```

---

## Formulas

Formulas define reusable workflows that generate beads with dependencies.

### View formulas
```bash
bd formula list                  # All available formulas
bd formula show shiny            # View specific formula
bd formula show shiny-enterprise
```

### Key formulas

**shiny** (base workflow):
```
design → implement → review → test → submit
```

**shiny-enterprise** (extends shiny, expands implement with rule-of-five):
```
design → implement.draft → implement.refine-1 → implement.refine-2 →
         implement.refine-3 → implement.refine-4 → review → test → submit
```

Rule-of-five refinements:
1. draft - Initial attempt, breadth over depth
2. refine-1 - Correctness (fix errors)
3. refine-2 - Clarity
4. refine-3 - Edge cases
5. refine-4 - Excellence (polish)

### Run a formula
```bash
gt formula run shiny-enterprise --target gt-xxx --var feature="Caching layer"
```

This generates child beads with wired dependencies:
```
gt-xxx.1  design
gt-xxx.2  implement.draft
gt-xxx.3  implement.refine-1
...
gt-xxx.9  submit
```

### Formula locations (in order of precedence)
1. `.beads/formulas/` (project)
2. `~/.beads/formulas/` (user)
3. `$GT_ROOT/.beads/formulas/` (town)

---

## Example: Epic with shiny-enterprise

```bash
# 1. Create epic
bd create "Add Redis caching" --type epic
# Returns: gt-cache01

# 2. Run formula
gt formula run shiny-enterprise --target gt-cache01 --var feature="Redis caching"

# 3. Check ready work
bd ready --parent gt-cache01
# Shows: gt-cache01.1 (design) ready

# 4. Sling first step
gt sling gt-cache01.1 gastown --args "consider invalidation strategies"

# 5. Track progress
bd show gt-cache01
bd epic status gt-cache01
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Ready work | `bd ready` |
| All work | `bd list` |
| Work details | `bd show <id>` |
| My hooked work | `gt hook` |
| My full context | `gt prime` |
| Create task | `bd create "Title"` |
| Create epic | `bd create "Title" --type epic` |
| Create child | `bd create "Title" --parent <id>` |
| Sling work | `gt sling <id> <rig>` |
| Hand off | `gt handoff` |
| Done | `gt done` |
| List formulas | `bd formula list` |
| Run formula | `gt formula run <name> --target <id>` |
