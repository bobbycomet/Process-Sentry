<p align="center">
  <img src="https://github.com/bobbycomet/Process-Sentry/blob/main/sentry.png?raw=true" width="50%">
</p>

# Process Sentry v3.1.0

A predictive, adaptive process priority daemon for Linux.

ProcessSentry quietly watches your system in the background and automatically throttles processes that are hogging CPU, memory, or disk I/O, before they make your desktop feel sluggish. It learns over time which programs are repeat offenders and gets faster at reining them in. Made for Windows switchers to Ubuntu to have an easier time. No terminal needed to figure out why the CPU spiked and then fix it. This is part of the full Griffin tool set. It is already integrated into Grix. [Grix preview](https://github.com/bobbycomet/Grix-Preview) [ Full Griffin Vision](https://bobbycomet.github.io/Griffin-Linux-Landing-Page/) 

> **Best used with:** [Kernel Autotune V2](https://github.com/bobbycomet/kernel-autotune-V2)
> Kernel Autotune + Process Sentry together give Ubuntu/Debian users a smoother gaming and desktop experience similar to specialized distributions, without changing your base system. Works with mainline, Xanmod, and Liquorix kernels.

---

## Quick Start

### Install the .deb package (recommended)

[**Download sentry-pkg.deb**](https://github.com/bobbycomet/Process-Sentry/releases/download/v3.1.0/sentry-pkg.deb)

```bash
sudo apt install ./sentry-pkg.deb
```

Or open the .deb file with your distro's graphical package manager (GDebi, Discover, GNOME Software) and click Install. The service starts automatically and runs in the background from now on.

### Check that it's running

```bash
systemctl status process-sentry
```

You should see `active (running)`. ProcessSentry is now protecting your system; no further configuration is required for most users.

### Stop or disable it

```bash
sudo systemctl stop process-sentry      # stop until next reboot
sudo systemctl disable process-sentry   # don't start on boot
```

### Temporarily pause throttling (kill switch)

If you ever need to suspend all throttling without stopping the service. For example, before running a benchmark, create the kill switch file:

```bash
sudo touch /etc/no-auto-throttle
```

Remove it to resume normal operation:

```bash
sudo rm /etc/no-auto-throttle
```

---

## What Does It Actually Do?

When a process spikes in CPU usage, hammers the disk, or balloons in memory, ProcessSentry:

1. Lowers its scheduling priority using `nice` (CPU) and `ionice` (disk I/O).
2. Moves it into a cgroup slice (`sentry.slice`) with hard CPU quota and I/O weight limits, if your kernel supports cgroup v2.
3. Optionally applies a soft memory ceiling (`memory.high`) to throttled processes when memory pressure is detected.
4. Restores the process to normal as soon as it calms down.
5. Remembers repeat offenders — processes that misbehave often get throttled faster next time.

It **never kills processes**. It never touches system-critical processes, audio servers, display compositors, your desktop shell, games, or [Appify](https://github.com/bobbycomet/appify) PWA apps. If something gets throttled that shouldn't be, see [Excluding Processes](https://github.com/bobbycomet/Process-Sentry/blob/main/sentry-pkg/etc/process-sentry/config.yaml). You can configure the file at any time; this file is set up for most general uses.

---

## Log File

All activity is logged to:

```
/var/log/process-sentry.log
```

To watch it live:

```bash
sudo tail -f /var/log/process-sentry.log
```

At the default `INFO` level, you'll see throttle/unthrottle events and habit saves. Switch to `DEBUG` in the config to see every decision the daemon makes.

---

## Configuration

The config file lives at:

```
/etc/process-sentry/config.yaml
```

It is plain YAML. Edit it with any text editor (you need root):

```bash
sudo nano /etc/process-sentry/config.yaml
```

After saving, restart the service for changes to take effect:

```bash
sudo systemctl restart process-sentry
```

A fully-documented example `config.yaml` is included in the package. The sections are:

| Section | What it controls |
|---|---|
| `intervals` | How often the daemon scans processes and how it adapts to load |
| `thresholds` | CPU/I/O/memory/PSI limits that trigger throttling, with per-mode overrides |
| `prediction` | How aggressively it pre-throttles known offenders |
| `learning` | How fast habit scores rise, fall, and decay |
| `cgroup` | CPU quota, I/O weight, and optional memory ceiling for throttled processes |
| `gaming` | Throttle-pause window and CPU threshold boost during gaming sessions |
| `exclusions` | Processes that are never touched |
| `appify_exempt` | Whether to exempt Appify PWA apps from throttling |
| `logging` | Log verbosity level |

---

## Common Configuration Changes

### Make throttling less aggressive

Raise the thresholds so processes get more headroom before being throttled:

```yaml
thresholds:
  cpu_percent: 70.0       # default: 50.0
  io_mbps: 50.0           # default: 25.0
  memory_percent: 25.0    # default: 18.0
```

### Make throttling more aggressive

Lower the thresholds and speed up learning:

```yaml
thresholds:
  cpu_percent: 35.0
  io_mbps: 15.0
  memory_percent: 12.0

learning:
  up_rate: 0.15     # learn bad habits faster (default: 0.08)
  down_rate: 0.01   # forget good behavior slower (default: 0.03)
```

### Change polling speed

For a very responsive desktop (polls every second under load):

```yaml
intervals:
  base: 3.0    # check every 3 seconds normally
  min: 0.5     # check twice a second under heavy load
  max: 10.0    # relax to every 10 seconds when idle
```

For a battery-saving laptop where you want less CPU overhead from the daemon itself:

```yaml
intervals:
  base: 8.0
  min: 3.0
  max: 30.0
```

### Tune the EMA (load smoothing)

The `ema_alpha` value controls how quickly the load average responds to changes. Higher = reacts faster to spikes; lower = smoother and less reactive:

```yaml
intervals:
  ema_alpha: 0.5    # faster reaction (default: 0.3)
  # ema_alpha: 0.1  # very smooth, slow to react
```

### Soften cgroup limits

If you find that throttled processes are too severely limited under cgroup v2:

```yaml
cgroup:
  cpu_quota_throttled: 85    # allow 85% of a core (default: 70)
  io_weight_throttled: 30    # less aggressive I/O deprioritisation (default: 10)
```

### Enable the memory cgroup ceiling

By default, `memory_high_percent` is `0` (disabled). Set it to apply a soft RAM ceiling to throttled processes. Use with caution; setting it too low can cause thrashing inside the cgroup slice.

```yaml
cgroup:
  memory_high_percent: 40    # throttled slice gets at most 40% of system RAM
```

### Tune gaming behaviour

When a game is detected, ProcessSentry pauses new throttle decisions for a configurable window and raises the CPU threshold so background work gets more headroom:

```yaml
gaming:
  throttle_pause_seconds: 30      # how long to pause throttling after a game is detected (default: 30)
  cpu_threshold_multiplier: 1.5   # multiply the CPU threshold while gaming (default: 1.5)
```

Set `throttle_pause_seconds: 0` to disable the pause window entirely.

---

## Excluding Processes

### Never throttle a specific program

Add the program name (or the start of it) to `never_touch_prefixes`:

```yaml
exclusions:
  never_touch_prefixes:
    - myapp        # "myapp", "myapp-helper", "myappd", etc., all ignored
    - blender      # never throttle Blender renders
    - ffmpeg       # never throttle video encoding jobs
```

Matching is case-insensitive and prefix-based on the process name (not the full path). Be as specific as needed to avoid accidentally excluding unrelated programs.

### Mark a game or launcher as a gaming process

Gaming processes are completely excluded from throttling and trigger GameMode (if installed):

```yaml
exclusions:
  gaming_markers:
    - ryujinx      # Nintendo Switch emulator
    - rpcs3        # PS3 emulator
    - dolphin      # GameCube/Wii emulator
    - .AppImage    # treat all AppImages as games (broad — use carefully)
```

Gaming markers are matched anywhere in the process name or its full command line, so they catch games launched through wrappers.

### Appify PWA apps

If you use [Appify](https://github.com/bobbycomet/appify) to run web apps as desktop apps, those browser processes are automatically exempt from throttling. This is controlled by:

```yaml
appify_exempt: true    # default: true — set to false to disable
```

Appify apps are identified by their browser command line containing `~/.pwa_manager/profiles/`, so they are distinguished from regular browser windows without needing you to list them manually. This means cloud gaming will get a bit of a boost as well, since it will be the foreground application.

---

## How the Learning System Works

ProcessSentry keeps a habit score (0.0–1.0) for each process name, stored in:

```
/var/lib/process-sentry/habits.yaml
```

- Score **rises** (`up_rate`) each time a process gets throttled.
- Score **falls** (`down_rate`) each cycle a process stays within limits.
- Score **decays passively** (`decay_rate`) over time on a timed interval, so programs that eventually clean up their act stop being penalised.

Once a process's habit score exceeds `pre_throttle_confidence` (default: `0.75`), it gets pre-emptively throttled at the start of each cycle, before it even spikes. A secondary trigger also fires early throttling if a process's recent CPU history average exceeds 80% of the current threshold, even if its habit score hasn't crossed the confidence threshold yet.

To reset all learned habits:

```bash
sudo rm /var/lib/process-sentry/habits.yaml
sudo systemctl restart process-sentry
```

To inspect current habits:

```bash
sudo cat /var/lib/process-sentry/habits.yaml
```

---

## Power Modes

ProcessSentry detects what kind of system it's running on and automatically adjusts thresholds. Mode is re-checked roughly once per minute (not every poll cycle) to avoid unnecessary overhead.

| Mode | Detected when | CPU threshold | I/O threshold | Mem threshold |
|---|---|---|---|---|
| Desktop | Plugged in, not a handheld | config value | config value | config value |
| Laptop | Running on battery | 60% | config × 1.3 | config value |
| Handheld | Steam Deck / ROG Ally / etc. | 80% | config × 2.0 | 30% |

Handheld mode is very lenient because the system is usually running games. Laptop battery mode is stricter on CPU to preserve charge.

The per-mode threshold values in the config also support a multiplier syntax. For example, setting `io_mbps: "1.5x"` under the `laptop` section applies 1.5× the base I/O threshold instead of a fixed value.

---

## Gaming Integration with GameMode

When a process matching `gaming_markers` is detected, ProcessSentry requests GameMode optimisations (CPU governor, scheduler tweaks, etc.) via a D-Bus call to `com.feralinteractive.GameMode`. This requires `gamemode` to be installed:

```bash
sudo apt install gamemode
```

No configuration needed, detection and the GameMode request are automatic. A debounce timer prevents rapid toggling if a game briefly disappears from the process list. When games have been absent for two consecutive scans, the GameMode request is released.

---

## Building From Source

### Prerequisites

ProcessSentry is a single Python 3 script. You need Python 3.8 or later and two third-party libraries:

```bash
pip install pyyaml psutil
```

Or via your distro's package manager:

```bash
# Debian / Ubuntu
sudo apt install python3-yaml python3-psutil

# Fedora / RHEL
sudo dnf install python3-pyyaml python3-psutil

# Arch
sudo pacman -S python-yaml python-psutil
```

### Install the script

```bash
sudo cp sentryv3.py /usr/local/bin/sentryv3.py
sudo chmod +x /usr/local/bin/sentryv3.py
```

### Create required directories

```bash
sudo mkdir -p /etc/process-sentry
sudo mkdir -p /var/lib/process-sentry
sudo cp config.yaml /etc/process-sentry/config.yaml
```

### Install and enable the systemd service

Save the following as `/etc/systemd/system/process-sentry.service`:

```ini
[Unit]
Description=Process Sentry v3
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/sentryv3.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Then enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable process-sentry
sudo systemctl start process-sentry
```

---

## Advanced Usage

### Running in the foreground for debugging

Stop the service first so they don't conflict:

```bash
sudo systemctl stop process-sentry
sudo python3 /usr/local/bin/sentryv3.py
```

### Verbose debug logging

In `config.yaml`:

```yaml
logging:
  level: DEBUG
```

Then restart the service. Every throttle/unthrottle decision, every pre-throttle prediction, and every cgroup operation will be written to the log.

### Disabling cgroup v2 (use nice/ionice only)

Set quotas to passthrough values:

```yaml
cgroup:
  cpu_quota_throttled: 100
  io_weight_throttled: 100
```

ProcessSentry will still apply `nice(15)` and `ionice` to throttled processes, just without the hard cgroup limits.

### Disabling PSI memory pressure checks

Set the threshold to `0` to ignore `/proc/pressure/memory` entirely:

```yaml
thresholds:
  memory_pressure_avg10: 0
```

PSI requires kernel 4.20+ with `CONFIG_PSI=y`. If the file doesn't exist, ProcessSentry skips PSI checks automatically, even without this change.

### Disabling habit decay (permanent memory)

To make the daemon remember bad actors indefinitely until manually cleared:

```yaml
learning:
  decay_rate: 0.0
```

### Disabling predictive pre-throttling

Set the confidence threshold above the habit cap so it's never reached:

```yaml
prediction:
  pre_throttle_confidence: 1.1    # habit_cap is 1.0, so this never fires
```

---

## File Reference

| Path | Purpose |
|---|---|
| `/usr/local/bin/sentryv3.py` | Main daemon script |
| `/etc/process-sentry/config.yaml` | Configuration (edit this) |
| `/var/lib/process-sentry/habits.yaml` | Learned process habits (auto-managed) |
| `/var/log/process-sentry.log` | Log file |
| `/run/process-sentry.pid` | PID file (auto-managed) |
| `/etc/no-auto-throttle` | Kill switch — create to pause, delete to resume |
| `/etc/systemd/system/process-sentry.service` | Systemd service unit |

---

## Troubleshooting

**"Must run as root"** — The daemon requires root to adjust process priorities and manage cgroups. Always start it via systemd or with `sudo`.

**"cgroup v2 not detected, using nice/ionice fallback"** — Your kernel or init setup doesn't expose cgroup v2. Throttling still works via nice/ionice, just without hard CPU quota enforcement.

**A process I care about is being throttled** — Add it to `never_touch_prefixes` in the config and restart the service. See [Excluding Processes](#excluding-processes).

**The daemon is using noticeable CPU itself** — Increase the `base` and `min` intervals to poll less frequently. Even `min: 2.0` makes a significant difference on very busy systems.

**Habits file is huge** — Habits are pruned automatically when they decay below `0.001`. You can also delete the file manually to start fresh; it will be recreated on next save.

**A game triggers GameMode but I don't have it installed** — This is harmless. The D-Bus call fails silently if `gamemode` isn't present. Install it with `sudo apt install gamemode` if you want the feature.

**An Appify PWA is being throttled** — Ensure `appify_exempt: true` is set in your config (it is by default). If the issue persists, check that the app's browser process has `~/.pwa_manager/profiles/` in its command line with `ps aux | grep pwa_manager`.

---

## What's New in v3.1.0

### Bug Fixes

- **GameMode integration corrected** — Previous versions incorrectly called `gamemoded -r`, which attempted to start a second daemon instance. The fix uses a proper D-Bus call (`gdbus`) to `com.feralinteractive.GameMode.RegisterGame`, which is the correct way to request GameMode as a client.
- **Config merging fixed** — User config files are now deep-merged with defaults instead of shallow-merged. This means new config sections added in future updates (like `gaming:` and `appify_exempt`) are no longer silently dropped when an older config file is present.
- **CPU percentage calculation corrected** — CPU usage is now computed from raw `cpu_times` deltas rather than relying on psutil's `cpu_percent()`, giving accurate per-process readings across varying poll intervals instead of inflated or stale values.
- **PID reuse detection added** — The process class cache is now invalidated when a PID is reused by a different process (detected by name change), preventing a new process from inheriting the wrong throttle classification.
- **Habit decay now time-scaled** — Decay was previously applied at a flat rate regardless of how much time had elapsed since the last decay pass. It now scales by elapsed seconds, so the effective decay rate stays consistent even when the daemon is busy or the poll interval varies.
- **IO delta guard against negative values** — A kernel rounding edge case could produce a tiny negative IO byte delta. This is now clamped to zero to prevent negative IO rates from corrupting throttle decisions.
- **cgroup CPU quota formula fixed** — The quota now correctly accounts for the number of CPU cores (`quota_us = pct/100 * period_us * cpu_count`), so a 70% quota on a 16-core machine no longer starves throttled processes to a fraction of one core.
- **Mode detection rate-limited** — Battery and handheld checks are now performed at most once per minute rather than on every poll loop, reducing unnecessary syscall overhead on busy systems.
- **GameMode debounce added** — GameMode is no longer released after a single game-free scan. A two-scan debounce prevents rapid toggling if a game briefly disappears from the process list.
- **Temp file naming fixed for habit saves** — The temporary file used during atomic habit saves is now named `habits.yaml.tmp` rather than `habits.tmp`, avoiding potential collisions with other files in the same directory.

### New Features

- **`gaming` config section** — A new `gaming:` block controls how ProcessSentry behaves while a game is running: `throttle_pause_seconds` (default: 30) suppresses new throttle decisions for a window after game detection, and `cpu_threshold_multiplier` (default: 1.5) raises the CPU threshold during play to give background work more headroom.
- **`appify_exempt` option** — Processes launched by [Appify](https://github.com/bobbycomet/appify) as PWA desktop apps are now automatically exempt from throttling. They are identified by their browser command line containing `~/.pwa_manager/profiles/`, which distinguishes them from regular browser windows without any manual configuration. Set `appify_exempt: false` to disable.
- **`memory_high_percent` cgroup option** — A new `cgroup: memory_high_percent` setting applies a soft memory ceiling (`memory.high`) to the throttled cgroup slice. Disabled by default (`0`). Only applied to throttled processes, not to normal-priority ones.
- **Threshold multiplier syntax** — Per-mode threshold overrides now support a multiplier suffix (e.g. `io_mbps: "1.3x"`) in addition to absolute values, making it easier to express relative adjustments without knowing the base value.
- **Secondary predictive trigger** — In addition to the habit-score threshold, a process is now pre-emptively throttled if its recent CPU history average exceeds 80% of the current threshold — catching processes that are clearly trending toward a spike before their habit score has had time to build up.
- **`memory.high` floor** — When `memory_high_percent` is set, the computed byte value is floored at 256 MiB to prevent the throttled slice from being made so small that normal processes inside it thrash constantly.
- **Expanded default exclusion list** — The built-in `never_touch_prefixes` list now covers a much broader range of common applications out of the box, including IDEs, creative tools, communication apps, browsers, terminals, and system monitors, so fewer users need to manually add exclusions.

Process Sentry (aka Sentry) is part of the Griffin Linux project. The name Process Sentry, the Griffin Linux name, and associated icons are protected under the GPLv3 to preserve the integrity of the branding in all distributed versions.
