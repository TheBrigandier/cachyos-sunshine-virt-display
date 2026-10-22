# Sunshine + KDE Wayland virtual display (CachyOS)

Stream the correct resolution, aspect ratio, and frame rate to any Moonlight
client — no dummy HDMI plug required.

Normally, streaming from a headless or docked machine means either plugging
in a dummy HDMI adapter (a fixed resolution that likely won't match your
client) or fighting with software-emulated displays. This project instead
creates a `krfb-virtualmonitor` display sized and clocked exactly to match
whatever client connects, and has Sunshine capture that virtual display via
its `kwin` capture mode — so the stream always matches the client's native
resolution, aspect ratio, and frame rate.

## Contents

- [Requirements](#requirements)
- [Sunshine config file locations](#sunshine-config-file-locations)
- [Setup](#setup)
- [How it works](#how-it-works)
- [Known issues / FAQ](#known-issues--faq)
- [Debugging a failed start](#debugging-a-failed-start)

## Requirements

- **KDE Plasma on Wayland.** This project relies on two KDE-specific pieces
  — KWin's screen-capture portal (used by Sunshine's `kwin` capture mode)
  and `kscreen-doctor` (used to read and restore your monitor layout) — so
  it won't work on X11 or on non-KDE desktops.
- **Sunshine** (latest stable). Provides the streaming server and the
  Application/prep-command hooks these scripts plug into.
- **`krfb`, `kscreen`, and `jq`:**
  ```
  sudo pacman -S krfb kscreen jq
  ```
  - `krfb` provides `krfb-virtualmonitor`, the binary that actually
    creates the virtual display.
  - `kscreen` provides `kscreen-doctor`, used to enable/disable monitors
    and to read/restore their layout.
  - `jq` parses `kscreen-doctor --json` output when snapshotting and
    restoring your monitor layout.

## Sunshine config file locations

These are the two Sunshine config files this project reads or writes:

| File | Purpose |
| --- | --- |
| `~/.config/sunshine/sunshine.conf` | Main config. `output_name` is what these scripts rewrite on start/stop. |
| `~/.config/sunshine/apps.json` | Application list. Editing this directly is faster than the web UI when adding several similar entries — e.g. one per-client app tile with a different scale argument (see [Setup step 4](#setup)). Its top-level `"env"` key is global to all apps; there's no per-app equivalent. |

## Setup

A quick primer if you haven't used Sunshine's "Applications" feature before:
each Application is a tile that shows up in Moonlight, and it can define up
to three commands — a **prep command** that runs *before* the stream
starts, an **undo command** that runs *after* the stream ends, and the
**command** that's actually launched (a game, Steam, a desktop session,
etc.). This project hooks into the prep/undo commands to create and tear
down the virtual display; you still choose whatever `Command` you want.

1. **Set Sunshine's capture method to `kwin`.** By default Sunshine
   auto-detects how to grab your screen, and on KDE Wayland that
   auto-detected method won't capture the virtual display these scripts
   create — you have to force it to use KDE's own `kwin` capture backend
   instead.

   - Open Sunshine's web UI in a browser: `https://your-ip:47990` (replace
     `your-ip` with the IP or hostname of the machine running Sunshine; if
     you're on the same machine, `https://localhost:47990` works too).
   - Log in if prompted, then go to the **Configuration** tab.
   - Click into the **Advanced** sub-tab.
   - Find **Force Capture Method** and set it to **`kwin`**.
   - Scroll down and click **Save**, then restart Sunshine so the setting
     takes effect (restart the systemd service if you've installed it —
     see step 5 — otherwise just relaunch the `sunshine` process).

2. **Install the scripts to `~/.local/bin`.** That directory is
   user-owned (no `sudo` needed) and already on `PATH` on most desktop
   Linux setups, including CachyOS, so once copied the scripts can be
   referenced directly by path from Sunshine's config, as shown in step 3.

   ```
   cp scripts/sunshine-start-vmon.sh scripts/sunshine-stop-vmon.sh ~/.local/bin/
   chmod +x ~/.local/bin/sunshine-start-vmon.sh ~/.local/bin/sunshine-stop-vmon.sh
   ```

   The `chmod +x` is required — without the executable bit set, Linux will
   refuse to run the scripts and Sunshine's prep/undo commands will fail.

3. **Create an Application for Sunshine to launch:**

   - In the web UI, go to the **Applications** tab and add a new app.
   - Fill in these fields:

     | Field | Value |
     | --- | --- |
     | Prep command (Do) | `$HOME/.local/bin/sunshine-start-vmon.sh` |
     | Undo command | `$HOME/.local/bin/sunshine-stop-vmon.sh` |
     | Command | whatever you want to launch, e.g. `setsid steam steam://open/bigpicture` — or leave it blank to just land in a normal desktop session on the virtual display |

   - Save the app.

   > Sunshine's UI field doesn't expand `~` or environment variables other
   > than the literal string `$HOME`, so use `$HOME` exactly as shown
   > above. If you need the literal path for some other reason, run
   > `echo $HOME` in a terminal.

4. **(Optional) Per-client DPI scaling.** Skip this unless you stream to
   multiple devices with different DPI needs (e.g. a laptop and a tablet)
   and want each to get a different UI scale.

   - Create one Application per device, following step 3, but append a
     scale factor as an argument to the prep command:

     | Field | Value |
     | --- | --- |
     | Prep command (Do) | `$HOME/.local/bin/sunshine-start-vmon.sh 1.5` |
     | Undo command | `$HOME/.local/bin/sunshine-stop-vmon.sh` (same as before — no argument needed) |

   - Name each tile after the device it's for (`Desktop (MacBook)`,
     `Desktop (iPad)`, `Desktop (TV)`, ...) and set the scale factor you
     want for that device.
   - In Moonlight, pick the matching tile per client — it remembers your
     last choice per host. Omit the argument entirely to use scale `1`.

   **Why per-device tiles instead of a per-device setting?** Sunshine's
   prep-command scripts only receive the resolution the client requested,
   not which client connected, so there's no built-in way to detect the
   device automatically. `apps.json`'s top-level `"env"` key could in
   theory pass a variable through, but it's global to every Application (a
   sibling of `"apps"` in the file, not a per-app override), so it can't
   vary by device either. Passing the scale as a CLI argument on a
   per-device Application tile is the workaround — no JSON editing
   required, since the Prep command field is exposed normally in the web
   UI.

5. **(Recommended) Install the systemd user service.** CachyOS's `sunshine`
   pacman package ships no systemd unit at all. Installing one means the
   stop script still runs (via `ExecStopPost`) to restore your monitors even
   if Sunshine crashes instead of exiting cleanly.

   ```
   mkdir -p ~/.config/systemd/user
   cp systemd/sunshine.service ~/.config/systemd/user/
   systemctl --user daemon-reload
   systemctl --user enable --now sunshine
   ```

   The unit file goes in `~/.config/systemd/user/`, systemd's per-user unit
   directory — matching the `cp` destination above. It uses systemd's `%h`
   specifier for paths, so it works as-is for any user — no editing needed.

6. **(Optional) Autologin with immediate lock.** Since `sunshine.service`
   only starts once a graphical session exists ([step 5](#setup)), a fresh
   boot needs someone to physically log in before Sunshine comes up —
   defeating the point of a headless/remote setup. Autologin fixes that, but
   an autologin session normally comes up fully unlocked on the physical
   console; the second piece below locks it immediately so that isn't a
   problem.

   - **Enable autologin** for your user through your login manager's own
     setting — e.g. CachyOS's Settings app → Login Screen, or your login
     manager's autologin option directly (SDDM's `[Autologin]` section in
     `/etc/sddm.conf.d/`, or Plasma's own login manager settings on distros
     that use it instead of SDDM). This part is distro/login-manager
     specific, so it's not scripted here — use whatever your system
     provides.
   - **Install the lock-on-start unit** so the session locks the instant the
     graphical session comes up:
     ```
     cp systemd/lock-on-start.service ~/.config/systemd/user/
     systemctl --user daemon-reload
     systemctl --user enable lock-on-start.service
     ```
     Like `sunshine.service`, this is tied to `graphical-session.target`
     (via `PartOf`/`WantedBy`), so it fires on every graphical login —
     autologin at boot or a normal manual login alike. Rather than relying
     on unit ordering, it waits (event-driven, via `gdbus wait`) for KWin to
     publish the screen locker's `org.freedesktop.ScreenSaver` D-Bus name,
     then calls `Lock()` on it directly. That is the earliest moment a lock
     can take effect, and it's well before plasmashell draws the desktop.
     Two timing traps this avoids:
     - Ordering after `graphical-session.target` is too slow: on one test
       system, `plasma-powerdevil.service` (pulled in by that target) took
       7+ seconds to start, leaving the desktop visible and unlocked.
     - Ordering after `plasma-kwin_wayland.service`/`plasma-ksmserver.service`
       alone is too early on Plasma 6.8+. KWin's unit counts as started
       before KWin itself has initialized, so a `loginctl lock-session` sent
       at that point is silently dropped and the session stays unlocked.
       Plasma 6.7 happened to hide this because ksmserver waited on kcminit.

     Confirmed on Plasma 6.7.2/CachyOS: the session
     goes straight to the lock screen on boot with no visible unlocked
     desktop, and Sunshine's virtual-monitor capture keeps streaming
     normally while locked — connecting via Moonlight shows the real lock
     screen (and the desktop after unlocking), not a frozen frame.

## How it works

1. **Moonlight connects.** Sunshine determines the resolution and frame
   rate the client requested, and exposes them as the environment
   variables `SUNSHINE_CLIENT_WIDTH`, `SUNSHINE_CLIENT_HEIGHT`, and
   `SUNSHINE_CLIENT_FPS` before running the Prep command.
2. **The start script creates the virtual display.** It reads those
   variables and tells `krfb-virtualmonitor` to create a new display at
   that exact resolution — this is what lets the stream match the client
   instead of a fixed dummy-plug resolution. It also registers a custom
   `kscreen-doctor` mode at the client's requested frame rate and switches
   the virtual display to it, so the stream isn't locked to whatever
   default refresh rate `krfb-virtualmonitor` would otherwise pick.
3. **Physical monitors are disabled and the virtual display goes
   primary,** so KWin treats it as the main screen and your desktop
   renders onto it at the client's resolution and frame rate.
4. **Sunshine captures and streams it.** Because Sunshine is set to the
   `kwin` capture method ([Setup step 1](#setup)), it captures the virtual
   display specifically, at the exact resolution created above — no
   scaling or letterboxing.
5. **On disconnect, the stop script cleans up:** it kills the virtual
   display and restores your physical monitors to their pre-stream layout.

## Known issues / FAQ

- **Sunshine crashes mid-session without the systemd service installed**
  - What happens: physical monitors stay disabled until you manually run
    the stop script.
  - Why: without the systemd unit's `ExecStopPost` hook
    ([Setup step 5](#setup)), nothing runs the stop script if Sunshine
    itself dies unexpectedly.
  - Fix: run `~/.local/bin/sunshine-stop-vmon.sh` by hand to restore your
    monitors, and consider installing the systemd service to prevent this
    going forward.

- **Why does the log show a random password being generated?**
  - Context: `krfb-virtualmonitor` requires a VNC password on its command
    line, so the start script generates a fresh random one on every
    invocation (`start.log` will show it, and it's visible to local users
    via `ps` for the lifetime of the process — inherent to how
    `krfb-virtualmonitor` takes it as a CLI arg).
  - Why it's fine here: Sunshine captures the virtual display through
    KWin's `kwin` capture mode, not by connecting to `krfb-virtualmonitor`'s
    VNC server, so nothing actually authenticates with this password in
    normal use. It exists only because the binary requires one to launch.

- **Field names may differ across Plasma/kscreen versions**
  - Context: the snapshot/restore logic's assumed `kscreen-doctor --json`
    field names (`name`, `enabled`, `priority`, `pos.x`/`pos.y`, `scale`,
    `rotation`, `currentModeId`,
    `modes[].id`/`.size.width`/`.size.height`/`.refreshRate`) have only
    been verified against a live single-monitor CachyOS system.
  - If you're on a different version and restore silently skips a field:
    run `kscreen-doctor --json | jq .`, compare the field names, and
    adjust the `jq` queries in `sunshine-stop-vmon.sh` accordingly.

## Debugging a failed start

Sunshine does **not** forward the prep-cmd script's stdout/stderr into its
own log — a failure there just shows up as
`[...sunshine-start-vmon.sh] exited with code [1]`, with no further
detail. To make failures debuggable, both scripts write their own logs
(including `krfb-virtualmonitor`'s raw output from the start script) to:

```
$XDG_STATE_HOME/sunshine-vmon/start.log   # typically ~/.local/state/sunshine-vmon/start.log
$XDG_STATE_HOME/sunshine-vmon/stop.log
```

These are written to `$XDG_STATE_HOME` (persistent storage) rather than
`$XDG_RUNTIME_DIR`, since the latter is tmpfs and gets wiped on
logout/reboot — exactly when you'd want to inspect a crash. (The pidfile
and monitor-layout snapshot are genuinely ephemeral session state, so
those do stay in `$XDG_RUNTIME_DIR`.)

When something fails:

1. Check `start.log` first — it's overwritten fresh on every invocation,
   so it always reflects the most recent attempt.
2. If Sunshine reported exit code 1, that specifically means the virtual
   display never registered with KWin within the ~10s poll window.
3. The logfile will show why: a missing `krfb-virtualmonitor` binary,
   port 5905 already in use, a permission/portal failure, or the process
   just hanging.
