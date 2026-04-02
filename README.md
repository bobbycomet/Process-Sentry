# ProcessSentry v3

**A predictive, adaptive process priority daemon for Linux.**  
ProcessSentry quietly watches your system in the background and automatically throttles processes that are hogging CPU, memory, or disk I/O, before they make your desktop feel sluggish. It learns over time which programs are repeat offenders and gets faster at reining them in.

---

## Quick Start

### Install the .deb package (recommended)

```bash
sudo apt install ./sentry-pkg.deb
```

Or open the `.deb` file with your distro's graphical package manager (e.g., **GDebi**, **Discover**, **GNOME Software**) and click Install.

That's it. The service starts automatically and runs in the background from now on.

### Check that it's running

```bash
systemctl status process-sentry
```

You should see `active (running)`. ProcessSentry is now protecting your system — no further configuration is required for most users.

### Stop or disable it

```bash
sudo systemctl stop process-sentry      # stop until next reboot
sudo systemctl disable process-sentry   # don't start on boot
```

### Temporarily pause throttling (kill switch)

If you ever need to suspend all throttling without stopping the service — for example, before running a benchmark — create the kill switch file:

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

1. **Lowers its scheduling priority** using `nice` (CPU) and `ionice` (disk I/O).
2. **Moves it into a cgroup slice** (`sentry.slice`) with hard CPU quota and I/O weight limits, if your kernel supports cgroup v2.
3. **Restores the process to normal** as soon as it calms down.
4. **Remembers repeat offenders** — processes that misbehave often get throttled faster next time.

It never kills processes. It never touches system-critical processes, audio servers, display compositors, your desktop shell, or games. If something gets throttled that shouldn't be, see [Excluding Processes](#excluding-processes).

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

At the default `INFO` level you'll see throttle/unthrottle events and habit saves. Switch to `DEBUG` in the config to see every decision the daemon makes.

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

A fully-documented example `config.yaml` is included in the package. Every key has an explanation. The sections are:

| Section | What it controls |
|---|---|
| `intervals` | How often the daemon scans processes |
| `thresholds` | CPU / I/O / memory limits that trigger throttling |
| `prediction` | How aggressively it pre-throttles known offenders |
| `learning` | How fast habit scores rise and fall |
| `cgroup` | CPU quota and I/O weight for throttled processes |
| `exclusions` | Processes that are never touched |

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

The `ema_alpha` value controls how quickly the load average responds to changes. Higher = reacts faster to spikes, lower = smoother and less reactive:

```yaml
intervals:
  ema_alpha: 0.5    # faster reaction (default: 0.3)
  # ema_alpha: 0.1  # very smooth, slow to react
```

### Soften cgroup limits

If you find throttled processes are too severely limited under cgroup v2:

```yaml
cgroup:
  cpu_quota_throttled: 85    # allow 85% of a core (default: 70)
  io_weight_throttled: 30    # less aggressive I/O deprioritisation (default: 10)
```

---

## Excluding Processes

### Never throttle a specific program

Add the program name (or the start of it) to `never_touch_prefixes`:

```yaml
exclusions:
  never_touch_prefixes:
    - myapp        # "myapp", "myapp-helper", "myappd" etc. all ignored
    - blender      # never throttle Blender renders
    - ffmpeg       # never throttle video encoding jobs
```

Matching is case-insensitive and prefix-based on the process name (not the full path). A prefix of `my` would match `myapp`, `myserver`, `mydaemon`, etc., so be as specific as needed.

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

Gaming markers are matched anywhere in the process name **or** its full command line, so they catch games launched through wrappers.

---

## How the Learning System Works

ProcessSentry keeps a habit score (0.0–1.0) for each process name, stored in:

```
/var/lib/process-sentry/habits.yaml
```

- **Score rises** (`up_rate`) each time a process gets throttled.
- **Score falls** (`down_rate`) each cycle a process stays within limits.
- **Score decays passively** (`decay_rate`) over time, so programs that eventually clean up their act stop being penalised.

Once a process's habit score exceeds `pre_throttle_confidence` (default: 0.75), it gets pre-emptively throttled at the start of each cycle, before it even spikes. This is the "predictive" part.

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

ProcessSentry detects what kind of system it's running on and automatically adjusts thresholds:

| Mode | Detected when | CPU threshold | I/O threshold | Mem threshold |
|---|---|---|---|---|
| **Desktop** | Plugged in, not a handheld | config value | config value | config value |
| **Laptop** | Running on battery | 60% | config × 1.3 | config value |
| **Handheld** | Steam Deck / ROG Ally / etc. | 80% | config × 2.0 | 30% |

Handheld mode is very lenient because the system is usually running games. Laptop battery mode is stricter on CPU to preserve charge. Mode is re-detected every poll cycle, so plugging in or unplugging takes effect within seconds.

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

Copy the script to a location on the system PATH:

```bash
sudo cp sentryv3.py /usr/local/bin/sentryv3.py
sudo chmod +x /usr/local/bin/sentryv3.py
```

### Create required directories

```bash
sudo mkdir -p /etc/process-sentry
sudo mkdir -p /var/lib/process-sentry
```

Copy the config file:

```bash
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

### Why does it need a .service file?

ProcessSentry must run as root (it needs to change process priorities, write to cgroup, and read `/proc` for all users). On any modern Linux system, the right way to run a privileged background process automatically is a **systemd service unit**.

The service file tells systemd:

- **`After=network.target`** — start after basic networking is up (not strictly required, but avoids boot-order edge cases).
- **`Type=simple`** — the process runs in the foreground and manages itself; systemd just watches it directly.
- **`Restart=always` / `RestartSec=5`** — if the daemon crashes for any reason, systemd restarts it after 5 seconds automatically. This makes it self-healing.
- **`WantedBy=multi-user.target`** — start it during a normal multi-user (desktop or server) boot.

Without the service file you would have to manually run `sudo python3 sentryv3.py` in a terminal every time you boot, and it would stop when that terminal closes.

---

## Advanced Usage

### Running in the foreground for debugging

```bash
sudo python3 /usr/local/bin/sentryv3.py
```

Stop the service first so they don't conflict:

```bash
sudo systemctl stop process-sentry
```

### Verbose debug logging

In `config.yaml`:

```yaml
logging:
  level: DEBUG
```

Then restart the service. Every throttle/unthrottle decision, every pre-throttle prediction, and every cgroup operation will be written to the log.

### Disabling cgroup v2 (use nice/ionice only)

If your system doesn't support cgroup v2 or you prefer the simpler fallback, you can ensure cgroup is never used by setting quotas to passthrough values:

```yaml
cgroup:
  cpu_quota_throttled: 100
  io_weight_throttled: 100
```

ProcessSentry will still apply `nice(15)` and `ionice` to throttled processes, just without the hard cgroup limits.

### Disabling PSI memory pressure checks

Set the threshold to 0 to ignore `/proc/pressure/memory` entirely:

```yaml
thresholds:
  memory_pressure_avg10: 0
```

PSI requires kernel 4.20+ with `CONFIG_PSI=y`. If the file doesn't exist, ProcessSentry skips PSI checks automatically even without this change.

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

### Gaming integration with GameMode

When a process matching `gaming_markers` is detected, ProcessSentry automatically calls `gamemoded -r` to request GameMode optimisations (CPU governor, scheduler tweaks, etc.). This requires `gamemode` to be installed:

```bash
sudo apt install gamemode
```

No configuration needed — detection and the GameMode request are automatic. When the last gaming process exits, the request is dropped and GameMode releases its hold.

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

**Habits file is huge** — Habits are pruned automatically when they decay below 0.001. You can also delete the file manually to start fresh; it will be recreated on next save.
