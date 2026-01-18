# Conversation Log

## User

```
# AGENTS.md instructions for /workspace/gastown

<INSTRUCTIONS>
# Agent Instructions

See **CLAUDE.md** for complete agent context and instructions.

This file exists for compatibility with tools that look for AGENTS.md.

> **Recovery**: Run `gt prime` after compaction, clear, or new session

Full context is injected by `gt prime` at session start.

</INSTRUCTIONS>
```

## User

```
<environment_context>
  <cwd>/workspace/gastown</cwd>
  <approval_policy>never</approval_policy>
  <sandbox_mode>danger-full-access</sandbox_mode>
  <network_access>enabled</network_access>
  <shell>bash</shell>
</environment_context>
```

## User

```
Analyze the code of gastown. Go in depth. Help me to understand the core data structures. Help me understand the core concepts so that I can use it effectively. Assume I am a senior engineer and very adept with the terminal and tmux.
```

## Assistant

Below is an in-depth, code-referenced walkthrough of Gas Town’s core concepts and the primary data structures that make the system work. I’ve organized it for a senior engineer who wants to reason about the system quickly and effectively.

---

## High-level architecture (what the system is doing)

Gas Town is a **multi-agent orchestration workspace** that persists agent work state and coordination metadata in a git-backed “beads” ledger. Its README frames the top-level roles and topology: **Town** (workspace root), **Rigs** (per-repo containers), **Crew** (your persistent workspace), **Polecats** (ephemeral workers), **Hooks** (worktrees used for persistence), and **Convoys** (work bundles). The **Mayor** is the orchestration coordinator and is the primary interface. These concepts are spelled out directly in the README, along with the canonical workflow and example commands you’ll use to run the system day-to-day.【F:README.md†L5-L211】

---

## Core data structures (what the system *stores* and *moves around*)

### 1) **Town & rig configuration models (identity + behavior)**
These structs define the stable identities of towns and rigs, plus behavioral settings like default agent models, patrol schedules, and merge queue settings:

- **TownConfig**: town identity, owner, public name, versioning.  
- **MayorConfig**: town-wide behavioral config (theme, daemon/deacon settings).  
- **TownSettings**: agent defaults, role-specific agent aliases, and agent runtime registry.  
- **RigsConfig / RigEntry**: registry of rigs (git URL, added time, beads config).  
- **RigConfig / RigSettings**: per-rig identity and behavior (including merge queue, runtime/agent selections).  

These are the canonical schema types and are central to understanding how Gas Town is configured at both the town and rig level.【F:internal/config/types.go†L11-L232】

### 2) **Rig model**
The `Rig` struct is the in-memory representation of a repo managed by Gas Town. It stores identity, paths, configuration, and agent inventory (polecats, crew) plus boolean flags for agents like witness/refinery. It’s the “core object” tying repo structure to orchestration state.  
Also note `AgentDirs`, the standardized directory layout for rig subagents, and `BeadsPath()` which defines where all beads operations should target (rig root), emphasizing the bead redirect system for safe storage.【F:internal/rig/types.go†L8-L94】

### 3) **Beads issue model (work-state graph)**
The beads “Issue” is the **primary persistent work unit**, and it includes fields for dependencies, blocks, labels, assignee, etc., along with agent-specific slots like `HookBead`, `RoleBead`, and `AgentState`.  
This struct represents the backbone of work tracking: dependencies and relationships are encoded directly into the beads issue graph and are used by convoys and mail.  
Gas Town wraps the `bd` CLI with a `Beads` struct that provides the operational interface (with safe environment isolation, explicit `BEADS_DIR`, etc.).【F:internal/beads/beads.go†L25-L208】

### 4) **Mail message model (agent coordination)**
Agents communicate via **mail messages** that are backed by beads. A `Message` captures sender/recipient, priority, delivery mode, and routing type (direct, queue, or channel).  
Queue and channel semantics are explicitly modeled; a message can be queued, broadcast, or direct, and those are mutually exclusive. This is the heart of how agents are assigned work and notified of state changes.【F:internal/mail/types.go†L12-L239】

### 5) **Protocol messages between Witness and Refinery**
The merge flow between verification (Witness) and integration (Refinery) uses protocol message payloads.  
Key types:

- `MERGE_READY`: Witness → Refinery, work verified  
- `MERGED`, `MERGE_FAILED`, `REWORK_REQUEST`: Refinery → Witness  

Each has a typed payload with branch/issue/polecat/rig metadata, timestamps, and (for failures) error and conflict info. This defines the inter-agent protocol for merge lifecycle operations.【F:internal/protocol/types.go†L1-L182】

### 6) **Merge queue data (Refinery)**
Refinery’s merge queue is defined by:

- `MergeQueueConfig`: queue behavior (target branch, conflict strategy, tests, concurrency).  
- `MRInfo`: the in-flight merge request state (issue ID, branch, priority, convoy info, etc.).  
- `Engineer`: the merge-queue processor holding pointers to beads, git, and mail routing.

This is the data layer for the “merge automation” pipeline in Gas Town.【F:internal/refinery/engineer.go†L23-L219】

### 7) **Convoy UI/analytics model**
Convoys are represented in the TUI with `Convoy` and `ConvoyState`. The convoy panel fetches convoys from the town-level beads database, calculates tracked item completion (via dependencies), and exposes “in progress” vs. “landed” lists.  
This structure is crucial for monitoring multi-issue work bundles and gives you the high-level project dashboard view.【F:internal/tui/feed/convoy.go†L25-L229】

### 8) **Agent runtime state**
Each agent’s runtime state is stored via a generic `StateManager[T]`, persisted as JSON under `.runtime/`. This is how agents are tracked as `running`, `stopped`, or `paused`, and it’s used by different sub-agents (witness, refinery, deacon, etc.).【F:internal/agent/state.go†L13-L76】

### 9) **Workspace discovery**
Workspace detection uses a “primary marker” (`mayor/town.json`) and a secondary marker (`mayor/`) to locate the town root by walking up the filesystem. This logic explains how `gt` commands can be run from anywhere in a rig or worktree and still resolve the correct town root.【F:internal/workspace/find.go†L14-L160】

---

## Core concepts mapped to the code

### 1) **Beads as the durable source of truth**
Beads issues (and their dependencies) are the **canonical storage** for work state, allowing the system to survive agent restarts and reorganize tasks. The `Issue` struct shows how relationships and metadata are persisted, while the beads CLI wrapper enforces consistent access patterns and environment isolation.【F:internal/beads/beads.go†L25-L208】

### 2) **Convoys as bundling and progress tracking**
Convoys are work bundles (top-level issues) that “track” sets of bead IDs. The TUI’s convoy fetcher uses beads and SQLite to compute status and completion metrics, which is how `gt convoy list` and the dashboard give you progress visibility.【F:internal/tui/feed/convoy.go†L25-L229】

### 3) **Agent coordination through mail**
All agent-to-agent and coordinator communication is modeled as mail, with delivery mode and routing fields allowing direct messages, queue-based assignment, or channel broadcasts. This is how your orchestration turns into actionable assignments and notifications.【F:internal/mail/types.go†L12-L239】

### 4) **Merge lifecycle via protocol messages**
Witness and Refinery coordinate merges using typed protocol payloads. These are high-signal, structured messages that mark when work is verified, merged, failed, or needs rework, and they carry explicit metadata for automation and auditing.【F:internal/protocol/types.go†L1-L182】

### 5) **Rig + Town as the core ownership boundaries**
A rig encapsulates a repo and its agents; the town encapsulates all rigs and global behavior. Configuration types show how defaults cascade from town settings to rig settings, and rig types show how agent directories and flags are tracked in-memory.【F:internal/config/types.go†L11-L232】【F:internal/rig/types.go†L8-L94】

---

## Practical usage guidance (workflow mental model)

### Day-to-day orchestration flow
1. **Attach to the Mayor** and describe desired outcomes.  
2. **Create convoys** to bundle related work.  
3. **Sling beads** to polecats (workers).  
4. **Track convoys** and mail for progress updates.  

This is documented in the README’s canonical workflow and examples.【F:README.md†L134-L211】

### Why this matters as you scale
- Work units are **persistent beads** rather than ephemeral agent context.
- Assignments are **mail-driven**, so work distribution is explicit and auditable.
- Merge and verification are **protocolized**, allowing automation across many agents.

---

## Commands run (for transparency)
* `sed -n '1,200p' README.md`【F:README.md†L1-L200】
* `sed -n '1,200p' internal/beads/beads.go`【F:internal/beads/beads.go†L1-L208】
* `sed -n '1,200p' internal/rig/types.go`【F:internal/rig/types.go†L1-L94】
* `sed -n '1,200p' internal/config/types.go`【F:internal/config/types.go†L1-L232】
* `sed -n '1,200p' internal/agent/state.go`【F:internal/agent/state.go†L1-L76】
* `sed -n '1,200p' internal/workspace/find.go`【F:internal/workspace/find.go†L1-L160】
* `sed -n '1,200p' internal/mail/types.go`【F:internal/mail/types.go†L1-L239】
* `sed -n '1,200p' internal/protocol/types.go`【F:internal/protocol/types.go†L1-L182】
* `sed -n '1,200p' internal/refinery/engineer.go`【F:internal/refinery/engineer.go†L1-L219】
* `sed -n '1,200p' internal/tui/feed/convoy.go`【F:internal/tui/feed/convoy.go†L1-L229】

---

## User

```
Explain the correct way to spawn work in the forms of convoys and beads. How is this done when you are attached to the mayor, for example?
```

## Assistant

Below is a **focused, code‑referenced explanation** of how to spawn work using **convoys** and **beads**, including how this works when you’re attached to the **Mayor**.

---

## 1) Mental model: beads are work items; convoys are bundles of beads

- **Beads** are the core work items (issues) stored in the beads ledger. They are the atomic unit of work that gets assigned to agents (e.g., via `gt sling`).【F:README.md†L76-L81】
- **Convoys** are higher‑level tracking bundles that group multiple bead IDs into a single unit of progress reporting and coordination (think “epic” or multi‑issue batch).【F:README.md†L72-L75】

This means: **you create beads (issues), then create a convoy that “tracks” those beads, then assign beads to agents.**

---

## 2) Correct way to spawn work (convoys + beads)

### A. Create a convoy that tracks bead IDs
From the README example, the **canonical workflow** is:

```bash
gt convoy create "Feature X" gt-abc12 gt-def34 --notify --human
```

This creates a convoy titled *Feature X* and associates multiple bead IDs (`gt-abc12`, `gt-def34`) with it for tracking and coordination.【F:README.md†L155-L166】

### B. Sling beads to an agent (assign work)
Once you have beads (issue IDs), you assign a specific bead to an agent with:

```bash
gt sling gt-abc12 myproject
```

That dispatches the bead to a worker in the target rig (`myproject`).【F:README.md†L162-L168】

### C. Track progress with convoys
To monitor status and completion:

```bash
gt convoy list
```

This uses the convoy tracking structure to give you progress on the bundle of beads.【F:README.md†L165-L169】

---

## 3) Doing this while attached to the Mayor (recommended path)

When you’re **attached to the Mayor**, you generally:

1. **Tell the Mayor what you want built** (in natural language).
2. The Mayor **creates a convoy**, then **slings beads** to agents on your behalf.

This is the intended “Mayor workflow.” The README’s sequence diagram shows the Mayor creating a convoy and assigning beads to an agent, which then stores state in hooks and reports back.【F:README.md†L136-L151】

**In practice, while in the Mayor session**, you can still run the CLI commands directly:

```bash
gt convoy create "Auth System" gt-x7k2m gt-p9n4q --notify
gt sling gt-x7k2m myproject
gt convoy list
```

These are explicitly called out as the recommended workflow and commands to use within the Mayor session.【F:README.md†L174-L199】

---

## 4) Minimal / non‑tmux usage (for completeness)

If you are *not* using tmux, the same command sequence still applies:

```bash
gt convoy create "Fix bugs" gt-abc12
gt sling gt-abc12 myproject
gt convoy list
```

This “minimal mode” still uses convoys + beads as the correct work‑spawning workflow.【F:README.md†L201-L211】

---

## Summary (short)

**Correct workflow**:
1. Create bead IDs (issues) in beads.
2. Create a **convoy** that tracks those bead IDs.
3. **Sling** bead IDs to agents (by rig).
4. Track progress via `gt convoy list`.

**When attached to the Mayor**, you can either:
- **Tell the Mayor** what to do and let it orchestrate, *or*
- Run the same CLI commands yourself inside the Mayor session, which is the standard approach documented in the README.【F:README.md†L136-L199】

---

If you want, I can show how beads (issues) are created/updated at the API layer (e.g., via the `bd` CLI wrapper in `internal/beads`) or map these commands to the specific Go code paths in `cmd/gt`.
