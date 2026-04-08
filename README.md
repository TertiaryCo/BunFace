# BunFace - Face Authentication for Linux

**Face recognition authentication for Linux — sudo, screensaver, and full GDM login screen.**

Built with MediaPipe BlazeFace (detection) + Silent-Face Anti-Spoofing (liveness) + dlib (recognition) + PyQt5 (GUI).

---

## Features

- **Face authentication for `sudo`** — replaces password prompt with live face scan
- **GDM login screen integration** — face auth on the login screen, password always available as fallback
- **Anti-spoofing** — Silent-Face MiniFASNet rejects printed photos and screen replays
- **Rolling window verification** — analyses 10 consecutive frames, requires 70% match — robust to blinks and lighting changes
- **Live camera feed** — PyQt5 GUI shows real-time camera, liveness bar, match progress
- **Graceful fallback** — face auth failure always falls through to password, never locks you out
- **Headless mode** — works over SSH without a display

---

## System Requirements

| Component | Requirement |
|---|---|
| OS | Ubuntu 22.04 / 24.04 |
| Display manager | GDM3 (for login screen integration) |
| Display server | Wayland |
| Python | 3.10+ |
| Camera | Any V4L2-compatible webcam |
| RAM | 4GB+ recommended |
| GPU | Optional (CUDA speeds up liveness check) |

---

## Quick Install

```bash
git clone https://github.com/yourusername/faceauth-linux.git
cd faceauth-linux
chmod +x setup.sh
./setup.sh
```

The setup script will:
1. Install system and Python dependencies
2. Download Silent-Face Anti-Spoofing models via git-lfs
3. Install all source files to `~/Python-Project/`
4. Configure sudo PAM
5. Optionally configure GDM login screen
6. Walk you through face registration

---

## Manual Install

If you prefer to install manually, follow the [Standard Operating Procedure](docs/SOP.md).

---

## Project Structure

```
faceauth-linux/
├── src/
│   ├── liveness.py        # Silent-Face anti-spoofing (shared module)
│   ├── register.py        # Face registration with PyQt5 camera
│   ├── auth.py            # PAM entry point (sudo / terminal)
│   ├── auth_gui.py        # PyQt5 popup for sudo auth
│   └── gdm_auth_gui.py    # Fullscreen PyQt5 for GDM login
├── scripts/
│   └── face-login         # Bash wrapper called by PAM
├── config/
│   └── gdm-password       # GDM PAM configuration
├── docs/
│   ├── SOP.md             # Step-by-step manual setup guide
│   └── ARCHITECTURE.md    # System architecture deep dive
├── setup.sh               # Automated installer
├── requirements.txt
└── README.md
```

---

## How It Works

```
User attempts login / sudo
          ↓
PAM calls face-login wrapper
          ↓
gdm_auth_gui.py opens (fullscreen or popup)
          ↓
MediaPipe BlazeFace detects face in live feed
          ↓
Silent-Face MiniFASNet checks liveness
(rejects photos, screens, masks)
          ↓
dlib compares face embedding to registered encoding
across 10-frame rolling window (70% match required)
          ↓
EXIT 0 → access granted    EXIT 1 → password prompt
```

---

## Rolling Window Verification

Instead of checking a single captured frame, FaceAuth continuously scans the live feed:

- Collects 10 frames where a real face is detected
- Each frame is checked for liveness first — spoof frames are ignored
- If 7 out of 10 frames match the registered face → access granted
- Robust to blinks, head turns, and lighting variation
- Up to 4 spoof frames are ignored before triggering a spoof block

---

## Security Notes

- Face encodings are stored locally at `~/Python-Project/Face_data/encoding.npy`
- No data leaves your machine
- Anti-spoofing uses two MiniFASNet models in ensemble — both must agree the face is real
- Password fallback is always available — press `ESC` during face auth
- Failed face auth falls through to password, never locks you out
- PAM `optional` flag means face auth failure is never fatal

---

## Troubleshooting

**Black camera feed**
```bash
# Test which backend works
python3 -c "import cv2; cap = cv2.VideoCapture(0, cv2.CAP_V4L2); print(cap.isOpened()); cap.release()"
```

**Liveness models not loading**
```bash
ls ~/Python-Project/models/*.pth
# Should show two .pth files ~1.8MB each
```

**Qt platform error on Wayland**
```bash
export QT_QPA_PLATFORM=wayland
python3 ~/Python-Project/register.py
```

**Permission denied on log file**
```bash
sudo chmod 666 /tmp/face_auth.log /tmp/gdm_face_auth.log
```

**Restore original PAM (emergency)**
```bash
# Sudo
sudo cp /etc/pam.d/sudo.backup.* /etc/pam.d/sudo
# GDM
sudo cp /etc/pam.d/gdm-password.backup.* /etc/pam.d/gdm-password
```

---

## Logs

```bash
# sudo / terminal auth log
cat /tmp/face_auth.log

# GDM login screen log
cat /tmp/gdm_face_auth.log

# Watch live
tail -f /tmp/face_auth.log
```

---

## Re-register Face

```bash
QT_QPA_PLATFORM=wayland python3 ~/Python-Project/register.py
```

---

## License

MIT License. See [LICENSE](LICENSE) for details.

---

## Acknowledgements

- [Silent-Face-Anti-Spoofing](https://github.com/minivision-ai/Silent-Face-Anti-Spoofing) by Minivision AI
- [face_recognition](https://github.com/ageitgey/face_recognition) by Adam Geitgey
- [MediaPipe](https://github.com/google/mediapipe) by Google