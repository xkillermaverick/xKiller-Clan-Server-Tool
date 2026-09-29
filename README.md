# xKiller Clan Server Tool

> **Status: Active Development — v2 Rewrite In Progress**
> The current codebase (v1) is the original VB.NET foundation. A full rewrite in C# is planned. See the roadmap below.

---

## What This Is

The xKiller Clan Server Tool is a **Windows-native game server management application** built for the xKiller Clan. It runs in the system tray, monitors and controls game servers, integrates with Discord, and connects to the xKiller Engine backend for clan-wide features like achievements, role sync, and notifications.

This is an **xKiller ecosystem product** — it is designed to work with `engine.xkillerclan.com` and the xKiller Discord server. It is not intended to be a generic server manager for the general public.

---

## v1 — What Exists Now (VB.NET)

The original tool was built in Visual Basic .NET and focused on Teamspeak DNS management. It established several core systems that are worth carrying forward:

- Windows Service start / stop / restart controller
- Process-based server control (for executables that don't run as services)
- System tray + balloon tip notification architecture
- Registry-based startup control (run on boot)
- INI-based settings persistence
- Account-based feature gating (Guest vs. logged-in users)
- Diagnostics and structured error logging (49 distinct error codes)
- Anti-spam protection for backend polling
- Discord.Net dependency already imported (stub, awaiting implementation)
- xKiller Engine connectivity check via `engine.xkillerclan.com`

### What Gets Removed in v2
- All TSDNS (Teamspeak DNS) logic — deprecated by Teamspeak years ago
- The 20-domain DNS update loop
- Hardcoded `tsdns_settings.ini` file format
- TSDNS username/password authentication

---

## v2 — Full Rewrite Plan (C#)

### Why C#
The entire .NET ecosystem — Windows apps, Discord bots, REST APIs, services — is maintained and documented primarily in C#. VB.NET and C# run on the same runtime, so all existing logic transfers. The rewrite also gives a clean slate to remove obsolete code and apply modern patterns.

### Architecture

The tool is built around a modular engine system. Each major concern is a separate engine:

| Engine | Responsibility |
|---|---|
| `CoreEngine` | App lifecycle, startup, settings, update checker |
| `ServerEngine` | Multi-server process and service control, crash detection, auto-restart |
| `DiscordEngine` | Integrated Discord bot + webhook notifications |
| `NotificationEngine` | Windows 11 Toast Notifications (replaces old balloon tips) |
| `AchievementEngine` | Achievement tracking, unlock logic, sync to backend |
| `DiagnosticsEngine` | Structured logging with severity levels |
| `SettingsEngine` | JSON-based configuration, server profiles |
| `EngineConnector` | xKiller Engine backend communication (auth, updates, sync) |

### Settings Format (v2)
INI files are replaced with JSON server profiles. Each game server gets its own profile:

```json
{
  "profiles": [
    {
      "name": "Palworld Main",
      "type": "process",
      "executable": "C:\\Servers\\Palworld\\PalServer.exe",
      "autoRestart": true,
      "maxRestartAttempts": 3,
      "restartDelaySeconds": 30
    },
    {
      "name": "Minecraft Survival",
      "type": "service",
      "serviceName": "MinecraftServer",
      "autoRestart": true,
      "maxRestartAttempts": 5,
      "restartDelaySeconds": 10
    }
  ]
}
```

### Role System (v2)
The old binary account/guest system expands into a full role hierarchy synced with Discord roles:

| Role | Permissions |
|---|---|
| Owner | All features, all servers, all settings |
| Admin | Start/stop/restart all servers, view diagnostics |
| Moderator | View server status, trigger broadcasts |
| Guest | View server status only |

---

## Features Roadmap

### Core (Phase 1)
- [ ] Multi-server profiles (process + service modes)
- [ ] Start / stop / restart with configurable retry logic
- [ ] Crash detection and auto-restart
- [ ] Windows 11 Toast Notifications (images, buttons, progress bars)
- [ ] System tray with per-server status indicators
- [ ] JSON configuration with profile editor UI
- [ ] Registry startup control
- [ ] Auto-updater via xKiller Engine
- [ ] Structured diagnostics log viewer (Info / Warning / Error / Critical)

### Discord Integration (Phase 2)
- [ ] Integrated Discord bot (no separate process — lives inside the tool)
- [ ] `/server status` — live status of all servers
- [ ] `/server start [name]` — start a server (Admin+)
- [ ] `/server stop [name]` — stop a server (Admin+)
- [ ] `/server restart [name]` — restart a server (Admin+)
- [ ] `/server logs [name]` — last 20 lines of server log (Admin+)
- [ ] `/players list` — who is online across all servers
- [ ] `/achievements` — clan achievement progress
- [ ] Automated channel notifications: server online/offline, crash + restart, player milestones, achievement unlocks, update available
- [ ] Discord role sync with internal role system

### Achievements (Phase 3)
Achievements are tracked per-clan and displayed in Discord and the web dashboard.

| Achievement | Trigger |
|---|---|
| Iron Admin | 30 consecutive days of server uptime |
| First Responder | Restarted a crashed server within 60 seconds |
| Ghost Protocol | Server ran 7 days with zero crashes |
| Overclock | Hit max player capacity 10 times |
| Always On | 365 cumulative days of uptime across all servers |
| Night Shift | Server running continuously midnight–6am for 30 days |

### Windows 11 Design Compliance (Phase 1+)
- Mica / Acrylic material backgrounds
- Rounded corners and Fluent Design icons
- Light and dark mode synced to OS theme
- Snap layout awareness

### Web Dashboard (Phase 4)
- Public-facing server status page
- Admin panel: start/stop/restart from browser
- Achievement showcase
- Player activity log
- Tech stack: C# ASP.NET Core API + lightweight HTML/JS frontend

---

## xKiller Engine Integration

The tool connects to `engine.xkillerclan.com` for:

- **Authentication** — issues tokens, validates Discord role claims
- **Update distribution** — hosts versioned releases for the auto-updater
- **Achievement sync** — stores and calculates achievement state clan-wide
- **Connectivity check** — validated on startup before enabling online features

All engine-dependent features degrade gracefully if the engine is unreachable. Core server control works fully offline.

---

## Technology Stack

| Component | Technology |
|---|---|
| Desktop Application | C# / WinForms or WPF (.NET 8+) |
| Discord Bot (integrated) | Discord.Net |
| Notifications | Microsoft.Toolkit.Uwp.Notifications |
| Configuration | System.Text.Json |
| Logging | Microsoft.Extensions.Logging |
| Backend Communication | HttpClient (async) |
| UI Theme | Windows App SDK / Fluent Design |
| Web Dashboard API | C# ASP.NET Core |

---

## Project History

- **v1 (VB.NET)** — Original tool focused on Teamspeak DNS management for the xKiller Clan server. Established the engine architecture, tray integration, service control, and diagnostics systems. Version 4.5.28.4.
- **v2 (C# — In Progress)** — Full rewrite. Removes obsolete TSDNS logic. Modernizes for Windows 11. Adds multi-server support, integrated Discord bot, achievements, and xKiller Engine sync.

---

## Contributing

This is an xKiller Clan internal project. Contributions from clan members are welcome. External PRs may be considered but are not the primary focus.

---

*Built by xKillerMaverick for the xKiller Clan.*
