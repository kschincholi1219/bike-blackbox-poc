# 2. SYSTEM ARCHITECTURE

## 🏗️ Complete Block Diagram

```
                    ┌──────────────────────────────────────────────────────────────┐
                    │              BIKE BLACKBOX SYSTEM - COMPLETE BLOCK DIAGRAM   │
                    └──────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                          EXTERNAL SENSORS & MODULES                         │
    │                                                                             │
    │  ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
    │  │  Front Cam   │              │  Rear Cam    │              │ GPS Module   │
    │  │  (USB/CSI)   │              │  (USB/CSI)   │              │ (u-blox M8)  │
    │  │              │              │              │              │ (UART TTL)   │
    │  │ 1080p/30fps  │              │ 1080p/30fps  │              │ 1Hz NMEA     │
    │  │ H.264 stream │              │ H.264 stream │              │ ±5m accuracy │
    │  └──────┬───────┘              └──────┬───────┘              └──────┬───────┘
    │         │                             │                             │
    │         │ USB 3.0 (high bandwidth)   │ USB 3.0 (high bandwidth)   │ UART (/dev/ttyUSB0)
    │         │                             │                             │
    └─────────┼─────────────────────────────┼─────────────────────────────┼──────────┘
              │                             │                             │
              │         ┌───────────────────┘                             │
              │         │                                                 │
              ▼         ▼                                                 ▼
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                      COMPUTE CORE (SBC - Single Board Computer)            │
    │                                                                             │
    │  ┌──────────────────────────────────────────────────────────────────────┐  │
    │  │                   OPERATING SYSTEM (Linux)                           │  │
    │  │  - Ubuntu 20.04 LTS / Debian Bullseye                              │  │
    │  │  - Systemd for service management                                  │  │
    │  │  - Journalctl for log collection                                   │  │
    │  └──────────────────────────────────────────────────────────────────────┘  │
    │                                                                             │
    │  ┌──────────────────────────────────────────────────────────────────────┐  │
    │  │              MIDDLEWARE & DEVICE DRIVER LAYER                        │  │
    │  │                                                                      │  │
    │  │  ┌─────────────────────────────────────────────────────────────┐   │  │
    │  │  │ Video Capture (V4L2 / libcamera)                           │   │  │
    │  │  │ - Query camera capabilities                               │   │  │
    │  │  │ - Set resolution, FPS, encoding                          │   │  │
    │  │  │ - Frame buffer management                                │   │  │
    │  │  └─────────────────────────────────────────────────────────────┘   │  │
    │  │                                                                      │  │
    │  │  ┌─────────────────────────────────────────────────────────────┐   │  │
    │  │  │ Serial Communication (pyserial)                            │   │  │
    │  │  │ - UART init & baud rate (9600 baud typical)              │   │  │
    │  │  │ - NMEA sentence parsing                                   │   │  │
    │  │  │ - Timeout & error handling                                │   │  │
    │  │  └─────────────────────────────────────────────────────────────┘   │  │
    │  │                                                                      │  │
    │  │  ┌─────────────────────────────────────────────────────────────┐   │  │
    │  │  │ Hardware Accelerators                                       │   │  │
    │  │  │ - H.264 encoding (MMAL on RPi, NVENC on Jetson)           │   │  │
    │  │  │ - Reduces CPU overhead                                     │   │  │
    │  │  └─────────────────────────────────────────────────────────────┘   │  │
    │  └──────────────────────────────────────────────────────────────────────┘  │
    │                                                                             │
    │  ┌──────────────────────────────────────────────────────────────────────┐  │
    │  │                    APPLICATION LAYER (Python 3)                     │  │
    │  │                                                                      │  │
    │  │  ┌─────────────────────┐      ┌──────────────────────────────┐    │  │
    │  │  │ Camera Manager      │      │ GPS Logger                   │    │  │
    │  │  │ (dual_camera.py)    │      │ (gps_logger.py)              │    │  │
    │  │  │                     │      │                              │    │  │
    │  │  │ • Front stream      │      │ • NMEA ingest               │    │  │
    │  │  │ • Rear stream       │      │ • UART polling              │    │  │
    │  │  │ • Encoding          │      │ • CSV logging               │    │  │
    │  │  │ • FPS throttling    │      │ • Time sync                 │    │  │
    │  │  │ • Error recovery    │      │ • Epoch offset calc         │    │  │
    │  │  └──────────┬──────────┘      └──────────────┬───────────────┘    │  │
    │  │             │                                 │                    │  │
    │  │             └────────────────┬────────────────┘                    │  │
    │  │                              │                                    │  │
    │  │                    ┌─────────▼──────────┐                        │  │
    │  │                    │ Storage Manager    │                        │  │
    │  │                    │ (storage_mgr.py)   │                        │  │
    │  │                    │                    │                        │  │
    │  │                    │ • Circular buffer  │                        │  │
    │  │                    │ • Disk usage check │                        │  │
    │  │                    │ • Delete oldest    │                        │  │
    │  │                    │ • Atomic ops       │                        │  │
    │  │                    │ • File validation  │                        │  │
    │  │                    └─────────┬──────────┘                        │  │
    │  │                              │                                    │  │
    │  │                    ┌─────────▼──────────┐                        │  │
    │  │                    │ Watchdog & Recovery│                        │  │
    │  │                    │ (watchdog.py)      │                        │  │
    │  │                    │                    │                        │  │
    │  │                    │ • Health monitor   │                        │  │
    │  │                    │ • Graceful shutdown│                        │  │
    │  │                    │ • Crash detection  │                        │  │
    │  │                    │ • Boot recovery    │                        │  │
    │  │                    └──────────┬─────────┘                        │  │
    │  │                              │                                    │  │
    │  │                    ┌─────────▼──────────┐                        │  │
    │  │                    │ Main Orchestrator  │                        │  │
    │  │                    │ (main.py)          │                        │  │
    │  │                    │                    │                        │  │
    │  │                    │ • Service startup  │                        │  │
    │  │                    │ • Logging setup    │                        │  │
    │  │                    │ • Error handling   │                        │  │
    │  │                    │ • Systemd binding  │                        │  │
    │  │                    └────────────────────┘                        │  │
    │  │                                                                      │  │
    │  └──────────────────────────────────────────────────────────────────────┘  │
    │                                                                             │
    │  CPU/Memory Budget (typical):                                             │
    │  • Front Camera:  15% CPU, 100 MB RAM                                     │
    │  • Rear Camera:   15% CPU, 100 MB RAM                                     │
    │  • GPS Logger:     2% CPU,  20 MB RAM                                     │
    │  • Storage Mgr:    5% CPU,  50 MB RAM                                     │
    │  • Watchdog:       1% CPU,  10 MB RAM                                     │
    │  • System:        10% CPU, 400 MB RAM                                     │
    │  • Available:     42% CPU, 336 MB RAM (on 8GB Raspberry Pi 4B)           │
    │                                                                             │
    └─────────────────────────────────────────────────────────────────────────────┘
              │                                                   │
              │ SD Card Interface (µSD → UHS-II or USB adapter)  │
              │                                                   │
              ▼                                                   ▼
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                         PERSISTENT STORAGE (SD CARD)                        │
    │                                                                             │
    │  Device: /dev/mmcblk0 or /dev/sda (USB adapter)                           │
    │  Capacity: 256–512 GB (industrial-grade, MLC/TLC, wear-leveled)           │
    │  Filesystem: ext4 (journaled, recovery-friendly)                          │
    │  Partitioning: Single /blackbox partition                                 │
    │                                                                             │
    │  /blackbox/                                                               │
    │  ├── video/                                                               │
    │  │   ├── front/                                                           │
    │  │   │   ├── 2026-07-06_10-00-00.mp4  (1 min chunk)                      │
    │  │   │   ├── 2026-07-06_10-01-00.mp4                                    │
    │  │   │   └── ... (up to 60 files per hour)                              │
    │  │   └── rear/                                                            │
    │  │       ├── 2026-07-06_10-00-00.mp4                                    │
    │  │       └── ...                                                          │
    │  ├── gps/                                                                 │
    │  │   ├── 2026-07-06_10.csv  (hourly GPS log)                            │
    │  │   └── 2026-07-06_11.csv                                             │
    │  ├── logs/                                                                │
    │  │   ├── system.log  (systemd journal dump)                              │
    │  │   ├── camera.log  (camera errors)                                     │
    │  │   └── gps.log     (GPS debug output)                                  │
    │  └── state/                                                               │
    │      ├── last_boot.txt  (timestamp of last clean boot)                   │
    │      ├── disk_usage.json (current disk state)                           │
    │      └── recovery.lock   (recovery mode flag)                           │
    │                                                                             │
    └─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🖥️ Compute Platform Options

### Option A: Raspberry Pi 4B (8GB) - **RECOMMENDED for POC**
- **CPU**: Broadcom BCM2711 Quad-core ARM Cortex-A72 @ 1.5 GHz
- **RAM**: 8 GB LPDDR4 @ 3200 MHz
- **GPU**: VideoCore VI (H.264 hardware encoder/decoder)
- **USB**: 2× USB 3.0, 4× USB 2.0 (via hub)
- **Power**: 5V/5A typical (8W continuous)
- **Cost**: $75
- **Pros**: Low cost, proven, hardware video encoding (MMAL), GPIO, extensive community
- **Cons**: ARM-only, limited AI capabilities, thermal throttling under load

### Option B: NVIDIA Jetson Nano 2GB
- **CPU**: NVIDIA Tegra X1 Mobile Processor (4× ARM Cortex-A57 @ 1.43 GHz)
- **GPU**: 128-core Maxwell (NVIDIA CUDA compatible)
- **RAM**: 2–4 GB LPDDR4
- **Video**: H.264/H.265 hardware encoding (NVDEC/NVENC)
- **Power**: 5W (extremely low)
- **Cost**: $59–$99
- **Pros**: AI-capable, low power, compact, NVIDIA ecosystem
- **Cons**: Limited RAM, slower than Pi 4B for video, fewer GPIO/I2C ports

### Option C: x86-64 Mini-PC (e.g., Intel NUC)
- **CPU**: Intel Core i3/i5 (quad-core, 2+ GHz)
- **RAM**: 8–16 GB DDR4
- **GPU**: Intel Quick Sync Video (H.264/H.265 encoding)
- **USB**: 4× USB 3.0+
- **Power**: 15–25W
- **Cost**: $200–$400
- **Pros**: Fastest, maximum flexibility, x86 software ecosystem
- **Cons**: Overkill for POC, higher power draw, larger form factor

### **Selection Rationale**
For POC: **Raspberry Pi 4B (8GB)**
- Proven for video capture tasks
- Hardware H.264 encoding (low CPU impact)
- Adequate RAM for dual streams + GPS
- Cost-effective
- Thermal management manageable with passive heatsink

---

## 📹 Camera Interface Design

### Physical Interface
```
Compute Module (USB 3.0 hubs)
        │
        ├─→ USB Hub 1 (powered)
        │     ├─→ USB Camera 1 (Front, 1080p)
        │     └─→ USB Camera 2 (Rear, 1080p)
        │
        ├─→ USB Hub 2 (powered, optional)
        │     └─→ GPS Module (TTL-to-USB adapter)
        │
        └─→ SD Card Adapter
```

### Camera Specifications
- **Model**: OV5647 or IMX219 (MIPI CSI) OR USB UVC cameras
- **Resolution**: 1920×1080 (1080p) native, can downsample to 720p in software
- **Frame Rate**: 30 fps @ 1080p (typical)
- **Interface**: USB 3.0 (recommended) or MIPI CSI (if using RPi camera module)
- **Bitrate (output)**: 5 Mbps per camera @ 720p H.264
- **Power**: <500 mA each (via USB powered hub)

### Capture Pipeline
```python
Front Camera Thread:
  1. Initialize V4L2 device (/dev/video0)
  2. Set format: YUYV 1920×1080 @ 30 fps
  3. Enable hardware encoding (MMAL)
  4. Loop:
     - Capture frame
     - Encode to H.264
     - Write to circular buffer
     - Check FPS, sleep if needed

Rear Camera Thread:
  (Identical to front, /dev/video1)
```

---

## 📍 GPS Module Interface

### Physical Connection
```
u-blox M8 GPS Module (UART)
        │
        ├─ TX (3.3V CMOS)  → RX (GPIO serial / USB-TTL converter)
        ├─ RX (3.3V CMOS)  ← TX (GPIO serial / USB-TTL converter)
        ├─ GND             → GND
        └─ +5V (or 3.3V)   ← Power

Data Flow:
  GPS Module → UART (/dev/ttyUSB0 or /dev/ttyAMA0)
            → NMEA sentences (ASCII strings)
            → GPS Logger (Python pyserial)
            → Parse $GPRMC, $GPGGA
            → Extract: timestamp, lat, lon, alt, speed, quality
            → CSV log file
```

### NMEA Sentence Parsing
```
Example $GPRMC:
$GPRMC,082949.000,A,3723.2475,N,12158.3416,W,0.295,286.30,060612,,,A*24

Fields:
  - Timestamp: 08:29:49 UTC
  - Status: A (active)
  - Latitude: 37°23.2475' N
  - Longitude: 121°58.3416' W
  - Speed: 0.295 knots
  - Course: 286.30°
```

### GPS Logger Implementation
```
1. Open serial port (9600 baud, 8N1)
2. Read NMEA sentences in loop
3. Parse $GPRMC for timestamp & position
4. Buffer incoming GPS data (circular queue, 1 Hz typical)
5. Write to CSV file at regular intervals (every 100 fixes or on timeout)
6. Handle timeout gracefully (use last known fix + local clock)
```

---

## ⚡ Power Supply & Battery Design

### Power Budget
```
Component              Typical Current @ 5V   Notes
───────────────────────────────────────────────────────
Raspberry Pi 4B        1.5 A (7.5W)           Varies with load
Front Camera           0.4 A (2W)             USB powered
Rear Camera            0.4 A (2W)             USB powered
GPS Module             0.1 A (0.5W)           Very low
SD Card Interface      0.05 A (0.25W)         Minimal
USB Hubs               0.2 A (1W)             Self-powered or passive
───────────────────────────────────────────────────────
TOTAL TYPICAL          ≈ 2.7 A (13.5W)        Peak could be 3.5A

Continuous Operation:  ~12W average
Peak Transient:        ~18W (both cameras encoding simultaneously)
```

### Battery Selection
- **Capacity**: 10,000–20,000 mAh (37–74 Wh)
- **Type**: Lithium-ion (LiPo or 18650-based)
- **Output**: 5V/2A continuous minimum, 5A peak capable
- **Runtime**: ~5–7 hours typical bike ride
- **Examples**:
  - Anker PowerCore 10000 ($25): 10,000 mAh → ~4.5 hours runtime
  - RavPower 26800 ($40): 26,800 mAh → ~12 hours runtime

### Power Delivery Design
```
Battery (5V output)
     │
     ├─→ DC-DC Converter (5V → 5V regulated, 3A capable)
     │     │
     │     ├─→ Powered USB Hub 1 (cameras)
     │     ├─→ Powered USB Hub 2 (optional)
     │     ├─→ Compute Module (direct 5V)
     │     └─→ GPS Module (via TTL adapter)
     │
     └─→ Graceful Shutdown Circuit
           │
           ├─→ Monitor battery voltage
           ├─→ If V < 4.5V (10% capacity remaining)
           │   → Signal compute module (SIGTERM)
           │   → Wait 30 seconds for clean shutdown
           │   → Force power off if still alive
           │
           └─→ Watchdog timer (external circuit optional)
               │
               └─→ Hardware interrupt to compute module
                   (forces reboot if deadlock detected)
```

### Graceful Shutdown Strategy
1. **Battery Monitor Thread** continuously measures voltage
2. When V drops to ~4.5V (≈10% capacity), trigger graceful shutdown:
   - Send `SIGTERM` to main process
   - Wait up to 30 seconds for GPS & camera threads to finish
   - Flush all file buffers (fsync)
   - Close SD card cleanly
   - Power off compute module

---

## 🔌 Electrical Schematic (Text Form)

```
┌─────────────────────────────────────────────────────────────────┐
│ POWER DISTRIBUTION NETWORK                                      │
│                                                                 │
│  +5V Battery (10Ah nominal)                                    │
│       │                                                         │
│       ├─→ [Fuse 15A] ──→ [Reverse polarity diode] ────→       │
│       │                                                         │
│       ├─→ [DC-DC Buck Converter] ─→ 5V_regulated               │
│       │   (input: 4.2V–12V, output: 5.0V ±5%, 3A capable)    │
│       │                                                         │
│       │   5V_regulated                                         │
│       │       ├─→ [TVS Diode] ─→ USB Hub 1 (2.5A capacity)   │
│       │       │                    ├→ Front Camera             │
│       │       │                    └→ Rear Camera              │
│       │       │                                                │
│       │       ├─→ [TVS Diode] ─→ USB Hub 2 (1.5A capacity)   │
│       │       │                    └→ GPS via TTL adapter     │
│       │       │                                                │
│       │       ├─→ [TVS Diode] ─→ RPi 4B (direct 5V)           │
│       │       │   (RPi includes internal 3.3V LDO)            │
│       │       │                                                │
│       │       └─→ [Bulk Capacitor 1000µF/10V] ──→ GND         │
│       │           (smoothing for transients)                   │
│       │                                                         │
│       └─→ [Battery Monitor ADC]                               │
│           (measure voltage via GPIO, trigger SIGTERM)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Key Components:
- TVS Diode: Clamp inductive spikes from camera power on/off
- Bulk Capacitor: Absorb transient current peaks
- Battery Monitor: ADC on RPi GPIO (via voltage divider 1:2)
- DC-DC Converter: Stabilize 5V supply, handle input voltage variation
```

---

## 🔄 Data Flow Diagrams

### Video Capture Pipeline
```
Front Camera Stream:
┌────────────────┐
│ USB Camera 1   │ (raw H.264 bitstream)
└────────┬───────┘
         │ V4L2 interface
         ▼
┌────────────────────────────────────────┐
│ Camera Manager (front_camera.py)       │
│                                        │
│ • V4L2 capture thread                 │
│ • Frame buffer (size: 10 frames)      │
│ • Encode: H.264 @ 5 Mbps              │
│ • FPS throttle: 30 fps                │
│ • Error handling: USB disconnects      │
└────────┬───────────────────────────────┘
         │ MP4 packets (1 Mbps chunks)
         ▼
┌────────────────────────────────────────┐
│ Storage Manager                        │
│ (storage_mgr.py)                       │
│                                        │
│ • Circular buffer (120 × 1-min files) │
│ • Monitor disk usage                   │
│ • Delete oldest when threshold hit    │
│ • Atomic file ops (temp + rename)     │
└────────┬───────────────────────────────┘
         │ 1-minute segments
         ▼
     /blackbox/video/front/
     ├─ 2026-07-06_10-00-00.mp4
     ├─ 2026-07-06_10-01-00.mp4
     └─ ...
```

### GPS Logger Pipeline
```
GPS Module (u-blox M8):
┌────────────────┐
│ UART TX        │ NMEA sentences
└────────┬───────┘
         │ 9600 baud, 8N1
         ▼
┌────────────────────────────────────────┐
│ GPS Logger (gps_logger.py)             │
│                                        │
│ • UART rx_thread (blocking read)      │
│ • NMEA sentence parser                │
│ • GPS data queue (FIFO, 100 fixes)   │
│ • Time sync (GPS vs system clock)     │
│ • CSV write interval: every 10 fixes  │
└────────┬───────────────────────────────┘
         │ CSV lines
         ▼
     /blackbox/gps/
     ├─ 2026-07-06_10.csv (timestamp, lat, lon, alt, speed, qual)
     ├─ 2026-07-06_11.csv
     └─ ...
```

### Circular Buffer Deletion Logic
```
CURRENT STATE:
/blackbox/video/front/
├─ 2026-07-06_10-00-00.mp4  (oldest, 9 min old)
├─ 2026-07-06_10-01-00.mp4
├─ 2026-07-06_10-02-00.mp4
├─ ...
└─ 2026-07-06_11-09-00.mp4  (newest, just written)

ON EACH NEW VIDEO CHUNK WRITE:
1. Check disk usage: du -s /blackbox/video/ → 2.4 GB
2. If 2.4 GB > THRESHOLD (2.5 GB):
   - Find oldest file in /front/ and /rear/
   - Check if file exists and is readable
   - Move to temp: mv file.mp4 file.mp4.deleting
   - Delete: rm file.mp4.deleting
   - Log deletion: timestamp, filename, freed_bytes
   - Repeat until usage < THRESHOLD

RESULT:
/blackbox/video/front/
├─ 2026-07-06_10-01-00.mp4  (now oldest)
├─ 2026-07-06_10-02-00.mp4
├─ ...
└─ 2026-07-06_11-09-00.mp4
```

---

## 🔐 Filesystem & Storage Layout

### Partition Scheme
```
Device: /dev/mmcblk0 (256 GB SD card)

Partition 1: /dev/mmcblk0p1 (1 GB)
  - Format: ext4
  - Mount: /boot
  - Contents: Linux kernel, device tree, bootloader
  - Reserve: ~10% for OS & utilities

Partition 2: /dev/mmcblk0p2 (255 GB)
  - Format: ext4 (journaled, recovery-friendly)
  - Mount: /blackbox
  - Contents: video, GPS logs, system logs, state
  - Reserved blocks: 5% (for filesystem recovery)
```

### Directory Structure
```
/blackbox/
├── video/
│   ├── front/
│   │   ├── 2026-07-06_10-00-00.mp4  (1 min, ~5 MB)
│   │   ├── 2026-07-06_10-01-00.mp4
│   │   ├── 2026-07-06_10-02-00.mp4
│   │   └── ... (up to 60 files = 1 hour)
│   │
│   └── rear/
│       ├── 2026-07-06_10-00-00.mp4
│       └── ...
│
├── gps/
│   ├── 2026-07-06_10.csv  (hourly, 3600 fixes × ~80 bytes = 288 KB)
│   ├── 2026-07-06_11.csv
│   └── ... (retained for last 1 hour)
│
├── logs/
│   ├── system.log  (systemd journal, rotated daily)
│   ├── camera.log  (camera errors, rotated on size)
│   ├── gps.log     (GPS ingest debug)
│   └── storage.log (deletion events)
│
└── state/
    ├── last_boot.txt  (ISO timestamp)
    ├── disk_usage.json (current usage stats)
    ├── recovery.lock   (created if last boot was unclean)
    └── config.json    (system configuration)
```

### Filesystem Mount Options
```bash
# /etc/fstab entry for blackbox partition:

# File System Mount Point Type Options Dump Pass
/dev/mmcblk0p2 /blackbox ext4 \
  defaults,\
  noatime,\
  nodiratime,\
  commit=60,\
  errors=remount-ro 0 2

# noatime/nodiratime: Reduce writes for access time tracking
# commit=60: Sync journal every 60 seconds (balance safety vs. performance)
# errors=remount-ro: If filesystem error, remount read-only (safer than continuing)
```

---

## 📊 Time Synchronization Strategy

### Challenge
- Cameras capture frames with local system timestamps
- GPS module provides GNSS time (UTC, high accuracy ±1ms)
- System clock may drift without NTP (bike environment, no network)
- **Goal**: Keep timestamps within 100ms across video and GPS

### Solution
```
Boot Sequence:
1. System boots with stale system clock (RTC battery backup minimal)
2. GPS module starts acquiring satellites (~30–60 seconds cold start)
3. Once GPS fix acquired:
   - Read GPS time from $GPRMC NMEA sentence
   - Compare with system time: offset = gps_time - sys_time
   - If |offset| > 100ms:
     → Adjust system clock: ntpdate or timedatectl --set-time
     → Log adjustment: "Clock offset: +450 ms"
   - Else: Just log offset for monitoring

During Operation:
4. Sample GPS time every 60 seconds
5. Monitor clock drift (should be <50 ppm on Pi with decent crystal)
6. If drift > 1 second, trigger re-sync
7. All video chunks embed:
   - Frame timestamp (UTC from system clock)
   - GPS timestamp (from concurrent GPS log entry)
   - Metadata in MP4: mdat box includes timestamp track

On Recovery (after power loss):
8. RTC time may be stale; repeat boot sync process
9. Validate recovered video/GPS files:
   - Check timestamp continuity
   - Flag large gaps (>2 second) for review
```

### Implementation
```python
# time_sync.py (pseudocode)

import subprocess
import time
from datetime import datetime
import gps_parser  # Custom NMEA parser

class TimeSynchronizer:
    def __init__(self, max_offset_ms=100):
        self.max_offset_ms = max_offset_ms
        self.offset = 0
        
    def sync_with_gps(self, gps_sentence):
        """Sync system time with GPS NMEA sentence"""
        try:
            gps_time = gps_parser.parse_rmc(gps_sentence)['timestamp']  # UTC seconds
            sys_time = time.time()  # UTC seconds
            self.offset = (gps_time - sys_time) * 1000  # ms
            
            if abs(self.offset) > self.max_offset_ms:
                subprocess.run(['sudo', 'timedatectl', 'set-time', 
                              datetime.utcfromtimestamp(gps_time).isoformat()],
                              check=True)
                print(f"Clock adjusted by {self.offset:.1f} ms")
            else:
                print(f"Clock offset: {self.offset:.1f} ms (acceptable)")
        except Exception as e:
            print(f"Time sync error: {e}")
    
    def get_timestamp(self):
        """Return current UTC timestamp adjusted by GPS offset"""
        return time.time() + (self.offset / 1000.0)
```

---

## 🔧 Hardware Integration Checklist

- [ ] Raspberry Pi 4B (8GB) mounted with heatsink + fan
- [ ] Two USB cameras (front + rear) with stable mounting brackets
- [ ] u-blox M8 GPS module on USB-TTL adapter, GPS antenna (external, clear sky view)
- [ ] Industrial SD card (256 GB, MLC, wear-leveled)
- [ ] USB powered hubs (2×, 2.5A+ each)
- [ ] Battery pack (10,000+ mAh) with stable 5V output
- [ ] Wiring harness with USB extensions (bike-friendly routing, strain relief)
- [ ] Weatherproof enclosure (IP65+, ventilated for thermal management)
- [ ] Power management circuit (battery monitor, graceful shutdown)
- [ ] Watchdog timer (external, optional but recommended)

---

**Document**: 02-SYSTEM-ARCHITECTURE.md  
**Version**: 1.0  
**Date**: 2026-07-06  
**Status**: Complete
