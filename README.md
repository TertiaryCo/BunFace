# BunFace - Face Authentication System for Ubuntu

## 🔐 Overview

**BunFace** is a real-time face authentication system for Ubuntu and Debian-based Linux systems. It enables password-less authentication using facial recognition, similar to Windows Hello.

The system uses a stored face embedding and compares it with a live camera feed to grant or deny access.

---

## ⚙️ How It Works

### 🧾 Registration Phase

* User captures their face using the webcam
* System extracts facial features (embedding)
* Embedding is stored locally as a `.npy` file

### 🔍 Authentication Phase

* Camera scans for a face in real-time
* Captured face is converted into embedding
* Compared with stored embedding using Euclidean distance
* If distance < threshold → ✅ Access granted
* Else → ❌ Access denied

---

## 📂 Files Description

### 1. Authentication Script

* File: `auth1.py`
* Purpose: Verifies user identity

Key Features:

* Loads stored face embedding
* Captures live face within 5 seconds
* Uses distance threshold (`0.6`) for matching
* Logs activity to `/tmp/face_auth.log`
* Returns exit codes (important for PAM integration):

  * `0` → Success
  * `1` → Failure

👉 See code: 

---

### 2. Registration Script

* File: `register1.py`
* Purpose: Registers user face

Key Features:

* Opens live camera dashboard
* Manual capture (`press 'c'`)
* **Automatic capture after 30 seconds if no input is given**
* Detects face and extracts embedding
* Saves embedding to:

  ```
  /home/mach1/Python-Project/Face_data/encoding.npy
  ```

👉 See code: 

---

## 🛠️ Tech Stack

* **Python 3**
* **OpenCV (cv2)** – Camera handling
* **face_recognition** – Face encoding & detection
* **NumPy** – Numerical operations
* **V4L2 backend** – Linux camera support

---

## 📦 Installation

### 1. Install Dependencies

```bash
sudo apt update
sudo apt install python3 python3-pip cmake libx11-dev
pip3 install opencv-python face_recognition numpy
```

### 2. Project Setup

```bash
mkdir -p /home/mach1/Python-Project/Face_data
```

---

## 🚀 Usage

### 🔹 Step 1: Register Face

```bash
python3 register1.py
```

* Press **`c`** to capture manually
* OR wait **30 seconds for automatic capture**

---

### 🔹 Step 2: Authenticate

```bash
python3 auth1.py
```

* System scans for face (5 seconds window)
* Grants or denies access

---

## 📁 Directory Structure

```
BunFace/
│── auth1.py              # Authentication script
│── register1.py          # Face registration script
│── Face_data/
│   └── encoding.npy      # Stored face embedding
```

---

## 🧠 Core Logic

### Face Matching Formula

```
distance = ||known_embedding - current_embedding||
```

* If `distance < 0.6` → Match
* Else → No match

---

## 📜 Logging

All authentication events are logged in:

```
/tmp/face_auth.log
```

Example logs:

```
AUTH START
Embedding loaded
Opening camera...
Face detected
Distance: 0.43
AUTH SUCCESS
```

---

## ⚠️ Limitations

* Works best in good lighting conditions
* Single-user system (one embedding)
* No liveness detection (can be spoofed with images)

---

## 🔮 Future Improvements

* Multi-user support
* Anti-spoofing (liveness detection)
* GUI interface
* PAM integration for full Linux login
* Deep learning-based recognition

---

## 🤝 Contributing

Pull requests are welcome. For major changes, open an issue first.

---

## ⭐ Support

If you like this project, give it a ⭐ on GitHub!

# BunFace - Face Detection Security System for Ubuntu

A real-time face authentication security system like Windows Hello for Ubuntu
(and Debian-based) Linux.  Built on OpenCV, Python 3, and systemd.



