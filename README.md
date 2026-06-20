# Facial Recognition Attendance System

A Offline-based desktop application for **offline attendance tracking using facial recognition**. The system detects faces via a pre-trained DNN and recognizes individuals using LBPH histograms. All data is stored locally in SQLite, attendance is exportable to CSV, and the app provides real-time feedback through a clean GUI.

>  **Prototype Deployment** — This system is currently being piloted by **two small businesses in Kalibo, Aklan** as a proof-of-concept attendance tracking solution.

![Python](https://img.shields.io/badge/Python-3.8+-blue)
![OpenCV](https://img.shields.io/badge/OpenCV-4.5+-green)
![Tkinter](https://img.shields.io/badge/Tkinter-built--in-orange)
![SQLite](https://img.shields.io/badge/SQLite-embedded-lightgrey)
![Status](https://img.shields.io/badge/status-working-brightgreen)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Package Structure](#package-structure)
- [Core Design Patterns](#core-design-patterns)
- [Key Features](#key-features)
- [Workflow & Lifecycle](#workflow--lifecycle)
- [Code Highlights](#code-highlights)
- [Setup & Installation](#setup--installation)
- [Troubleshooting](#troubleshooting)
- [Data Management](#data-management)
- [Acknowledgements](#acknowledgements)
- [License](#license)

---

## Architecture Overview

The application follows a modular, layered design with clear separation of concerns:

```
GUI Layer (Tkinter, ttk)
        |
Business Logic (AttendanceApp, Registration, Session Management)
        |
Face Manager (DNN detection + LBPH recognition)
        |
Data Layer (SQLite, CSV export)
        |
Image Storage / Model File
```

- **GUI Layer** — Built with Tkinter; provides three tabs: Registration, Attendance, and About.
- **Face Manager** — Encapsulates all OpenCV operations: face detection using an SSD-based DNN and recognition using LBPH.
- **Database Layer** — SQLite stores persons, sessions, and attendance logs. All queries are abstracted in `database.py`.
- **File Storage** — Face crops are saved as JPEGs under `images/<person_id>/`; the LBPH model is persisted as `face_model.yml`; attendance sessions are exported as CSV files.
- **Background Threading** — Registration training runs in a separate thread to keep the interface responsive.

---

## Package Structure

```
project_root/
├── app/
│   ├── __init__.py
│   ├── attendance_app.py    # Main application class (GUI + logic)
│   └── about_app.py         # About tab content
│
├── face/
│   ├── __init__.py
│   └── face_manager.py      # DNN detection + LBPH recognition + training
│
├── data/
│   ├── __init__.py
│   └── database.py          # SQLite CRUD operations
│
├── models/                  # Pre-trained DNN files (must be downloaded)
│   └── dnn/
│       ├── deploy.prototxt
│       └── res10_300x300_ssd_iter_140000_fp16.caffemodel
│
├── images/                  # Stored face crops (auto-created)
├── exports/                 # CSV exports (auto-created)
├── main.py                  # Application entry point
├── wipe_db_script.py        # Utility to reset all data
└── requirements.txt
```

---

## Core Design Patterns

### FaceManager – Encapsulation of Computer Vision Logic

`FaceManager` centralises all face-related operations: loading the DNN network, detecting faces in a frame, extracting and resizing face ROIs, performing LBPH recognition, and training the model on newly registered persons. It manages the persistence of the LBPH model and loads it automatically at startup.

```python
class FaceManager:
    def __init__(self):
        self.face_net = cv2.dnn.readNetFromCaffe(...)
        self.recognizer = cv2.face.LBPHFaceRecognizer_create()
        self.load_model()
```

### Database Abstraction

All SQLite operations are wrapped in simple functions in `database.py` (e.g., `add_person`, `log_attendance`, `get_all_sessions`). The module handles connection management using context managers, ensuring transactions are properly committed or rolled back.

### Threading for Responsive Registration

Registration involves capturing multiple face crops (~4 seconds) and then training the LBPH model. Training runs in a background thread while a timer updates the UI with elapsed time.

```python
thread = threading.Thread(target=self._do_registration, daemon=True)
thread.start()
self._update_processing_timer()   # updates status label every 500ms
```

### Polymorphism – Single Display Method

A single `display_frame()` method handles video feeds across both tabs, with configurable target dimensions.

---

## Key Features

- **Face Registration** — Capture multiple face images while the user moves their head; train the LBPH model incrementally.
- **Attendance Sessions** — Start a named session; recognised persons are automatically logged once per session.
- **Real-time Feedback** — Displays detected faces with names/status on the video feed; a side panel logs every event.
- **CSV Export** — Export any past session to a CSV file with full timestamps.
- **Offline Operation** — No external network calls; all data stays on the local machine.
- **Reset Functionality** — Wipe all persons, sessions, images, and the trained model with a single click.

---

## Workflow & Lifecycle

### Registration Flow

1. User enters a name and clicks **Register Person**.
2. The camera starts a 4-second capture phase; the user slowly moves their head while the system collects face crops (every 0.25 s).
3. A progress bar and countdown are overlaid on the video feed.
4. If at least 10 faces are captured, training runs in a background thread.
5. On completion, the model is updated and the new person is stored in the database.

### Attendance Session Flow

1. User enters a session name and clicks **Start Attendance**.
2. A session record is created in SQLite and an empty CSV is initialised.
3. For each frame, faces are detected and recognised.
4. If a recognised person is not yet logged in the session, their name and timestamp are appended to the CSV and the database.
5. The video feedback shows:
   - 🟢 **Green** — "Registered in session"
   - 🟡 **Yellow** — "Already registered"
   - 🔴 **Red** — Unknown face
6. Click **Stop Attendance** to end the session; the CSV is finalised with start/end times.

---

## Code Highlights

### Face Detection with DNN

The DNN expects a 300×300 blob; detections are filtered by a confidence threshold and bounding boxes are scaled back to the original frame dimensions.

```python
blob = cv2.dnn.blobFromImage(cv2.resize(frame, (300, 300)), 1.0, (300, 300), (104.0, 177.0, 123.0))
self.face_net.setInput(blob)
detections = self.face_net.forward()
```

### LBPH Recognition and Confidence

LBPH returns a label and confidence score (lower = better). If confidence is below the threshold (default `70`), the person is considered recognised.

```python
label, confidence = self.recognizer.predict(face_roi)
if confidence < CONFIDENCE_THRESHOLD:
    name = get_person_name_by_id(label)
```

### Overlay Feedback During Registration

A semi-transparent overlay displays the app name, status message, countdown, captured face count, and a progress bar.

```python
overlay = frame.copy()
cv2.rectangle(overlay, (10, 10), (w - 10, 190), (0, 0, 0), -1)
frame = cv2.addWeighted(overlay, 0.45, frame, 0.55, 0)
```

### CSV Export with Metadata

Each export writes session metadata followed by a header row and all attendance records.

```python
writer.writerow(["Session ID", session_id])
writer.writerow(["Session Name", session_name])
writer.writerow(["Start Time", start_time])
writer.writerow(["End Time", end_time])
writer.writerow([])
writer.writerow(["Name", "Timestamp"])
writer.writerows(records)
```

### Thread-Safe Status Updates

UI updates from the registration background thread are scheduled via `root.after()` to keep Tkinter responsive.

```python
def _update_processing_timer(self):
    if not self._reg_timer_running:
        return
    elapsed = time.time() - self._reg_start_time
    self.set_reg_status(f"Processing registration... {elapsed:.1f}s", fg="blue")
    self.root.after(500, self._update_processing_timer)
```

---

## Setup & Installation

### Prerequisites

- Python 3.8 or higher
- A working webcam

### 1. Clone the repository

```bash
git clone <repository-url>
cd facial-recognition-attendance
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

**`requirements.txt`:**

```
opencv-python>=4.5
opencv-contrib-python>=4.5   # required for LBPH
Pillow>=9.0
```

### 3. Download the DNN model files

The application uses the **res10_300x300_ssd_iter_140000_fp16** model. Download the following two files and place them in `models/dnn/`:

- [`deploy.prototxt`](https://github.com/opencv/opencv/blob/master/samples/dnn/face_detector/deploy.prototxt)
- [`res10_300x300_ssd_iter_140000_fp16.caffemodel`](https://github.com/opencv/opencv_3rdparty/raw/dnn_samples_face_detector_20170830/res10_300x300_ssd_iter_140000_fp16.caffemodel)

### 4. Run the application

```bash
python main.py
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| **Camera not opening** | Check that your webcam is connected and not in use by another application. |
| **"No face crops captured" during registration** | Ensure good lighting and that your face is clearly visible. Move your head slowly side-to-side. |
| **Recognition is poor** | Re-register the person under the same lighting conditions used during attendance. You can also adjust `CONFIDENCE_THRESHOLD` in `face_manager.py`. |
| **Model files missing** | The app will crash if the DNN `.prototxt` or `.caffemodel` are not in the correct location. Download them as described above. |
| **Performance issues** | DNN detection runs on CPU. Consider reducing video resolution (default is 800×600) in `attendance_app.py` if frame rate is too low. |

---

## Data Management

| Resource | Location |
|----------|----------|
| Database | `database.db` — persons, sessions, attendance |
| Face images | `images/<person_id>/` — used for retraining the model |
| Trained model | `models/face_model.yml` — LBPH recognizer state |
| CSV exports | `exports/` — one CSV file per session |

To reset everything completely and start fresh:

```bash
python wipe_db_script.py
```

This deletes the database, the `images/` and `exports/` folders, and the model file.

---

## Acknowledgements

- [OpenCV](https://opencv.org/) — face detection and recognition
- [Tkinter](https://docs.python.org/3/library/tkinter.html) — GUI framework
- [SQLite](https://www.sqlite.org/) — lightweight embedded storage

---

## License

This project is provided for **educational and non-commercial use**.
