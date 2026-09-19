# Teams Multi-Instance

A [Windhawk](https://windhawk.net/) mod that lets you run multiple independent instances of the new Microsoft Teams desktop app on Windows — for example your own company tenant and a client tenant where you are a guest, side by side in two full Teams windows, without switching organisations.

Submitted to the official Windhawk mod repository: [ramensoftware/windhawk-mods#5562](https://github.com/ramensoftware/windhawk-mods/pull/5562).

## The problem

The new Teams (`ms-teams.exe`) allows only one instance per Windows session. Launching it again just activates the already-running window. Teams' built-in multi-account support gives you cross-tenant notifications, but the main window still shows one organisation at a time.

## How it works

Teams enforces single-instancing with two named kernel mutexes:

```
Teams-Tfw-instance
Teams-Tfw-server
```

The mod hooks `NtCreateMutant` / `NtOpenMutant` in `ntdll.dll` and appends the process ID to names matching a configurable pattern, so every launch believes it is the first one:

```
Teams-Tfw-instance -> Teams-Tfw-instance_18368
Teams-Tfw-server   -> Teams-Tfw-server_18368
```

Only mutexes are hooked — sections and events are deliberately left alone, since those fire on the DLL-loader path and hooking them destabilises the process.

Each `ms-teams.exe` process uses its own PID as the salt. Only the top-level Teams process creates or opens the `Teams-Tfw-*` objects; the child `ms-teams.exe` processes never touch them, so no shared salt is needed.

The salting decision is made once, when a Teams process starts (on the process's initial thread), and kept for the life of the process via an environment marker that also records the settings in effect at the time. This means:

- A Teams instance that was already running when the mod was enabled is left untouched.
- An instance started with the mod active keeps its salt and settings across mod updates and Windhawk restarts.
- Settings changes only apply to Teams processes started afterwards.

Teams registers its tray icon with a fixed GUID, and Windows allows only one icon per GUID, so a second instance would get no icon at all. The mod also hooks `Shell_NotifyIconW` in salted processes, drops the GUID so the icon is identified by its window instead, and appends the PID to the tooltip.

## Installation

### From the Windhawk mod repository

Once the mod is published, search for **Teams Multi-Instance** in Windhawk and click *Install*.

### Manual (local mod)

1. Install [Windhawk](https://windhawk.net/).
2. Open Windhawk → *Explore* → *Create a new mod*.
3. Replace the template with the contents of [`teams-multi-instance.wh.cpp`](teams-multi-instance.wh.cpp).
4. Press **Compile Mod** (Ctrl+B), then **Exit Editing Mode**.
5. Make sure the mod is enabled.

## Usage

1. Enable the mod.
2. Quit Teams completely (tray icon → Quit) and start it again, so the first instance is launched with the mod active.
3. Start Teams again. Every additional launch opens a new independent instance. Sign in to a different account or tenant in each one.

Each instance gets its own tray icon, with the process ID in the tooltip (e.g. `Microsoft Teams [18368]`). Additional icons usually land in the tray overflow (the `^` arrow next to the clock). Quit each instance from its own tray icon or from inside its window — closing a window only minimises it. If instances get stuck, `taskkill /f /im ms-teams.exe` ends all of them.

**Restart Teams after enabling, disabling, or changing the settings of the mod.**

## Settings

| Setting | Default | Description |
|---|---|---|
| Object name patterns | `Teams-Tfw-` | Comma-separated substrings. Named mutexes matching any of these get salted. Applies to Teams processes started after the change. |
| Separate tray icon per instance | on | Drop the fixed tray-icon GUID so every instance gets its own tray icon, and show the process ID in the tooltip. |
| Log only | off | Log every named mutex Teams creates or opens without modifying anything. For troubleshooting; restart Teams after changing. |

## Limitations

- **All instances share the same Teams profile** (`%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache`, including the WebView2 user-data folder). Two Teams processes writing it concurrently is exactly what the single-instance lock exists to prevent. This can corrupt the profile, which means signing in again and rebuilding the cache. Avoid signing in/out or changing settings in more than one instance, and use at your own risk.
- Notification clicks and `teams://` links are routed by Windows activation and may land in a different instance than the one you expect.
- Helpers that are not `ms-teams.exe` (the Teams Meeting Add-in in Outlook, the updater) are not hooked and cannot reach a salted instance's objects; meeting-join from Outlook may target the wrong instance or none.
- Not supported by Microsoft.

## Troubleshooting

Turn on **Log only**, then quit and restart Teams. Open Windhawk's log (or [DbgView](https://learn.microsoft.com/sysinternals/downloads/debugview)): the mod lists every named mutex the new Teams process creates or opens without changing any of them. The interesting objects are created during startup, so a running instance shows nothing useful. If a future Teams build renames the single-instance objects, add the new name to **Object name patterns**, turn Log only off, and restart Teams again.

A healthy run looks like this (one block per instance):

```
[Wh_ModInit]: Init
[InitSalt]: Salt: 18368
[NtCreateMutant_Hook]: [create-mutant] Teams-Tfw-instance -> Teams-Tfw-instance_18368
[NtCreateMutant_Hook]: [create-mutant] Teams-Tfw-server -> Teams-Tfw-server_18368
```

Child `ms-teams.exe` processes log `Init` and `Salt: <pid>` but no renames. After a mod update or Windhawk restart, running instances log `Mod reloaded in an already salted process; keeping salt <pid>`.

## Alternatives

If you would rather not hook anything, the officially supported route is to install `https://teams.microsoft.com` as a PWA in a separate browser profile per tenant (Edge: `⋯` → *Apps* → *Install this site as an app*). Each profile keeps its own sign-in and runs alongside the desktop client.

[TonCunha/multi-microsoft-teams](https://github.com/TonCunha/multi-microsoft-teams) takes a different approach: a launcher that manages separate profiles per instance.

## License

MIT
