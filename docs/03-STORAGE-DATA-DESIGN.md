# 3. STORAGE & DATA DESIGN

## 📊 Video Encoding Specifications

### Codec Selection: H.264 (AVC)
- **Standard**: ITU-T H.264 / ISO/IEC MPEG-4 Part 10
- **Profile**: Baseline (simplest, lowest CPU overhead)
- **Level**: 4.0 (supports 1920×1080 @ 30fps)
- **Entropy Coding**: CAVLC (simpler than CABAC, lower latency)
- **Key Frame Interval**: 2 seconds (120 frames @ 30fps)
  - Allows seeking every 2 seconds
  - Trade-off: ~5% bitrate increase vs. longer intervals

### Resolution & Frame Rate Options

| Quality Level | Resolution | FPS | Bitrate/cam | 1-hr (2 cams) | Use Case |
|---|---|---|---|---|---|
| **Low** | 480p (854×480) | 20 | 2 Mbps | 900 MB | Low-light, storage-critical |
| **Medium** | 720p (1280×720) | 30 | 5 Mbps | **2.25 GB** | **POC Target** |
| **High** | 1080p (1920×1080) | 30 | 8 Mbps | 3.6 GB | Production, maximum detail |
| **Ultra** | 1440p (2560×1440) | 30 | 12 Mbps | 5.4 GB | Not recommended for 1-hr retention |

**POC Selection Rationale**:
- 720p @ 30fps → Sweet spot between detail and storage
- Sufficient for license plate recognition (at close range)
- Fits easily in 256 GB SD card for 100+ hours
- Low CPU load on Raspberry Pi / ESP32-P4

### Bitrate Calculations

```
Formula: Bitrate = Resolution × Color Depth × FPS × Compression Ratio

Example (720p @ 30fps H.264):
  Raw: 1280 × 720 × 12 bits/px × 30 fps = 3.32 Gbps
  After H.264 compression (60:1 ratio): 3.32 / 60 = ~55 Mbps (max)
  Practical encoding target: 5 Mbps (overhead budget)
  Achieved in tests: 4.5–5.2 Mbps

Two cameras total:
  Peak: 10 Mbps (both streaming simultaneously)
  Average: 7–8 Mbps (accounting for scene complexity variation)
```

---

## 💾 Storage Requirements & Calculations

### 1-Hour Dual Stream Breakdown

```
Scenario: 720p @ 30fps, H.264 Baseline, 5 Mbps per camera

Calculation:
  Time: 1 hour = 3600 seconds
  Front camera: 5 Mbps × 3600 s = 18,000 Megabits = 2,250 MB = 2.25 GB
  Rear camera:  5 Mbps × 3600 s = 18,000 Megabits = 2,250 MB = 2.25 GB
  ───────────────────────────────────────────────
  Total for 1 hour: 4.5 GB (dual stream)

  BUT: Circular buffer deletes OLDEST CHUNK when NEWEST is written
  Effective storage used: ~2.25 GB (only newest hour retained)
```

### Storage Retention on Different SD Capacities

| SD Capacity | Retention Duration | Events Recordable | Cost |
|---|---|---|---|
| 64 GB | 28.4 hours | ~28 separate 1-hour rides | $20 |
| **128 GB** | 56.9 hours | ~57 rides | **$35** |
| **256 GB** | 113.8 hours | ~114 rides | **$65** |
| 512 GB | 227.6 hours | ~228 rides | $120 |

**Recommendation**: 256 GB industrial-grade SD card
- Longest retention window (113+ hours of history)
- Wear leveling built-in
- SLC/MLC technology for endurance
- Cost-effective at ~$65

---

## 🔄 Circular Buffer Strategy

### Problem
"Rolling 1-hour retention" means:
- Oldest footage auto-deleted when newest footage written
- Disk usage stays below threshold (e.g., 2.5 GB)
- No manual cleanup required
- Graceful degradation if disk full

### Solution: Timestamp-Based Deletion

```python
CIRCULAR_BUFFER_THRESHOLD_MB = 2500  # 2.5 GB limit
FILE_SEGMENT_DURATION_MIN = 1        # 1-minute chunks
TARGET_RETENTION_HOURS = 1           # Latest 1 hour

class CircularBuffer:
    def __init__(self):
        self.front_files = []  # List of (filename, timestamp, size_mb)
        self.rear_files = []
        self.last_cleanup_time = 0
    
    def on_new_video_chunk(self, camera, filename, size_mb, timestamp):
        """Called when a new 1-min video chunk is written"""
        
        # 1. Register new file
        if camera == 'front':
            self.front_files.append((filename, timestamp, size_mb))
        else:
            self.rear_files.append((filename, timestamp, size_mb))
        
        # 2. Check disk usage
        total_mb = sum(s for _, _, s in self.front_files) + \
                   sum(s for _, _, s in self.rear_files)
        
        # 3. Delete oldest files if threshold exceeded
        while total_mb > CIRCULAR_BUFFER_THRESHOLD_MB:
            
            # Find oldest file (both cameras)
            front_oldest = self.front_files[0] if self.front_files else None
            rear_oldest = self.rear_files[0] if self.rear_files else None
            
            oldest = None
            if front_oldest and (not rear_oldest or front_oldest[1] < rear_oldest[1]):
                oldest = ('front', front_oldest)
                self.front_files.pop(0)
            elif rear_oldest:
                oldest = ('rear', rear_oldest)
                self.rear_files.pop(0)
            else:
                break  # No more files to delete
            
            if oldest:
                cam, (old_fname, old_ts, old_sz) = oldest
                self._delete_file(old_fname)
                total_mb -= old_sz
                print(f"Deleted {cam}:{old_fname} (freed {old_sz} MB)")
    
    def _delete_file(self, filename):
        """Atomically delete file with verification"""
        try:
            # Rename to .deleting (atomic on ext4)
            temp_name = f"{filename}.deleting"
            os.rename(filename, temp_name)
            
            # Verify deletion
            os.remove(temp_name)
            print(f"Successfully deleted: {filename}")
        except Exception as e:
            print(f"ERROR deleting {filename}: {e}")
            # Log for manual review, but continue
```

### File Naming Convention
```
Front camera:
  /blackbox/video/front/2026-07-06_10-00-00.mp4  ← timestamp = YYYY-MM-DD_HH-MM-SS
  /blackbox/video/front/2026-07-06_10-01-00.mp4  ← 1 minute later
  /blackbox/video/front/2026-07-06_10-02-00.mp4  ← etc.

Rear camera:
  /blackbox/video/rear/2026-07-06_10-00-00.mp4
  /blackbox/video/rear/2026-07-06_10-01-00.mp4
  /blackbox/video/rear/2026-07-06_10-02-00.mp4

Benefits:
  • Lexicographic ordering = chronological ordering
  • Easy to find files by date/time
  • Recovery tools can parse timestamp directly
  • No database required (filesystem is the DB)
```

---

## 📍 GPS Logging Schema

### CSV Format
```csv
timestamp,latitude,longitude,altitude_m,speed_kmh,satellites,fix_quality
2026-07-06T10:00:01Z,37.7749,-122.4194,10.5,0.0,12,3
2026-07-06T10:00:02Z,37.7749,-122.4194,10.5,2.3,12,3
2026-07-06T10:00:03Z,37.7749,-122.4195,10.6,4.1,11,3
```

### Field Definitions

| Field | Type | Range | Source | Notes |
|---|---|---|---|---|
| **timestamp** | ISO 8601 | 2026-07-06T10:00:01Z | GPS $GPRMC | UTC, 1 Hz precision |
| **latitude** | float | -90 to +90 | GPS $GPRMC | Degrees decimal |
| **longitude** | float | -180 to +180 | GPS $GPRMC | Degrees decimal |
| **altitude_m** | float | -10000 to +10000 | GPS $GPGGA | Meters above sea level |
| **speed_kmh** | float | 0 to 500 | GPS $GPRMC | Kilometers per hour |
| **satellites** | int | 0 to 24 | GPS $GPGGA | Visible satellites |
| **fix_quality** | int | 0 = invalid, 1 = GPS, 2 = DGPS, 3 = RTK | GPS $GPGGA | Accuracy indicator |

### Example NMEA Sentences Parsed
```
$GPRMC (Recommended Minimum Navigation Information)
$GPRMC,082949.000,A,3723.2475,N,12158.3416,W,0.295,286.30,060612,,,A*24
       ↓           ↓ ↓ ↓           ↓  ↓           ↓   ↓
       timestamp   status lat/lon    speed course date fix

$GPGGA (Global Positioning System Fix Data)
$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
       ↓       ↓ ↓ ↓   ↓  ↓ ↓ ↓ ↓
       time    lat lon quality sats hdop alt

Parsed into CSV row:
  timestamp: 082949.000 → 2026-07-06T08:29:49Z
  lat/lon: 3723.2475N, 12158.3416W → 37.7541, -121.9724
  altitude: 545.4 m
  speed: 0.295 knots → 0.546 km/h
  satellites: 08
  fix_quality: 1 (GPS fix)
```

### File Rotation
```
Hourly GPS logs:
  /blackbox/gps/2026-07-06_10.csv  (10:00:00 – 10:59:59, 3600 fixes)
  /blackbox/gps/2026-07-06_11.csv  (11:00:00 – 11:59:59, 3600 fixes)
  /blackbox/gps/2026-07-06_12.csv  (12:00:00 – 12:59:59, 3600 fixes)

Size per file:
  3600 fixes × 85 bytes/line ≈ 306 KB per hour
  
Circular buffer cleanup:
  • Keep only latest 1 hour of GPS logs (matching video retention)
  • Delete 2026-07-06_10.csv when 2026-07-06_12.csv becomes complete
```

---

## 🕐 Time Synchronization Strategy

### Challenge
Bike environment = no NTP/Internet access
- System clock drifts (RTC battery depleted, oscillator error)
- GPS provides accurate UTC time (±1ms typical)
- Video timestamps must align with GPS timestamps (±100ms)

### Boot-Time Sync
```
1. System boots with stale system clock (RTC may be days old)
   Scenario: 2026-07-06 08:30:00 (stale RTC)
             2026-07-06 14:32:15 (actual GPS time)
             Offset: -6 hours 2 minutes

2. GPS module starts acquiring fix (~30-60 sec cold start)
   While waiting: system proceeds with stale time
   Video/GPS timestamps will be misaligned initially

3. Once GPS fix acquired:
   Parse $GPRMC sentence → GPS time = 2026-07-06 14:32:15
   Compare with system time → offset = 6.03 hours
   If |offset| > 100 ms: ADJUST system clock
   
   Adjustment:
   $ timedatectl set-time '2026-07-06 14:32:15'
   or
   $ date -s '2026-07-06 14:32:15'

4. Log adjustment:
   ✓ Clock corrected at 2026-07-06T14:32:15Z (+6.03 hours)
   
5. Continue normal operation:
   Video chunks timestamped with corrected system clock
   GPS logs timestamped with GPS NMEA sentences
   Drift monitoring: resync every 60 minutes (or if drift > 1 sec)
```

### Continuous Drift Monitoring
```python
class TimeSync:
    def __init__(self):
        self.last_sync_time = time.time()
        self.last_sync_gps_time = None
        self.offset_ms = 0
    
    def sample_gps(self, gps_sentence):
        """Sample GPS time every 60 seconds"""
        gps_time = parse_nmea(gps_sentence)  # Returns Unix timestamp
        sys_time = time.time()
        
        # Calculate drift
        if self.last_sync_gps_time:
            elapsed_gps = (gps_time - self.last_sync_gps_time)  # Should be ~60s
            elapsed_sys = (sys_time - self.last_sync_time)      # Actual elapsed
            
            drift_ppm = (elapsed_sys - elapsed_gps) / elapsed_gps * 1e6
            print(f"Clock drift: {drift_ppm:.1f} ppm")
            
            # If drift > 1 second over 60 seconds (16,667 ppm), resync
            if abs(drift_ppm) > 16667:
                self.resync_clock(gps_time)
        
        self.last_sync_time = sys_time
        self.last_sync_gps_time = gps_time
    
    def resync_clock(self, gps_time):
        """Resync system clock if drift detected"""
        subprocess.run(['timedatectl', 'set-time', 
                       datetime.utcfromtimestamp(gps_time).isoformat() + 'Z'],
                       check=True)
        print(f"Clock resynced at {gps_time}")
```

### Timestamp Embedding in Video
```
MP4 File Structure:
┌─────────────────────────────────┐
│ ftyp (file type)                │
├─────────────────────────────────┤
│ mdat (media data)               │
│   [H.264 bitstream]             │ ← Actual video frames
├─────────────────────────────────┤
│ moov (movie metadata)           │
│   trak (track)                  │
│     tkhd (track header)         │  ← Duration, width, height
│     edts (edit list)            │  ← Media time mapping
│     mdia (media)                │
│       mdhd (media header)       │  ← Timescale (30000 = 30fps)
│       minf (media info)         │
│         stbl (sample table)     │
│           stts (time-to-sample) │  ← Frame timestamps
│           stsz (sample sizes)   │
│           stco (chunk offsets)  │
└─────────────────────────────────┘

FFmpeg command to embed timestamps:
$ ffmpeg -f v4l2 -r 30 -input_format yuyv422 \
         -i /dev/video0 \
         -c:v h264_nvenc \
         -b:v 5M \
         -start_number 0 \
         -metadata creation_time="$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
         output.mp4
```

---

## 📁 Filesystem & Wear Considerations

### ext4 Filesystem Configuration
```bash
# Format SD card with ext4
$ sudo mkfs.ext4 -L blackbox -m 5 /dev/mmcblk0p2

# Options:
#   -L blackbox     → Label partition
#   -m 5            → Reserve 5% for filesystem recovery

# Mount with optimized options
cat >> /etc/fstab <<EOF
/dev/mmcblk0p2  /blackbox  ext4  defaults,noatime,nodiratime,commit=60,errors=remount-ro  0  2
EOF

# Explanation:
#   noatime         → Don't update file access time (reduces writes)
#   nodiratime      → Don't update directory access time
#   commit=60       → Sync journal every 60 seconds (vs. default 5)
#   errors=remount-ro → If error, remount read-only (safer than crash)
```

### SD Card Wear Management

```
Wear Pattern Analysis:

1. Video Files (Continuous Writes)
   - 1-minute chunks, ~300 MB each
   - Written sequentially (low wear pattern)
   - Deleted after 60 files (~1 hour)
   
2. GPS Logs (Periodic Writes)
   - ~10 KB per second (low bitrate)
   - Buffered every 100 fixes or 10 seconds
   - Wear: LOW (mostly reads during rotation)

3. System Logs
   - Minimal writes (errors only)
   - Wear: MINIMAL

Estimated Card Lifespan:
  SD Card: 256 GB, industrial-grade SLC
  Endurance: 3000 P/E cycles per cell (guaranteed)
  Effective writes per year: ~10 TB (typical bike usage)
  
  Lifespan: (256 GB × 3000 P/E) / 10 TB/yr = ~77 years
  (Practical reality: 5-10 years due to other factors)
```

### Backup & Recovery
```bash
# Periodic backup of GPS logs (manual or automated)
$ rsync -av /blackbox/gps/ /external/gps_backup/

# Monitor filesystem health
$ smartctl -a /dev/mmcblk0  # Requires smartmontools
$ df -h /blackbox            # Disk usage
$ e2fsck -n /dev/mmcblk0p2   # Dry-run filesystem check

# Emergency recovery after sudden shutdown
$ sudo fsck -y /dev/mmcblk0p2  # Force filesystem repair
```

---

## 📦 Complete Storage Layout

```
/blackbox/ (256 GB, ext4)
├── video/ (2.25 GB max active)
│   ├── front/
│   │   ├── 2026-07-06_10-00-00.mp4  (5 MB, H.264 1-min chunk)
│   │   ├── 2026-07-06_10-01-00.mp4
│   │   ├── 2026-07-06_10-02-00.mp4
│   │   └── ... (up to 60 files per hour)
│   │
│   └── rear/
│       ├── 2026-07-06_10-00-00.mp4
│       ├── 2026-07-06_10-01-00.mp4
│       └── ... (same as front)
│
├── gps/ (306 KB max active)
│   ├── 2026-07-06_10.csv  (3600 fixes, 306 KB)
│   ├── 2026-07-06_11.csv
│   └── 2026-07-06_12.csv  (oldest deleted when new hour starts)
│
├── logs/ (10 MB max active)
│   ├── system.log      (systemd journal, rotated daily)
│   ├── camera.log      (camera errors, rotated at 10 MB)
│   ├── gps.log         (GPS debug output)
│   └── storage.log     (deletion events, one per day)
│
└── state/ (1 MB)
    ├── last_boot.txt        (ISO timestamp of last boot)
    ├── disk_usage.json      (current usage snapshot)
    ├── recovery.lock        (exists if unclean shutdown detected)
    └── config.json          (system settings)
```

---

**Document**: 03-STORAGE-DATA-DESIGN.md  
**Version**: 1.0  
**Date**: 2026-07-06  
**Status**: Complete