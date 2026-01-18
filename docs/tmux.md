# Tmux usage in Gas Town

Gas Town relies on tmux as the default runtime host for agent sessions. It uses tmux to
create, monitor, and control long-running agents (Mayor, Deacon, Witness, Refinery,
Polecats, and Crew) while keeping persistent terminal sessions for inspection and
interaction. Tmux is optional (there is a degraded no-tmux mode), but it is the
preferred path for observability and session management.

## Why tmux is central

Gas Town treats tmux as the **observable source of truth** for whether agents are
running. Agent liveness and session presence are derived by querying tmux sessions
rather than trusting stale state in beads. This is why most lifecycle actions
(start, stop, revive, health check) go through tmux primitives.

## Session UI integrations (tmux-specific UX)

Gas Town includes several tmux-integrated UI elements that are configured on every
Gas Town tmux session:

### 1) Status bar theming and identity
Each session gets a deterministic theme (per rig) and a distinctive status bar format
for role/rig context. Themes are defined as reusable palettes and applied via tmux
options, which makes Mayor/Deacon/rig sessions visually distinct.

- Theme palette + per-role themes: `Theme`, `DefaultPalette`, `MayorTheme`, `DeaconTheme`.【F:internal/tmux/theme.go†L9-L63】
- Themed status style/format applied by `ConfigureGasTownSession`.【F:internal/tmux/tmux.go†L1112-L1145】

### 2) Dynamic status-right (mail + status line)
Sessions use a dynamic status-right hook that runs `gt status-line` periodically to
show live status and mail indicators in the tmux bar.

- Dynamic status binding via `SetDynamicStatus` (status-right + refresh interval).【F:internal/tmux/tmux.go†L1089-L1110】

### 3) Mail click popup and notification banners
Mail is surfaced inside tmux using a click binding and optional notification banners.

- Click status-right to open a popup showing the first unread message (`gt mail peek`).【F:internal/tmux/tmux.go†L1147-L1160】
- Notification banners injected into the session for high-visibility notices.【F:internal/tmux/tmux.go†L792-L825】

### 4) Session navigation bindings (C-b n/p) and feed binding (C-b a)
Gas Town remaps tmux key bindings to work with its session topology.

- `C-b n/p` cycles within Gas Town sessions (Mayor ↔ Deacon, crew, polecats), while
  non-GT sessions keep default tmux navigation. This is set via `SetCycleBindings`.【F:internal/tmux/tmux.go†L1191-L1233】
- `C-b a` opens the activity feed (`gt feed --window`) in a dedicated tmux window.
  This is wired via `SetFeedBinding`.【F:internal/tmux/tmux.go†L1236-L1254】

## TUI elements integrated with tmux

Yes—Gas Town includes a TUI (Bubble Tea based) activity feed, and it is explicitly
integrated with tmux:

- The `gt feed` command launches an interactive TUI dashboard by default and documents
  the TUI layout (agent tree, convoy panel, event stream, keyboard navigation).【F:internal/cmd/feed.go†L43-L128】
- The same command supports `--window` which opens the feed in a dedicated tmux window
  inside the current tmux session, and tmux bindings are configured to jump to it
  with `C-b a` when in a Gas Town session.【F:internal/cmd/feed.go†L85-L128】【F:internal/cmd/feed.go†L196-L276】【F:internal/tmux/tmux.go†L1236-L1254】

This means the TUI is **tmux-aware** and can be treated as a persistent dashboard
window inside the session topology.

## Common usage patterns

### Start or attach to a crew session
Use `gt crew` to create/attach to a crew tmux session (default behavior) or run without
creating a tmux session via `--no-tmux`.

### Launch the activity feed in its own window
From a tmux-based session:

```bash
gt feed --window
```

Then jump back and forth with `C-b a` (feed) and `C-b n/p` (cycle). The `--window`
flag is tmux-gated (it will error outside tmux) and is intended for persistent
monitoring in the session’s window list.【F:internal/cmd/feed.go†L196-L276】【F:internal/tmux/tmux.go†L1236-L1254】

### Use mail notifications inside tmux
When mail is delivered, Gas Town can display a notification banner and a clickable
status bar entry for a mail popup.

- Notification banner injection via `SendNotificationBanner`.【F:internal/tmux/tmux.go†L792-L825】
- Status-right click popup via `SetMailClickBinding`.【F:internal/tmux/tmux.go†L1147-L1160】

## Summary

- Gas Town’s default operational mode is tmux-based for persistent agent sessions and
  liveness detection.
- The tmux UI is enhanced with **themes, status bar info, mail popups, and key bindings**.
- The **activity feed TUI is tmux-integrated**, can live in a dedicated window, and is
  exposed via tmux bindings for quick access.

