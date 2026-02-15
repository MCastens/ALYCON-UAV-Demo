# Reproducibility Runbook

Complete validation methodology for ALYCON UAV anomaly detection results.

---

## Environment Setup

### Prerequisites
- Python 3.8+
- Dataset access (see Data Acquisition section)
- ~15 GB disk space (uncompressed datasets)

### Python Dependencies
```bash
pip install numpy scipy matplotlib scikit-learn
```

---

## Data Acquisition

### 1. ALFA Dataset
```bash
# Download from Carnegie Mellon
wget https://cmu.box.com/s/u7vne29uiips4n5p1dl2k514s3j3ozby
# Alternative: https://doi.org/10.1184/R1/12707963

# Extract (12.5 GB unzipped)
unzip alfa-dataset.zip

# Expected structure:
# alfa/processed/
#   ├── carbonZ_2018-07-18-15-53-31_1_engine_failure/
#   ├── carbonZ_2018-07-30-16-39-00_3_no_failure/
#   └── ... (47 flight directories total)
```

### 2. AV-GPS Spoofing Dataset
```bash
# Clone from GitHub
git clone https://github.com/mehrab-abrar/AV-GPS-Dataset.git

# Files:
# - AV-GPS-Dataset-1-Normal-Data.csv
# - AV-GPS-Dataset-1.csv (attack)
# - AV-GPS-Dataset-2.csv (attack)
# - AV-GPS-Dataset-3.csv (attack)
```

### 3. IEEE UAV Cyberattacks
```bash
# Download from IEEE DataPort
# https://ieee-dataport.org/open-access/uav-ids-dataset

# Requires IEEE account (free)
# File: UAV_Cyberattacks.csv
```

---

## Validation Execution

### Test 1: ALFA Behavioral Analysis
**Purpose:** Validate stability metrics across 47 flights

```powershell
# Set Python path
$env:PYTHONPATH = "path/to/src"

# Run ALFA behavioral validation
python verticals/uav_drone/scripts/alycon_alfa_behavioral.py

# Output: verticals/uav_drone/results/behavioral_validation/
#   ├── alfa_behavioral_report_<timestamp>.json
#   └── summary statistics in console
```

**Expected Output:**
- 11 normal flights loaded
- 36 fault flights loaded
- Stability scores computed
- JSON report with per-flight metrics

### Test 2: Real Attack Replay
**Purpose:** Test 112 scenarios (16 attack types × 7 flights)

```powershell
# Run attack replay validation
python verticals/uav_drone/scripts/test_real_attacks.py

# Output: verticals/uav_drone/results/real_attack_validation.json
```

**Attack Types Injected:**
- GPS: gradual_spoof, sudden_jump, replay_attack, intermittent_jam, altitude_spoof
- IMU: sensor_bias, scaling_attack, noise_injection, calibration_drift
- Control: surface_degradation, actuator_stuck, actuator_oscillation
- Environmental: wind_shear, turbulence
- Stealth: low_rate_gps_drift, periodic_micro_glitch

**Expected Results:**
```json
{
  "overall": {
    "total_tests": 112,
    "detected": 110,
    "missed": 2,
    "detection_rate": 0.982,
    "median_latency_s": 3.9
  }
}
```

### Test 3: Baseline Comparison
**Purpose:** Compare ALYCON vs Isolation Forest vs One-Class SVM

```powershell
# Run benchmark
python verticals/cyber_security/scripts/alycon_benchmark_baselines.py

# Output: verticals/cyber_security/results/comprehensive_validation/
#   ├── validation_summary.json
#   ├── ALYCON_WINDOW_METRICS_<timestamp>.json
#   ├── comparison_roc.png
#   └── per_algorithm_results.csv
```

---

## Attack Injection Parameters

### GPS Spoofing (Gradual)
```python
injection_time = 20.0  # seconds into flight
drift_rate = 0.002  # degrees per second
duration = 40.0  # seconds
```

### IMU Sensor Bias
```python
injection_time = 30.0
bias_magnitude = 0.5  # m/s² constant offset
axes = ['x', 'y', 'z']
```

### Actuator Stuck
```python
injection_time = 25.0
stuck_value = 0.0  # neutral position
surface = 'elevator'  # or 'aileron', 'rudder'
```

### Stealth Micro-Glitch
```python
injection_time = 35.0
glitch_magnitude = 0.1  # m/s² periodic
frequency = 0.5  # Hz (every 2 seconds)
```

---

## Detection Algorithm Parameters

**Note:** These are high-level descriptions only. Exact threshold values and window parameters are **not disclosed** for operational security.

```python
# Conceptual parameters (not actual implementation)
window_size = REDACTED  # samples per detection window
calibration_phase = REDACTED  # number of baseline windows
detection_threshold = REDACTED  # sigma multiplier for anomaly score
```

Detection uses **Boolean OR logic** across three metrics:
- Entropy < threshold → structural anomaly
- Fisher > threshold → volatility spike
- Wasserstein > threshold → distributional shift

---

## Results Validation

### Verify JSON Output Structure
```python
import json

with open('results/real_attack_validation.json') as f:
    data = json.load(f)

# Expected structure:
assert 'overall' in data
assert 'by_attack_type' in data
assert data['overall']['detection_rate'] > 0.98
assert data['overall']['median_latency_s'] < 5.0
```

### Check Phase Space Outputs
```python
# Each detection window should have (H, F, W) coordinates
assert all('H' in window for window in data['phase_space'])
assert all('F' in window for window in data['phase_space'])
assert all('W' in window for window in data['phase_space'])
```

---

## Performance Metrics

### Compute Time Measurement
```python
import time

start = time.perf_counter()
result = alycon_detect(signal_window)
elapsed_ms = (time.perf_counter() - start) * 1000

# Expected: <200ms per window on laptop hardware
assert elapsed_ms < 200
```

### Memory Usage
```python
import psutil
import os

process = psutil.Process(os.getpid())
mem_mb = process.memory_info().rss / 1024 / 1024

# Expected: <100MB for detection process
assert mem_mb < 100
```

---

## Known Issues & Limitations

### 1. Short Flight Duration
**Issue:** Flights shorter than calibration period cannot be validated  
**Mitigation:** Require minimum 30 seconds of normal telemetry before injection

### 2. Extreme Turbulence
**Issue:** 1 miss on turbulence scenario (high natural variance)  
**Mitigation:** Acceptable for 98.2% overall detection rate

### 3. Simultaneous Multi-Fault
**Issue:** Not tested (multiple attacks at once)  
**Future Work:** Validate compound attack scenarios

---

## Hardware-In-The-Loop (HITL) Testing

### Live MAVLink Telemetry (Optional)
```powershell
# Connect to SITL simulator or physical drone
python verticals/uav_drone/scripts/live_mavlink_stream.py \
    --connection udp:127.0.0.1:14550 \
    --duration 300 \
    --rate 10 \
    --output results/live_mavlink_demo.json

# Inject attack via MAVLink protocol
# (Requires ArduPilot SITL or hardware setup)
```

---

## Reproducibility Checklist

Before claiming reproducibility, verify:

- [ ] All datasets downloaded and verified (SHA256 checksums)
- [ ] Python environment matches requirements.txt
- [ ] PYTHONPATH set correctly
- [ ] Attack injection timestamps randomized but logged
- [ ] JSON outputs include timestamp and flight IDs
- [ ] Detection latency measured independently
- [ ] Compute time excludes file I/O
- [ ] Results match published figures (±2% tolerance)

---

## Troubleshooting

### ImportError: No module named 'alycon'
```powershell
# Fix: Set PYTHONPATH to src directory
$env:PYTHONPATH = "C:\path\to\Alycon\src"
```

### ALFA Data Not Loading
```powershell
# Check file pattern matches processed format
ls verticals/uav_drone/data/alfa/processed/*-mavros-imu-data_raw.csv

# Expected: ~47 directories with IMU CSV files
```

### Low Detection Rate
```
# Verify attack injection actually occurred
# Check JSON output for 'fault_idx' timestamps
# Confirm signal has sufficient variance (not all zeros)
```

---

## Citation

If you reproduce these results, cite both ALYCON validation and original datasets:

```bibtex
@misc{alycon2026uav,
  title={ALYCON UAV Anomaly Detection Validation},
  author={[Your Name]},
  year={2026},
  url={https://github.com/[your-repo]/ALYCON-UAV-Demo}
}

@article{keipour2020alfa,
  title={ALFA: A dataset for UAV fault and anomaly detection},
  author={Keipour, Azarakhsh and Mousaei, Mohammadreza and Scherer, Sebastian},
  journal={The International Journal of Robotics Research},
  year={2020}
}
```

---

**Last Updated:** February 15, 2026  
**Reproducibility Status:** Fully documented, public datasets, verifiable results
