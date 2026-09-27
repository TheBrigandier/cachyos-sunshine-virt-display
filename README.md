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
  Application/Command Preparations hooks these scripts plug into.
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
| `~/.config/sunshine/apps.json` | Application list. Editing this directly is faster than the web UI when adding several similar entries — e.g. one per-client app tile with a different scale argument (see [Setup step 6](#setup)). Its top-level `"env"` key is global to all apps; there's no per-app equivalent. |

## Setup

A quick primer if you haven't used Sunshine's **Applications** page before:
each application is a tile that shows up in Moonlight and has a **Command**
to launch (a game, Steam, or a desktop session) plus optional **Command
Preparations**. The preparations contain a **Do Command** that runs before
the stream starts and an **Undo Command** that runs after it ends; these
scripts create and tear down the virtual display.

1. **Install the scripts to `~/.local/bin`.** That directory is user-owned
   (no `sudo` needed) and is commonly on `PATH`, including on CachyOS.

   ```
   cp scripts/sunshine-start-vmon.sh scripts/sunshine-stop-vmon.sh ~/.local/bin/
   chmod +x ~/.local/bin/sunshine-start-vmon.sh ~/.local/bin/sunshine-stop-vmon.sh
   ```

   The `chmod +x` is required; without it, Sunshine cannot run the scripts.

2. **Configure Sunshine's capture method.** By default
   Sunshine auto-detects how to grab your screen, and on KDE Wayland that
   auto-detected method won't capture the virtual display these scripts
   create — you have to force it to use KDE's own `kwin` capture backend
   instead.

   - Open Sunshine's web UI in a browser: `https://your-ip:47990` (replace
     `your-ip` with the IP or hostname of the machine running Sunshine; if
     you're on the same machine, `https://localhost:47990` works too).
   - Log in if prompted, then open the **Configuration** page.
   - In the **Advanced** tab, find **Force a Specific Capture Method**
     and set it to **KWin Screencast**. This forces Sunshine to use KDE's
     capture backend, which can capture the virtual display created by the
     scripts.
   - Click **Save**, then restart Sunshine so the settings take effect.

3. **Choose one Command Preparations scope.** Do not configure both global
   and per-application **Command Preparations**.

   - **Global Command Preparations:** use this only when every Sunshine
     connection should use the virtual display. On the **Configuration**
     page, open the **General** tab, find **Command Preparations**, click
     **+ Add**, and enter the following values:

     | Field | Value |
     | --- | --- |
     | Do Command | `sunshine-start-vmon.sh` |
     | Undo Command | `sunshine-stop-vmon.sh` |

     The scripts are available by name after step 1 installs `~/.local/bin` on your PATH.

   - **Per-application Command Preparations:** use this when only selected
     applications should use the virtual display. On the **Applications**
     page, edit an existing application or click **Add New**. In its
     **Command Preparations** section, set:

     | Field | Value |
     | --- | --- |
     | Do Command | `sunshine-start-vmon.sh` |
     | Undo Command | `sunshine-stop-vmon.sh` |
     | Command | whatever you want to launch, e.g. `setsid steam steam://open/bigpicture` — or leave it blank to land in a normal desktop session on the virtual display |

     Save the application after adding the commands. Sunshine does not expand
     `~` or `$HOME` in these fields, so use the script names shown above.

4. **(Recommended) Add monitor cleanup to Sunshine's user service.** Add the
   stop script as an `ExecStopPost` hook so monitors are restored if Sunshine
   exits unexpectedly.

   ```
   systemctl --user edit app-dev.lizardbyte.app.Sunshine.service
   ```

   Add:

   ```ini
   [Service]
   ExecStopPost=%h/.local/bin/sunshine-stop-vmon.sh
   ```

   Then reload systemd:

   ```
   systemctl --user daemon-reload
   ```

   The `%h` specifier expands to your home directory.

5. **(Recommended) Select the display for multi-monitor setups.**

   Open the **Configuration** page, select the **Audio/Video**
   tab, find **Display Id**,
   and follow Sunshine's display-selection instructions for the screen you
   want to capture.

   During a virtual-monitor session, the scripts temporarily select the
   virtual display and restore the previous target when streaming ends.

6. **(Optional) Use per-client DPI scaling.**
   Skip this unless you stream to
   multiple devices with different DPI needs (e.g. a laptop and a tablet)
   and want each to get a different UI scale.

   Per-client scaling uses the per-application **Command Preparations** from
   step 5 instead of the global **Command Preparations** from step 2. Remove
   the global preparations before configuring these application-specific
   commands.

   - Create one Application per device, following step 5, but append a
     scale factor as an argument to the **Do Command**:

     | Field | Value |
     | --- | --- |
     | Do Command | `sunshine-start-vmon.sh 1.5` |
     | Undo Command | `sunshine-stop-vmon.sh` (same as before — no argument needed) |

   - Name each tile after the device it's for (`Desktop (MacBook)`,
     `Desktop (iPad)`, `Desktop (TV)`, ...) and set the scale factor you
     want for that device.
   - In Moonlight, pick the matching tile per client — it remembers your
     last choice per host. Omit the argument entirely to use scale `1`.

   **Why per-device tiles instead of a per-device setting?** Sunshine's
   **Do Command** scripts only receive the resolution the client requested,
   not which client connected, so there's no built-in way to detect the
   device automatically. `apps.json`'s top-level `"env"` key could in
   theory pass a variable through, but it's global to every Application (a
   sibling of `"apps"` in the file, not a per-app override), so it can't
   vary by device either. Passing the scale as a CLI argument on a
   per-device Application tile is the workaround — no JSON editing
   required, since the **Do Command** field is exposed inside
   **Command Preparations** in the web UI.

7. **(Optional) Start Sunshine at login and lock an autologin session.**
   Enable Sunshine with the graphical session so it starts after you log in.
   For a headless setup, combine this with autologin; the lock-on-start unit
   then locks the session immediately instead of leaving the physical console
   unlocked.

   - **Enable Sunshine at login** so it starts with the graphical session:
     ```
     systemctl --user enable app-dev.lizardbyte.app.Sunshine.service
     ```

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
     This is tied to `graphical-session.target` (via `PartOf`/`WantedBy`), so
     it fires on every graphical login —
     autologin at boot or a normal manual login alike. Its `After=` is
     deliberately pinned to `plasma-kwin_wayland.service` and
     `plasma-ksmserver.service` specifically, **not**
     `graphical-session.target` itself — on one test system,
     `plasma-powerdevil.service` (also pulled in by that target) took 7+
     seconds to start, which delayed the target and left the desktop
     visible and unlocked the whole time. Ordering on KWin/ksmserver
     directly means the lock fires within about a second of them being up,
     regardless of how long slower, lock-irrelevant units elsewhere in the
     session take to start. Confirmed on Plasma 6.7.2/CachyOS: the session
     goes straight to the lock screen on boot with no visible unlocked
     desktop, and Sunshine's virtual-monitor capture keeps streaming
     normally while locked — connecting via Moonlight shows the real lock
     screen (and the desktop after unlocking), not a frozen frame.

## How it works

1. **Moonlight connects.** Sunshine determines the resolution and frame
   rate the client requested, and exposes them as the environment
   variables `SUNSHINE_CLIENT_WIDTH`, `SUNSHINE_CLIENT_HEIGHT`, and
   `SUNSHINE_CLIENT_FPS` before running the **Do Command** in **Command
   Preparations**.
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
   `kwin` capture method ([Setup step 2](#setup)), it captures the virtual
   display specifically, at the exact resolution created above — no
   scaling or letterboxing.
5. **On disconnect, the stop script cleans up:** it kills the virtual
   display and restores your physical monitors to their pre-stream layout.

## Known issues / FAQ

- **Sunshine crashes mid-session before the service cleanup hook is installed**
  - What happens: physical monitors stay disabled until you manually run
    the stop script.
  - Why: without the service's `ExecStopPost` hook
    ([Setup step 4](#setup)), nothing runs the stop script if Sunshine
    itself dies unexpectedly.
  - Fix: run `~/.local/bin/sunshine-stop-vmon.sh` by hand to restore your
    monitors, then add the service cleanup hook in Setup step 4.

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
    `modes[].id`/`.size.width`/`.size.height`/`.refreshRate`)

- **Using multiple monitors**
  - This setup has been tested on a multi-monitor CachyOS system. To apply
    the correct screen, open the **Configuration** page, select the
    **Audio/Video** tab, and configure **Display Id**.

## Debugging a failed start

Sunshine does **not** forward the Do Command script's stdout/stderr into its
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
