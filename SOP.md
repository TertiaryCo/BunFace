# Standard Operating Procedure — FaceAuth Linux

**Version:** 1.0  
**Tested on:** Ubuntu 22.04 / 24.04, GDM3, Wayland  
**Estimated setup time:** 20–30 minutes

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install System Dependencies](#2-install-system-dependencies)
3. [Set Up Python Environment](#3-set-up-python-environment)
4. [Download Anti-Spoofing Models](#4-download-anti-spoofing-models)
5. [Install Source Files](#5-install-source-files)
6. [Register Your Face](#6-register-your-face)
7. [Configure sudo PAM](#7-configure-sudo-pam)
8. [Configure GDM Login Screen](#8-configure-gdm-login-screen)
9. [Testing](#9-testing)
10. [Uninstall](#10-uninstall)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Prerequisites

Before starting, verify your system meets all requirements.

### 1.1 Check Ubuntu version
```bash
lsb_release -a
```
Must be Ubuntu 22.04 or 24.04.

### 1.2 Check display manager
```bash
cat /etc/X11/default-display-manager
```
Must output `/usr/sbin/gdm3`. If not, GDM login integration will not work (sudo auth still works).

### 1.3 Check display server
```bash
echo $XDG_SESSION_TYPE
```
Must output `wayland`.

### 1.4 Check camera
```bash
ls /dev/video*
v4l2-ctl --list-devices
```
At least one camera device must be present.

### 1.5 Check camera quality
```bash
python3 -c "
import cv2, numpy as np, time
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
time.sleep(2)
for _ in range(10): cap.read()
ret, frame = cap.read()
cap.release()
gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
print(f'Sharpness: {cv2.Laplacian(gray, cv2.CV_64F).var():.1f} (need > 50)')
print(f'Resolution: {frame.shape[1]}x{frame.shape[0]}')
"
```
Sharpness above 50 is required. Above 100 is good.

---

## 2. Install System Dependencies

```bash
sudo apt-get update
sudo apt-get install -y \
    python3 python3-pip python3-venv python3-dev \
    build-essential cmake \
    libopencv-dev \
    v4l-utils \
    git git-lfs \
    libgl1-mesa-glx \
    libglib2.0-0 \
    libgtk-3-dev \
    libatlas-base-dev \
    gfortran \
    libboost-all-dev \
    libqt5wayland5 \
    qtwayland5
```

**Verify:**
```bash
python3 --version   # must be 3.10+
git lfs version     # must be installed
```

---

## 3. Set Up Python Environment

### 3.1 Create install directory
```bash
mkdir -p ~/Python-Project
mkdir -p ~/Python-Project/models
mkdir -p ~/Python-Project/Face_data
```

### 3.2 Create virtual environment
```bash
cd ~/Python-Project
python3 -m venv Penv
```

### 3.3 Install Python packages
```bash
# Activate venv
source ~/Python-Project/Penv/bin/activate

# Install PyTorch (CPU version — add --index-url for CUDA if you have GPU)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu

# Install remaining packages
pip install \
    opencv-python \
    face-recognition \
    mediapipe \
    PyQt5 \
    numpy

deactivate
```

**Verify all imports work:**
```bash
~/Python-Project/Penv/bin/python3 -c "
import cv2, face_recognition, mediapipe, torch
from PyQt5.QtWidgets import QApplication
print('All imports OK')
print('PyTorch:', torch.__version__)
"
```

---

## 4. Download Anti-Spoofing Models

### 4.1 Clone Silent-Face repo with LFS
```bash
git lfs install
git lfs clone https://github.com/minivision-ai/Silent-Face-Anti-Spoofing.git \
    ~/Python-Project/sfas
```

### 4.2 Copy model files
```bash
cp ~/Python-Project/sfas/resources/anti_spoof_models/*.pth \
   ~/Python-Project/models/
```

### 4.3 Verify models
```bash
ls -lh ~/Python-Project/models/*.pth
```
Expected output — two files, each ~1.8MB:
```
-rw-rw-r-- 1 user user 1.8M ... 2.7_80x80_MiniFASNetV2.pth
-rw-rw-r-- 1 user user 1.8M ... 4_0_0_80x80_MiniFASNetV1SE.pth
```

### 4.4 Test liveness module
```bash
~/Python-Project/Penv/bin/python3 ~/Python-Project/liveness.py
```
Expected output:
```
[LIVENESS] Stripping 'module.' prefix from MiniFASNetV2 weights
[LIVENESS] Loaded: MiniFASNetV2
[LIVENESS] Stripping 'module.' prefix from MiniFASNetV1SE weights
[LIVENESS] Loaded: MiniFASNetV1SE
[LIVENESS] MiniFASNetV2: real_prob=0.8xx
[LIVENESS] MiniFASNetV1SE: real_prob=0.7xx
[LIVENESS] Average: 0.7xx → REAL
Score: 0.7xx | Real: True
```

---

## 5. Install Source Files

### 5.1 Copy source files
From the cloned `faceauth-linux` directory:
```bash
INSTALL_DIR=~/Python-Project
USERNAME=$(whoami)

cp src/liveness.py     $INSTALL_DIR/liveness.py
cp src/register.py     $INSTALL_DIR/register.py
cp src/auth.py         $INSTALL_DIR/auth.py
cp src/auth_gui.py     $INSTALL_DIR/auth_gui.py
cp src/gdm_auth_gui.py $INSTALL_DIR/gdm_auth_gui.py
```

### 5.2 Update paths to match your username
```bash
USERNAME=$(whoami)
SFAS_DIR=~/Python-Project/sfas

sed -i "s|/home/mach1|/home/$USERNAME|g" $INSTALL_DIR/*.py
sed -i "s|SFAS_DIR.*=.*\".*sfas\"|SFAS_DIR   = \"$SFAS_DIR\"|g" \
    $INSTALL_DIR/liveness.py
```

### 5.3 Install wrapper script
```bash
sudo cp scripts/face-login /usr/local/bin/face-login
sudo sed -i "s|/home/mach1|/home/$USERNAME|g" /usr/local/bin/face-login
sudo sed -i "s|/run/user/1000|/run/user/$(id -u)|g" /usr/local/bin/face-login
sudo chmod +x /usr/local/bin/face-login
```

### 5.4 Set up log files
```bash
sudo touch /tmp/face_auth.log /tmp/gdm_face_auth.log
sudo chmod 666 /tmp/face_auth.log /tmp/gdm_face_auth.log
```

---

## 6. Register Your Face

```bash
QT_QPA_PLATFORM=wayland \
~/Python-Project/Penv/bin/python3 \
~/Python-Project/register.py
```

A camera window will open. Look at the camera and press `Space` or `Enter` to capture.

**Verify registration:**
```bash
ls -lh ~/Python-Project/Face_data/encoding.npy
```
File must exist and be non-zero.

**Tips for good registration:**
- Sit in normal lighting — avoid backlighting
- Look directly at the camera
- Be at a comfortable distance (50–80cm)
- Capture in the lighting conditions where you normally use your computer

---

## 7. Configure sudo PAM

> **WARNING:** Always keep a terminal open while editing PAM files. A broken PAM config can lock you out of sudo.

### 7.1 Back up current sudo PAM
```bash
sudo cp /etc/pam.d/sudo /etc/pam.d/sudo.backup.$(date +%Y%m%d%H%M%S)
```

### 7.2 Add face auth line
```bash
USERNAME=$(whoami)
FACE_LINE="auth    optional    pam_exec.so /home/$USERNAME/Python-Project/Penv/bin/python3 /home/$USERNAME/Python-Project/auth.py"

# Add after first line of sudo PAM
sudo sed -i "1a $FACE_LINE" /etc/pam.d/sudo
```

### 7.3 Verify sudo PAM
```bash
cat /etc/pam.d/sudo
```
The face auth line must appear near the top.

### 7.4 Test in a new terminal
```bash
# Open a NEW terminal and test — do not close existing terminal
sudo ls
```
The face auth GUI should appear. If it works, you're done.

**If sudo breaks — restore from your existing open terminal:**
```bash
# Note: do NOT use sudo here, use tee instead
cat /etc/pam.d/sudo.backup.* | tee /etc/pam.d/sudo
```

---

## 8. Configure GDM Login Screen

> **WARNING:** This modifies the GDM login PAM. If it breaks, boot to recovery mode and restore the backup.

### 8.1 Back up GDM PAM
```bash
sudo cp /etc/pam.d/gdm-password \
        /etc/pam.d/gdm-password.backup.$(date +%Y%m%d%H%M%S)
```

### 8.2 Install new GDM PAM config
```bash
USERNAME=$(whoami)
sudo cp config/gdm-password /etc/pam.d/gdm-password
sudo sed -i "s|/home/mach1|/home/$USERNAME|g" /etc/pam.d/gdm-password
```

### 8.3 Test without logging out
Lock your screen first (safer than full logout):
```bash
loginctl lock-session
```
The face auth overlay should appear fullscreen. Look at the camera to unlock.

Press `ESC` at any time to skip face auth and use password instead.

### 8.4 Test full login
```bash
gnome-session-quit --logout
```
After logout, click your username. Face auth overlay should appear before the password field.

### 8.5 Emergency recovery
If GDM login breaks, boot to recovery mode:
1. Hold `Shift` during boot to open GRUB
2. Select **Advanced options** → **Recovery mode**
3. Select **root** shell
4. Run:
```bash
mount -o remount,rw /
cp /etc/pam.d/gdm-password.backup.* /etc/pam.d/gdm-password
reboot
```

---

## 9. Testing

Run each test in order. Do not proceed to the next test if the current one fails.

### 9.1 Camera test
```bash
~/Python-Project/Penv/bin/python3 -c "
import cv2, time
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
time.sleep(2)
for _ in range(5): cap.read()
ret, frame = cap.read()
cap.release()
print('Camera OK:', ret, frame.shape)
cv2.imwrite('/tmp/test_camera.jpg', frame)
print('Image saved to /tmp/test_camera.jpg')
"
```
Open `/tmp/test_camera.jpg` — should show your face.

### 9.2 Liveness test
```bash
~/Python-Project/Penv/bin/python3 ~/Python-Project/liveness.py
```
Score must be above 0.6 and show `REAL`.

### 9.3 Registration test
```bash
ls -lh ~/Python-Project/Face_data/encoding.npy
```
File must exist.

### 9.4 Auth GUI test
```bash
QT_QPA_PLATFORM=wayland \
~/Python-Project/Penv/bin/python3 \
~/Python-Project/auth_gui.py
```
GUI popup must open, show camera feed, and grant access when you look at it.

### 9.5 sudo test
```bash
# Open a new terminal
sudo ls
```
Face auth popup must appear. After face match, `ls` output must appear without password.

### 9.6 Spoof test
Hold a printed photo or phone screen in front of the camera:
```bash
QT_QPA_PLATFORM=wayland \
~/Python-Project/Penv/bin/python3 \
~/Python-Project/auth_gui.py
```
Must show `SPOOF DETECTED` and deny access.

### 9.7 Lock screen test
```bash
loginctl lock-session
```
Face auth overlay must appear. Look at camera to unlock. Press ESC to use password.

---

## 10. Uninstall

### 10.1 Restore sudo PAM
```bash
sudo cp /etc/pam.d/sudo.backup.* /etc/pam.d/sudo
```

### 10.2 Restore GDM PAM
```bash
sudo cp /etc/pam.d/gdm-password.backup.* /etc/pam.d/gdm-password
```

### 10.3 Remove wrapper script
```bash
sudo rm /usr/local/bin/face-login
```

### 10.4 Remove project files (optional)
```bash
rm -rf ~/Python-Project/liveness.py \
       ~/Python-Project/register.py \
       ~/Python-Project/auth.py \
       ~/Python-Project/auth_gui.py \
       ~/Python-Project/gdm_auth_gui.py \
       ~/Python-Project/Face_data/encoding.npy \
       ~/Python-Project/models/*.pth
```

---

## 11. Troubleshooting

### Black camera feed in GUI
OpenCV's `imshow` is broken on Wayland. The PyQt5-based GUI should not have this issue. If you see a black feed in the PyQt5 window:
```bash
# Test camera directly
~/Python-Project/Penv/bin/python3 -c "
import cv2, time
cap = cv2.VideoCapture(0, cv2.CAP_V4L2)
cap.set(cv2.CAP_PROP_FOURCC, cv2.VideoWriter_fourcc(*'MJPG'))
time.sleep(2)
for _ in range(10): cap.read()
ret, f = cap.read()
cap.release()
print('brightness:', f.mean() if ret else 'FAILED')
"
```
If brightness is above 50, the camera works and the issue is warm-up frames. Already handled in source.

### Qt platform plugin error
```bash
# Remove OpenCV's conflicting Qt plugins
rm -rf ~/Python-Project/Penv/lib/python3.10/site-packages/cv2/qt/plugins
# Always run with:
export QT_QPA_PLATFORM=wayland
```

### Liveness models not loading — `No module named 'src'`
The `sfas` directory path in `liveness.py` is wrong:
```bash
grep "SFAS_DIR" ~/Python-Project/liveness.py
# Should point to ~/Python-Project/sfas
```
Fix:
```bash
sed -i "s|SFAS_DIR.*=.*|SFAS_DIR   = \"$HOME/Python-Project/sfas\"|g" \
    ~/Python-Project/liveness.py
```

### Permission denied on log file
```bash
sudo chmod 666 /tmp/face_auth.log /tmp/gdm_face_auth.log
```

### Face not detected
- Improve lighting — avoid backlight
- Move closer to camera (50–80cm)
- Increase `min_detection_confidence` threshold from 0.6 to 0.5 in source files

### Too many false rejections (low match ratio)
Lower the threshold slightly in `auth_gui.py` and `gdm_auth_gui.py`:
```python
THRESHOLD   = 0.65   # increase from 0.6 (more lenient)
GRANT_RATIO = 0.6    # decrease from 0.7 (fewer frames needed)
```

### GDM face auth not appearing after login
Check PAM config:
```bash
grep face /etc/pam.d/gdm-password
sudo /usr/local/bin/face-login
cat /tmp/gdm_face_auth.log
```

### Wayland socket not found for GDM
Check the runtime directory:
```bash
ls /run/user/$(id -u)/wayland-*
```
Update `/usr/local/bin/face-login` if UID is not 1000:
```bash
sudo sed -i "s|/run/user/1000|/run/user/$(id -u)|g" /usr/local/bin/face-login
```
