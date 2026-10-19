# Data Directory

## Overview

Sample data is **not included** in this repository due to size and confidentiality constraints. The raw telemetry CSV files can range from 5 MB to 200+ MB depending on the collection window and number of drives.

## Expected Data Format

The analysis notebooks expect a **CSV file** with:

- One row per motor per timestamp (long format)
- A `timestamp` column (ISO 8601)
- A `drive_ip` column identifying each motor
- 50+ numeric telemetry columns covering the feature groups below

### Feature Groups

| Group | Description | Typical Columns | Units |
|---|---|---|---|
| **Position** | Absolute and relative shaft position from drive and encoder | 4–5 columns | Increments, user units |
| **Speed** | Motor speed from drive controller and output frequency | 3–4 columns | rpm, Hz |
| **Torque & Current** | Motor torque, output current (apparent and active) | 4–5 columns | %, A |
| **Thermal** | Motor winding temperature, heat sink temperature and utilization | 3–4 columns | °C, % |
| **Electrical** | DC bus voltage, output voltage setpoint, power, energy | 4–5 columns | V, kW, kWh |
| **Utilization** | Motor utilization, electro-mechanical utilization, total utilization | 3–4 columns | % |
| **Status** | Tracking error, homing status, encoder connection status | 2–3 columns | Bitfield / enum |

> **Note**: Many features come in both raw (integer/scaled) and engineering-unit versions. The analysis notebooks handle both and identify redundant pairs automatically.

### Total Column Count

| Stage | Columns |
|---|---|
| Raw CSV | ~52 |
| After removing constants & duplicates | ~25 useful numeric |
| After correlation clustering (ρ > 0.9) | **9 essential** (96% variance) |
| After multi-method feature selection | **16 comprehensive** (99.9% variance) |

## Generating Synthetic Test Data

To generate compatible test data for running the notebooks, use the script below. It creates correlated feature groups so that the PCA and correlation-clustering analyses produce meaningful results (random independent columns would show no structure).

```python
import pandas as pd
import numpy as np

np.random.seed(42)
n_drives = 50
n_timestamps = 200
timestamps = pd.date_range("2026-01-01", periods=n_timestamps, freq="5s")

rows = []
for drive_id in range(1, n_drives + 1):
    # Base signals (independent latent variables)
    speed_base = np.random.normal(1450, 50, n_timestamps)
    torque_base = np.random.normal(35, 10, n_timestamps)
    temp_base = np.random.normal(75, 5, n_timestamps)
    voltage_base = np.random.normal(560, 30, n_timestamps)
    position_base = np.cumsum(np.random.normal(0, 100, n_timestamps))

    for t in range(n_timestamps):
        s, tq, tm, v, p = speed_base[t], torque_base[t], temp_base[t], voltage_base[t], position_base[t]
        rows.append({
            "timestamp": timestamps[t].isoformat(),
            "drive_ip": f"192.168.1.{drive_id + 70}",
            # Position group (highly correlated)
            "PositionUser": p,
            "PositionModulo": p % 360,
            "PositionSys": p * 1.0001 + np.random.normal(0, 0.01),
            "EncoderPos": p * 1.0002 + np.random.normal(0, 0.01),
            # Speed group (highly correlated)
            "SpeedRaw": int(s * 10),
            "SpeedRPM": s,
            "SpeedSys": s * 1.001 + np.random.normal(0, 0.1),
            "OutputFreq_Hz": s / 30 + np.random.normal(0, 0.05),
            # Torque group (correlated)
            "TorqueRaw": int(tq * 100),
            "TorquePct": tq,
            "TorqueRelActual": tq * 0.98 + np.random.normal(0, 0.5),
            "TorqueRelMin": tq * 0.6 + np.random.normal(0, 1),
            "TorqueRelMax": tq * 1.4 + np.random.normal(0, 1),
            "ActiveCurrent_A": tq * 0.35 + np.random.normal(0, 0.2),
            "TotalCurrent_A": tq * 0.38 + np.random.normal(0, 0.3),
            "RelActiveCurrent": tq * 0.7 + np.random.normal(0, 0.5),
            "RelTotalCurrent_%": tq * 0.75 + np.random.normal(0, 0.5),
            "RelExcitCurrent": np.random.normal(50, 3),
            "ExcitCurrent_A": np.random.normal(2.5, 0.2),
            # Thermal group (correlated)
            "MotorTempRaw": int(tm * 10),
            "MotorTemp_C": tm,
            "HeatsinkTemp_C": tm * 0.6 + np.random.normal(0, 1),
            "HeatsinkUtil_%": tm * 0.8 + np.random.normal(0, 2),
            "MotorUtilRaw": int(tq * 2),
            "MotorUtilPct": tq * 2.0 + np.random.normal(0, 1),
            # Electrical group (partially correlated)
            "DcBusVoltage_V": v,
            "VoltageRef_V": v * 0.7 + np.random.normal(0, 5),
            "ElecMechUtil_%": tq * 1.5 + np.random.normal(0, 2),
            "TotalUtil_%": tq * 1.6 + np.random.normal(0, 2),
            # Tracking / status group (independent)
            "TrackingError": np.random.normal(0, 0.01),
            "TrackingErrorScaled": np.random.normal(0, 0.001),
            "TrackingErrorSys": np.random.normal(0, 0.01),
            "HomingStatus": 0,        # constant → will be removed
            "HomingStatusScaled": 0,   # constant → will be removed
            "ConfigUserUnit": 1.0,     # constant → will be removed
        })

df = pd.DataFrame(rows)
df.to_csv("data/sample_data.csv", index=False)
print(f"Generated {len(df)} rows x {len(df.columns)} columns")
```

> **Note**: The synthetic data includes correlated feature groups by design — position columns track together, speed columns track together, etc. This ensures the PCA and correlation clustering analyses produce realistic cluster structures. With fully random independent columns, no meaningful reduction would be found.

> **Important**: The hardcoded feature lists in the analysis notebooks (Set A, Set B, and memory estimation cells) reflect results from the original production data. With synthetic data, the dynamic analysis cells will run correctly but produce different specific results. The hardcoded cells are preserved as documentation of the real-data findings.
