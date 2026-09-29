# xKiller Clan Server Tool — Roadmap

> **Last updated:** 2026-09-29  
> **Current version:** v1 (VB.NET) — v2 C# rewrite in progress  
> **Repository:** `xkillermaverick/xKiller-Clan-Server-Tool`  
> **Branch:** `master` — all commits go directly to master, no feature branches or PRs.

---

## Status Tags

| Tag | Meaning |
|---|---|
| `[SHIPPED v1]` | In the current VB.NET build |
| `[RETIRE]` | Remove in v2 — already approved (see notes) |
| `[REWRITE]` | Exists in v1 but must be rebuilt in C# for v2 |
| `[NEW]` | Does not exist in v1 — add in v2 |
| `[EVALUATE]` | Idea from archived Palworld manager or backlog — assess before committing |
| `[RESEARCH PENDING]` | Needs investigation before any design decision |
| `[DO NOT PROCEED]` | Blocked — requires owner decision or license review first |

---

## Project Overview

The xKiller Clan Server Tool is a **Windows-native game server management application** for the xKiller Clan. It runs in the system tray, monitors and controls game servers, integrates with Discord, and connects to `engine.xkillerclan.com` for authentication, updates, and achievement sync. All engine-dependent features must degrade gracefully offline — core server control works with no internet connection.

The **xKiller Palworld Server Manager** (archived) is a reference source only. Its code and branding cannot be reused without a license review. Do not redirect users to it or any replacement.

---

## Phase 0 — Pre-Rewrite Inventory (Start Here)

Complete this before writing any C# code. Every item in Phases 1–5 depends on the inventory being accurate.

### 0.1 V1 Feature Map `[REWRITE]`

For every screen, engine, background task, config value, and data file in v1:
- Name and source file
- What it does
- Disposition: **retain**, **rewrite**, **repair then rewrite**, or **retire** (retire needs owner approval)
- Which v2 engine owns it
- Acceptance test that confirms parity in v2

**Known v1 components:**

| Component | Source File | v2 Destination | Disposition |
|---|---|---|---|
| Windows Service start/stop/restart | `ServerToolEngine.vb` | `ServerEngine` | Rewrite |
| Process-based server control | `ServerToolEngine.vb` | `ServerEngine` | Rewrite |
| System tray + balloon tips | `frmServerToolMain.vb` | `NotificationEngine` | Rewrite (upgrade to Toast) |
| Registry startup control | `ServerToolEngine.vb` | `CoreEngine` | Rewrite |
| INI-based settings | `SettingEngine.vb` | `SettingsEngine` | Rewrite (replace with JSON) |
| Guest vs. account gating | `frmServerToolMain.vb` | `EngineConnector` + role system | Rewrite |
| Structured error logging (49 codes) | `ServerToolEngine.vb` | `DiagnosticsEngine` | Rewrite (carry all 49 codes forward) |
| Anti-spam backend polling | `ServerToolEngine.vb` | `EngineConnector` | Rewrite |
| Discord.Net stub | `DiscordEngine.vb` | `DiscordEngine` | Rewrite (implement fully) |
| Engine connectivity check | `ServerToolEngine.vb` | `EngineConnector` | Rewrite |
| TSDNS logic (20-domain loop) | `ServerToolEngine.vb` | — | **RETIRE** |
| `tsdns_settings.ini` format | `SettingEngine.vb` | — | **RETIRE** |
| TSDNS username/password auth | `ServerToolEngine.vb` | — | **RETIRE** |

### 0.2 Palworld Manager Evaluation `[EVALUATE]`

Review the archived manager's features. For each:
- **Adopt:** bring the concept into the Clan Server Tool with a named v2 destination
- **Adapt:** describe the modification and its v2 destination
- **Reject:** document why it doesn't fit

Known ideas from the archived manager:
- Automated server restarts with in-game countdown messages
- Crash monitoring and automatic recovery
- Email notifications on server events
- Player detection
- Backup management

**License check required before reusing any code or assets. Do not use xKiller clan branding on anything derived from the archived project until the license is confirmed to permit it.** `[DO NOT PROCEED without license review]`

---

## Phase 1 — Core Engine Rewrite (C#)

### 1.1 Engine Architecture `[REWRITE]`

| Engine | Responsibility |
|---|---|
| `CoreEngine` | App lifecycle, startup, mode detection, settings load, update check |
| `ServerEngine` | Multi-server process and service control, crash detection, auto-restart, configurable retry |
| `DiscordEngine` | Integrated Discord bot + webhook notifications |
| `NotificationEngine` | Windows 11 Toast Notifications (images, buttons, progress bars) |
| `AchievementEngine` | Achievement tracking, unlock logic, sync to backend |
| `DiagnosticsEngine` | Structured logging with severity levels; all 49 v1 error codes carried forward |
| `SettingsEngine` | JSON server profiles, settings editor UI, INI migration |
| `EngineConnector` | xKiller Engine backend communication; graceful offline degradation |

Each engine is independently testable. A failing optional engine must not crash or block `ServerEngine` or `CoreEngine`.

### 1.2 Feature Toggles and Memory `[NEW]`
- Every optional feature has an explicit on/off setting
- Turning a feature off stops its background polling, scheduled work, and associated processes — not just its UI
- Measure actual idle and active memory before and after toggling each optional feature; record the results in the settings documentation
- Settings survive restart

### 1.3 Settings — JSON Profiles `[REWRITE]`
- Replace INI files with JSON server profiles
- Each server profile contains: name, type (process or service), executable or service name, autoRestart, maxRestartAttempts, restartDelaySeconds
- Migration converts existing INI settings without data loss
- Rollback available if migration fails

### 1.4 Role System `[REWRITE]`

| Role | Permissions |
|---|---|
| Owner | All features, all servers, all settings |
| Admin | Start/stop/restart all servers, view diagnostics |
| Moderator | View server status, trigger broadcasts |
| Guest | View server status only |

Roles sync with Discord roles via `EngineConnector`. Role sync failure must not prevent basic server control.

### 1.5 Auto-Updater `[REWRITE]`
- Check for updates via `engine.xkillerclan.com`
- Replace only changed files (diff-based), not a full reinstall
- User-initiated manual check in addition to automatic detection
- User settings kept outside core files so updates don't overwrite configuration

### 1.6 Windows 11 Design `[NEW]`
- Mica / Acrylic material backgrounds
- Rounded corners and Fluent Design icons
- Light and dark mode synced to OS theme
- Snap layout awareness

---

## Phase 2 — Discord Integration

### 2.1 Integrated Discord Bot `[NEW]`

Bot lives inside the tool — no separate process:

- `/server status` — live status of all servers
- `/server start [name]` — Admin+
- `/server stop [name]` — Admin+
- `/server restart [name]` — Admin+
- `/server logs [name]` — last 20 lines of server log, Admin+
- `/players list` — who is online across all servers
- `/achievements` — clan achievement progress

Automated channel notifications:
- Server online / offline
- Crash detected + auto-restart triggered
- Player milestones
- Achievement unlocks
- Update available

Discord role sync maps Discord roles to the internal role system.

### 2.2 Integration Safety `[NEW]`
- Discord integration has its own off switch
- Bot token and permissions in settings — never hardcoded
- Bot failure must not crash or block server control
- Required Discord permissions documented in README

---

## Phase 3 — Achievements

### 3.1 Achievement System `[NEW]`

| Achievement | Trigger |
|---|---|
| Iron Admin | 30 consecutive days of server uptime |
| First Responder | Restarted a crashed server within 60 seconds |
| Ghost Protocol | Server ran 7 days with zero crashes |
| Overclock | Hit max player capacity 10 times |
| Always On | 365 cumulative days of uptime across all servers |
| Night Shift | Server running midnight–6am continuously for 30 days |

Achievements sync to `engine.xkillerclan.com`. If the engine is unreachable, achievements accrue locally and sync when connection is restored.

---

## Phase 4 — Web Dashboard

### 4.1 Dashboard `[NEW]`
- Public-facing server status page
- Admin panel: start/stop/restart from browser (requires engine auth)
- Achievement showcase
- Player activity log
- Stack: C# ASP.NET Core API + lightweight HTML/JS frontend

---

## Phase 5 — Palworld-Specific Integration

### 5.1 Palworld Server Support `[EVALUATE]`

After Phase 0.2 evaluation, implement approved features. For each:
- Confirm it fits the multi-server engine model
- Name the `ServerEngine` sub-component that owns it
- Confirm license before reusing any archived code

Candidates from the archived manager (pending evaluation):
- Automated restarts with in-game countdown messages
- Crash monitoring and auto-recovery
- Email notifications
- Player detection
- Backup management

---

## Handoff Rules

- **Start with Phase 0** — do not write new C# until the inventory is done
- Every item needs: v1 source file(s), v2 destination engine, data migration plan, rollback plan, acceptance test
- Optional features: measure resource use before claiming any memory improvement
- Items marked `[EVALUATE]` or `[RESEARCH PENDING]` must not be implemented until evaluation or research is complete and documented
- Do not remove any v1 behavior without owner approval and a documented migration path

---

## Open Backlog

Ideas captured — assign to a phase before implementing:

- Notifications across all surfaces: tray, Discord, email, web
- In-game broadcast messages on scheduled restarts (games supporting RCON or REST API)
- RCON / REST API support per game type
- Scheduled maintenance windows with advance player warnings
- Player count history and graphs
- Mod update detection and notification
- Community feature evaluation: what other open-source server managers do well

---

*This roadmap is the handoff document for the next development session. Complete Phase 0 before touching Phase 1.*
