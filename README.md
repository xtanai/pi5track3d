# Pi5Track3D

**RAW10 mono ingest and on‑edge preprocessing on Raspberry Pi 5** with synchronized multi‑camera capture for real‑time 3D hand/gesture tracking. Pi5Track3D handles **camera I/O, undistortion/normalization, optional on‑Pi keypoints**, and publishes **ROI‑reduced point clouds / keypoint tracks** to a host over **Gigabit LAN**. Designed to pair with **TDMStrobe** (IR throw/fill, A/B/C/D phases) for deterministic lighting.

> **Status:** early prototype. APIs and wiring may change.

---

## Why Pi5Track3D?

* **Deterministic capture**: synchronized global‑shutter sensors, TDM strobe phases
* **Low latency**: on‑Pi preproc (undistort, normalize, crop, optional 2D keypoints)
* **Bandwidth‑friendly**: transmit only **ROI point clouds / keypoints**, not raw video
* **Modular**: 2× cameras per Pi (stereo). Scale to **3–4 stereos** via LAN
* **Marker‑optional**: gloveless by default; **wristband** improves stability; **fingertip markers** reach best precision

---

## System Overview

```
[ TDMStrobe (RP2040) ] ── TRIG A/B ─► [ Pi5Track3D ] ── LAN ─► [ Pi5Fusion (Host PC) ] ──► MotionCoder
                                 │               │                         │
                            2× MIPI‑CSI      ROI point clouds /        Multi‑stereo fusion,
                         (global‑shutter)    keypoints + refs          calibration, 3D key‑pose
```

* **Pi 5** ingests **RAW10 mono** from **2× MIPI‑CSI** (e.g., OV9281), performs **undistort/normalize**, optional **2D keypoints**, and sends **ROI‑compressed data** to the host.
* **TDMStrobe** provides **A/B (optional C/D)** IR pulses and **2‑wire sync** for shutter/LED timing, ensuring clean Z across views.
* **AprilTag reference boards**: two **removable L‑brackets** around the work area with tags on multiple faces. This stabilizes **Z‑scale** and improves re‑localization.

---

## Features

* **RAW10 mono ingest** (global shutter) with GPU/NEON‑assisted preprocessing
* **On‑edge ops**: undistortion, normalization, ROI cropping, temporal filtering
* **Optional 2D keypoints** on‑Pi; otherwise, forward crops to host for keypointing
* **ROI point clouds** (hand‑only) with embedded **reference points** (AprilTag/wrist/fingertip aids)
* **Hard/soft sync**: EXPOSE/FLASH in from TDMStrobe; phase‑aligned capture
* **LAN publisher**: Zero‑copy UDP/ZeroMQ/TCP (configurable)

---

## Bill of Materials (BOM)

**Core**

* **Raspberry Pi 5 (8 GB RAM)** — recommended; 4 GB is possible, 16 GB is optional
* **Cooling**: Pi 5 **active cooler** (heatsink + fan)
* **Storage**: microSD (≥ 64 GB) *or* NVMe (preferred for logs)

**Cameras (Stereo per Pi)**

* 2× **global‑shutter mono** modules (e.g., OV9281 1280×800 @ up to 120 fps)
* 2× **lenses** matched to FOV (target 60–90° for precision; 120° only for bring‑up)
* **850 nm band‑pass filters** (camera‑safe IR)
* **Rigid stereo mount** with baseline ~200–300 mm (context‑dependent)

**Lighting / Sync**

* **TDMStrobe** controller (RP2040/Pico) with A/B (C/D optional) phases
* **IR emitters**: prototype 120°; production 60° (throw) + 90° (fill)
* 2‑wire **Click‑Stereo** sync cables from TDMStrobe to camera rig

**Power**

* **Central 24 V PSU** sized for **all rigs + lighting** (strings × current × 1.4)
* **Buck 24→5.1 V** for each Pi 5 (≥ 5 A, low ripple). Prefer a **shared 24 V bus** to minimize wall‑warts
* Optional **UPS/hold‑up** for safe shutdown

**Reference & Aids**

* **Two aluminum L‑brackets** (removable) with **AprilTags** on multiple faces
* **Wristband markers** (optional) for arm pose stability
* **Fingertip/nail markers** (optional) for peak precision

**Cabling/Hardware**

* Short MIPI cables, quality LAN (Cat6), strain relief, black matte baffles

---

## Data Flow

1. **Capture** RAW10→DNG‑like buffers; timestamps + phase IDs from TDM
2. **Preprocess**: undistort, normalize, crop to hand ROI
3. **(Optional) 2D keypoints** on‑Pi; otherwise, forward crops to host for keypointing
4. **Publish** over LAN to **Pi5Fusion**: ROI point clouds / keypoints + **reference set** (AprilTag corners, wrist band, fingertip aids)
5. **Pi5Fusion (Host PC)**: multi‑stereo calibration & **fusion** (bundle adjustment / filtering), **precise 3D key‑pose** estimation
6. **Output to MotionCoder**: clean, low‑latency 3D key‑pose stream (joints + confidences)

---

## Synchronization

* **Global shutter** cameras recommended
* **TDM phases**: A/B per frame (C/D optional). Pulses must sit **inside exposure**
* Pi5Track3D listens to **EXPOSE/FLASH** and phase lines; timestamps all frames
* Host enforces **frame‑to‑frame phase consistency** and discards cross‑lit frames

---

## Configuration

* **Pi5Track3D (edge)**: camera IDs, intrinsics/extrinsics, FOV, baseline
* **Lighting**: phase map, nominal pulse widths (Throw/Fill), guard times
* **Networking**: transport (UDP/ZeroMQ/TCP), MTU, topics
* **ROI policy**: detector thresholds, crop sizes, hysteresis
* **Markers**: AprilTag families, board layout, wrist/fingertip options
* **Pi5Fusion (host)**: rig registry (3–4 stereo pairs), inter‑rig extrinsics, fusion weights, outlier rejection, smoothing window, output FPS/format

---

## Quick Start

1. Mount the **L‑brackets** with AprilTags; verify coverage from both cameras
2. Wire **TDMStrobe** (A/B sync) to the stereo rig; set Throw/Fill pulses
3. Power from the **central 24 V PSU**; feed **5.1 V** to the Pi via buck
4. Start Pi5Track3D; load the rig config; check undistortion/normalization
5. Verify **histogram 70–80% FS** at target exposure; adjust strobe pulses
6. Enable **ROI** and (optional) **on‑Pi keypoints**; stream to host

---

## Best Practices

* Prefer **60–90° optics** and **solid baselines (≥ 0.2 m)** for Z precision
* Keep **exposure short** (no blur); use strobe for light, not longer shutter
* Fix **gain** and **gamma=1** (linear); avoid auto‑everything
* Calibrate carefully (Charuco + bundle adjust); use **NIR band‑pass**
* Keep rigs **mechanically stiff**; avoid vibration & thermal drift

---

## Roadmap

* On‑Pi NEON/GPU kernels for faster undistort/normalize
* Lightweight on‑Pi detector for ROI without full keypointing
* **Pi5Fusion**: multi‑rig calibration tool (AprilTag board solve, BA), per‑rig latency compensation, timebase alignment
* Configurable ROS/ZeroMQ bridges; log‑ring buffers
* Calibration helper for L‑brackets (tag layout generator)
* MotionCoder adapter: joint remapping, confidence gating, state machine hooks

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files (rig plates, brackets) may use **CERN‑OHL‑S**.

---

## Safety

* **850 nm IR is invisible and hazardous**—follow **IEC 62471** guidelines.
* **Never look into emitters.** Use black matte **baffles/shields**, aim emitters away from faces, and add **interlocks** (LEDs off on loss of sync / open cover / presence detection).
* Keep **exposure short** (strobe pulses inside the camera exposure) and the **average power low**.
* Use **850 nm band-pass filters** on cameras to reduce required LED power.

**Solution Strategies:**

* **Option A – Side/Rear placement (recommended):**
  Mount stereo pairs **left/right and slightly behind** the workspace, aimed toward the monitor/work area. Add **one or two top stereo pairs** for occlusion-free coverage. This directs IR **away from eyes** while keeping the scene well lit. Future refinement: integrate cameras cleanly by recess-mounting one pair near the center of the table and another near the back edge for a slimmer, more robust design.

* **Option B – Front placement with HMD:**
  If all stereo pairs must face forward, operate with a **closed VR headset** (no see-through optics) so eyes are **occluded**. Still use baffles and interlocks to protect bystanders without headsets.

* **Option C – IR-filtering safety glasses:**
  Use **visible-light-transmitting eyewear** that strongly attenuates **near-IR (≈ 780–950 nm)** (e.g., specified optical density at **850 nm**), so users retain normal vision while IR exposure is reduced.

* **Option D – Side-shield eyewear (“horse blinkers” idea):**
  Provide **IR-blocking safety glasses with side shields** for operators/visitors when emitters face forward. Choose eyewear rated for **near-IR attenuation** and ensure a snug fit to block off-axis light.

### Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.
