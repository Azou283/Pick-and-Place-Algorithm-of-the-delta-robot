# ♻️ Automated Waste Sorting System

A pick-and-place robot that detects and sorts waste in real time using computer vision and a delta robot arm.

---

## 🧠 How It Works

A camera streams live video of a conveyor belt to a **Raspberry Pi 5**, which runs a **YOLOv8n** model to detect and classify waste objects. The Pi then sends pick/drop coordinates over **serial (UART)** to an **Arduino**, which drives a **delta robot** to physically sort the item into the correct bin.

```
Camera → Raspberry Pi (YOLO detection) → Arduino (Inverse Kinematics) → Delta Robot
```

---

## 🔧 Hardware

| Component | Role |
|---|---|
| Raspberry Pi 5 | Computer vision + coordination |
| Arduino | Motor control + robot sequencing |
| USB/CSI Camera | Live video feed above conveyor belt |
| Delta Robot (3× NEMA 23 steppers) | Pick-and-place arm |
| Gripper | Object grasping via digital signal |

---

## 🗂️ System Pipeline

### Step 1 — Object Detection (Raspberry Pi)

- A YOLOv8n model processes each camera frame continuously.
- For each detection, it returns:
  - **Class label** — `plastic`, `metal`, `paper`, `glass`, or `cardboard`
  - **Confidence score** — detections below **60%** are discarded
  - **Bounding box center** in pixel coordinates

### Step 2 — Pixel → Real-World Coordinate Conversion

- A **homography matrix H** (3×3) is computed at the start of each session using a calibration grid placed on the belt.
- OpenCV's `findHomography()` accounts for lens distortion and perspective.
- Any detected pixel coordinate `(u, v)` is converted to millimeter coordinates `(X, Y)` via a single matrix multiplication.
- `Z` is fixed — the belt surface is flat at a known constant height.

### Step 3 — Serial Command (Pi → Arduino)

The Pi sends a compact UART message:

```
X,Y,Z,CLASS\n
```

**Example:**
```
120.5,45.2,-150.0,plastic
```

- The Pi waits up to **5 seconds** for a `DONE` acknowledgment.
- If no reply arrives, a warning is logged and the loop continues.

### Step 4 — Delta Robot Motion (Arduino)

1. Parses the serial message into `X`, `Y`, `Z`, and `CLASS`.
2. Looks up the **drop bin position** for the given waste class.
3. Runs **analytical inverse kinematics** to convert `(X, Y, Z)` → motor angles `(θ₁, θ₂, θ₃)`.
4. Drives all three NEMA 23 steppers **simultaneously** via `AccelStepper`.
5. If the target is outside the reachable workspace, the move is skipped and an error is sent back to the Pi.

**Pick-and-place sequence:**
```
Rise to safe height → Descend to object → Grip → Rise → 
Travel over bin → Descend → Release → Return home
```

### Step 5 — Loop

- On completion, the Arduino sends `DONE` back to the Pi.
- The Pi unblocks and processes the next frame.
- The cycle repeats continuously.

---

## 📦 Dependencies

### Raspberry Pi
```bash
pip install ultralytics opencv-python pyserial
```

### Arduino
- [AccelStepper](https://www.airspayce.com/mikem/arduino/AccelStepper/) library

---

## ⚙️ Configuration

Key parameters are defined in one place for easy tuning:

| Parameter | Description |
|---|---|
| `SERIAL_PORT` | Port used for Pi ↔ Arduino communication |
| `CONFIDENCE_THRESHOLD` | Minimum detection confidence (default: `0.60`) |
| `CAMERA_RESOLUTION` | Input frame size |
| `WORKSPACE_DIMENSIONS` | Physical bounds of the robot's reachable area |

---

## 🚀 Startup Sequence

1. Arduino initializes steppers, moves to home position, and broadcasts `READY`.
2. Pi loads the YOLO model, opens the camera, and establishes the serial connection.
3. Calibration grid is placed on the belt → homography matrix is computed.
4. Main detection loop begins.

---

## 📁 Project Structure

```
├── pi/
│   ├── main.py              # Main detection and coordination loop
│   ├── calibration.py       # Homography calibration routine
│   └── model/               # YOLOv8s weights
├── arduino/
│   ├── robot_controller.ino # Serial parsing, IK, motor control
│   └── delta_kinematics.h   # Inverse kinematics geometry
└── README.md
```
