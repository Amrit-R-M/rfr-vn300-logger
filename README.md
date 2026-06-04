# RFR VN-300 Data Logger
Full data logger for the VectorNav VN-300 Dual GNSS/INS unit for Rutgers Formula Racing. Logs all sensor data to a timestamped CSV file at 10Hz during test drive days.

## How to Run
**Windows:** `py -3.11 rfr_vn300_logger.py --port COM3`
**Raspberry Pi:** `python3 rfr_vn300_logger.py --port /dev/ttyUSB0`

## Requirements
`py -3.11 -m pip install ./vectorNav/python`

---

## How It Works
The VN-300 sends 3 separate data packets over a serial cable at 10Hz. Each packet contains different sensor data. The script reads each packet as it arrives and writes it as a row in the CSV file. This is why some columns are empty in certain rows — each row only contains data from whichever packet arrived at that moment. This is completely normal.

- **Packet 1** fills attitude, acceleration, gyroscope, magnetometer, temperature, pressure
- **Packet 2** fills raw GNSS position, velocity, fix status, satellite count
- **Packet 3** fills INS fused position and velocity — always use this for analysis

---

## Parameters Logged

### Attitude — How the car is oriented
| Parameter | Description |
|-----------|-------------|
| yaw | Which direction the car is pointing. 0 = North, 90 = East, 180 = South, 270 = West |
| pitch | Nose tilting up or down. Goes negative under acceleration, positive under braking |
| roll | Car leaning left or right. Goes positive in right turns, negative in left turns |

### Acceleration — Forces acting on the car
| Parameter | Description |
|-----------|-------------|
| ax | Forward and backward force in m/s². Negative when braking, positive when accelerating |
| ay | Left and right cornering force in m/s². Tells you how hard the car is pushing through a corner |
| az | Vertical force in m/s². At rest reads 9.8 (gravity). Increases at speed as the car generates downforce. KEY parameter for the aero team |

### Gyroscope — How fast the car is rotating
| Parameter | Description |
|-----------|-------------|
| gx | How fast the car is rolling over bumps in rad/s |
| gy | How fast the nose is pitching up or down in rad/s |
| gz | How fast the car is turning through corners in rad/s. KEY parameter for the dynamics team |

### Delta Values — Change between each reading
| Parameter | Description |
|-----------|-------------|
| dvx, dvy, dvz | How much the velocity changed since the last reading in m/s |
| dtx, dty, dtz | How much the car rotated since the last reading in radians |
| dt | How much time passed between the last two readings in seconds |

### Magnetometer — Built in compass
| Parameter | Description |
|-----------|-------------|
| mx, my, mz | Magnetic field strength in Gauss on each axis. Used internally by the VN-300 to calculate heading. May be affected by the steel chassis and electric motor |

### Environment
| Parameter | Description |
|-----------|-------------|
| temp_c | Internal sensor temperature in Celsius |
| pressure_pa | Air pressure in Pascals. Changes with altitude and weather |

### Raw GNSS — Position direct from satellites
| Parameter | Description |
|-----------|-------------|
| gnss_fix | GPS lock status. Must be 3 (full 3D lock) before the car moves |
| gnss_sats | Number of satellites locked. Want 8 or more before driving |
| gnss_lat, gnss_lon | Raw GPS latitude and longitude direct from satellites. Accurate to 2-3 meters |
| gnss_alt | Raw GPS altitude in meters above sea level |
| gnss_vn, gnss_ve, gnss_vd | Raw GPS velocity in North, East, and Down directions in m/s |
| gnss_speed | Total ground speed in m/s calculated from all three velocity components |

### INS Fused — GPS and IMU combined ⭐ Always use this for analysis
| Parameter | Description |
|-----------|-------------|
| ins_lat, ins_lon | Fused latitude and longitude. GPS and IMU combined, accurate to under 1 meter. Always use this instead of raw GNSS for lap analysis and track mapping |
| ins_alt | Fused altitude in meters. More stable than raw GPS altitude |
| ins_vn, ins_ve, ins_vd | Fused velocity North, East, Down in m/s. Smoother than raw GPS because IMU fills the gaps between GPS updates |

---

## Lap Timer
The script detects when the car crosses the start/finish line and logs lap times to the terminal in real time.

**How it works:**
1. A GPS coordinate is set for the start/finish line before each test day
2. Every 0.1 seconds the script calculates the distance between the car and that coordinate using the Haversine formula
3. When the car comes within 5 meters of the point a lap is triggered
4. A 20 second cooldown prevents multiple triggers on the same crossing
5. Lap times are printed to the terminal as the car completes each lap

**To set your start/finish coordinate:**
Update these values at the top of the script before each test day:
```python
START_LAT = 40.52653
START_LON = -74.46526
THRESHOLD_METERS = 5.0
COOL_DOWN_SECONDS = 20
```
