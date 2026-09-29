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
| `[PLANNED]` | Approved idea — ready to implement when the phase arrives |
| `[RESEARCH PENDING]` | Needs investigation before any design decision |
| `[DO NOT PROCEED]` | Blocked — requires owner decision first |

---

## Project Overview

The xKiller Clan Server Tool is a **Windows-native game server management application** for the xKiller Clan. It runs in the system tray, monitors and controls game servers, integrates with Discord, and connects to `engine.xkillerclan.com` for authentication, updates, and achievement sync. All engine-dependent features must degrade gracefully offline — core server control works with no internet connection.

**Target distribution:** Microsoft Store (MSIX packaged) + direct sideload installer. The developer account is already registered.

**Windows compatibility target:**
- **Windows 11** — primary target; WinUI 3 / Windows App SDK, full Fluent Design
- **Windows 10** — fully supported; WinUI 3 runs on Win10 1809+
- **Windows 7** — best-effort via a legacy WinForms UI variant (see Phase 6)

The **xKiller Palworld Server Manager** is archived. Its ideas are being reimplemented from scratch in C# — no original code is being reused. Credits for inspiration are noted in the About page (see Phase 7).

---

## Phase 0 — Pre-Rewrite Inventory (Start Here)

Complete this before writing any C# code. Every item in Phases 1–7 depends on the inventory being accurate.

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

### 0.2 Palworld Manager — Ideas Adopted `[PLANNED]`

The archived xKiller Palworld Server Manager is the inspiration source for the following features. All are being **reimplemented from scratch in C#** — no original Python code is being reused. The MIT license of the original repo does not apply to a clean reimplementation of ideas. Credits appear on the About page.

| Feature | v2 Destination |
|---|---|
| Automated server restarts with in-game countdown messages | `ServerEngine` + RCON/REST per game type |
| Crash monitoring and automatic recovery | `ServerEngine` |
| Email notifications on server events | `NotificationEngine` |
| Discord webhook notifications on crash | `DiscordEngine` |
| Automated server updates | `ServerEngine` + `CoreEngine` |
| Automated backups | `ServerEngine` |
| Player detection | `ServerEngine` |

---

## Phase 1 — Core Engine Rewrite (C#)

### 1.1 Engine Architecture `[REWRITE]`

| Engine | Responsibility |
|---|---|
| `CoreEngine` | App lifecycle, startup, mode detection, settings load, update check |
| `ServerEngine` | Multi-server process and service control, crash detection, auto-restart, configurable retry |
| `DiscordEngine` | Integrated Discord bot + webhook notifications |
| `NotificationEngine` | Windows 11 Toast Notifications; email notifications; tray notifications |
| `AchievementEngine` | Achievement tracking, unlock logic, sync to backend |
| `DiagnosticsEngine` | Structured logging with severity levels; all 49 v1 error codes carried forward |
| `SettingsEngine` | JSON server profiles, settings editor UI, INI migration |
| `EngineConnector` | xKiller Engine backend communication; graceful offline degradation |

Each engine is independently testable. A failing optional engine must not crash or block `ServerEngine` or `CoreEngine`.

### 1.2 Feature Toggles and Memory `[NEW]`
- Every optional feature has an explicit on/off setting
- Turning a feature off stops its background polling, scheduled work, and associated processes — not just its UI
- Measure actual idle and active memory before and after toggling each optional feature; record results in the settings documentation
- Settings survive restart

### 1.3 Settings — JSON Profiles `[REWRITE]`
- Replace INI files with JSON server profiles
- Each profile: name, type (process or service), executable or service name, autoRestart, maxRestartAttempts, restartDelaySeconds
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
- User settings kept outside core files so updates never overwrite configuration

---

## Phase 2 — Packaging and Windows Compatibility

### 2.1 MSIX Packaging and Microsoft Store `[NEW]`

The app is distributed as an MSIX package — the required format for Microsoft Store submission. The developer account is already registered.

**Two distribution paths:**
- **Microsoft Store** — submitted for certification when the app is ready
- **Direct sideload** — MSIX installer hosted on xkillerclan.com for users who prefer not to use the Store

**Microsoft Store certification notes — known flag areas:**

| Capability | Why it's needed | How to declare it |
|---|---|---|
| Windows Service control | Start/stop/restart game server services | Declare `runFullTrust` capability in manifest; explain use case in certification notes |
| Process launch/terminate | For servers that run as executables, not services | Same — `runFullTrust` covers this |
| Registry write (startup) | Run on boot option | Declare restricted capability; document it is user-initiated and opt-in |
| Elevated privileges | Some server operations require admin rights | Request elevation only for specific actions — never run the whole app elevated; drop back to standard after the action completes |
| Network access | Engine connectivity, Discord bot, update check | Standard `internetClient` + `internetClientServer` capabilities |

**Key certification rule:** The app must request elevation only when needed for a specific action, not run as administrator by default. Design all admin-required operations as explicit, user-triggered actions that elevate, complete, and drop back.

### 2.2 Windows Version Compatibility `[NEW]`

| Windows Version | UI Layer | Status |
|---|---|---|
| Windows 11 | WinUI 3 / Windows App SDK — Mica, Fluent Design, Toast | Primary target |
| Windows 10 (1809+) | WinUI 3 — same codebase, Acrylic where Mica unavailable | Fully supported |
| Windows 7 | WinForms legacy UI variant | Best-effort (see Phase 6) |

**Runtime detection:** On startup, `CoreEngine` detects the Windows version and launches the appropriate UI variant. The user can also manually switch between the modern (WinUI 3) and legacy (WinForms) UI from settings.

### 2.3 Dual UI — Modern and Legacy `[NEW]`

Both UI variants connect to the same backend engines. Switching UI does not restart the engines or lose server state.

- **Modern UI (WinUI 3):** Full Fluent Design, Mica/Acrylic, Toast notifications, Snap layout awareness — Windows 10/11
- **Legacy UI (WinForms):** Functional, clean, no Fluent Design dependencies — Windows 7 compatible
- **Communication between variants:** Both variants read/write the same JSON config and engine state. If a user runs the legacy variant on one machine and the modern variant on another (e.g. managing the same server from two PCs), they stay in sync through the shared config and engine backend
- **Switch without restart:** The user can toggle between modern and legacy UI from within the app — engines keep running, no data lost

---

## Phase 3 — Discord Integration

### 3.1 Integrated Discord Bot `[NEW]`

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
- Scheduled restart warnings (with countdown)

Discord role sync maps Discord roles to the internal role system.

### 3.2 Integration Safety `[NEW]`
- Discord integration has its own off switch
- Bot token and permissions in settings — never hardcoded
- Bot failure must not crash or block server control
- Required Discord permissions documented in README

---

## Phase 4 — Achievements

### 4.1 Achievement System `[NEW]`

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

## Phase 5 — Web Dashboard

### 5.1 Dashboard `[NEW]`
- Public-facing server status page
- Admin panel: start/stop/restart from browser (requires engine auth)
- Achievement showcase
- Player activity log
- Stack: C# ASP.NET Core API + lightweight HTML/JS frontend

---

## Phase 6 — Windows 7 Legacy Support

### 6.1 WinForms Legacy UI `[NEW]`

Windows 7 cannot run WinUI 3 or the Windows App SDK. A WinForms-based UI variant provides best-effort support:

- All core server control features available (start/stop/restart, crash detection, auto-restart)
- System tray support — WinForms tray is natively compatible with Windows 7
- Balloon tip notifications (Toast not available on Win7)
- JSON config compatibility — same profiles as the modern variant
- Discord bot and engine connectivity work if .NET version supports it on Win7
- Features that are technically impossible on Win7 are clearly labeled as unavailable — no silent failures

**Known Win7 limitations to document:**
- WinUI 3 / Mica / Acrylic — not available
- Toast Notifications — not available (balloon tips used instead)
- Microsoft Store distribution — not applicable; sideload only
- Some .NET 8+ APIs may not be available — the legacy variant targets the highest .NET version compatible with Windows 7 (.NET Framework 4.8 or .NET 6 with compatibility shims)

### 6.2 Cross-Variant Communication `[NEW]`

The modern (WinUI 3) and legacy (WinForms) variants can run on different machines managing the same server infrastructure:

- Both variants read/write the same JSON config format
- Engine state is shared through `engine.xkillerclan.com` when online
- A user on Windows 7 (legacy UI) and a user on Windows 11 (modern UI) see the same server status and can both trigger actions — subject to their role permissions
- Offline: each variant operates independently; state reconciles when the engine connection is restored

---

## Phase 7 — About Page and Credits

### 7.1 About Page `[NEW]`

The About page documents the tool's history, version, and credits:

- App name, version, and build date
- Owner: xKillerMaverick / xKiller Clan
- Link to xkillerclan.com
- Credits section acknowledging inspirations:
  - **[Palworld-Dedicated-Server-Manager](https://github.com/Andrew1175/Palworld-Dedicated-Server-Manager) by Andrew1175** — automated restart, crash monitoring, Discord webhook, backup, and email notification concepts originated here
  - Any additional contributors or open-source projects that influenced the tool
- License information for the xKiller Clan Server Tool itself
- Link to the GitHub repository

**Note:** Credits are given voluntarily — the clean C# reimplementation of ideas carries no legal obligation to credit. It is the right thing to do.

---

## Handoff Rules

- **Start with Phase 0** — do not write new C# until the inventory is done
- Every item needs: v1 source file(s), v2 destination engine, data migration plan, rollback plan, acceptance test
- Optional features: measure resource use before claiming any memory improvement
- Elevation: request admin rights only for specific actions — never run the whole app elevated
- Items marked `[RESEARCH PENDING]` must not be implemented until research is complete and documented
- Do not remove any v1 behavior without owner approval and a documented migration path
- Legacy (WinForms) and modern (WinUI 3) variants must stay in sync on config format — never let them diverge

---

## Open Backlog

Ideas captured — assign to a phase before implementing:

- Notifications across all surfaces: tray, Discord, email, web
- In-game broadcast messages on scheduled restarts (RCON or REST API per game)
- RCON / REST API support per game type
- Scheduled maintenance windows with advance player warnings
- Player count history and graphs
- Mod update detection and notification
- Community feature evaluation: what other open-source server managers do well
- Evaluate additional game server types beyond Palworld

---

*This roadmap is the handoff document for the next development session. Complete Phase 0 before touching Phase 1.*
