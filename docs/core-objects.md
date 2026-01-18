# Gas Town Core Objects Reference

This document provides a comprehensive technical reference for all core objects in Gas Town, including their data structures, relationships, and usage patterns.

## Overview

Gas Town is an **AI agent orchestration system** that coordinates multiple Claude Code instances across Git repositories. The system provides:

- **Accountability** - Every action is attributed to a specific agent
- **Durability** - Work survives agent crashes via beads-backed state
- **Scalability** - Coordinate agents across multiple repositories
- **Quality tracking** - Historical data on agent performance

## Core Principles

| Principle | Meaning |
|-----------|---------|
| **GUPP** (Gas Town Universal Propulsion Principle) | "If there is work on your Hook, YOU MUST RUN IT" - agents self-propel |
| **MEOW** (Molecular Expression of Work) | Breaking goals into detailed, trackable instructions |
| **NDI** (Nondeterministic Idempotence) | Guarantee completion despite unreliable individual operations |
| **ZFC** (Zero False Commands) | Trust agent self-reports, don't infer state via regex |

---

## 1. Town

The **town** is the root workspace directory (`~/gt/`) serving as the central coordination hub for all agents and projects.

### Data Structure

```go
// File: internal/config/types.go
type TownConfig struct {
    Type       string    `json:"type"`       // "town"
    Version    int       `json:"version"`    // schema version
    Name       string    `json:"name"`       // town identifier
    Owner      string    `json:"owner"`      // owner email
    PublicName string    `json:"public_name"`
    CreatedAt  time.Time `json:"created_at"`
}
```

### Key Configuration Files

| File | Purpose |
|------|---------|
| `mayor/town.json` | Town identity (name, owner) |
| `mayor/rigs.json` | Registry of all rigs |
| `settings/config.json` | Default agent, role-specific agents |
| `mayor/daemon.json` | Heartbeat and patrol cycles |

### Commands

```bash
gt install [path]        # Create town
gt doctor                # Health check
gt doctor --fix          # Auto-repair
gt status                # Overall town status
```

---

## 2. Rig

A **rig** is a project-specific Git repository container. Each rig has its own Witness, Refinery, Polecats, and Crew.

### Data Structure

```go
// File: internal/rig/types.go
type Rig struct {
    Name        string              `json:"name"`        // Rig identifier (directory name)
    Path        string              `json:"path"`        // Absolute path to rig directory
    GitURL      string              `json:"git_url"`     // Remote repository URL
    LocalRepo   string              `json:"local_repo"`  // Optional local reference repo
    Config      *config.BeadsConfig `json:"config"`      // Rig-level configuration
    Polecats    []string            `json:"polecats"`    // Active polecat names
    Crew        []string            `json:"crew"`        // Crew member names
    HasWitness  bool                `json:"has_witness"`
    HasRefinery bool                `json:"has_refinery"`
    HasMayor    bool                `json:"has_mayor"`
}
```

### Rig Configuration

```go
// File: internal/rig/manager.go
type RigConfig struct {
    Type          string       `json:"type"`                     // "rig"
    Version       int          `json:"version"`                  // schema version
    Name          string       `json:"name"`                     // rig name
    GitURL        string       `json:"git_url"`                  // repository URL
    LocalRepo     string       `json:"local_repo,omitempty"`     // optional local reference
    DefaultBranch string       `json:"default_branch,omitempty"` // main, master, etc.
    CreatedAt     time.Time    `json:"created_at"`
    Beads         *BeadsConfig `json:"beads,omitempty"`
}

type BeadsConfig struct {
    Prefix     string `json:"prefix"`                // issue prefix (e.g., "gt")
    SyncRemote string `json:"sync_remote,omitempty"` // git remote for bd sync
}
```

### Directory Layout

```
<rig>/                          Project container (NOT a git clone)
├── config.json                 Rig identity
├── .beads/                     Redirect to mayor/rig/.beads
├── .repo.git/                  Bare repo (shared by all worktrees)
├── mayor/rig/                  Mayor's clone (canonical beads)
│   └── .beads/                 Rig-level beads database
├── refinery/rig/               Worktree on main branch
├── witness/                    Witness agent (no clone)
├── crew/                       Persistent human workspaces
│   └── <name>/rig/             Individual crew workspace
└── polecats/                   Ephemeral worker worktrees
    └── <name>/rig/             Individual polecat worktree
```

### Operational States

| State | Meaning | Set Via | Persistence |
|-------|---------|---------|-------------|
| **OPERATIONAL** | Normal operation | Default | — |
| **PARKED** | Local pause (daemon won't restart) | `gt rig park` | Wisp layer (ephemeral) |
| **DOCKED** | Global shutdown (synced via git) | `gt rig dock` | Rig identity bead |

### Commands

```bash
gt rig add <name> <git-url>    # Create and register rig
gt rig list                    # List all rigs
gt rig status <rig>            # Show detailed status
gt rig boot <rig>              # Start witness and refinery
gt rig shutdown <rig>          # Stop all agents
gt rig park <rig>              # Pause locally
gt rig dock <rig>              # Pause globally (persisted)
gt rig unpark <rig>            # Resume from parked
gt rig undock <rig>            # Resume from docked
```

---

## 3. Beads

**Beads** is an AI-native issue tracking system stored in Git. Beads are the fundamental unit of work tracking.

### Core Data Structure

```go
// File: internal/beads/beads.go
type Issue struct {
    ID          string   `json:"id"`           // Unique ID (e.g., "gt-123")
    Title       string   `json:"title"`        // Human-readable title
    Description string   `json:"description"`  // Structured fields as "key: value"
    Status      string   `json:"status"`       // open, in_progress, hooked, closed
    Priority    int      `json:"priority"`     // 0-4 (0=highest)
    Type        string   `json:"issue_type"`   // task, bug, agent, queue, etc.
    CreatedAt   string   `json:"created_at"`
    CreatedBy   string   `json:"created_by"`
    Parent      string   `json:"parent"`       // Parent issue ID
    Children    []string `json:"children"`     // Child issue IDs
    DependsOn   []string `json:"depends_on"`   // Dependencies
    Blocks      []string `json:"blocks"`
    BlockedBy   []string `json:"blocked_by"`
    Labels      []string `json:"labels"`       // Tags like "gt:agent"
    Assignee    string   `json:"assignee"`     // Agent identity
}
```

### Bead Types

| Type | ID Format | Label | Purpose |
|------|-----------|-------|---------|
| **Task/Bug/Feature** | `gt-123` | — | Standard work items |
| **Agent** | `gt-<rig>-<role>-<name>` | `gt:agent` | Agent lifecycle tracking |
| **Queue** | `hq-q-<name>` | `gt:queue` | Work queues (claim-based) |
| **Channel** | `hq-channel-<name>` | `gt:channel` | Pub/sub message streams |
| **Group** | `hq-group-<name>` | `gt:group` | Mailing lists |
| **Merge Request** | Auto-generated | `gt:merge-request` | Code merges to main |
| **Molecule** | Auto-generated | `gt:molecule` | Workflow templates |
| **Escalation** | Auto-generated | `gt:escalation` | Incident tracking |

### ID Prefix Conventions

- `hq-*` = Town-level beads (groups, channels, mayor, deacon)
- `gt-*` = Rig-level beads (tasks, polecats, witnesses)
- Custom prefixes per rig (configured in `routes.jsonl`)

### Storage Layout

```
.beads/
├── beads.db           # SQLite database (source of truth)
├── *.jsonl            # Exported beads for git sync
├── routes.jsonl       # Prefix→path routing
└── formulas/          # Workflow templates
```

### Commands

```bash
bd list                        # List open issues
bd show <id>                   # Show issue details
bd create "Title"              # Create new issue
bd update <id> --status=done   # Update issue
bd close <id>                  # Close issue
bd sync                        # Sync with git
bd ready                       # Work with no blockers
```

---

## 4. Formulas

**Formulas** are TOML-based workflow definitions with validation, cycle detection, and execution planning.

### Data Structure

```go
// File: internal/formula/types.go
type Formula struct {
    Name        string            `toml:"formula"`     // Formula identifier
    Description string            `toml:"description"` // Documentation
    Type        FormulaType       `toml:"type"`        // workflow|convoy|expansion|aspect
    Version     int               `toml:"version"`     // Schema version
    Steps       []Step            `toml:"steps"`       // For workflows
    Vars        map[string]Var    `toml:"vars"`        // Template variables
}

type Step struct {
    ID          string   `toml:"id"`          // Unique step ID
    Title       string   `toml:"title"`       // Human-readable title
    Description string   `toml:"description"` // Instructions
    Needs       []string `toml:"needs"`       // Dependencies (other step IDs)
}

type Var struct {
    Description string `toml:"description"`
    Required    bool   `toml:"required"`
    Default     string `toml:"default"`
}
```

### Formula Types

| Type | Purpose | Use Case |
|------|---------|----------|
| **WORKFLOW** | Sequential steps with dependencies | Multi-step implementation |
| **CONVOY** | Parallel legs with optional synthesis | Coordinated parallel work |
| **EXPANSION** | Template-based with `{{variable}}` substitution | Reusable patterns |
| **ASPECT** | Cross-cutting concerns (before/after advice) | Security audits, logging |

### Example: Workflow Formula

```toml
# File: .beads/formulas/shiny.formula.toml
formula = "shiny"
type = "workflow"
version = 1

description = "Engineer in a Box - Design before you code. Review before you ship."

[[steps]]
id = "design"
title = "Design {{feature}}"
description = "Think carefully about architecture before writing code."

[[steps]]
id = "implement"
title = "Implement {{feature}}"
needs = ["design"]
description = "Write the code for {{feature}}. Follow the design."

[[steps]]
id = "review"
title = "Review implementation"
needs = ["implement"]
description = "Review the implementation. Does it match the design?"

[[steps]]
id = "test"
title = "Test {{feature}}"
needs = ["review"]
description = "Write and run tests."

[[steps]]
id = "submit"
title = "Submit for merge"
needs = ["test"]
description = "Submit for merge. Final check: git status, git diff."

[vars]
[vars.feature]
description = "The feature being implemented"
required = true
```

### Resolution Hierarchy

Formulas are resolved in order (most specific wins):

1. **Project-level**: `<project>/.beads/formulas/`
2. **Town-level**: `~/.beads/formulas/`
3. **System-level**: Compiled into `gt` binary

### Commands

```bash
gt formula list                # List available formulas
gt formula show <name>         # Display formula details
bd cook <formula>              # Create protomolecule from formula
```

---

## 5. Molecules

**Molecules** are active workflow instances created from formulas. They decompose work into interdependent steps (beads) and track progress.

### Data Structure

```go
// File: internal/beads/molecule.go
type MoleculeStep struct {
    Ref          string         `json:"ref"`          // Step reference
    Title        string         `json:"title"`        // Step title
    Instructions string         `json:"instructions"` // Prose instructions
    Needs        []string       `json:"needs"`        // Dependencies
    WaitsFor     []string       `json:"waits_for"`    // Dynamic conditions
    Tier         string         `json:"tier"`         // haiku, sonnet, opus
    Type         string         `json:"type"`         // task, wait
    Backoff      *BackoffConfig `json:"backoff"`      // For wait-type steps
}
```

### Lifecycle Pipeline

```
Formula (TOML source)
    │ bd cook
    ▼
Protomolecule (frozen template)
    │ bd mol pour
    ▼
Molecule (active workflow with step beads)
    │ bd mol squash
    ▼
Digest (archived summary)
```

### Molecule Variants

| Variant | Storage | Purpose |
|---------|---------|---------|
| **Molecule** | Persistent in `.beads/` | Tracked workflows with audit trail |
| **Wisp** | Ephemeral (not synced) | Temporary patrol cycles |

### Commands

```bash
bd mol pour <proto>            # Create persistent molecule
bd mol wisp <proto>            # Create ephemeral wisp
bd mol current                 # Show current step
bd mol progress <id>           # Show completion progress
bd mol squash <id>             # Archive completed molecule
bd close <step> --continue     # Close step and auto-advance
```

---

## 6. Agent Roles

### Overview Table

| Role | Scope | Purpose | Lifecycle |
|------|-------|---------|-----------|
| **Mayor** | Town | Global coordinator | Singleton, persistent |
| **Deacon** | Town | Daemon watchdog | Singleton, persistent |
| **Witness** | Rig | Polecat lifecycle manager | One per rig, persistent |
| **Refinery** | Rig | Merge queue processor | One per rig, persistent |
| **Polecat** | Rig | Ephemeral task worker | Transient, Witness-managed |
| **Crew** | Rig | Persistent human worker | Long-lived, user-managed |
| **Dog** | Town | Deacon helper | Ephemeral, Deacon-managed |

### Polecat

**Purpose**: Execute discrete work items, then self-destruct.

```go
// File: internal/polecat/types.go
type Polecat struct {
    Name      string    // Pooled name (polecat-01 through polecat-50)
    Rig       string    // Parent rig
    State     State     // working, done, stuck
    ClonePath string    // polecats/<name>/rig/
    Branch    string    // polecat/<name>/<issue>@<timestamp>
    Issue     string    // Currently assigned issue ID
}

type State string
const (
    StateWorking State = "working"  // Active session
    StateDone    State = "done"     // Ready for cleanup
    StateStuck   State = "stuck"    // Needs help
)
```

**Lifecycle**:
1. Witness spawns polecat with issue assignment
2. Polecat creates git worktree and branch
3. Polecat implements solution
4. Polecat submits merge request
5. Polecat sends `POLECAT_DONE` mail to Witness
6. Witness nukes polecat when cleanup_status=clean

### Witness

**Purpose**: Monitor polecats, auto-spawn ready issues, handle completions.

```go
// File: internal/witness/types.go
type Witness struct {
    RigName           string
    State             State           // Running, Stopped, Paused
    MonitoredPolecats []string
    Config            WitnessConfig
}

type WitnessConfig struct {
    MaxWorkers   int    // Default: 4
    SpawnDelayMs int    // Default: 5000
    AutoSpawn    bool   // Default: true
}
```

### Refinery

**Purpose**: Process merge requests from polecats, handle conflicts.

```go
// File: internal/refinery/types.go
type MergeRequest struct {
    ID           string
    Branch       string      // polecat/<worker>/<issue>@<timestamp>
    Worker       string      // Who created this MR
    IssueID      string      // Original work item
    TargetBranch string      // Usually "main"
    Status       MRStatus    // open, in_progress, closed
    CloseReason  CloseReason // merged, rejected, conflict
}
```

**Merge Flow**:
1. Polecat creates merge-request bead
2. Refinery picks up ready MRs
3. Refinery merges to main (or creates conflict task)
4. Refinery closes source issue
5. Refinery notifies worker

### Deacon

**Purpose**: Monitor all agents, unhook stale beads, restart stuck sessions.

```go
// File: internal/deacon/heartbeat.go
type Heartbeat struct {
    Timestamp       time.Time
    Cycle           int64
    LastAction      string
    HealthyAgents   int
    UnhealthyAgents int
}
```

**Health Check Loop**:
1. Read heartbeat to decide if poke needed
2. Send HEALTH_CHECK nudges to agents
3. Track consecutive failures
4. Force-kill after threshold (with cooldown)
5. Unhook stale beads from dead agents

---

## 7. Hooks

A **hook** is the durability primitive - work attached to an agent survives session restarts.

### How Hooks Work

```go
// Bead hooked to an agent
status = "hooked"
assignee = "gastown/polecats/max"

// Attachment fields (optional molecule)
attached_molecule = "gt-xyz-123"
attached_at = "2026-01-18T10:30:00Z"
```

### Hook Slot Rules

- Only ONE bead can be hooked per agent
- Auto-replaces completed beads
- Requires `--force` to replace incomplete work
- Stale hooks (agent dead) are auto-unhooked by Deacon

### Commands

```bash
gt hook                        # Show my hook status
gt hook <bead-id>              # Attach work to my hook
gt hook --clear                # Clear my hook
gt hook show <agent>           # Show another agent's hook
```

---

## 8. Mail System

The mail system provides inter-agent communication via beads-backed messages.

### Message Structure

```go
// File: internal/mail/types.go
type Message struct {
    ID        string      // Unique ID
    From      string      // Sender address
    To        string      // Direct recipient
    Subject   string      // Brief summary
    Body      string      // Full content
    Timestamp time.Time
    Read      bool        // Closed in beads = read
    Priority  Priority    // low, normal, high, urgent
    Type      MessageType // task, notification, reply
    Delivery  Delivery    // queue or interrupt
    Queue     string      // For queue messages
    Channel   string      // For channel broadcasts
}
```

### Addressing Patterns

| Pattern | Example | Behavior |
|---------|---------|----------|
| **Direct** | `gastown/crew/max` | Single recipient |
| **Queue** | `queue:work-items` | Claim-based (one worker) |
| **Channel** | `channel:alerts` | Broadcast with retention |
| **List** | `list:ops-team` | Fan-out to members |
| **Group** | `@witnesses` | Pattern-based expansion |

### Commands

```bash
gt mail inbox                  # Check inbox
gt mail send <to> "Subject"    # Send message
gt mail read <id>              # Read message
gt nudge <session> "Message"   # Real-time interrupt
```

---

## 9. Relationships Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                           TOWN (~gt/)                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │   Mayor     │  │   Deacon    │  │     .beads/ (hq-*)      │ │
│  │  (coord)    │◄─┤  (watchdog) │  │  groups, channels, mail │ │
│  └─────────────┘  └──────┬──────┘  └─────────────────────────┘ │
│                          │ monitors                             │
├──────────────────────────┼──────────────────────────────────────┤
│                          ▼                                      │
│  ┌──────────────────── RIG ────────────────────────────────┐   │
│  │  ┌─────────┐  ┌──────────┐  ┌────────────────────────┐  │   │
│  │  │ Witness │  │ Refinery │  │    .beads/ (gt-*)      │  │   │
│  │  │ (patrol)│  │ (merge)  │  │  tasks, MRs, agents    │  │   │
│  │  └────┬────┘  └────┬─────┘  └────────────────────────┘  │   │
│  │       │ manages    │ merges                              │   │
│  │       ▼            ▼                                     │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              POLECATS (ephemeral)               │    │   │
│  │  │  polecat-01  polecat-02  polecat-03  ...        │    │   │
│  │  │     │            │            │                 │    │   │
│  │  │     ▼            ▼            ▼                 │    │   │
│  │  │  [worktree]  [worktree]  [worktree]             │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │                CREW (persistent)                │    │   │
│  │  │  crew/joe    crew/max    (user-managed)         │    │   │
│  │  └─────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Request
    │
    ▼
gt convoy create "Feature" gt-123
    │
    ▼
gt sling gt-123 <rig>
    │
    ├──► Witness receives notification
    │        │
    │        ▼
    │    Witness spawns Polecat
    │        │
    │        ▼
    │    Polecat works on gt-123
    │        │
    │        ▼
    │    Polecat creates merge-request
    │        │
    │        ▼
    │    Refinery merges to main
    │        │
    │        ▼
    │    Issue closed, convoy updated
    │
    ▼
User notified via mail
```

---

## 10. Key Files Reference

| Component | Key Files |
|-----------|-----------|
| **Town** | `internal/config/types.go`, `internal/config/loader.go` |
| **Rig** | `internal/rig/manager.go`, `internal/rig/types.go` |
| **Beads** | `internal/beads/beads.go`, `internal/beads/fields.go` |
| **Formula** | `internal/formula/parser.go`, `internal/formula/types.go` |
| **Molecule** | `internal/beads/molecule.go` |
| **Polecat** | `internal/polecat/manager.go`, `internal/polecat/session_manager.go` |
| **Witness** | `internal/witness/manager.go`, `internal/witness/handlers.go` |
| **Refinery** | `internal/refinery/manager.go`, `internal/refinery/engineer.go` |
| **Deacon** | `internal/deacon/manager.go`, `internal/deacon/heartbeat.go` |
| **Mayor** | `internal/mayor/manager.go` |
| **Mail** | `internal/mail/router.go`, `internal/mail/types.go` |
| **Tmux** | `internal/tmux/tmux.go` |
| **Hooks** | `internal/cmd/hook.go` |
