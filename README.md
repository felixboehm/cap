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
| **cpulimit** | required — CPU % capping in `call` | `brew install cpulimit` |
| `renice`, `nettop`, `pmset`, `pgrep`, `pfctl`, `dseditgroup` | built in | — |

`cap` warns on every run if `cpulimit` is missing. The sync-client network block uses the built-in `pfctl` — no extra install.

## Usage

```
cap call          quiet things for a call: pause daemons, cpu-cap heavy
                  apps, block sync clients' network; never the live call
cap off           release everything (resume, unblock, drop caps)
cap status        show what is paused / capped / blocked
cap suggest [sec] list current CPU/RAM/net hogs; with [sec], loop every sec
cap add <pat>     remember a process/app (path substring) to cap from now on
cap list          show what call mode caps vs. protects
cap setup         one-time: create the throttle group + pf anchor (needs sudo)
```

### Running without a password

`call`/`off` need root (pausing daemons, the `pf` block, the group relaunch). To avoid `sudo` prompts:

1. Run the one-time setup (does the file-writing bits that must *not* be passwordless):
   ```sh
   sudo cap setup
   ```
2. It prints a `sudoers` drop-in — install it (validate with `visudo -c`). It NOPASSWDs only process/firewall commands (`pfctl`, `kill`, `renice`, `pkill`, `cpulimit`, `open`), **not** `tee`/file writes — so it can't be turned into a passwordless root shell.
3. Then just run `cap call` / `cap off` — no `sudo`, no password.

## How it works

- **Call detection** — `cap` reads `pmset -g assertions` for the `coreaudiod` input (mic) assertion. The owning process is the active call, and its entire `.app` bundle (audio, video, and screen-share renderers) is exempt — not just the mic-holding PID.
- **What `call` does** — three treatments:
  - **Daemons** (Spotlight, backups, iCloud, photo analysis): hard-paused with `SIGSTOP`, resumed with `SIGCONT` on `off`.
  - **Heavy background apps** (`Spotify`, `Docker`, …): `renice` + `cpulimit` to a CPU %.
  - **Sync clients** (`owncloud`, `OpenCloud`): outbound network **blocked** (see below). The call app itself is always exempt.

## Sync-client network block

macOS has no by-name per-process network filter for scripts, so `cap` uses the built-in `pf`, matched on a **group**:

1. `cap` creates a `capnet` group (one-time) and, in `call`, relaunches each running sync client under it via `sudo -g capnet open -a …` — **only if it's already running** (cap never starts apps).
2. A `pf` anchor (`block drop out group capnet`) is loaded; `cap off` flushes it.

Caveat: the relaunch is required because traffic is matched by group, and you can't tag an already-running process — so a sync client must be (re)started under the group for the block to apply.

## What gets capped vs. protected

`cap list` prints the current lists. Defaults:

- **Protected (never touched):** `kernel_task`, `launchd`, `WindowServer`, `coreaudiod`, `loginwindow`, the session itself, and the live call app's whole bundle.
- **Daemons (paused):** Spotlight (`mds_stores`, `mdworker`, `*spotlight*`), backups (`backupd`), iCloud (`cloudd`, `bird`), photo analysis.
- **Heavy apps (cpu-capped):** `Spotify`, `Docker`, `qemu`, `java`. Interactive tools you use during calls (Teams, Chrome, VS Code) are deliberately excluded.
- **Sync clients (network-blocked):** `owncloud`, `OpenCloud`.

## Configuration

Overrides live in `~/.cap/`:

- `~/.cap/config.sh` — shell file sourced at startup; override any tunable (`CPU_THRESHOLD`, `APP_LIMIT`, `RENICE_VAL`, `THROTTLE_GROUP`, `SYNC_APPS`, the `PROTECTED_*` / `CAP_*` lists, …).
- `~/.cap/extra.list` — extra path/name substrings to cap, one per line (managed via `cap add`).

## Limitations

- **`call`/`off` need root** (pausing daemons, the `pf` block, the group relaunch). Run `cap setup` once + install the printed sudoers drop-in to use them without a password.
- The sync-client block only applies to clients **(re)started under the `capnet` group** — `cap call` relaunches running ones automatically.
- `cap setup` appends a `cap` anchor to `/etc/pf.conf` and (on `call`) enables `pf`.
- An automatic "arm on call start / disarm on call end" daemon is not yet built; modes are run manually.
