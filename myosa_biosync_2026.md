---
publishDate: 2026-05-24
title: BioSync, Real-Time Multi-Parameter Bio-Responsive Health Monitoring & Adaptive Alert System
excerpt: BioSync is a wrist-worn, battery-powered health monitoring system built on the MYOSA ESP32 platform that continuously tracks heart rate, SpO₂, stress index, motion, and environment, and alerts the wearer and remote caregivers the moment a health anomaly is detected.
image: BioSync/cover.png
tags:
  - health-monitoring
  - wearable
  - iot
---
<p align="center">
  <img src="/cover.png" width="800"><br/>
  <i>cover.png</i>
</p>

---

## Acknowledgements

We sincerely thank our faculty mentor and the institution for supporting the development of this assistive healthcare project. Special appreciation goes to the MYOSA Event 5.0 platform for encouraging innovative embedded system solutions focused on real-world problems.

---

## Overview

Every 2 seconds, someone dies from a cardiovascular event :  not because medicine lacks solutions, but because the warning signs went unnoticed. BioSync exists to change that.

BioSync is a wrist-worn, real-time health monitoring system built entirely on the MYOSA ESP32 platform. It continuously tracks heart rate (BPM), blood oxygen saturation (SpO₂), HRV-based Stress Index, fall detection, motion, temperature, humidity, and barometric pressure, then classifies the wearer's state as Normal, Caution, or Critical every second.

When an anomaly is detected, BioSync responds across three simultaneous layers:

* On-arm :  OLED display updates + buzzer + vibration motor fire within 1 second
* Local network :  React dashboard (running on a laptop) turns amber or red, visible on any browser on the same WiFi
* Remote :  MQTT stream to EMQX broker, accessible globally using Grafana.

The ESP32 has exactly one WiFi responsibility: publish a 300-byte JSON vitals packet every 2 seconds via MQTT. All visualisation, fleet management, and trend charting lives in the React dashboard, keeping the ESP32 lean, stable, and focused entirely on sensing and alerting.

**Key features:**

* 6-sensor I2C fusion, MAX30102, MPU6050, BMP180, APDS9960, DHT11, SSD1306, all on a single bus
* Scientifically grounded 4-component Stress Index (HRV + HR elevation + motion + heat) computed entirely on-device
* Three-state alert system (Normal / Caution / Critical) with buzzer, vibration motor, and flashing OLED overlay
* Gesture-controlled OLED screens (left/right swipe via APDS9960)
* Purpose-built React dashboard with fleet view, per-device trend charts, alert timeline, and admin auth
* Full demo mode (stress ramp + SpO₂ drop simulations) for live conference presentation
* Live Grafana healthcare dashboard

---

## Demo / Examples

### Images

<p align="center">
  <img src="/cover.png" width="800"><br/>
  <i>BioSync Prototype</i>
</p>

<p align="center">
  <img src="/biosync_hardware.png" width="800"><br/>
  <i>BioSync Hardware</i>
</p>

<p align="center">
  <img src="/local_dashboard_home_page.png" width="800"><br/>
  <i>Local Dashboard Home Page</i>
</p>

<p align="center">
  <img src="/grafana_dashboard_1.png" width="800"><br/>
  <i>Grafana Dashboard</i>
</p>

### Videos

**Presentation**

https://github.com/user-attachments/assets/b8237d21-2828-4da1-aef9-1e99e509c851

**Demo**


https://github.com/user-attachments/assets/298202d3-5555-4bc8-a8f7-e69f5dcea371

---

## Features (Detailed)

### 1. Multi-Sensor Fusion on a Single I2C Bus

BioSync drives five I2C sensors and one GPIO sensor simultaneously from a single ESP32:

| Sensor | Role | Interface | Address |
|--------|------|-----------|---------|
| MAX30102 | BPM + SpO₂ + HRV | I2C | 0x57 |
| MPU6050 | Accel magnitude, fall detection, motion label | I2C | 0x69 (MYOSA AD0=HIGH) |
| BMP180 | Barometric pressure + ambient temperature | I2C | 0x77 |
| APDS9960 | Gesture detection (left/right swipe) | I2C | 0x39 |
| SSD1306 OLED | 128×32 display | I2C | 0x3C |
| DHT11 | Temperature + humidity | GPIO27 | :  |

All sensors are polled concurrently on Core 0 using millis()-based timing with no blocking delay() calls in the main loop. Core 1 handles WiFi and MQTT exclusively, so sensor acquisition is never interrupted by network activity.

### 2. BPM, SpO₂, and HRV (RMSSD) from MAX30102

Heart Rate is detected using the SparkFun checkForBeat() PBA algorithm on the IR channel. Each detected R-peak timestamps a new R-R interval. A personal BPM baseline is accumulated over the first 30 seconds of wear (minimum 5 readings) and locked :  this baseline feeds the HR elevation component of the Stress Index.

SpO₂ uses the SparkFun maxim_heart_rate_and_oxygen_saturation() algorithm from the MYOSA kit. 100 samples of RED + IR are buffered; after each calculation the last 25 are kept and 75 more collected :  giving continuous SpO₂ updates roughly every 3–4 seconds. Only values in the range 70–100% with a valid flag are accepted.

HRV (RMSSD) is computed from the last 20 R-R intervals:

```
RMSSD = sqrt( mean( (RR[n] – RR[n-1])² ) )
```

Lower RMSSD = reduced autonomic variability = higher stress load. This is the highest-weighted input (40%) to the Stress Index.

Finger detection threshold: IR channel > 50,000 counts. When no finger is detected, BPM and SpO₂ are cleared and the Stress Index HRV/HR components contribute zero.

### 3. Stress Index :  4-Component Weighted Fusion

The Stress Index is a single 0–100 score computed every second from four physiological and environmental inputs, each normalised independently to [0, 1]:

```
StressIndex =
  0.40 × hrv_score      ( 1 – RMSSD/80,           clamped 0–1 )
+ 0.30 × hr_elevation   ( (BPM – baseline) / 60,  clamped 0–1 )
+ 0.20 × motion_score   ( |accel_mag – 1.0| / 2,  clamped 0–1 )
+ 0.10 × heat_score     ( (temp_c – 28) / 10,     clamped 0–1 )
```

### 4. Three-State Alert Classification

Every second, the alert state machine evaluates all parameters in priority order:

CRITICAL fires immediately if:
* Fall detected (accel magnitude > 2.5 g)
* SpO₂ < 90% (with valid reading)
* BPM > 150 or BPM < 40
* Stress Index > 70

CAUTION fires if:
* SpO₂ < 94%
* BPM > 110
* Stress Index > 41

NORMAL otherwise.

### 5. Actuator Response :  Buzzer and Vibration Motor

Both actuators are GPIO-driven through BC547 NPN transistors (no direct GPIO current through the motor or buzzer):

| State | Buzzer (GPIO25) | Vibration Motor (GPIO26) |
|-------|-----------------|--------------------------|
| NORMAL | OFF | OFF |
| CAUTION | OFF | 3 × 200 ms pulses, every 30 s |
| CRITICAL | Continuous ON | Continuous ON |

The CAUTION vibration pattern is fully non-blocking, implemented with millis() comparison. The 30-second repeat cycle resets automatically when state changes.

### 6. OLED Display :  Three Screens

The 128×32 SSD1306 supports four text rows at size 1 (6×8 px per character).

Screen 1 :  Full Vitals (default):

```
BPM:74    SpO2:98%
STR: 32  OK
T:28.4C   H:63%
MOV:LOW   1013hPa
```

Row 2 shows a 50-pixel progress bar for the Stress Index with a live state label.

Screen 2 :  Alert History:

```
-- RECENT ALERTS --
00:14 CAU spo2:94%
00:09 CRI spo2:87%
00:02 CAU stre:68
```

Displays the last 3 events from the alert log with uptime timestamps.

Critical Overlay (overrides all screens):

```
!!!CRITICAL ALERT!!!
SpO2:87%  BPM:74
STR:82  FALL:NO
SEEK HELP NOW
```

Flashes at 2 Hz by alternating WHITE-on-BLACK and BLACK-on-WHITE. Cannot be dismissed by gesture :  persists until state clears.

Gesture switching: Left or right swipe on the APDS9960 cycles between Screen 1 and Screen 2. The APDS9960 sensor object lives inside oled.cpp since it is the only consumer of gesture data.

### 7. MQTT Streaming to React Dashboard

Core 1 runs a dedicated FreeRTOS task that handles WiFi connection and MQTT publishing only :  no web server, no HTML serving. Every 2 seconds it publishes this JSON payload:

```json
{
  "device_id":    "BS-001",
  "bpm":          74,
  "spo2":         98,
  "hrv_rmssd":    42.3,
  "stress_index": 32,
  "temperature":  28.4,
  "humidity":     63,
  "pressure_hpa": 1013,
  "motion":       "LOW",
  "fall_detected": false,
  "finger_on":    true,
  "spo2_valid":   true,
  "state":        "NORMAL",
  "timestamp":    1748300000
}
```

Field names are case-sensitive and match the React dashboard parser exactly. The device_id field in both the topic path and the JSON body enables multi-device fleet mode with zero firmware changes beyond config.h.

Auto-reconnect is implemented with a 5-second cooldown on both WiFi and MQTT :  no blocking retries in the Core 1 loop.

### 8. Grafana Dashboard

Grafana is a visualization platform used in BioSync. It provides a professional healthcare monitoring dashboard similar to modern ICU patient monitoring systems.

Grafana receives processed data from InfluxDB and visualizes it using:
* Gauge panels
* Stat panels
* Time-series graphs
* Alert indicators
* Live trend analysis

Grafana Architecture:

```
ESP32 Sensors
      ↓
MQTT Protocol
      ↓
Mosquitto Broker
      ↓
Node-RED Processing
      ↓
InfluxDB Database
      ↓
Grafana Dashboard
```

---

## Usage Instructions

### Hardware Setup

1. Wire all 5 I2C sensors to GPIO21 (SDA) and GPIO22 (SCL) on the ESP32.
2. Wire DHT11 DATA pin to GPIO27 with a 10 kΩ pull-up resistor to 3.3 V.
3. Connect MAX30102 flush to the inner wrist face with a foam light gasket.
4. Wire buzzer and vibration motor each through a BC547 NPN transistor to GPIO25 and GPIO26 respectively.

### Flashing the Firmware

1. Install Arduino IDE 2.x and the Espressif ESP32 board package.
2. Copy MYOSA kit libraries to your Arduino `libraries/` folder:

```
Adafruit_SSD1306-master/
Adafruit-GFX-Library-master/
SparkFun_MAX3010x_Sensor_Library-master/
SparkFun_APDS-9960_Sensor_Arduino_Library-V_1.4.2/
MPU6050/
BMP085/
```

3. Install from Arduino Library Manager:

```
PubSubClient       by Nick O'Leary
ArduinoJson        by Benoit Blanchon
DHT sensor library by Adafruit
Adafruit Unified Sensor by Adafruit
```

4. Open `BioSync/BioSync.ino` in Arduino IDE.
5. Edit `config.h` :  set your hotspot credentials.
6. Select board: ESP32 Dev Module, port: your USB-serial port.
7. Upload. Open Serial Monitor at 115200 baud to watch the boot sequence and sensor readings.

### Running the React Dashboard

```bash
cd biosync-dashboard
npm install
npm run dev
```

---

## Tech Stack

### 1. Enterprise Backend & Cloud Analytics

* **Espressif Systems ESP32** :  Main IoT Controller
* **MQTT Protocol** :  Real-time IoT Communication
* **Mosquitto MQTT Broker** :  MQTT Broker for messaging routing
* **Node-RED** :  Data Processing, AI Logic & Automation
* **InfluxDB Cloud** :  Cloud Time-Series Database for persistent medical records
* **Grafana Cloud** :  Medical-grade Cloud Dashboard Visualization

### 2. Firmware

* **ESP32 Arduino Core** :  FreeRTOS dual-core, WiFi stack
* **Arduino IDE** :  ESP32 Programming environment
* **SparkFun Libraries** :  MAX30105 (Pulse Ox), APDS9960 (Gesture)
* **Adafruit Libraries** :  SSD1306 (OLED), DHT sensor library
* **I2Cdev** :  MPU6050, BMP085
* **PubSubClient & ArduinoJson** :  MQTT and Payload formatting

### 3. Frontend Dashboard (Local/Fleet)

* **React 18 + Vite** :  Local fleet dashboard
* **MQTT.js (WebSocket)**
* **Recharts & Tailwind CSS v4**

---

## Requirements / Installation

### Hardware Requirements

* ESP32 Development Board (From MYOSA Kit)
* MAX30102 Heart Rate & SpO₂ Sensor
* DHT11 Temperature & Humidity Sensor
* BMP180 Pressure Sensor (From MYOSA Kit)
* MPU6050 Accelerometer & Gyroscope (From MYOSA Kit)
* SSD1306 OLED Display (From MYOSA Kit)
* APDS9960 Gesture Sensor (From MYOSA Kit)
* Buzzer & Vibration Motor (10 mm coin)
* Power Chain: 3.7 V 500 mAh Li-Po Battery x2 :  Charging and Boost Converter Module
* Misc: BC547 NPN Transistor, 1 kΩ Resistor, Capacitor, Flyback Diode, Wires, USB Cable

### Software Requirements

* **Arduino IDE** :  Programming ESP32 firmware
* **VS Code** :  Source code editor and development environment
* **Mosquitto MQTT** :  Core MQTT Communication routing
* **Node.js** :  Runtime environment required for Node-RED and React
* **Node-RED** :  Visual data flow processing and AI logic
* **InfluxDB Cloud Account** :  Remote Cloud Database for time-series metrics
* **Grafana Cloud Account** :  Dashboard Visualization and alert monitoring

### Installation

**STEP 1: Install Arduino IDE & Flash Firmware**

1. Download [Arduino IDE](https://www.arduino.cc/en/software).
2. Install the required libraries via the Library Manager or ZIP import:

```
WiFi, PubSubClient, ArduinoJson
DHT Sensor Library, Adafruit Unified Sensor
Adafruit BMP085, Adafruit MPU6050, Adafruit SSD1306, Adafruit GFX
MAX30105, Adafruit APDS9960
```

3. Edit `config.h` in the firmware to set your WiFi and MQTT Broker details.
4. Select board ESP32 Dev Module and upload.

**STEP 2: Install Mosquitto MQTT Broker**

1. Download [Mosquitto](https://mosquitto.org/download/).
2. Run the broker from your terminal:

```bash
mosquitto -v
```

3. Default MQTT Port is 1883.

**STEP 3: Install Node.js**

1. Download Node.js.
2. Verify installation in your terminal:

```bash
node -v
npm -v
```

**STEP 4: Install Node-RED**

1. Install Node-RED globally via npm:

```bash
npm install -g --unsafe-perm node-red
```

2. Start Node-RED:

```bash
node-red
```

3. Open http://localhost:1880 in your browser.

**STEP 5: Create InfluxDB Cloud**

1. Open InfluxDB Cloud and create an account.
2. Create the following configurations:
   * Organization: (Your Choice)
   * Bucket: biosync
   * API Token: Generate and save this for Node-RED and Grafana.

**STEP 6: Create Grafana Cloud**

1. Open Grafana Cloud and create an account.
2. Add a new Datasource and select InfluxDB.
3. Connect using your InfluxDB URL, Token, Organization, and the biosync Bucket.
4. Import the BioSync Dashboard template to instantly visualize all data.

**STEP 7: Local Fleet Dashboard**

To run the local React priority dashboard alongside the cloud system:

```bash
cd biosync-dashboard
npm install
npm run dev
```

Open the provided local IP (port 3000) and login with admin / biosync2026.

---

## File Structure

```
│   biosync_hardware.png
│   cover.png
│   demo.mp4
│   grafana_dashboard_1.png
│   grafana_dashboard_2.png
│   local_dashboard_home_page.png
│   local_dashboard_login.png
│   myosa_biosync_2026.md
│   presentation.mp4
│
├───biosync_firmware
│       alerts.cpp
│       alerts.h
│       biosync.ino
│       config.h
│       data.h
│       health_analytics.cpp
│       health_analytics.h
│       mqtt.cpp
│       mqtt.h
│       oled.cpp
│       oled.h
│       sensor_bmp180.cpp
│       sensor_bmp180.h
│       sensor_dht11.cpp
│       sensor_dht11.h
│       sensor_max30102.cpp
│       sensor_max30102.h
│       sensor_mpu6050.cpp
│       sensor_mpu6050.h
│       stress.cpp
│       stress.h
│       wifi_manager.cpp
│       wifi_manager.h
│
└───biosync_local_react_dashboard
    ├───public
    │       biosync_favicon.png
    │       biosync_horizontal.png
    │       biosync_vertical.png
    │
    └───src
        │   App.css
        │   App.jsx
        │   Auth.jsx
        │   DeviceView.jsx
        │   FleetView.jsx
        │   index.css
        │   main.jsx
        │   mqtt.js
        │
        └───components
                AlertLog.jsx
                EditDeviceModal.jsx
                SparkLine.jsx
                StateBadge.jsx
                VitalCard.jsx
```
