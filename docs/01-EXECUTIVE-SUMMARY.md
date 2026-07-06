# 1. EXECUTIVE SUMMARY

## Vision
Design and prototype a **resilient, self-contained bike dashcam system** that captures continuous dual-camera video and GPS tracking with minimal computational overhead and graceful degradation on power loss.

---

## 🎯 Problem Statement

Cyclists face unique challenges in incident documentation:
- **Need proof** of accidents/conflicts without external witnesses
- **Limited power** (battery-constrained edge device)
- **Vibration & weather** exposure demands rugged design
- **Privacy concerns** require local-only storage (no cloud)
- **Data loss risk** from sudden power interruptions (typical bike electronics scenario)

This POC addresses these through a **distributed, modular system** with:
- Dual high-definition cameras (front + rear)
- Continuous GPS tracking for incident location
- Rolling 1-hour video retention (space-efficient)
- Crash-resilient storage (graceful recovery on power loss)

---

## 📋 Core Design Principles

| Principle | How Achieved | Rationale |
|-----------|--------------|-----------|
| **Local-Only Storage** | SD card, no cloud dependency | Privacy + reliability in poor connectivity |
| **Rolling Retention** | Circular buffer, auto-delete old chunks | Limited storage vs. continuous recording |
| **Modularity** | Independent camera/GPS/storage services | Easy to test, debug, and upgrade |
| **Resilience** | Watchdog, atomic file ops, recovery logic | Bikes experience unexpected power loss |
| **Low Power** | Optimized codec, hardware accel, selective recording | 4-8 hour battery life typical |
| **Simplicity** | Minimal external dependencies, edge-native design | Reduces attack surface, improves reliability |

---

## ✅ Success Criteria (POC Phase)

| Criterion | Target | Acceptance |
|-----------|--------|-----------|
| **Dual Video Capture** | Front + rear @ 720p 30fps | Both streams capture without frame drops |
| **1-Hour Rolling Retention** | Oldest footage auto-deleted | Disk usage stays below threshold |
| **Continuous GPS Logging** | 1 Hz @ accuracy ±5m | GPS data saved to SD, timestamps aligned to video |
| **Power Loss Recovery** | Graceful shutdown + restart | No major file corruption after abrupt power cut |
| **Operational Duration** | 4+ hour test | System operates continuously without crashes |
| **Storage Efficiency** | ≤256 GB for 1-hour dual stream | Achievable with H.264 @ 720p |

---

## 🏗️ System Architecture (High-Level)

```
┌─────────────────────────────────────────────────────────┐
│                    COMPUTE MODULE                        │
│  (Raspberry Pi 4B / Jetson Nano / x86 SBC)             │
│                                                          │
│  ┌─────────────┐  ┌────────────┐  ┌──────────────────┐ │
│  │ Front Cam   │  │ Rear Cam   │  │ GPS Module       │ │
│  │ (USB/CSI)   │  │ (USB/CSI)  │  │ (UART)           │ │
│  └──────┬──────┘  └──────┬─────┘  └────────┬─────────┘ │
│         │                 │                  │          │
│  ┌──────▼─────────────────▼──────────────────▼────────┐ │
│  │   SOFTWARE ORCHESTRATION (systemd, custom daemon) │ │
│  │                                                    │ │
│  │  ┌─────────────────┐  ┌────────────────────────┐ │ │
│  │  │ Camera Manager  │  │ GPS Logger             │ │ │
│  │  │ (dual streams)  │  │ (continuous ingest)    │ │ │
│  │  └────────┬────────┘  └────────────┬───────────┘ │ │
│  │           │                        │             │ │
│  │  ┌────────▼────────────────────────▼──────────┐  │ │
│  │  │  Storage Manager (Circular Buffer)         │  │ │
│  │  │  - Segment video into 1-min chunks         │  │ │
│  │  │  - Monitor disk usage                      │  │ │
│  │  │  - Delete oldest chunks when limit hit     │  │ │
│  │  │  - Atomic file operations                  │  │ │
│  │  └────────┬─────────────────────────────────┘  │ │
│  │           │                                    │ │
│  │  ┌────────▼──────────────────────────────────┐ │ │
│  │  │ Watchdog & Recovery                       │ │ │
│  │  │ - Monitor system health                   │ │ │
│  │  │ - Validate file integrity on boot        │ │ │
│  │  │ - Graceful shutdown handler               │ │ │
│  │  └──────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
            ┌─────────────────────┐
            │  SD CARD (256-512GB)│
            │                     │
            │  /blackbox/         │
            │    ├── video/       │
            │    ├── gps/         │
            │    ├── logs/        │
            │    └── state/       │
            └─────────────────────┘
```

---

## 💾 Storage & Data Design Summary

### Video Encoding
- **Codec**: H.264 (hardware-accelerated on Raspberry Pi/Jetson)
- **Resolution**: 720p (1280×720)
- **Frame Rate**: 30 fps
- **Bitrate**: ~5 Mbps per camera (dual = 10 Mbps total)
- **File Format**: MP4 (H.264 video + AAC audio optional)

### 1-Hour Retention
- **File Segmentation**: 1-minute MP4 chunks (60 files per camera per hour)
- **Total Storage**: ~2.25 GB for 1 hour (both cameras)
- **Circular Buffer**: Delete oldest file when new file written AND total > 2.5 GB
- **Overwrite Strategy**: Rename temp file atomically only when complete

### GPS Logging
- **Format**: CSV (timestamp, lat, lon, alt, speed, fix_quality, satellites)
- **Frequency**: 1 Hz
- **File Rotation**: Hourly rotation (one CSV per hour of operation)
- **Retention**: Same rolling policy as video (1-hour history)

### Time Synchronization
- **Reference**: GPS time (NMEA $GPRMC sentences)
- **Fallback**: System NTP + local clock offset
- **Sync Strategy**: Calibrate system clock against GPS on startup, apply offset to all timestamps
- **Video Metadata**: Embed timestamps in MP4 (moov atom) + frame metadata

---

## 🧠 Software Architecture Overview

### Modular Services

1. **Camera Manager** (front_camera.py, rear_camera.py)
   - Stream capture via OpenCV/FFmpeg
   - Frame buffering and encoding
   - Real-time performance monitoring
   - Graceful handling of USB disconnects

2. **GPS Logger** (gps_logger.py)
   - UART ingest from u-blox M8
   - NMEA parsing ($GPRMC, $GPGGA)
   - CSV file writing with atomic operations
   - Timestamp calibration

3. **Storage Manager** (storage_manager.py)
   - Circular buffer implementation
   - Disk usage monitoring
   - Atomic file operations (temp + rename)
   - Cleanup logic for 1-hour retention

4. **Watchdog & Recovery** (watchdog.py, recovery.py)
   - Monitor process health
   - Handle SIGTERM gracefully (30-sec shutdown window)
   - Validate file integrity on boot
   - Replay orphaned logs after crash

5. **Main Orchestrator** (main.py)
   - Coordinate all services
   - Manage systemd integration
   - Handle boot-time recovery

### Boot-Time Recovery
```
1. System boots (after power loss)
   ↓
2. Validate SD card filesystem (fsck if needed)
   ↓
3. Check for orphaned video chunks (incomplete MP4 headers)
   ↓
4. Check for orphaned GPS logs (incomplete lines)
   ↓
5. Start Camera Manager, GPS Logger, Storage Manager
   ↓
6. Resume normal operation
```

---

## 📊 Storage Calculations

| Scenario | Duration | 2-Camera Total | 256 GB Duration |
|----------|----------|----------------|-----------------|
| Low (480p 20fps, 2 Mbps) | 1 hour | 900 MB | ~291 hours |
| **Medium (720p 30fps, 5 Mbps)** | **1 hour** | **2.25 GB** | **102 hours** |
| High (1080p 30fps, 8 Mbps) | 1 hour | 3.6 GB | 64 hours |

**POC Target**: Medium quality (720p, 30fps) = **2.25 GB per hour**

---

## ⚙️ Implementation Phases

### Phase 1: MVP (Weeks 1–6)
- Single front camera + GPS logging
- Circular buffer prototype
- Local testing on Linux workstation

**Deliverable**: Proof that core loop works (capture → segment → delete oldest)

### Phase 2: Hardening (Weeks 7–10)
- Add rear camera (dual stream)
- Watchdog & graceful shutdown
- Recovery logic post-power-loss
- Hardware testing (Raspberry Pi + real GPS module)

**Deliverable**: System survives power loss without major data corruption

### Phase 3: Optimization (Weeks 11–13)
- Performance tuning (frame drop analysis, CPU/memory profiling)
- Event locking (optional feature: preserve footage on collision)
- Mobile app prototype (view latest clips, export GPX)

**Deliverable**: Production-ready prototype, ready for field trials

---

## 🛡️ Risk Mitigation Summary

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **SD card wear-out** | Data loss | Use industrial-grade SLC/MLC, wear leveling, log rotation |
| **Filesystem corruption** | Unrecoverable data | Atomic file ops, periodic fsck, journaling filesystem (ext4) |
| **Timestamp drift** | GPS/video mismatch | NTP sync at boot, GPS discipline, frame metadata |
| **Frame drops** | Video gaps | Hardware encoder, priority threads, buffer tuning |
| **Power loss mid-write** | Corrupted chunks | Temp file + atomic rename, validation on boot |
| **Dual camera contention** | CPU overload | Separate encoder threads, load balancing, FPS negotiation |

---

## 🧪 Validation Strategy

1. **Functional Tests** (Week 7)
   - Verify dual capture at target FPS
   - Confirm 1-hour deletion triggers correctly
   - Validate GPS sync with video timestamps

2. **Stress Tests** (Week 9)
   - Temperature extremes (-10°C to +60°C)
   - Vibration simulation (bike handling)
   - Abrupt power cuts (simulated 5 times)
   - SD card removal/reinsertion

3. **Integrity Tests** (Week 10)
   - Timestamp alignment (video ↔ GPS ≤ 100ms)
   - No dropped frames over 4-hour run
   - File validation after recovery

4. **Endurance Tests** (Week 12)
   - 24-hour continuous operation
   - SD card wear metrics
   - CPU/memory trend analysis

---

## 💰 Rough Cost Estimate (Bill of Materials)

| Component | Unit Cost | Qty | Subtotal | Notes |
|-----------|-----------|-----|----------|-------|
| Raspberry Pi 4B (8GB) | $75 | 1 | $75 | Compute core |
| USB 3.0 Hub + connectors | $15 | 1 | $15 | Camera attachment |
| USB Cameras (1080p) | $30 | 2 | $60 | Front + rear |
| u-blox M8 GPS Module | $40 | 1 | $40 | UART interface |
| Industrial SD Card (256GB) | $80 | 1 | $80 | Continuous write endurance |
| Power Bank (10000mAh) | $25 | 1 | $25 | ~5 hour runtime |
| Mounting hardware, cables | $30 | 1 | $30 | Bike integration |
| **Total** | | | **$325** | Fully functional POC |

**Alternative Jetson Nano** (Edge AI capable): +$100 (subtotal ~$425)

---

## 📈 Success Metrics (End of POC)

- ✅ Dual 720p video capture without frame drops
- ✅ Rolling 1-hour retention verified (oldest auto-deleted)
- ✅ Continuous GPS logging with ≤100ms timestamp drift
- ✅ Recovery from 5× abrupt power cuts with no major data loss
- ✅ 4+ hour continuous operational test
- ✅ Storage efficiency ≤256 GB per hour (both cameras)
- ✅ BOM costing <$400 USD

---

## 🚀 Next Steps

1. **Approve architecture** and hardware selections
2. **Procure BOM components** (lead time ~1–2 weeks)
3. **Begin Phase 1 development** on workstation Linux environment
4. **Implement MVP** (single camera + GPS + circular buffer)
5. **Validate on Raspberry Pi** hardware
6. **Progress to Phase 2** (dual camera, watchdog, recovery)
7. **Conduct stress & integrity tests** per validation plan
8. **Document learnings** for production design

---

**Document**: 01-EXECUTIVE-SUMMARY.md  
**Version**: 1.0  
**Date**: 2026-07-06  
**Status**: Ready for Development Kickoff
