# Bike Black Box POC - Documentation Index

## Quick Links

### Core Design Documents
1. **[Executive Summary](01-EXECUTIVE-SUMMARY.md)** – Vision, success criteria, design overview
2. **[System Architecture](02-SYSTEM-ARCHITECTURE.md)** – Hardware block diagram, platform options, power budget
3. **[Storage & Data Design](03-STORAGE-DATA-DESIGN.md)** – Video encoding, circular buffer, GPS schema
4. **[Software Architecture](04-SOFTWARE-ARCHITECTURE.md)** – Modular services, capture pipelines, recovery logic

### Platform Analysis
5. **[ESP32-P4 Analysis](ESP32-P4-ANALYSIS.md)** – Viability, power efficiency, comparison with RPi/Jetson

### Implementation & Testing
6. **[Implementation Roadmap](#coming-soon)** – Phased milestones, effort estimates, BOM
7. **[Risks & Mitigation](#coming-soon)** – Technical risks, failure modes
8. **[Validation & Test Plan](#coming-soon)** – Functional, stress, integrity tests

### Reference Materials  
9. **[Reference Materials](#coming-soon)** – Pseudocode, GPS schema, folder structure

---

## Reading Order

### For Decision Makers
1. Read [Executive Summary](01-EXECUTIVE-SUMMARY.md) (5 min)
2. Skim [System Architecture](02-SYSTEM-ARCHITECTURE.md) hardware section (10 min)
3. Check [ESP32-P4 Analysis](ESP32-P4-ANALYSIS.md) verdict (5 min)

### For Architects
1. [Executive Summary](01-EXECUTIVE-SUMMARY.md) (15 min)
2. [System Architecture](02-SYSTEM-ARCHITECTURE.md) in full (30 min)
3. [Storage & Data Design](03-STORAGE-DATA-DESIGN.md) – Storage section (20 min)
4. [Software Architecture](04-SOFTWARE-ARCHITECTURE.md) – Overview (25 min)

### For Developers
1. [System Architecture](02-SYSTEM-ARCHITECTURE.md) – Diagram + platform selection (20 min)
2. [Storage & Data Design](03-STORAGE-DATA-DESIGN.md) – Complete (45 min)
3. [Software Architecture](04-SOFTWARE-ARCHITECTURE.md) – Complete (60 min)
4. Reference Materials (pseudocode samples) (30 min)

### For QA/Testing
1. [Executive Summary](01-EXECUTIVE-SUMMARY.md) – Success criteria (10 min)
2. [Validation & Test Plan](#coming-soon) – Complete (60 min)
3. [Storage & Data Design](03-STORAGE-DATA-DESIGN.md) – Time sync section (15 min)

---

## Key Decisions Made

| Decision | Choice | Rationale |
|----------|--------|----------|
| **Compute Platform** | Raspberry Pi 4B (POC) / ESP32-P4 (production) | RPi for familiarity, ESP32-P4 for power |
| **Video Codec** | H.264 Baseline | Hardware-accelerated, proven, low latency |
| **Resolution/FPS** | 720p @ 30fps | Balance quality, storage, and CPU load |
| **Storage Retention** | Circular 1-hour buffer | Space-efficient, meets requirement |
| **Filesystem** | ext4 journaled | Crash-safe, recovery-friendly |
| **GPS Logging** | CSV format, hourly rotation | Simple, parseable, no dependencies |
| **Time Sync** | GPS-disciplined system clock | Accurate without network |

---

## Success Criteria (POC Phase)

✅ Dual 720p video capture without frame drops  
✅ Rolling 1-hour retention (oldest auto-deleted)  
✅ Continuous GPS logging with ≤100ms timestamp drift  
✅ Graceful recovery from 5× abrupt power cuts  
✅ 4+ hour continuous operational test  
✅ Storage efficiency ≤256 GB per hour (both cameras)  
✅ BOM costing <$400 USD  

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|----------|
| **OS** | Linux (Ubuntu 20.04) | Proven, Python support, vast libraries |
| **Language** | Python 3.9+ | Rapid development, GPIO libraries |
| **Video Capture** | OpenCV + FFmpeg | Hardware encoding support, flexible |
| **GPS** | pyserial + NMEA parser | Lightweight, no external APIs |
| **Storage** | ext4 filesystem | Journaling, crash recovery |
| **Process Mgmt** | systemd | Boot integration, automatic restart |
| **Watchdog** | Custom daemon + hardware watchdog | Deadlock detection, recovery |

---

## Files & Folders

```
repo/
├── README.md                           (this file)
├── docs/
│   ├── 01-EXECUTIVE-SUMMARY.md         ← Start here
│   ├── 02-SYSTEM-ARCHITECTURE.md       ← Hardware & platforms
│   ├── 03-STORAGE-DATA-DESIGN.md       ← Encoding & retention
│   ├── 04-SOFTWARE-ARCHITECTURE.md     ← Code structure & pseudocode
│   ├── 05-IMPLEMENTATION-ROADMAP.md    ← Phases & milestones (coming soon)
│   ├── 06-RISKS-MITIGATION.md          ← Failure modes & solutions (coming soon)
│   ├── 07-VALIDATION-TEST-PLAN.md      ← Test checklist (coming soon)
│   ├── 08-REFERENCE-MATERIALS.md       ← Code samples, GPS schema (coming soon)
│   └── ESP32-P4-ANALYSIS.md            ← Alternative platform
├── src/
│   └── (source code - to be added in Phase 1)
├── tests/
│   └── (test harness - to be added in Phase 2)
└── hardware/
    └── (BOM, schematics - to be added in Phase 1)
```

---

## Phase Timeline

- **Phase 1 (Weeks 1–6)**: MVP on RPi, single camera + GPS
- **Phase 2 (Weeks 7–10)**: Dual camera, watchdog, recovery, hardware tests
- **Phase 3 (Weeks 11–13)**: Performance tuning, optional features, field trials

---

## Contact & Questions

For questions about:
- **Architecture** → See [System Architecture](02-SYSTEM-ARCHITECTURE.md) FAQ section
- **Storage** → See [Storage & Data Design](03-STORAGE-DATA-DESIGN.md) Design Principles
- **Code** → See [Software Architecture](04-SOFTWARE-ARCHITECTURE.md) pseudocode
- **Platform choice** → See [ESP32-P4 Analysis](ESP32-P4-ANALYSIS.md) Recommendation

---

**Last Updated**: 2026-07-06  
**Status**: POC Design Complete, Ready for Development  
**Maintainer**: Embedded Systems Architecture Team