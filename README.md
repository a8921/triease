# TrisEase — IoT Jaw Physiotherapy Device

<p align="center">
  <img src="docs/banner.png" alt="TrisEase" width="600"/>
</p>

<p align="center">
  <b>Smart jaw exercise monitoring for trismus rehabilitation</b><br>
  IIT Bhilai × AIIMS Raipur &nbsp;|&nbsp; TIH IoT TDP 2025
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-prototype%20v1.0-brightgreen"/>
  <img src="https://img.shields.io/badge/hardware-Raspberry%20Pi%205-red"/>
  <img src="https://img.shields.io/badge/cloud-ThingSpeak-blue"/>
  <img src="https://img.shields.io/badge/patent-202421035151-orange"/>
</p>

---

## What is TrisEase?

Trismus is a jaw-opening restriction affecting patients after head-and-neck cancer treatment, facial trauma, or radiation therapy. Current rehabilitation relies on manual exercises with no objective measurement.

TrisEase is a wearable IoT device that:
- Measures jaw opening distance in real time (VL53L0X ToF sensor)
- Counts exercise repetitions automatically
- Monitors applied force to prevent over-exertion
- Streams data to the cloud for remote clinician review
- Displays live metrics on an onboard OLED

**Patent:** 202421035151 | **PI:** Dr. Jose Immanuel R. (IIT Bhilai) | **Clinical partner:** Dr. Manish Raghani (AIIMS Raipur)

---

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│                  TRISEASE DEVICE                     │
│                                                      │
│  VL53L0X ──I2C──┐                                   │
│  OLED SH1106 ───┤  Raspberry Pi 5 / ESP32-S3        │
│  FSR + HX711 ───┘  (Python / C++ firmware)          │
│                                                      │
│  BLE ────────────────────► Flutter App (moto g45)   │
│  WiFi ───────────────────► ThingSpeak Cloud         │
│  HTTP :8080 ─────────────► Live Dashboard           │
└─────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
trisease/
├── firmware/
│   ├── raspberry-pi/       # Python prototype firmware (current)
│   │   ├── main.py         # Main loop: sensor, OLED, rep counter
│   │   ├── ble_server.py   # BLE GATT server (BlueZ 5.82 workaround)
│   │   ├── bt_agent.py     # Auto-accept BT pairing agent
│   │   ├── api_server.py   # HTTP API server (port 8080)
│   │   └── calibrate.py    # Interactive sensor calibration tool
│   └── esp32-s3/           # ESP32-S3 production firmware (PlatformIO)
│       ├── platformio.ini
│       └── src/main.cpp
├── hardware/
│   ├── wiring/             # Wiring diagrams and pin maps
│   └── bom/                # Bill of materials
├── dashboard/
│   └── TrisEase_Dashboard.html   # Live web dashboard
├── docs/
│   └── TrisEase_Handover.pdf     # Full project documentation
└── README.md
```

---

## Hardware

### Current Prototype (Raspberry Pi 5)

| Component | Spec | I2C Address |
|---|---|---|
| Raspberry Pi 5 | 8GB, Bookworm | — |
| VL53L0X ToF sensor | 0–200mm, ±3% | 0x29 |
| OLED SH1106G | 128×64px, 1.3" | 0x3C |
| FSR 402 | 0–10 kg | — |
| HX711 ADC | 24-bit | — |

### Production Target (ESP32-S3)

| Component | Spec | Notes |
|---|---|---|
| ESP32-S3-DevKitC-1-N32R8V | 32MB Flash + 8MB PSRAM | No errata rev 3.1 |
| VL53L0X bare breakout | NOT GY-53 module | GY-53 MCU causes I2C issues |
| FSR 402 short | 0–10 kg | Direct ADC, no HX711 needed |
| SSD1306 OLED 4-pin | 0.96" I2C | |
| LiPo 3.7V 2000mAh | JST PH 2.0 | ~8h battery life |
| TP4056 USB-C charger | With protection circuit | |

### Wiring (Pi 5)

```
Pi GPIO 2  (SDA) ──── VL53L0X SDA ──── OLED SDA
Pi GPIO 3  (SCL) ──── VL53L0X SCL ──── OLED SCL
Pi 3.3V          ──── VL53L0X VIN ──── OLED VCC
Pi GND           ──── VL53L0X GND ──── OLED GND

ESP32 DOUT ──── HX711 DOUT
ESP32 SCK  ──── HX711 SCK
ESP32 3.3V ──── HX711 VCC
ESP32 GND  ──── HX711 GND
HX711 E+   ──── FSR + 10kΩ pull-down to GND
```

---

## Setup

### Raspberry Pi — Fresh Flash

```bash
# 1. Flash SD card with Raspberry Pi Imager
#    WiFi: LOQ / 12345678 / 2.4GHz
#    SSH: enabled, username: pi

# 2. Install dependencies
sudo pip3 install luma.oled smbus2 requests --break-system-packages
sudo apt-get install -y python3-dbus python3-gi bluez
sudo raspi-config nonint do_i2c 0

# 3. Clone and run
git clone https://github.com/YOUR_USERNAME/trisease.git
cd trisease/firmware/raspberry-pi
sudo python3 main.py
```

### Verify I2C Devices

```bash
sudo i2cdetect -y 1
# Expected: 0x29 (VL53L0X), 0x3C (OLED SH1106)
```

### BLE Setup (before each run)

```bash
sudo btmgmt le on && sudo btmgmt connectable on
sudo hciconfig hci0 noleadv
sudo systemctl restart bluetooth
# Run bt_agent.py in a separate terminal for auto-pairing
sudo python3 bt_agent.py &
```

---

## Calibration

The sensor calibration maps raw VL53L0X readings to jaw-opening distance in mm.

```bash
python3 calibrate.py
# Interactive: measures CLOSED and OPEN positions
# Auto-updates constants in main.py
```

**Current calibration (GY-53 module):**
- RAW_CLOSED = 23.0 mm (jaw closed → sensor reads ~23mm)
- RAW_OPEN = 5.0 mm (jaw open → sensor reads ~5mm)
- Display formula: `display = (23.0 - raw) / 18.0 * 55.0`

> **Note:** GY-53 module limits range to ~60mm and causes I2C instability. Replace with bare VL53L0X breakout (₹200 from Robu.in) for stable 0–200mm range.

---

## Rep Counting Algorithm

```
State machine: CLOSED → OPEN → CLOSED = 1 rep

OPEN threshold:   > 35mm displayed distance
CLOSE threshold:  < 10mm displayed distance
Min open time:    0.3 seconds (debounce)
```

Validated: 34+ reps counted correctly in testing.

---

## Cloud — ThingSpeak

| Field | Data | Update Rate |
|---|---|---|
| Field 1 | Distance (mm) | Every 20s |
| Field 2 | Force (kg) | Every 20s |
| Field 3 | Rep count | Every 20s |
| Field 4 | Timestamp IST | Every 20s |

Channel: 3407797 | [View live data](https://thingspeak.com/channels/3407797)

---

## BLE GATT Service

Advertises as: **"TrisEase"**

| UUID | Type | Data Format |
|---|---|---|
| `12345678-...-012` | Service | — |
| `12345678-...-013` | Sensor Characteristic | `"55.0,0.00,3,1,14:32:07"` |
| `12345678-...-014` | Patient Characteristic | Read/Write |

Data format: `dist_mm,force_kg,reps,jaw_open(0/1),HH:MM:SS`

> **BlueZ 5.82 bug:** RegisterAdvertisement fails silently. Workaround using `hciconfig` raw HCI command is implemented in `ble_server.py`.

---

## HTTP API

```
GET http://<PI_IP>:8080/data

Response:
{
  "dist_mm": 42.3,
  "force_kg": 0.0,
  "reps": 7,
  "jaw_open": true,
  "max_dist": 55.0,
  "timestamp": "14:32:07"
}
```

---

## Flutter App Integration

Three connection methods available:

```yaml
# pubspec.yaml
dependencies:
  flutter_blue_plus: ^1.31.3   # BLE
  http: ^1.1.0                  # WiFi HTTP
```

**BLE:** Scan for "TrisEase", subscribe to sensor characteristic UUID `12345678-1234-1234-1234-123456789013`

**WiFi:** `GET http://192.168.137.245:8080/data` every 200ms

---

## Known Issues

| Issue | Root Cause | Fix |
|---|---|---|
| VL53L0X max range ~60mm | GY-53 onboard MCU interfering | Buy bare VL53L0X breakout |
| ESP32 StoreProhibited crash | Silicon errata rev 3.1 | ESP32-S3 ordered (no errata) |
| BLE pairing "incorrect PIN" | bt_agent.py not running | Run bt_agent.py before pairing |
| BLE ad error (BlueZ 5.82) | RegisterAdvertisement bug | hciconfig workaround (implemented) |
| Pi IP changes each session | DHCP | Check Windows hotspot settings |

---

## Roadmap

| Phase | Timeline | Milestone |
|---|---|---|
| v1.0 | Aug 2026 | Pi 5 prototype with BLE + ThingSpeak ✅ |
| v1.1 | Oct 2026 | ESP32-S3 firmware + bare VL53L0X |
| v1.2 | Nov 2026 | Flutter app + custom PCB design |
| v1.3 | Jan 2027 | PCB fabrication (5 units) |
| v2.0 | Mar 2027 | IRB approval + AIIMS Raipur patient trial |

Target: **TRL 5** by January 2027

---

## Team

| Role | Name | Institution |
|---|---|---|
| Principal Investigator | Dr. Jose Immanuel R. | IIT Bhilai |
| Clinical Partner | Dr. Manish Raghani | AIIMS Raipur |
| Hardware & Firmware | Ashish | IIT Bhilai |

**Funding:** TIH Foundation for IoT & IoE — Technology Development Program 2025

---

## License

This project is proprietary research under TIH Foundation TDP 2025.
Patent application: **202421035151** (filed).

© 2025 IIT Bhilai — TrisEase Project Team
