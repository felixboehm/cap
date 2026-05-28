# cap

A call-aware process throttler for macOS. When you're on a video call (or just want quiet), `cap` pauses background CPU hogs (Spotlight, backups, photo analysis), cpu-caps heavy apps, and quits your cloud-sync clients — then restores everything afterward. The live call app is detected and never touched.

It was built to stop heavy background load from starving **WindowServer**, whose display watchdog will otherwise time out and restart the GUI (a black screen / bounce to the login window that looks like a crash) during calls.

## Why

macOS has **App Nap** (passive, only naps occluded GUI apps) and the GUI-only commercial **App Tamer**, but nothing keys off "I'm on a call." `cap` does: it detects an active call via the audio (mic) power assertion and throttles everything noisy except the call itself.

## Install

```sh
git clone https://github.com/felixboehm/cap.git
ln -s "$PWD/cap/cap" ~/.local/bin/cap     # ~/.local/bin must be on your PATH
```

## Dependencies

| tool | role | install |
|---|---|---|
| **cpulimit** | required — CPU % capping in `call` | `brew install cpulimit` |
| `renice`, `nettop`, `pmset`, `pgrep`, `osascript` | built in | — |

`cap` warns on every run if `cpulimit` is missing.

## Usage

```
cap call          quiet things for a call: pause daemons, cpu-cap heavy
                  apps, quit sync clients; never the live call
cap off           release everything (resume, relaunch sync clients, drop caps)
cap status        show what is paused / capped / quit
cap suggest [sec] list current CPU/RAM/net hogs; with [sec], loop every sec
cap add <pat>     remember a process/app (path substring) to cap from now on
cap list          show what call mode does vs. protects
cap setup         one-time: install a sudoers drop-in for password-free call/off
```

### Running without a password

Quitting your own sync clients needs no privileges, but pausing system daemons does. To run `call`/`off` without a `sudo` prompt:

```sh
sudo cap setup     # once
```

It installs `/etc/sudoers.d/cap` (validated with `visudo -c`) that NOPASSWDs **only** process-control commands for your user — `kill`, `renice`, `pkill`, `cpulimit`. No file-write commands (`tee` etc.), so it can't be turned into a passwordless root shell. After that, just run `cap call` / `cap off`.

## How it works

- **Call detection** — `cap` reads `pmset -g assertions` for the `coreaudiod` input (mic) assertion. The owning process is the active call, and its entire `.app` bundle (audio, video, and screen-share renderers) is exempt — not just the mic-holding PID.
- **What `call` does** — three treatments:
  - **Daemons** (Spotlight, backups, iCloud, photo analysis): hard-paused with `SIGSTOP`, resumed with `SIGCONT` on `off`.
  - **Heavy background apps** (`Spotify`, `Docker`, …): `renice` + `cpulimit` to a CPU %.
  - **Sync clients** (`owncloud`, `OpenCloud`): **quit** (the simplest, most reliable "offline"), and relaunched on `cap off`. The call app itself is always exempt.

## Sync clients

macOS has no reliable, scriptable way to block one app's network from a shell (the only per-app network API is a packaged firewall extension, none of which expose a CLI). So `cap` takes the pragmatic route: a quit sync client can't sync. On `cap call` it quits each *running* sync client (graceful AppleScript quit, then force) and records it; on `cap off` it relaunches the ones it quit. `cap` never starts an app you didn't have open.

## What gets capped vs. protected

`cap list` prints the current lists. Defaults:

- **Protected (never touched):** `kernel_task`, `launchd`, `WindowServer`, `coreaudiod`, `loginwindow`, the session itself, and the live call app's whole bundle.
- **Daemons (paused):** Spotlight (`mds_stores`, `mdworker`, `*spotlight*`), backups (`backupd`), iCloud (`cloudd`, `bird`), photo analysis.
- **Heavy apps (cpu-capped):** `Spotify`, `Docker`, `qemu`, `java`. Interactive tools you use during calls (Teams, Chrome, VS Code) are deliberately excluded.
- **Sync clients (quit & relaunched):** `owncloud`, `OpenCloud`.

## Configuration

Overrides live in `~/.cap/`:

- `~/.cap/config.sh` — shell file sourced at startup; override any tunable (`CPU_THRESHOLD`, `APP_LIMIT`, `RENICE_VAL`, `SYNC_APPS`, the `PROTECTED_*` / `CAP_*` lists, …).
- `~/.cap/extra.list` — extra path/name substrings to cap, one per line (managed via `cap add`).

## Limitations

- **Pausing system daemons needs root.** Run `cap setup` once for password-free `call`/`off`; quitting/relaunching sync clients needs no privileges.
- Sync clients are **quit**, not throttled — a closed app is fully offline but its UI is gone until `cap off`.
- An automatic "arm on call start / disarm on call end" daemon is not yet built; modes are run manually.
