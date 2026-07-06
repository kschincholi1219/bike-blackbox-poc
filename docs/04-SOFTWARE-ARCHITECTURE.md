# 4. SOFTWARE ARCHITECTURE

## 🏗️ Modular Service Design

### Service Decomposition

```
┌─────────────────────────────────────────────────────────┐
│                 SYSTEMD ORCHESTRATOR                     │
│              (service manager, watchdog)                │
└────────┬───────────────────────────────────┬────────────┘
         │                                   │
    ┌────▼────────┐  ┌────────────────┐  ┌──▼──────────┐
    │  Main.py    │  │ Watchdog.py    │  │ Recovery.py │
    │  (launcher) │  │ (health mon)   │  │ (boot fix)  │
    └────┬────────┘  └────────────────┘  └─────────────┘
         │
    ┌────▼─────────────────────────────────────┐
    │    CORE SERVICES (Python asyncio/threads)│
    ├──────────────────────────────────────────┤
    │                                          │
    │  ┌─ Front_Camera_Service ─────────────┐ │
    │  │ • V4L2 device init                │ │
    │  │ • H.264 encoder (MMAL/VPU)        │ │
    │  │ • Frame buffer management         │ │
    │  │ • FPS throttling                  │ │
    │  │ • Error recovery (USB disconnect) │ │
    │  └──────────────┬────────────────────┘ │
    │                 │                      │
    │  ┌──────────────▼──────────────────┐  │
    │  │ Rear_Camera_Service             │  │
    │  │ (identical to Front)            │  │
    │  └──────────────┬──────────────────┘  │
    │                 │                      │
    │  ┌──────────────▼──────────────────┐  │
    │  │ GPS_Logger_Service              │  │
    │  │ • UART polling (9600 baud)      │  │
    │  │ • NMEA sentence parsing         │  │
    │  │ • CSV buffering & write         │  │
    │  │ • Time sync logic               │  │
    │  └──────────────┬──────────────────┘  │
    │                 │                      │
    │  ┌──────────────▼──────────────────┐  │
    │  │ Storage_Manager_Service         │  │
    │  │ • Circular buffer impl          │  │
    │  │ • Disk usage monitoring         │  │
    │  │ • Atomic file ops               │  │
    │  │ • Cleanup/deletion logic        │  │
    │  └──────────────┬──────────────────┘  │
    │                 │                      │
    │  ┌──────────────▼──────────────────┐  │
    │  │ Watchdog_Service                │  │
    │  │ • Process health checks         │  │
    │  │ • Heartbeat monitors            │  │
    │  │ • Graceful shutdown handler     │  │
    │  │ • Recovery coordination         │  │
    │  └─────────────────────────────────┘  │
    │                                          │
    └──────────────────────────────────────────┘
         │
    ┌────▼────────────────────────────────┐
    │    PERSISTENCE LAYER                │
    │  (SD Card filesystem, SD SPI iface) │
    │  /blackbox/{video,gps,logs,state}  │
    └─────────────────────────────────────┘
```

---

## 🎬 Camera Capture Pipeline (Pseudocode)

### Front Camera Service
```python
# front_camera.py
import cv2
import threading
import time
from collections import deque
from datetime import datetime

class FrontCameraService:
    def __init__(self, device_path="/dev/video0", target_fps=30, bitrate_mbps=5):
        self.device_path = device_path
        self.target_fps = target_fps
        self.bitrate_mbps = bitrate_mbps
        self.cap = None
        self.is_running = False
        self.frame_queue = deque(maxlen=10)  # Buffer up to 10 frames
        self.stats = {'frames_captured': 0, 'frames_dropped': 0, 'errors': 0}
        
    def start(self):
        """Initialize camera and start capture thread"""
        try:
            # Open V4L2 device
            self.cap = cv2.VideoCapture(self.device_path)
            self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
            self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
            self.cap.set(cv2.CAP_PROP_FPS, self.target_fps)
            self.cap.set(cv2.CAP_PROP_BUFFERSIZE, 1)  # Minimize latency
            
            self.is_running = True
            self.capture_thread = threading.Thread(target=self._capture_loop, daemon=True)
            self.capture_thread.start()
            print(f"[Front] Camera started: {self.device_path}")
        except Exception as e:
            print(f"[Front] Error initializing camera: {e}")
            self.stats['errors'] += 1
    
    def _capture_loop(self):
        """Main capture loop (runs in thread)"""
        frame_delay = 1.0 / self.target_fps  # ~33ms for 30fps
        
        while self.is_running:
            try:
                ret, frame = self.cap.read()
                if not ret:
                    self.stats['frames_dropped'] += 1
                    time.sleep(0.001)  # Brief sleep before retry
                    continue
                
                # Add timestamp to frame metadata
                timestamp = datetime.utcnow().isoformat()
                frame_info = {
                    'frame': frame,
                    'timestamp': timestamp,
                    'sequence': self.stats['frames_captured']
                }
                
                # Buffer frame
                if len(self.frame_queue) < self.frame_queue.maxlen:
                    self.frame_queue.append(frame_info)
                else:
                    self.stats['frames_dropped'] += 1
                
                self.stats['frames_captured'] += 1
                
                # FPS throttling
                time.sleep(frame_delay)
                
            except Exception as e:
                print(f"[Front] Capture error: {e}")
                self.stats['errors'] += 1
                time.sleep(0.01)
    
    def get_frame(self):
        """Retrieve next buffered frame (FIFO)"""
        if self.frame_queue:
            return self.frame_queue.popleft()
        return None
    
    def stop(self):
        """Gracefully stop capture"""
        self.is_running = False
        if self.capture_thread:
            self.capture_thread.join(timeout=2)
        if self.cap:
            self.cap.release()
        print(f"[Front] Stopped. Stats: {self.stats}")
    
    def get_stats(self):
        """Return capture statistics for monitoring"""
        return self.stats

# Rear camera: identical class (RearCameraService)

if __name__ == "__main__":
    front = FrontCameraService(device_path="/dev/video0")
    front.start()
    
    # Capture for 10 seconds
    for _ in range(300):  # 300 frames @ 30fps = 10 sec
        frame_info = front.get_frame()
        if frame_info:
            print(f"Got frame {frame_info['sequence']} at {frame_info['timestamp']}")
        time.sleep(0.033)
    
    front.stop()
```

---

## 📍 GPS Logger Pipeline (Pseudocode)

```python
# gps_logger.py
import serial
import threading
import csv
import time
from datetime import datetime
from collections import deque

class GPSLogger:
    def __init__(self, uart_port="/dev/ttyUSB0", baud=9600, csv_path="/blackbox/gps/"):
        self.uart_port = uart_port
        self.baud = baud
        self.csv_path = csv_path
        self.ser = None
        self.is_running = False
        self.gps_queue = deque(maxlen=100)  # Buffer GPS fixes
        self.csv_file = None
        self.csv_writer = None
        self.stats = {'fixes_received': 0, 'parse_errors': 0, 'writes': 0}
        
    def start(self):
        """Open UART port and start GPS ingest thread"""
        try:
            self.ser = serial.Serial(self.uart_port, self.baud, timeout=1)
            self.is_running = True
            self.ingest_thread = threading.Thread(target=self._ingest_loop, daemon=True)
            self.ingest_thread.start()
            print(f"[GPS] Logger started: {self.uart_port} @ {self.baud} baud")
        except Exception as e:
            print(f"[GPS] Error opening UART: {e}")
    
    def _ingest_loop(self):
        """Main GPS ingest loop (runs in thread)"""
        buffer_time = time.time()
        
        while self.is_running:
            try:
                # Read NMEA sentence from GPS module
                if self.ser.in_waiting:
                    line = self.ser.readline().decode('utf-8', errors='ignore').strip()
                    
                    # Parse NMEA sentences (filter for RMC + GGA)
                    if line.startswith('$GPRMC') or line.startswith('$GPGGA'):
                        parsed = self._parse_nmea(line)
                        if parsed:
                            self.gps_queue.append(parsed)
                            self.stats['fixes_received'] += 1
                    
                    # Buffer and write to CSV every 10 fixes or 10 seconds
                    if len(self.gps_queue) >= 10 or (time.time() - buffer_time) > 10:
                        self._flush_csv()
                        buffer_time = time.time()
                else:
                    time.sleep(0.1)
                    
            except Exception as e:
                print(f"[GPS] Ingest error: {e}")
                self.stats['parse_errors'] += 1
                time.sleep(0.5)
    
    def _parse_nmea(self, sentence):
        """Parse NMEA sentence into dict"""
        try:
            parts = sentence.split(',')
            
            if sentence.startswith('$GPRMC'):
                # RMC: Recommended Minimum Navigation
                # $GPRMC,082949.000,A,3723.2475,N,12158.3416,W,0.295,286.30,060612,,,A*24
                if len(parts) >= 9 and parts[2] == 'A':  # A = active (valid fix)
                    time_str = parts[1]  # 082949.000
                    lat_str = parts[3]   # 3723.2475
                    lat_dir = parts[4]   # N/S
                    lon_str = parts[5]   # 12158.3416
                    lon_dir = parts[6]   # E/W
                    speed_knots = float(parts[7])
                    
                    lat = self._dms_to_decimal(lat_str, lat_dir)
                    lon = self._dms_to_decimal(lon_str, lon_dir)
                    speed_kmh = speed_knots * 1.852
                    
                    return {
                        'timestamp': datetime.utcnow().isoformat() + 'Z',
                        'latitude': lat,
                        'longitude': lon,
                        'altitude_m': None,  # Will be filled by GGA
                        'speed_kmh': speed_kmh,
                        'satellites': None,
                        'fix_quality': 1
                    }
            
            elif sentence.startswith('$GPGGA'):
                # GGA: Global Positioning System Fix Data
                # $GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47
                if len(parts) >= 9:
                    lat_str = parts[2]
                    lat_dir = parts[3]
                    lon_str = parts[4]
                    lon_dir = parts[5]
                    fix_quality = int(parts[6])
                    satellites = int(parts[7])
                    altitude = float(parts[9]) if parts[9] else 0
                    
                    lat = self._dms_to_decimal(lat_str, lat_dir)
                    lon = self._dms_to_decimal(lon_str, lon_dir)
                    
                    return {
                        'timestamp': datetime.utcnow().isoformat() + 'Z',
                        'latitude': lat,
                        'longitude': lon,
                        'altitude_m': altitude,
                        'speed_kmh': 0,  # Not in GGA
                        'satellites': satellites,
                        'fix_quality': fix_quality
                    }
        except Exception as e:
            print(f"[GPS] Parse error on '{sentence}': {e}")
            self.stats['parse_errors'] += 1
        
        return None
    
    def _dms_to_decimal(self, dms_str, direction):
        """Convert DMS (degrees, minutes, seconds) to decimal"""
        # 3723.2475 = 37° 23.2475'
        degrees = int(dms_str[:-7])
        minutes = float(dms_str[-7:])
        decimal = degrees + minutes / 60.0
        
        if direction in ['S', 'W']:
            decimal = -decimal
        return decimal
    
    def _flush_csv(self):
        """Write buffered GPS data to CSV file"""
        if not self.gps_queue:
            return
        
        try:
            # Determine hourly filename
            now = datetime.utcnow()
            csv_filename = f"{self.csv_path}{now.strftime('%Y-%m-%d_%H')}.csv"
            
            # Open or create CSV file
            if not self.csv_file or self.csv_file.name != csv_filename:
                if self.csv_file:
                    self.csv_file.close()
                
                import os
                os.makedirs(self.csv_path, exist_ok=True)
                
                # Check if file exists to skip header
                file_exists = os.path.isfile(csv_filename)
                self.csv_file = open(csv_filename, 'a', newline='')
                self.csv_writer = csv.DictWriter(self.csv_file, 
                    fieldnames=['timestamp', 'latitude', 'longitude', 'altitude_m', 'speed_kmh', 'satellites', 'fix_quality'])
                
                if not file_exists:
                    self.csv_writer.writeheader()
            
            # Flush queue to CSV
            while self.gps_queue:
                row = self.gps_queue.popleft()
                self.csv_writer.writerow(row)
            
            self.csv_file.flush()  # Sync to disk
            self.stats['writes'] += 1
            
        except Exception as e:
            print(f"[GPS] CSV write error: {e}")
    
    def stop(self):
        """Gracefully stop GPS logger"""
        self._flush_csv()  # Final flush
        self.is_running = False
        if self.ingest_thread:
            self.ingest_thread.join(timeout=2)
        if self.csv_file:
            self.csv_file.close()
        if self.ser:
            self.ser.close()
        print(f"[GPS] Stopped. Stats: {self.stats}")

if __name__ == "__main__":
    gps = GPSLogger(uart_port="/dev/ttyUSB0")
    gps.start()
    
    # Run for 60 seconds
    time.sleep(60)
    
    gps.stop()
```

---

## 💾 Storage Manager Pseudocode

```python
# storage_manager.py
import os
import json
from datetime import datetime
from pathlib import Path

class StorageManager:
    def __init__(self, base_path="/blackbox/", threshold_mb=2500):
        self.base_path = Path(base_path)
        self.threshold_mb = threshold_mb
        self.front_files = []
        self.rear_files = []
        self.stats = {'deleted': 0, 'freed_mb': 0}
    
    def on_new_chunk(self, camera, filename, size_mb):
        """Called when a new video chunk is finalized"""
        try:
            # Register file
            if camera == 'front':
                self.front_files.append((filename, datetime.now().timestamp(), size_mb))
            else:
                self.rear_files.append((filename, datetime.now().timestamp(), size_mb))
            
            # Check disk usage
            total_mb = sum(s for _, _, s in self.front_files) + sum(s for _, _, s in self.rear_files)
            
            # Delete oldest files if threshold exceeded
            while total_mb > self.threshold_mb:
                oldest = self._find_oldest_file()
                if oldest:
                    cam, fname, ts, sz = oldest
                    self._delete_file(fname)
                    
                    if cam == 'front':
                        self.front_files = [(f, t, s) for f, t, s in self.front_files if f != fname]
                    else:
                        self.rear_files = [(f, t, s) for f, t, s in self.rear_files if f != fname]
                    
                    total_mb -= sz
                    self.stats['freed_mb'] += sz
                    self.stats['deleted'] += 1
                    print(f"[Storage] Deleted {cam}:{Path(fname).name} (freed {sz} MB)")
                else:
                    break
            
            # Log usage snapshot
            self._log_usage(total_mb)
            
        except Exception as e:
            print(f"[Storage] Error on new chunk: {e}")
    
    def _find_oldest_file(self):
        """Find oldest file across both cameras"""
        candidates = []
        for fname, ts, sz in self.front_files:
            candidates.append(('front', fname, ts, sz))
        for fname, ts, sz in self.rear_files:
            candidates.append(('rear', fname, ts, sz))
        
        if candidates:
            return min(candidates, key=lambda x: x[2])  # Sort by timestamp
        return None
    
    def _delete_file(self, filepath):
        """Atomically delete file"""
        try:
            temp_path = f"{filepath}.deleting"
            os.rename(filepath, temp_path)  # Atomic on ext4
            os.remove(temp_path)
            print(f"[Storage] ✓ Deleted {Path(filepath).name}")
        except Exception as e:
            print(f"[Storage] ✗ Delete failed for {filepath}: {e}")
    
    def _log_usage(self, total_mb):
        """Log disk usage to state file"""
        try:
            state = {
                'timestamp': datetime.utcnow().isoformat() + 'Z',
                'total_mb': total_mb,
                'threshold_mb': self.threshold_mb,
                'front_files': len(self.front_files),
                'rear_files': len(self.rear_files)
            }
            
            state_path = self.base_path / 'state' / 'disk_usage.json'
            state_path.parent.mkdir(parents=True, exist_ok=True)
            with open(state_path, 'w') as f:
                json.dump(state, f, indent=2)
        except Exception as e:
            print(f"[Storage] State log error: {e}")
```

---

## 🛡️ Watchdog & Recovery Logic

```python
# watchdog.py
import threading
import time
import signal
import os
from datetime import datetime

class Watchdog:
    def __init__(self, services, timeout_sec=30):
        self.services = services  # Dict of service_name -> service_object
        self.timeout_sec = timeout_sec
        self.is_running = False
        self.last_heartbeat = {name: time.time() for name in services.keys()}
    
    def start(self):
        """Start watchdog monitoring thread"""
        self.is_running = True
        self.monitor_thread = threading.Thread(target=self._monitor_loop, daemon=True)
        self.monitor_thread.start()
        print("[Watchdog] Started")
    
    def _monitor_loop(self):
        """Periodically check service health"""
        while self.is_running:
            for name, service in self.services.items():
                if hasattr(service, 'is_running'):
                    if service.is_running:
                        self.last_heartbeat[name] = time.time()
                    else:
                        elapsed = time.time() - self.last_heartbeat.get(name, 0)
                        if elapsed > self.timeout_sec:
                            print(f"[Watchdog] ✗ Timeout detected for {name}")
                            self._emergency_shutdown()
                            break
            
            time.sleep(5)  # Check every 5 seconds
    
    def _emergency_shutdown(self):
        """Graceful shutdown on deadlock detected"""
        print("[Watchdog] Initiating graceful shutdown...")
        
        # Send SIGTERM to all services (30 second grace period)
        for name, service in self.services.items():
            if hasattr(service, 'stop'):
                try:
                    service.stop()
                    print(f"[Watchdog] Stopped {name}")
                except Exception as e:
                    print(f"[Watchdog] Error stopping {name}: {e}")
        
        # Wait 30 seconds for clean shutdown
        time.sleep(30)
        
        # If still running, force exit
        print("[Watchdog] Forcing shutdown...")
        os._exit(1)
    
    def stop(self):
        self.is_running = False
        if self.monitor_thread:
            self.monitor_thread.join(timeout=2)
        print("[Watchdog] Stopped")

# Signal handlers for graceful shutdown
def setup_signal_handlers(services):
    def handle_sigterm(signum, frame):
        print(f"\n[Main] Received SIGTERM, gracefully shutting down...")
        for service in services.values():
            if hasattr(service, 'stop'):
                service.stop()
        print("[Main] Shutdown complete")
        os._exit(0)
    
    signal.signal(signal.SIGTERM, handle_sigterm)
    signal.signal(signal.SIGINT, handle_sigterm)  # Also handle Ctrl+C
```

---

## 🔧 Boot-Time Recovery Logic

```python
# recovery.py
import os
import json
from pathlib import Path
from datetime import datetime

class RecoveryManager:
    def __init__(self, base_path="/blackbox/"):
        self.base_path = Path(base_path)
    
    def check_unclean_shutdown(self):
        """Detect unclean shutdown on boot"""
        recovery_lock = self.base_path / 'state' / 'recovery.lock'
        
        if recovery_lock.exists():
            print("[Recovery] Unclean shutdown detected")
            return True
        
        return False
    
    def validate_video_files(self):
        """Validate MP4 video files for corruption"""
        print("[Recovery] Validating video files...")
        
        video_dir = self.base_path / 'video'
        corrupted = []
        
        for camera in ['front', 'rear']:
            cam_dir = video_dir / camera
            if cam_dir.exists():
                for mp4_file in cam_dir.glob('*.mp4'):
                    if not self._is_valid_mp4(mp4_file):
                        corrupted.append(str(mp4_file))
        
        if corrupted:
            print(f"[Recovery] Found {len(corrupted)} corrupted files")
            for fname in corrupted:
                print(f"  - {fname} (to be reviewed)")
        else:
            print("[Recovery] All video files valid")
        
        return corrupted
    
    def validate_gps_logs(self):
        """Validate GPS CSV logs for corruption"""
        print("[Recovery] Validating GPS logs...")
        
        gps_dir = self.base_path / 'gps'
        issues = []
        
        if gps_dir.exists():
            for csv_file in gps_dir.glob('*.csv'):
                try:
                    with open(csv_file, 'r') as f:
                        lines = f.readlines()
                        if len(lines) < 2:  # No data rows
                            issues.append((str(csv_file), 'empty'))
                except Exception as e:
                    issues.append((str(csv_file), str(e)))
        
        if issues:
            print(f"[Recovery] Found {len(issues)} GPS log issues")
            for fname, issue in issues:
                print(f"  - {fname}: {issue}")
        else:
            print("[Recovery] All GPS logs valid")
        
        return issues
    
    def _is_valid_mp4(self, filepath):
        """Check if MP4 file has valid header/footer"""
        try:
            with open(filepath, 'rb') as f:
                # Check for ftyp (file type) box
                header = f.read(8)
                if not header.startswith(b'\x00\x00\x00\x18ftyp'):
                    return False
                
                # Check file size > 1 MB (arbitrary minimum)
                f.seek(0, 2)  # Seek to end
                size = f.tell()
                return size > 1_000_000
        except:
            return False
    
    def create_recovery_lock(self):
        """Mark boot as in-progress (remove on clean shutdown)"""
        lock_path = self.base_path / 'state' / 'recovery.lock'
        lock_path.parent.mkdir(parents=True, exist_ok=True)
        with open(lock_path, 'w') as f:
            f.write(datetime.utcnow().isoformat())
    
    def remove_recovery_lock(self):
        """Mark shutdown as clean"""
        lock_path = self.base_path / 'state' / 'recovery.lock'
        if lock_path.exists():
            lock_path.unlink()
    
    def save_last_boot_time(self):
        """Log last successful boot timestamp"""
        boot_path = self.base_path / 'state' / 'last_boot.txt'
        boot_path.parent.mkdir(parents=True, exist_ok=True)
        with open(boot_path, 'w') as f:
            f.write(datetime.utcnow().isoformat() + 'Z')
```

---

## 🚀 Main Orchestrator (Pseudocode)

```python
# main.py
import logging
import sys
from pathlib import Path

from front_camera import FrontCameraService
from rear_camera import RearCameraService
from gps_logger import GPSLogger
from storage_manager import StorageManager
from watchdog import Watchdog, setup_signal_handlers
from recovery import RecoveryManager

def setup_logging():
    """Configure system logging to SD card"""
    log_dir = Path('/blackbox/logs')
    log_dir.mkdir(parents=True, exist_ok=True)
    
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
        handlers=[
            logging.FileHandler(log_dir / 'system.log'),
            logging.StreamHandler(sys.stdout)
        ]
    )

def main():
    logger = logging.getLogger('main')
    logger.info("=" * 60)
    logger.info("Bike Black Box System - Startup")
    logger.info("=" * 60)
    
    # 1. Recovery check
    recovery_mgr = RecoveryManager()
    recovery_mgr.create_recovery_lock()
    
    if recovery_mgr.check_unclean_shutdown():
        logger.warning("Unclean shutdown detected, validating files...")
        corrupted = recovery_mgr.validate_video_files()
        gps_issues = recovery_mgr.validate_gps_logs()
    
    recovery_mgr.save_last_boot_time()
    
    # 2. Initialize services
    logger.info("Initializing services...")
    
    front_cam = FrontCameraService(device_path="/dev/video0", target_fps=30, bitrate_mbps=5)
    rear_cam = RearCameraService(device_path="/dev/video1", target_fps=30, bitrate_mbps=5)
    gps_logger = GPSLogger(uart_port="/dev/ttyUSB0", baud=9600)
    storage_mgr = StorageManager(base_path="/blackbox/", threshold_mb=2500)
    
    services = {
        'front_camera': front_cam,
        'rear_camera': rear_cam,
        'gps_logger': gps_logger,
        'storage_manager': storage_mgr
    }
    
    # 3. Start services
    logger.info("Starting services...")
    front_cam.start()
    rear_cam.start()
    gps_logger.start()
    
    # 4. Start watchdog
    watchdog = Watchdog(services=services, timeout_sec=30)
    watchdog.start()
    setup_signal_handlers(services)
    
    # 5. Main loop (encode & store video, manage storage)
    logger.info("Entering main loop...")
    
    while True:
        try:
            # Process front camera frames
            front_frame = front_cam.get_frame()
            if front_frame:
                # Encode and segment (pseudocode)
                # ... H.264 encoding logic ...
                # On 1-minute boundary:
                # storage_mgr.on_new_chunk('front', filename, size_mb)
                pass
            
            # Process rear camera frames (identical)
            rear_frame = rear_cam.get_frame()
            if rear_frame:
                # Encode and segment
                pass
            
            # GPS is handled by background thread (automatic flushing)
            
            # Sleep brief interval
            time.sleep(0.01)
            
        except KeyboardInterrupt:
            logger.info("Received interrupt, shutting down...")
            break
        except Exception as e:
            logger.error(f"Main loop error: {e}")
            time.sleep(1)
    
    # 6. Graceful shutdown
    logger.info("Shutting down services...")
    watchdog.stop()
    front_cam.stop()
    rear_cam.stop()
    gps_logger.stop()
    recovery_mgr.remove_recovery_lock()
    
    logger.info("Shutdown complete")
    sys.exit(0)

if __name__ == "__main__":
    main()
```

---

**Document**: 04-SOFTWARE-ARCHITECTURE.md  
**Version**: 1.0  
**Date**: 2026-07-06  
**Status**: Complete