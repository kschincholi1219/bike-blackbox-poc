# UML Architecture Diagrams – Bike Black Box POC

This document provides professional PlantUML architecture diagrams for the Bike Black Box POC.
Each diagram is stored as a standalone `.puml` file in [`docs/diagrams/`](diagrams/) and is
embedded here for convenient reading.

> **Rendering tip** – PlantUML diagrams can be rendered in:
> - VS Code with the [PlantUML extension](https://marketplace.visualstudio.com/items?itemName=jebbs.plantuml)
> - GitHub/GitLab via the [kroki](https://kroki.io/) integration
> - The online editor at <https://www.plantuml.com/plantuml>
> - JetBrains IDEs with the PlantUML plugin

---

## Table of Contents

1. [System Architecture (Component Model)](#1-system-architecture-component-model)
2. [Software Services Architecture](#2-software-services-architecture)
3. [Data Flow Pipelines](#3-data-flow-pipelines)
4. [Boot-Time Recovery Sequence](#4-boot-time-recovery-sequence)
5. [Circular Buffer State Machine](#5-circular-buffer-state-machine)
6. [GPS Time Synchronisation Sequence](#6-gps-time-synchronisation-sequence)

---

## 1. System Architecture (Component Model)

**Source file:** [`docs/diagrams/system-architecture.puml`](diagrams/system-architecture.puml)

**Purpose:** Shows all hardware and software components, their interfaces, connections
(USB, UART, SD card), and data flow from sensors through the Raspberry Pi to persistent
storage.

```plantuml
@startuml system-architecture
!include diagrams/system-architecture.puml
@enduml
```

### Key architectural decisions visible in this diagram

| Decision | Detail | Reference |
|---|---|---|
| **Compute platform** | Raspberry Pi 4B, 8 GB RAM | [02-SYSTEM-ARCHITECTURE.md](02-SYSTEM-ARCHITECTURE.md) §Platform Options |
| **Video interface** | USB 3.0 (UVC cameras) via V4L2 | [04-SOFTWARE-ARCHITECTURE.md](04-SOFTWARE-ARCHITECTURE.md) §Camera Capture |
| **GPS interface** | UART `/dev/ttyUSB0` @ 9600 baud | [03-STORAGE-DATA-DESIGN.md](03-STORAGE-DATA-DESIGN.md) §GPS Schema |
| **Storage** | ext4 journaled, `/blackbox/` partition | [03-STORAGE-DATA-DESIGN.md](03-STORAGE-DATA-DESIGN.md) §Filesystem |
| **Hardware encoding** | VideoCore VI MMAL (H.264) | [02-SYSTEM-ARCHITECTURE.md](02-SYSTEM-ARCHITECTURE.md) §HW Accelerators |
| **Process management** | systemd unit file, `Restart=always` | [04-SOFTWARE-ARCHITECTURE.md](04-SOFTWARE-ARCHITECTURE.md) §Main Orchestrator |

### CPU / memory budget (Raspberry Pi 4B, 8 GB)

| Service | CPU | RAM |
|---|---|---|
| Front Camera | ~15 % | 100 MB |
| Rear Camera | ~15 % | 100 MB |
| GPS Logger | ~2 % | 20 MB |
| Storage Manager | ~5 % | 50 MB |
| Watchdog | ~1 % | 10 MB |
| OS / System | ~10 % | 400 MB |
| **Available headroom** | **~52 %** | **~7 368 MB** |

---

## 2. Software Services Architecture

**Source file:** [`docs/diagrams/software-services.puml`](diagrams/software-services.puml)

**Purpose:** Shows the five core Python services as a class/component model, including
their public interfaces, key attributes, dependency relationships, and the shared
`IService` interface they all implement.

```plantuml
@startuml software-services
!include diagrams/software-services.puml
@enduml
```

### Service summary

| Service | File | Role |
|---|---|---|
| `MainOrchestrator` | `main.py` | Boot, logging, start/stop all services |
| `FrontCameraService` | `front_camera.py` | V4L2 capture, H.264 encoding, frame queue |
| `RearCameraService` | `rear_camera.py` | Identical to Front; targets `/dev/video1` |
| `GPSLoggerService` | `gps_logger.py` | UART ingest, NMEA parse, CSV flush |
| `StorageManagerService` | `storage_mgr.py` | Circular buffer, deletion, state logging |
| `WatchdogService` | `watchdog.py` | Heartbeat monitoring, emergency shutdown |
| `RecoveryManager` | `recovery.py` | Boot validation, lock file management |
| `TimeSync` | `time_sync.py` | GPS-disciplined clock drift correction |

---

## 3. Data Flow Pipelines

**Source file:** [`docs/diagrams/data-flow-pipelines.puml`](diagrams/data-flow-pipelines.puml)

**Purpose:** Shows three parallel data pipelines – the video capture pipeline (front and
rear cameras → circular buffer), the GPS ingest pipeline (UART → NMEA parsing → CSV
storage), and the storage management flow (disk usage monitoring → atomic deletion).

```plantuml
@startuml data-flow-pipelines
!include diagrams/data-flow-pipelines.puml
@enduml
```

### Pipeline summary

#### Video Capture Pipeline

```
Front/Rear USB Camera
  → V4L2 Driver (/dev/videoN)
  → Frame Queue  (deque, maxlen=10, FIFO)
  → H.264 HW Encoder (MMAL, VideoCore VI, 5 Mbps)
  → MP4 Muxer (FFmpeg, 1-minute segments)
  → SD Card /video/{front,rear}/YYYY-MM-DD_HH-MM-SS.mp4
  → StorageManager.on_new_chunk()
```

#### GPS Ingest Pipeline

```
GPS Module (u-blox M8N, UART 9600 baud, 1 Hz)
  → pyserial.readline()
  → NMEA Parser ($GPRMC, $GPGGA → decimal lat/lon)
  → GPS Buffer (deque, maxlen=100)
  → CSV Writer (flush on 10 fixes or 10 s timeout)
  → SD Card /gps/YYYY-MM-DD_HH.csv
```

#### Storage Management Flow

```
on_new_chunk(camera, filename, size_mb)
  → Disk Usage Monitor (sum front + rear lists)
  → [total_mb > 2,500 MB?]
       Yes → Oldest-File Finder (min timestamp across both lists)
           → Atomic Deleter (rename → .deleting → remove)
           → Update disk_usage.json
       No  → Return (within budget)
```

---

## 4. Boot-Time Recovery Sequence

**Source file:** [`docs/diagrams/boot-recovery-sequence.puml`](diagrams/boot-recovery-sequence.puml)

**Purpose:** Shows the full lifecycle from systemd `ExecStart` through optional file
validation (if unclean shutdown is detected), service initialisation, steady-state
watchdog monitoring, and graceful SIGTERM shutdown with lock-file cleanup.

```plantuml
@startuml boot-recovery-sequence
!include diagrams/boot-recovery-sequence.puml
@enduml
```

### Recovery lock mechanism

The `recovery.lock` file acts as a "dirty boot" flag:

| Event | Action |
|---|---|
| Boot starts | `create_recovery_lock()` — writes current UTC timestamp |
| Clean shutdown | `remove_recovery_lock()` — unlinks the lock file |
| Next boot sees lock | `validate_video_files()` + `validate_gps_logs()` run before services start |
| Next boot sees no lock | Validation skipped; fast startup |

This design ensures that after any abrupt power loss the system automatically audits its
data before resuming recording.

---

## 5. Circular Buffer State Machine

**Source file:** [`docs/diagrams/circular-buffer-state.puml`](diagrams/circular-buffer-state.puml)

**Purpose:** Shows the four states of the circular-buffer lifecycle (`Idle`, `Writing`,
`Threshold_Exceeded`, `Deleting`), transition conditions, and the atomic delete pattern
that prevents partial-delete corruption on sudden power loss.

```plantuml
@startuml circular-buffer-state
!include diagrams/circular-buffer-state.puml
@enduml
```

### State transition summary

| From | Trigger | To |
|---|---|---|
| `Idle` | New 1-min chunk written | `Writing` |
| `Writing` | `total_mb ≤ 2,500 MB` | `Idle` |
| `Writing` | `total_mb > 2,500 MB` | `Threshold_Exceeded` |
| `Threshold_Exceeded` | Oldest file found & rename succeeds | `Deleting` |
| `Threshold_Exceeded` | Still above threshold after one deletion | `Threshold_Exceeded` (loop) |
| `Threshold_Exceeded` | File not found / rename fails | `Error` |
| `Deleting` | `total_mb ≤ 2,500 MB` after removal | `Idle` |
| `Deleting` | `total_mb > 2,500 MB` after removal | `Threshold_Exceeded` (loop) |
| `Error` | Exception logged | `Idle` |

### Retention calculation

```
Threshold: 2,500 MB
Per camera: 5 Mbps × 3,600 s = 2,250 MB / hour
Two cameras: 4,500 MB / hour

Effective retention: threshold / per-camera rate
  = 2,500 MB / 2,250 MB per hour ≈ 1 hour 6 minutes

Result: approximately the latest 1 hour of dual-stream footage
        is always preserved (slightly over-threshold for safety).
```

---

## 6. GPS Time Synchronisation Sequence

**Source file:** [`docs/diagrams/time-sync-sequence.puml`](diagrams/time-sync-sequence.puml)

**Purpose:** Shows the three-phase time synchronisation strategy – boot-time initial
sync once the first valid GPS fix arrives, continuous 60-second drift monitoring, and a
scheduled full resync every 60 minutes.

```plantuml
@startuml time-sync-sequence
!include diagrams/time-sync-sequence.puml
@enduml
```

### Sync trigger conditions

| Phase | Trigger | Action |
|---|---|---|
| Boot sync | First valid NMEA fix (`fix_quality ≥ 1`) | `timedatectl set-time` if `|offset| > 100 ms` |
| Drift monitor | Every GPS fix (~1 Hz), sampled every 60 s | Resync if `|drift_ppm| > 16,667` (> 1 s per 60 s) |
| Scheduled resync | Every 60 minutes | Full `timedatectl set-time` unconditionally |

### Why this matters

- The Raspberry Pi RTC can drift by several minutes after the coin-cell battery depletes.
- Without NTP (bike is offline), GPS is the only accurate time source.
- Video chunk filenames (`YYYY-MM-DD_HH-MM-SS.mp4`) must align with GPS CSV timestamps
  so that footage can be correlated with location data in post-processing.
- Target accuracy: **≤ 100 ms** timestamp drift between video and GPS records.

---

## Diagram Files

| Diagram | `.puml` file | Description |
|---|---|---|
| System Architecture | [`system-architecture.puml`](diagrams/system-architecture.puml) | Hardware + software component model |
| Software Services | [`software-services.puml`](diagrams/software-services.puml) | Class/component diagram for all services |
| Data Flow Pipelines | [`data-flow-pipelines.puml`](diagrams/data-flow-pipelines.puml) | Video, GPS, and storage management flows |
| Boot Recovery Sequence | [`boot-recovery-sequence.puml`](diagrams/boot-recovery-sequence.puml) | Startup, validation, and graceful shutdown |
| Circular Buffer State | [`circular-buffer-state.puml`](diagrams/circular-buffer-state.puml) | File lifecycle state machine |
| Time Sync Sequence | [`time-sync-sequence.puml`](diagrams/time-sync-sequence.puml) | GPS-disciplined clock sync phases |

---

**Related Documentation:**
- [System Architecture](02-SYSTEM-ARCHITECTURE.md) – Hardware block diagram, platform options
- [Storage & Data Design](03-STORAGE-DATA-DESIGN.md) – Encoding specs, circular buffer, GPS schema
- [Software Architecture](04-SOFTWARE-ARCHITECTURE.md) – Service pseudocode, recovery logic

**Last Updated:** 2026-07-07
**Status:** Complete
