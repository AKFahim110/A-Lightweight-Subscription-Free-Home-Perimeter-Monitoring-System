================================================================================
PerimeterGuard
A Lightweight, Subscription-Free Home Perimeter Monitoring System
================================================================================

Project Type : Semester Team Project
Course       : GS14805004 - Cloud Computing and Edge Intelligence Principles
               and Applications
Institution  : Southwest Jiaotong University

--------------------------------------------------------------------------------
1. OVERVIEW
--------------------------------------------------------------------------------

PerimeterGuard is a self-contained, GPU-accelerated edge-AI surveillance system
for residential and small-scale commercial perimeter monitoring. It performs
continuous person detection and zone-based intrusion alerting without relying on
cloud subscriptions, third-party NVR platforms, or proprietary home-automation
hubs.

The system combines:

  - YOLO11s for real-time person detection
  - ByteTrack for persistent multi-object tracking
  - Polygon-based Regions of Interest (ROIs) for zone-aware alerting
  - FastAPI for REST control and MJPEG streaming
  - SQLite for local event logging
  - Telegram Bot API for push notifications
  - Docker with NVIDIA GPU passthrough for reproducible deployment
  - TensorRT for optimized inference on consumer NVIDIA GPUs

All video processing, inference, logging, and alert generation happen locally.
Video frames never leave the edge device.

--------------------------------------------------------------------------------
2. KEY FEATURES
--------------------------------------------------------------------------------

- Continuous zone-aware detection
  Unlike motion-gated NVRs, the system performs continuous inference and
  evaluates detections against user-defined polygon ROIs.

- State-of-the-art detection and tracking
  YOLO11s detects persons; ByteTrack maintains track identities across frames,
  including during brief occlusion or missed detections.

- TensorRT acceleration
  Direct TensorRT inference achieves a 2.37x throughput improvement over native
  PyTorch on an NVIDIA RTX 4060 Laptop GPU.

- Non-blocking Telegram alerts
  Alerts and snapshots are dispatched through a bounded background queue, so
  network latency never blocks the inference pipeline.

- Browser-native dashboard
  Any device on the local network can view live MJPEG streams, edit ROI
  polygons, and inspect intrusion logs from a modern web browser.

- Single-container deployment
  The application runs as a Docker container with GPU passthrough. No manual
  CUDA/cuDNN installation is required on the host beyond the NVIDIA Container
  Toolkit.

- Local data sovereignty
  Snapshots, video clips, and SQLite records are stored locally. No cloud
  dependency is required for normal operation.

--------------------------------------------------------------------------------
3. SYSTEM ARCHITECTURE
--------------------------------------------------------------------------------

The system uses a four-layer architecture:

  1. Data Acquisition Layer
     - Reads frames from IP cameras using OpenCV with the FFmpeg backend.
     - Maintains a fresh frame buffer without causing camera backpressure.

  2. Inference and Analysis Layer
     - Runs YOLO11s inference and ByteTrack tracking.
     - Evaluates detected person centroids against configured polygon ROIs.
     - Maintains a per-camera intrusion state machine with hysteresis.

  3. Persistence and Notification Layer
     - Writes intrusion events to SQLite.
     - Saves snapshots and annotated video clips.
     - Dispatches Telegram alerts via a background daemon thread.

  4. Presentation Layer
     - Serves MJPEG streams and REST API endpoints through FastAPI/Uvicorn.
     - Provides browser-based dashboard, logs, and ROI configuration pages.

Each camera is handled by one CameraWorker daemon thread. Frame capture,
inference, MJPEG encoding, and video recording are time-gated independently to
avoid blocking and to keep each subsystem at its optimal rate.

For the full architecture description, data flow, and experimental results,
see:

  reports/Architecture_Design_Report.pdf

--------------------------------------------------------------------------------
4. PERFORMANCE SUMMARY
--------------------------------------------------------------------------------

Benchmark environment:

  CPU  : AMD Ryzen 7 7840H (8C/16T, Zen 4)
  RAM  : 32 GB DDR5
  GPU  : NVIDIA GeForce RTX 4060 Laptop GPU, 8 GB GDDR6
  CUDA : 12.8
  Model: YOLO11s, FP16, imgsz=640, confidence threshold 0.25
  Input: 1280x720 video source, 300 measured frames

Inference backend benchmark:

  Backend                     Mean Pipeline Latency    Estimated FPS
  ---------------------------------------------------------------------
  PyTorch + Ultralytics       18.06 ms                 55
  TensorRT + Ultralytics       9.68 ms                103
  Direct TensorRT              7.61 ms                131

Key result:

  Direct TensorRT is approximately 2.37x faster than native PyTorch in
  end-to-end pipeline throughput. The largest gains come from GPU-accelerated
  preprocessing and TensorRT layer fusion/kernel auto-tuning.

Multi-stream resource usage:

  - Two simultaneous 720p streams consume approximately 1.1 GB total VRAM.
  - CPU utilization remains below 15% on the Ryzen 7 7840H.
  - MJPEG fan-out supports multiple concurrent browser clients without
    per-client re-encoding.

--------------------------------------------------------------------------------
5. TESTED DEPLOYMENT SCENARIOS
--------------------------------------------------------------------------------

The system was validated across four scenarios with only ROI and camera-source
changes; the core inference stack was not retrained.

  1. Backyard perimeter monitoring
     - Wide-angle camera covering a backyard fence line.
     - Sidewalk pedestrians ignored; fence-line intrusion detected.

  2. Main gate / entry point monitoring
     - Narrow trapezoid ROI aligned with the gate opening.
     - Persistent track IDs kept a single person as one event even when they
       briefly left and re-entered the ROI.

  3. Indoor doorway monitoring
     - ROI covering the door threshold.
     - No false positives from furniture or swinging doors at default
       confidence threshold.

  4. Toddler / children monitoring (repurposed use case)
     - ROI around a balcony boundary.
     - Inverted alert logic detected "extrusion" rather than intrusion.

--------------------------------------------------------------------------------
6. REPOSITORY STRUCTURE
--------------------------------------------------------------------------------

PerimeterGuard/
|
+-- app/
|   +-- main.py               FastAPI entry point and API routes
|   +-- worker.py             Per-camera worker thread logic
|   +-- inference.py          YOLO / ByteTrack / TensorRT wrappers
|   +-- database.py           SQLite operations
|   +-- notifier.py           Telegram background notification thread
|
+-- config/
|   +-- config.yaml           Global and per-camera configuration
|
+-- models/
|   +-- yolo11s.pt            Default PyTorch weights
|   +-- yolo11s.engine        Optional TensorRT engine
|
+-- reports/
|   +-- Architecture_Design_Report.pdf
|
+-- docker-compose.yml
+-- Dockerfile
+-- requirements.txt
+-- README.txt                This file

Note: File and folder names may differ slightly depending on the final uploaded
code structure. Update this section if your repository layout changes.

--------------------------------------------------------------------------------
7. PREREQUISITES
--------------------------------------------------------------------------------

Hardware:

  - NVIDIA GPU recommended (GTX 10-series or newer; RTX 4060 tested).
  - IP cameras or local video sources accessible via RTSP/HTTP/USB.

Software:

  - Linux host recommended (Ubuntu 22.04 tested).
  - Docker Engine
  - Docker Compose
  - NVIDIA Container Toolkit
  - NVIDIA driver compatible with CUDA 12.8

Optional:

  - Telegram account and bot token for push notifications.
  - NVIDIA Jetson module for embedded deployment.

--------------------------------------------------------------------------------
8. QUICK START
--------------------------------------------------------------------------------

Step 1: Clone the repository

  git clone https://github.com/<your-username>/PerimeterGuard.git
  cd PerimeterGuard

Step 2: Prepare configuration

  Copy the example configuration file:

    cp config/config.yaml.example config/config.yaml

  Edit config/config.yaml to set:

    - Camera RTSP/HTTP sources
    - ROI polygon files
    - Telegram bot token and chat ID (optional)
    - Model backend: pytorch_yolo11s or tensorrt_yolo11s
    - Inference FPS, stream FPS, and recording FPS

Step 3: Build and start the container

  docker compose up --build -d

Step 4: Open the dashboard

  In a browser on the same LAN:

    http://<SERVER_IP>:8000/dashboard

  Other useful pages:

    http://<SERVER_IP>:8000/logs
    http://<SERVER_IP>:8000/configs

Step 5: Stop the system

  docker compose down

--------------------------------------------------------------------------------
9. CONFIGURATION NOTES
--------------------------------------------------------------------------------

The main configuration file controls:

  - Application device selection (CPU / CUDA)
  - Model backend and weights path
  - Per-camera source URL
  - ROI polygon YAML path
  - process_fps: inference rate ceiling
  - stream_fps: MJPEG publish rate
  - record_fps: alert video recording rate
  - patience_frames: hysteresis before ending an intrusion event
  - Telegram bot token and chat ID

ROI polygons are stored as YAML files containing a list of [x, y] pixel
coordinates. The browser-based ROI editor can create and update these files
through the /api/configs/roi endpoint.

Example ROI YAML structure:

  roi_polygon:
    - [120, 200]
    - [640, 180]
    - [700, 500]
    - [100, 520]

--------------------------------------------------------------------------------
10. API OVERVIEW
--------------------------------------------------------------------------------

The FastAPI server exposes the following main endpoints:

  GET   /dashboard
        Serves the main control dashboard HTML.

  GET   /logs
        Serves the intrusion history HTML page.

  GET   /configs
        Serves the camera and ROI configuration page.

  GET   /stream/{cam_id}
        Returns an MJPEG stream as multipart/x-mixed-replace.

  GET   /snapshot/{cam_id}
        Returns a single annotated JPEG snapshot.

  PATCH /api/cameras/{cam_id}
        Updates camera parameters such as ROI, FPS, or enabled state.

  POST  /api/configs/roi
        Saves an ROI polygon configuration.

  GET   /api/logs
        Returns intrusion records as JSON.

  GET   /api/media/{path}
        Serves recorded snapshots and video clips.

All endpoints are intended for use within a trusted local network. The current
prototype does not include authentication or TLS.

--------------------------------------------------------------------------------
11. LIMITATIONS
--------------------------------------------------------------------------------

- No authentication layer
  The service binds to 0.0.0.0 without API keys or TLS. Use only on a trusted
  LAN. For remote access, use a VPN such as WireGuard or Tailscale, or place
  the service behind an authenticated reverse proxy.

- Single-class detection
  Only the "person" class is used for alerting. Vehicles, animals, and package
  deliveries are not detected by the alert logic.

- Static ROI polygons
  ROIs are defined at configuration time and do not change dynamically based
  on time of day.

- No cross-camera re-identification
  Each camera runs an independent ByteTrack instance. A person moving between
  cameras receives a new track ID.

- LAN-only scalability
  Remote access beyond the local network requires port forwarding or a VPN.
  WebRTC/STUN/TURN support is not included.

--------------------------------------------------------------------------------
12. FUTURE WORK
--------------------------------------------------------------------------------

- Add authentication, API key enforcement, and TLS.
- Add time-based dynamic ROI scheduling.
- Extend to multi-class detection (vehicles, animals, packages).
- Add support for Google Coral TPU, Intel OpenVINO, and Hailo accelerators.
- Add motion-adaptive inference scheduling to reduce GPU load during idle
  periods.
- Add cross-camera re-identification using appearance embeddings.

--------------------------------------------------------------------------------
13. AUTHORS
--------------------------------------------------------------------------------

  [Your Name]        - Adalat Khan Fahim
  [Teammate Name]    - AI Rifat

Course project for GS14805004: Cloud Computing and Edge Intelligence Principles
and Applications, Southwest Jiaotong University, 2025-2026.

