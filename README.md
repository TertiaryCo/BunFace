# BunFace - Face Detection Security System for Ubuntu

A real-time face detection security system for Ubuntu
(and Debian-based) Linux.  Built on OpenCV, Python 3, and systemd.

---

## What It Does

The system continuously reads frames from your built-in or USB webcam,
runs a face detector on each frame, and applies a configurable security
policy.  When an unknown face is seen repeatedly, it fires desktop
notifications, plays an alert sound, optionally sends an email with a
snapshot, and — after a configurable threshold — locks the session using
`loginctl lock-session`.

A lightweight Flask web dashboard streams the annotated live feed and
provides a JSON API for event history and statistics.

---

## Project Structure

```
face-security/
├── main.py                  # Entry point (CLI argument parsing, startup)
├── security_monitor.py      # Central orchestrator (main detection loop)
├── detector.py              # Face detection engine (DNN or Haar cascade)
├── event_logger.py          # SQLite event DB + rotating log files
├── notifier.py              # Desktop / email / sound alerts
├── web_dashboard.py         # Flask MJPEG dashboard
├── config.json              # All runtime configuration
├── requirements.txt         # Python dependencies
├── face-security.service    # systemd unit file
├── sudoers-face-security    # sudo policy (least privilege)
├── install.sh               # One-command installer
├── models/                  # DNN model files (auto-downloaded)
├── logs/                    # Rotating logs + SQLite event DB
├── snapshots/               # Timestamped JPEG snapshots
└── known_faces/             # (Optional) reference photos for recognition
```

---

## Quick Start

### Step 1 — Clone and run the installer

```bash
git clone https://github.com/yourorg/face-security
cd face-security
chmod +x install.sh
./install.sh
```

The installer will ask for your password once via `sudo`.  It handles
everything: system packages, video group membership, Python virtualenv,
DNN model download, systemd service, and the sudoers policy.

> **Note:** If the installer adds you to the `video` group, you must log
> out and back in before the camera will be accessible.

### Step 2 — Verify the service is running

```bash
sudo systemctl status face-security
```

### Step 3 — Open the dashboard

```
http://127.0.0.1:5050
```

You will see the live annotated feed, detection statistics, and a
scrollable list of recent events.

### Step 4 — Run manually (for development)

```bash
cd /opt/face-security
source venv/bin/activate

# Run with an OpenCV window and dashboard
python3 main.py

# Headless (no window, e.g. on a server)
python3 main.py --no-window

# List available cameras
python3 main.py --list-cameras

# Override the detection model
python3 main.py --model haar
```

---

## Configuration Reference

All runtime behaviour is controlled by `config.json`.  The key sections are:

**`camera`** — Device index, resolution, and frame rate.  The system uses
the V4L2 backend on Linux for lowest latency.  `device_index: 0`
corresponds to `/dev/video0`, which is the built-in webcam on most laptops.

**`detection`** — Choose between `"dnn"` (the SSD ResNet-10 neural network,
more accurate) and `"haar"` (the classic Haar cascade, lighter on old
hardware).  Adjust `confidence_threshold` between 0.5 and 0.85 to trade
false positives for sensitivity.

**`security`** — The security policy.  `max_unknown_faces_before_lockdown`
controls how many consecutive unknown-face detections trigger the lockdown
command.  Set it to a higher number in a busy environment.

**`alerts`** — Toggle each notification channel independently.  For email,
it is safer to set credentials via environment variables
(`FACE_SEC_EMAIL_SENDER`, `FACE_SEC_EMAIL_PASSWORD`,
`FACE_SEC_EMAIL_RECIPIENT`) rather than in the config file.

**`dashboard`** — Set `host` to `"0.0.0.0"` to expose the dashboard on
your LAN.  Only do this on a trusted network or behind a reverse proxy with
TLS and authentication.

---

## Detection Models Explained

The DNN model (`res10_300x300_ssd_iter_140000.caffemodel`) is a Single
Shot Detector trained on a large dataset and converted to Caffe format by
the OpenCV team.  It is more robust to lighting variation and profile angles
than the Haar cascade, but requires about 3 MB of disk space and is slightly
heavier on CPU.  On a modern i5/i7 it runs comfortably at 15-30 FPS.

The Haar cascade (`haarcascade_frontalface_default.xml`) is the classic
Viola-Jones detector from 2001.  It is bundled inside the OpenCV Python
package so it never needs to be downloaded.  Choose this on a Raspberry Pi
or other resource-constrained device.

---

## Sudo Privilege Design

The security system runs as your **normal user account**, not as root.  This
is intentional — a root process could cause far more damage if the detection
loop were ever exploited.

The *only* elevated privilege granted is the ability to run
`loginctl lock-session` without a password, via the sudoers drop-in file
at `/etc/sudoers.d/face-security`.  The installer verifies the file's
syntax with `visudo -c` before placing it, so a bad edit can never lock
you out of sudo.

The systemd service additionally requests `CAP_SYS_NICE` through
`AmbientCapabilities` so the process can raise its own scheduling priority
without a full root context.

---

## Hardening the systemd Service

The service unit applies several systemd security directives:

- `NoNewPrivileges=yes` — The process can never gain more privileges than
  it starts with, even via setuid binaries.
- `ProtectSystem=strict` — The entire filesystem is read-only except for
  explicitly listed paths (logs, snapshots).
- `PrivateTmp=yes` — A private, isolated `/tmp` is provided.
- `DeviceAllow` — Only `/dev/video0` and `/dev/video1` are accessible;
  all other device nodes are blocked.

---

## Enabling Face Recognition (Optional)

The base system detects faces but does not identify them.  To enable
recognition you need to install the `face_recognition` library (which
depends on `dlib`):

```bash
sudo apt install cmake libopenblas-dev libx11-dev
/opt/face-security/venv/bin/pip install face_recognition
```

Then place one or more reference photos of known people in the
`known_faces/` directory, named after the person:

```
known_faces/
├── alice.jpg
└── bob.jpg
```

Finally, set `face_recognition_enabled: true` in `config.json`.  The
system will label recognised faces with their name (green box) and
still alert on unknown faces (red box).

---

## Useful Commands

```bash
# Watch the live service log
sudo journalctl -u face-security -f

# Query the event database directly
sqlite3 /opt/face-security/logs/events.db \
  "SELECT timestamp, event_type, face_count, notes FROM events ORDER BY id DESC LIMIT 20;"

# Stop the service temporarily
sudo systemctl stop face-security

# Permanently disable
sudo systemctl disable face-security

# Update (re-run the installer after pulling new code)
git pull && ./install.sh
```

---

## Troubleshooting

**"Camera at /dev/video0 is not accessible"**
Your user is not yet in the `video` group.  Run `sudo usermod -aG video $USER`
then log out and back in.

**No desktop notifications**
`notify-send` requires a running D-Bus session.  When running as a systemd
service, the `DBUS_SESSION_BUS_ADDRESS` environment variable in the unit
file must match your actual session.  You can find the correct value with:
`systemctl --user show-environment | grep DBUS`

**High CPU usage**
Reduce the detection frequency by increasing `detection_interval_ms` in
config.json, or switch from `dnn` to `haar`.

**Lockdown command fails**
Verify the sudoers file is installed correctly:
`sudo visudo -c -f /etc/sudoers.d/face-security`
And that the username in the file matches yours:
`cat /etc/sudoers.d/face-security`

---

