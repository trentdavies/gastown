# Tmux Integration

This document describes how Gas Town uses tmux as the process container for Claude Code agents.

## Overview

**Tmux is the process container; gt is the orchestrator.**

- **Tmux sessions** are where Claude Code agents run (one session per agent role)
- **gt CLI** manages session lifecycle: creates, monitors, sends messages, and cleans up sessions
- Sessions persist across network disconnects
- Each agent runs in isolation with its own environment

## Interaction Flow

```
gt CLI
    │
    ├──► builds startup command (with env vars)
    │
    ├──► creates tmux session
    │        │
    │        ▼
    │    Claude Code runs inside session
    │        │
    │        ├──► reads from stdin (tmux pane)
    │        ├──► writes to stdout (tmux pane)
    │        └──► runs gt commands internally
    │
    ├──► sends nudge messages via send-keys
    │
    └──► monitors session health via pane state
```

## Session Naming Convention

Gas Town uses a consistent naming scheme for all tmux sessions:

| Pattern | Example | Role |
|---------|---------|------|
| `hq-<role>` | `hq-mayor`, `hq-deacon` | Town-level agents |
| `gt-<rig>-<role>` | `gt-gastown-witness` | Rig-level agents |
| `gt-<rig>-<worker>` | `gt-gastown-polecat-01` | Workers |
| `gt-<rig>-crew-<name>` | `gt-gastown-crew-joe` | Crew members |

Defined in `internal/session/names.go`:

```go
const (
    HQPrefix = "hq-"  // Town-level agents
    Prefix   = "gt-"  // Rig-level agents
)
```

## Session Creation

### Preferred Method: NewSessionWithCommand

Creates a session that immediately runs a command, avoiding race conditions:

```go
// File: internal/tmux/tmux.go
func (t *Tmux) NewSessionWithCommand(name, workDir, command string) error {
    args := []string{
        "new-session",
        "-d",           // Detached
        "-s", name,     // Session name
        "-c", workDir,  // Working directory
        command,        // Command to run
    }
    _, err := t.run(args...)
    return err
}
```

**Why not NewSession + SendKeys?**

Using `NewSession` followed by `SendKeys` creates a race condition where the shell might not be ready to receive input. `NewSessionWithCommand` atomically creates the session with the command.

### Command Building

All agent startup commands are assembled via `internal/config/loader.go`:

```go
func BuildStartupCommand(envVars map[string]string, rigPath, prompt string) string {
    // 1. Build environment exports
    // 2. Append agent command (claude --dangerously-skip-permissions)
    // 3. Optionally append initial prompt

    // Result: "export GT_ROLE=... && claude --args 'prompt'"
}
```

### Environment Variables

Each session receives role-specific environment variables:

| Variable | Purpose | Example |
|----------|---------|---------|
| `GT_ROLE` | Agent role type | `polecat`, `witness`, `crew` |
| `GT_RIG` | Rig name | `gastown` |
| `GT_ROOT` | Town root | `/home/user/gt` |
| `BD_ACTOR` | Git attribution | `gastown/polecats/toast` |
| `GIT_AUTHOR_NAME` | Commit author | `gastown/polecats/toast` |

## The Nudge Pattern

The **nudge** is the canonical reliable delivery method for sending messages to Claude sessions.

### Why Nudge Exists

Raw `tmux send-keys` has issues:
- Claude's input handling differs from standard shells
- Long pastes can be truncated or garbled
- Mode state (vim INSERT mode) can intercept input
- Enter key timing can cause partial execution

### Implementation

```go
// File: internal/tmux/tmux.go
func (t *Tmux) NudgeSession(session, message string) error {
    // 1. Serialize nudges to same session (prevent garbled input)
    lock := getSessionNudgeLock(session)
    lock.Lock()
    defer lock.Unlock()

    // 2. Send text in literal mode (-l flag) for safety
    if _, err := t.run("send-keys", "-t", session, "-l", message); err != nil {
        return err
    }

    // 3. Wait 500ms for paste to complete (tested, required)
    time.Sleep(500 * time.Millisecond)

    // 4. Send Escape to exit vim INSERT mode if active
    t.run("send-keys", "-t", session, "Escape")

    // 5. Send Enter with retry (up to 3 attempts)
    for attempt := 0; attempt < 3; attempt++ {
        if _, err := t.run("send-keys", "-t", session, "Enter"); err == nil {
            return nil
        }
        time.Sleep(100 * time.Millisecond)
    }
    return fmt.Errorf("failed to send Enter after 3 attempts")
}
```

### Key Design Decisions

1. **Literal mode (`-l`)**: Prevents tmux from interpreting special characters
2. **500ms debounce**: Required for reliable paste completion
3. **Escape before Enter**: Handles vim mode edge cases
4. **Session mutex**: Serializes concurrent nudges to prevent interleaving
5. **Enter retry**: Handles transient tmux failures

### Usage

```bash
# From CLI
gt nudge <session> "Your message here"

# From Go code
t.NudgeSession("gt-gastown-polecat-01", "Please check your hook")
```

## Session Startup Flow

When an agent session starts, it receives a **beacon message** for discoverability:

```
[GAS TOWN] recipient <- sender • timestamp • topic[:mol-id]
```

Example:
```
[GAS TOWN] gastown/polecats/toast <- witness • 2026-01-18T15:42 • spawn:gt-123
```

This beacon appears in Claude's `/resume` picker, making sessions searchable.

### Full Startup Sequence

```go
// File: internal/session/startup.go
func StartSession(sessionID, workDir, startupCmd string) error {
    // 1. Create session with command
    t.NewSessionWithCommand(sessionID, workDir, startupCmd)

    // 2. Set environment variables
    t.SetEnvironment(sessionID, "GT_ROLE", role)
    t.SetEnvironment(sessionID, "GT_RIG", rig)

    // 3. Apply theming and bindings
    t.ConfigureGasTownSession(sessionID, theme, rig, worker, role)

    // 4. Wait for Claude to start
    t.WaitForCommand(sessionID, []string{"bash", "zsh"}, timeout)

    // 5. Accept permissions warning if needed
    t.AcceptBypassPermissionsWarning(sessionID)

    // 6. Send beacon message
    t.NudgeSession(sessionID, beacon)

    // 7. Send propulsion nudge (GUPP)
    t.NudgeSession(sessionID, "Run `gt prime` to check status and begin work.")
}
```

## Pane Operations

### Reading Pane State

```go
// Get the command running in a pane
t.GetPaneCommand(session)  // Returns "claude", "node", "bash", etc.

// Get pane identifier
t.GetPaneID(session)       // Returns pane ID like "%0"

// Get working directory
t.GetPaneWorkDir(session)  // Returns "/home/user/gt/gastown"

// Get process ID
t.GetPanePID(session)      // Returns PID of foreground process
```

### Health Monitoring

```go
// Check if Claude is actually running (not a zombie shell)
func (t *Tmux) IsClaudeRunning(session string) bool {
    cmd, _ := t.GetPaneCommand(session)
    // Look for: "node", "claude", or version pattern
    return strings.Contains(cmd, "node") ||
           strings.Contains(cmd, "claude") ||
           claudeVersionPattern.MatchString(cmd)
}

// Wait for process to be ready
t.WaitForCommand(session, excludeCommands, timeout)
t.WaitForShellReady(session, timeout)
t.WaitForRuntimeReady(session, readyCheck, timeout)
```

### Crash Detection

```go
// Set up pane-died hook to detect crashes
func (t *Tmux) SetPaneDiedHook(session, agentID string) error {
    hookCmd := fmt.Sprintf("gt session crashed %s", agentID)
    return t.run("set-hook", "-t", session, "pane-died", hookCmd)
}
```

## Cycle Bindings

Agents can cycle through related sessions using keyboard shortcuts:

```go
// File: internal/tmux/tmux.go
func (t *Tmux) SetCycleBindings(session string) error {
    // C-b n → gt cycle next for GT sessions
    // C-b p → gt cycle prev for GT sessions
    // Falls back to default tmux behavior for non-GT sessions
}
```

This allows operators to quickly navigate between rig agents.

## Session Types Summary

| Session Type | Created By | Working Directory | Purpose |
|--------------|------------|-------------------|---------|
| `hq-mayor` | `gt start` | `~/gt/mayor/` | Town coordinator |
| `hq-deacon` | `gt start` | `~/gt/deacon/` | Daemon watchdog |
| `gt-<rig>-witness` | `gt rig boot` | `~/gt/<rig>/witness/` | Polecat monitor |
| `gt-<rig>-refinery` | `gt rig boot` | `~/gt/<rig>/refinery/rig/` | Merge processor |
| `gt-<rig>-<polecat>` | Witness | `~/gt/<rig>/polecats/<name>/rig/` | Task worker |
| `gt-<rig>-crew-<name>` | `gt crew start` | `~/gt/<rig>/crew/<name>/rig/` | Human worker |

## Key Files

| File | Purpose |
|------|---------|
| `internal/tmux/tmux.go` | Core tmux wrapper (70+ functions) |
| `internal/session/startup.go` | Beacon/nudge message formatting |
| `internal/session/names.go` | Session naming conventions |
| `internal/config/loader.go` | Command building and agent resolution |

## Best Practices

### Do

- Use `gt nudge` for all message delivery to Claude sessions
- Use `NewSessionWithCommand` instead of `NewSession + SendKeys`
- Check `IsClaudeRunning()` before sending nudges
- Set pane-died hooks for crash detection

### Don't

- Use raw `tmux send-keys` for Claude sessions
- Assume a session is ready immediately after creation
- Send multiple nudges without serialization
- Rely on pane content parsing for state detection (ZFC principle)

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Nudge not received | Session not ready | Wait for `WaitForCommand` |
| Garbled input | Concurrent nudges | Use `NudgeSession` (has mutex) |
| Session zombie | Claude crashed, shell remains | Check `IsClaudeRunning()` |
| Wrong environment | Session created manually | Use `gt` commands, not raw tmux |
