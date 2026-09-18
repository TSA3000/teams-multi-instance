# Teams Multi-Instance

A [Windhawk](https://windhawk.net/) mod that lets you run multiple independent instances of the new Microsoft Teams desktop app on Windows — for example your own company tenant and a client tenant where you are a guest, side by side in two full Teams windows, without switching organisations.

## The problem

The new Teams (`ms-teams.exe`) allows only one instance per Windows session. Launching it again just activates the already-running window. Teams' built-in multi-account support gives you cross-tenant notifications, but the main window still shows one organisation at a time.

## How it works

Teams enforces single-instancing with two named kernel mutexes:

```
Teams-Tfw-instance
Teams-Tfw-server
```

The mod hooks `NtCreateMutant` / `NtOpenMutant` in `ntdll.dll` and appends a per-launch salt (the process ID) to those names, so every launch believes it is the first one:

```
Teams-Tfw-instance -> Teams-Tfw-instance_3544
Teams-Tfw-server   -> Teams-Tfw-server_3544
```

Child processes (WebView2 etc.) inherit the same salt through an environment variable, so objects shared inside one Teams process tree keep working. Only mutexes and semaphores are hooked — sections and events are deliberately left alone, since those fire on the DLL-loader path and hooking them destabilises the process.

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

Start Teams. Then start Teams again — a second, fully independent instance opens. Sign in to a different account or tenant in each one.

Each instance has its own tray icon; the second one is usually hidden in the tray overflow (the `^` arrow next to the clock). Quit each instance separately from its own tray icon or from inside its window. If instances get stuck, `taskkill /f /im ms-teams.exe` ends all of them.

## Settings

| Setting | Default | Description |
|---|---|---|
| Object name patterns | `Teams-Tfw-` | Comma-separated substrings. Named mutexes/semaphores matching any of these get salted. |
| Log only | off | Log every named mutex/semaphore Teams creates without modifying anything. For troubleshooting. |

## Limitations

- All instances share the same profile directory (`%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache`). Reading is fine, but avoid signing in/out or changing settings in one instance while another is running — treat one instance as "primary" for that.
- Notification clicks and `teams://` links are routed by Windows activation and may land in a different instance than the one you expect.
- Both tray icons look identical.
- Not supported by Microsoft. Use at your own risk.

## Troubleshooting

Turn on **Log only** and open Windhawk's log (or [DbgView](https://learn.microsoft.com/sysinternals/downloads/debugview)). The mod logs every named mutex/semaphore Teams creates without changing any of them. If a future Teams build renames the single-instance objects, add the new name to **Object name patterns**.

A healthy run looks like this (one block per instance):

```
[Wh_ModInit]: Init
[InitSalt]: Created salt: 3544
[NtCreateMutant_Hook]: [create-mutant] Teams-Tfw-instance -> Teams-Tfw-instance_3544
[NtCreateMutant_Hook]: [create-mutant] Teams-Tfw-server -> Teams-Tfw-server_3544
[Wh_ModInit]: Init
[InitSalt]: Inherited salt: 3544
```

## Alternatives

If you would rather not hook anything, the officially supported route is to install `https://teams.microsoft.com` as a PWA in a separate browser profile per tenant (Edge: `⋯` → *Apps* → *Install this site as an app*). Each profile keeps its own sign-in and runs alongside the desktop client.

## License

MIT
