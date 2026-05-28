# cap

A call-aware process throttler for macOS. When you're on a video call (or just want quiet), `cap` pauses or throttles the background processes that hog CPU — Spotlight indexing, backups, cloud sync, photo analysis — and restores them afterward. The live call app is detected and never touched.

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
| **cpulimit** | required — CPU % capping in `on`/`call` | `brew install cpulimit` |
| `renice`, `nettop`, `pmset`, `pgrep`, `pfctl`, `dnctl` | built in | — |

`cap` warns on every run if `cpulimit` is missing. Per-app network control (planned) uses the built-in `pfctl`/`dnctl` — no extra install.

## Usage

```
cap on            soft-cap background indexers/daemons (renice + cpulimit)
cap call          pause daemons + soft-cap heavy background apps; never the live call
cap off           release everything (resume, restore priority, drop caps)
cap status        show what is currently paused / capped / busy
cap suggest [sec] list current CPU/RAM/net hogs; with [sec], loop every sec
cap add <pat>     remember a process/app (path substring) to cap from now on
cap list          show what each mode caps vs. protects
```

Capping your own apps needs no privileges. Capping **system daemons** (`mds_stores`, `backupd`, …) needs root, so for the full effect:

```sh
sudo cap call     # entering a call
sudo cap off      # done
```

## How it works

- **Call detection** — `cap` reads `pmset -g assertions` for the `coreaudiod` input (mic) assertion. The owning process is the active call, and its entire `.app` bundle (audio, video, and screen-share renderers) is exempt — not just the mic-holding PID.
- **Throttling** — three escalating mechanisms:
  - `renice` — deprioritize (safe, reversible, never freezes).
  - `cpulimit` — cap to a CPU % (duty-cycled SIGSTOP/SIGCONT).
  - `SIGSTOP`/`SIGCONT` — hard pause/resume (used for background daemons in `call`).
- **`on` vs `call`** — `on` soft-caps the background daemons and leaves apps alone; `call` hard-pauses the daemons and soft-caps heavy background apps, while protecting the live call.

## What gets capped vs. protected

`cap list` prints the current lists. Defaults:

- **Protected (never touched, any mode):** `kernel_task`, `launchd`, `WindowServer`, `coreaudiod`, `loginwindow`, the session itself, and the live call app's whole bundle.
- **Daemons (paused in `call`, throttled in `on`):** Spotlight (`mds_stores`, `mdworker`, `*spotlight*`), backups (`backupd`), iCloud (`cloudd`, `bird`), photo analysis, and the `owncloud`/`OpenCloud` sync engines.
- **Background apps (soft-capped in `call`):** `Spotify`, `Docker`, `qemu`, `java`. Interactive tools you use during calls (Teams, Chrome, VS Code) are deliberately excluded.

## Configuration

Overrides live in `~/.cap/`:

- `~/.cap/config.sh` — shell file sourced at startup; override any tunable (`CPU_THRESHOLD`, `SOFT_LIMIT`, `APP_LIMIT`, `RENICE_VAL`, the `PROTECTED_*` / `CAP_*` lists, …).
- `~/.cap/extra.list` — extra path/name substrings to cap, one per line (managed via `cap add`).

## Limitations

- **Per-app network control is not wired up yet.** macOS has no by-name per-process network API for scripts; the workable route is built-in `pfctl`/`dnctl` (block + bandwidth shaping) matched on a throttle group, which requires launching the target app under that group (a one-time relaunch). This is planned, not yet implemented.
- Capping system daemons requires `sudo`.
- An automatic "arm on call start / disarm on call end" daemon is not yet built; modes are run manually.
