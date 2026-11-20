# Raspberry Pi System Health Monitoring

## Overview

This guide documents lightweight host-level health reporting from the Raspberry Pi 5 to the existing Uptime Kuma instance. A Python script reads Linux system metrics and sends them to a Uptime Kuma **Push** monitor every five minutes.

This complements the existing Ping, DNS, and HTTP monitors. Those checks confirm that devices and services respond; this push monitor reports selected resource conditions on the Pi itself.

The implementation uses Python 3 and systemd. It does not require another monitoring platform or expose a new inbound service.

## Architecture

```text
Raspberry Pi 5 (FrankiesPi)
  |
  +-- /usr/local/bin/rpi-health-push.py
  |     +-- CPU temperature
  |     +-- RAM utilization
  |     +-- root filesystem utilization
  |     +-- normalized 1-minute load average
  |
  +-- systemd timer (every 5 minutes)
        |
        +-- systemd oneshot service
              |
              +-- HTTPS/HTTP request to Uptime Kuma Push endpoint
                    |
                    +-- Raspberry Pi System Health monitor
```

The Push URL is a credential because it contains the monitor's unique token. Store it only on the Pi in a root-readable environment file. **Never commit the real URL or token to Git.**

## Metrics and thresholds

| Metric | Source | Warning threshold |
|---|---|---:|
| CPU temperature | `/sys/class/thermal/thermal_zone0/temp` | 75°C or higher |
| RAM utilization | `/proc/meminfo` (`MemAvailable`) | 90% or higher |
| Root filesystem utilization | Python `shutil.disk_usage("/")` | 90% or higher |
| 1-minute load per CPU core | `os.getloadavg()` divided by CPU count | 2.0 or higher |

The script sends `UP` when all metrics are below their warning thresholds and `DOWN` if one or more thresholds are reached. These are configurable operational thresholds, not hardware failure predictions. Review them if the workload or cooling changes.

## 1. Create the Uptime Kuma Push monitor

In Uptime Kuma, add a monitor with:

- Monitor type: **Push**
- Friendly name: `Raspberry Pi System Health`
- Heartbeat interval: `300` seconds
- Retries: `1`
- Upside Down Mode: Off

Save the monitor and copy its generated Push URL. Keep the URL private.

## 2. Store the Push URL securely

Create the environment file:

```bash
sudo nano /etc/default/rpi-health-push
```

Add one line, replacing the placeholder with the complete generated URL:

```bash
PUSH_URL="PASTE_YOUR_COMPLETE_UPTIME_KUMA_PUSH_URL_HERE"
```

There must be no spaces around the equals sign and no trailing semicolon. Restrict access:

```bash
sudo chmod 600 /etc/default/rpi-health-push
```

The file is intentionally excluded from the repository. Do not display or share its contents.

## 3. Install the health reporting script

Create the script:

```bash
sudo nano /usr/local/bin/rpi-health-push.py
```

Use the following Python code:

```python
import os
import urllib.parse
import urllib.request
import shutil

# Read CPU temperature
with open("/sys/class/thermal/thermal_zone0/temp") as f:
    temperature = int(f.read().strip()) / 1000

# Calculate RAM usage
memory = {}
with open("/proc/meminfo") as f:
    for line in f:
        key, value = line.split(":", 1)
        memory[key] = int(value.strip().split()[0])

ram_used = 100 * (
    memory["MemTotal"] - memory["MemAvailable"]
) / memory["MemTotal"]

# Calculate root filesystem usage
disk = shutil.disk_usage("/")
disk_used = 100 * disk.used / disk.total

# Calculate one-minute load per CPU core
load_per_core = os.getloadavg()[0] / (os.cpu_count() or 1)

# Health thresholds
warnings = []

if temperature >= 75:
    warnings.append("High CPU temperature")

if ram_used >= 90:
    warnings.append("High RAM usage")

if disk_used >= 90:
    warnings.append("High disk usage")

if load_per_core >= 2:
    warnings.append("High CPU load")

status = "down" if warnings else "up"

message = (
    f"Temp: {temperature:.1f}C | "
    f"RAM: {ram_used:.1f}% | "
    f"Disk: {disk_used:.1f}% | "
    f"Load/core: {load_per_core:.2f}"
)

if warnings:
    message += " | WARNING: " + ", ".join(warnings)

# Read the private Uptime Kuma Push URL
push_url = os.environ.get("PUSH_URL")

if not push_url:
    raise SystemExit("ERROR: PUSH_URL is not configured")

# Preserve the generated URL while updating its status and message
parts = urllib.parse.urlsplit(push_url)
params = urllib.parse.parse_qs(parts.query, keep_blank_values=True)
params.update({
    "status": [status],
    "msg": [message],
    "ping": [""]
})

url = urllib.parse.urlunsplit((
    parts.scheme,
    parts.netloc,
    parts.path,
    urllib.parse.urlencode(params, doseq=True),
    parts.fragment
))

# Send the health report
request = urllib.request.Request(url, method="GET")

with urllib.request.urlopen(request, timeout=15) as response:
    response.read()

print(f"Uptime Kuma report sent: {status.upper()}")
print(message)
```

Set permissions:

```bash
sudo chmod 755 /usr/local/bin/rpi-health-push.py
```

## 4. Test the report manually

Run the script while loading the protected environment file:

```bash
sudo bash -c 'source /etc/default/rpi-health-push && export PUSH_URL && python3 /usr/local/bin/rpi-health-push.py'
```

A successful run prints a status and current readings, for example:

```text
Uptime Kuma report sent: UP
Temp: 55.6C | RAM: 18.8% | Disk: 39.7% | Load/core: 0.00
```

Refresh Uptime Kuma and confirm that the Push monitor is UP and displays the metric message.

## 5. Create the systemd service

Create `/etc/systemd/system/rpi-health-push.service`:

```ini
[Unit]
Description=Send Raspberry Pi health metrics to Uptime Kuma

[Service]
Type=oneshot
EnvironmentFile=/etc/default/rpi-health-push
ExecStart=/usr/bin/python3 /usr/local/bin/rpi-health-push.py
```

The service is a oneshot task: it runs the script once and exits.

## 6. Create the systemd timer

Create `/etc/systemd/system/rpi-health-push.timer`:

```ini
[Unit]
Description=Run Raspberry Pi health report every five minutes

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min
Unit=rpi-health-push.service

[Install]
WantedBy=timers.target
```

Reload systemd and enable the timer:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rpi-health-push.timer
```

The first scheduled run is two minutes after boot. Subsequent runs are scheduled five minutes after the service's prior activation.

## 7. Verify operation

Check the timer schedule:

```bash
systemctl list-timers rpi-health-push.timer --all
```

Inspect recent service logs:

```bash
sudo journalctl -u rpi-health-push.service -n 10 --no-pager
```

A successful execution includes output similar to:

```text
Uptime Kuma report sent: UP
Temp: 57.9C | RAM: 20.0% | Disk: 39.7% | Load/core: 0.11
```

Check the service's most recent result:

```bash
sudo systemctl status rpi-health-push.service --no-pager
```

<img width="1375" height="826" alt="Screenshot 2026-11-19 at 11 48 40 AM" src="https://github.com/user-attachments/assets/fbed2219-e1fc-4585-aec3-2f19907c28f8" />

## Operational notes

- The monitor heartbeat is 300 seconds, matching the five-minute reporting interval. If reports stop, Uptime Kuma can detect a missed heartbeat.
- The Push URL is a secret. Keep `/etc/default/rpi-health-push` out of Git, backups shared publicly, screenshots, and support messages.
- The script reports current values in the monitor's status message. Uptime Kuma Push is a heartbeat/status mechanism, not a long-term time-series metrics database.
- A `DOWN` report indicates that at least one configured threshold was crossed; inspect the message and host directly to diagnose the cause.
- This setup monitors the Pi's root filesystem and host resource metrics. It does not replace application-specific checks, backups, or alert notifications.

## Skills demonstrated

- Python scripting for Linux system monitoring
- Reading system metrics from `/proc` and `/sys`
- HTTP requests and URL query parameter handling
- Secure environment-file handling
- systemd oneshot services and timers
- Uptime Kuma Push monitoring
- Operational threshold design and log-based verification
