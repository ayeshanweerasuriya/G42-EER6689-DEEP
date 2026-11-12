# Hardware Bill of Materials (BOM) & Electrical Architecture

**Project:** Decentralized Edge-Interlocking Railway Crossing Safety System (Group 42)  
**Target:** Solar-Powered, Off-Grid Deployment at Rural Railway Level Crossings (Sri Lanka)

---

## 1. System Bill of Materials (BOM) per Node

### 1.1 Device 1: Upstream Train Detection Node (`device-1-upstream`)

Positioned 500 m to 1,000 m ahead of the level crossing. Operates autonomously off solar power, sampling rail acoustics and structural track vibrations.

| Component | Part / Model | Specification / Description | Interface / Pinout | Qty |
|---|---|---|---|---|
| **Microcontroller** | Espressif ESP32-S3 DevKit-C | Dual-Core Xtensa LX7 @ 240 MHz, 512 KB SRAM, 8 MB Flash, 8 MB Octal PSRAM | SPI, I2C, I2S | 1 |
| **Acoustic Sensor** | INMP441 MEMS Microphone | Omnidirectional digital MEMS microphone, 24-bit I2S output, 61 dBA SNR, 16 kHz sampling | I2S (BCLK, WS, SD, 3.3V) | 1 |
| **Vibration Sensor** | MPU6050 6-Axis IMU | 3-axis accelerometer ($\pm 2g/\pm 4g$), 3-axis gyro, 16-bit ADC, $\ge 100\text{ Hz}$ I2C sampling | I2C (SDA, SCL, 3.3V) | 1 |
| **LoRa Transceiver** | Semtech SX1278 (Ra-02) | 433 MHz transceiver module, $+17\text{ dBm}$ Tx, $-148\text{ dBm}$ sensitivity, SPI interface | SPI (MOSI, MISO, SCK, NSS, DIO0, RST) | 1 |
| **LoRa Antenna** | 433 MHz Whip / Omni | $3\text{ dBi}$ gain, SMA male connector, tuned for $433\text{ MHz}$ | SMA to IPX cable | 1 |
| **Solar Panel** | 10W - 20W Monocrystalline | $18\text{V}$ $V_{mp}$, weatherproof aluminium frame, bypass diode | DC Solar Terminals | 1 |
| **Charge Controller** | CN3791 MPPT / TP4056 | Dedicated lithium-ion solar charge controller with overcharge / deep-discharge protection | Battery / Solar I/O | 1 |
| **Battery Storage** | 3.7V 18650 Li-Ion Pack | 2S or 1S2P configuration (~$5200\text{ mAh}$ total capacity), integrated BMS protection | DC Power Terminals | 1 |
| **Voltage Regulator** | Low-Dropout (LDO) / Buck | High-efficiency step-down converter ($V_{in} \to 3.3\text{V}$, quiescent current $< 30\,\mu\text{A}$) | 3.3V System Bus | 1 |
| **Mounting & Housing** | IP66 Weatherproof Enclosure | Sealed ABS junction box with mechanical steel rail-web clamp bracket | Mechanical mount | 1 |

---

### 1.2 Device 2: Crossing Danger-Zone Monitoring Node (`device-2-crossing`)

Installed roadside at the level crossing. Integrates optical vision, infrared illumination, mmWave radar, and relay actuation for sirens and strobe lights.

| Component | Part / Model | Specification / Description | Interface / Pinout | Qty |
|---|---|---|---|---|
| **Edge Compute & Cam** | ESP32-S3-CAM Module | Dual-Core Xtensa LX7 @ 240 MHz, 8 MB PSRAM, onboard DVP camera header, micro-SD slot | DVP, UART, SPI | 1 |
| **Image Sensor** | Omnivision OV2640 | 2-Megapixel camera module (IR-cut filter removed for $850\text{ nm}$ infrared sensitivity) | 24-pin DVP ribbon | 1 |
| **IR Illuminator** | 850 nm High-Power IR Board | 48-LED / Array $850\text{ nm}$ infrared illuminator board, $12\text{V DC}$, photoresistor night auto-trigger | 12V Switched DC | 1 |
| **mmWave Radar** | Hi-Link HLK-LD2410 | 24 GHz FMCW radar sensor, detects moving and micro-motion stationary targets up to $6\text{ m}$ | UART (TX, RX, 3.3V) | 1 |
| **Actuation Relays** | 2-Channel 5V Relay Module | Optocoupled relay board, 10A 250VAC / 30VDC rated contacts, active-low trigger | GPIO Control (IN1, IN2) | 1 |
| **Audible Warning** | 12V 110 dB Alert Siren | High-output industrial acoustic siren for level crossing road-user alert | 12V Relay Contact | 1 |
| **Visual Warning** | 12V Red Strobe Beacon | High-intensity flashing LED hazard beacon (visible up to $300\text{ m}$ in daytime) | 12V Relay Contact | 1 |
| **LoRa Transceiver** | Semtech SX1278 (Ra-02) | 433 MHz transceiver module, SPI interface, external antenna connection | SPI Bus | 1 |
| **LoRa Antenna** | 433 MHz High-Gain Omni | $5\text{ dBi}$ outdoor omnidirectional mast antenna with low-loss RG58 cable | SMA connector | 1 |
| **Solar Panel** | 20W Monocrystalline | $18\text{V}$ $V_{mp}$, $1.11\text{A}$ $I_{mp}$, toughened glass weatherproof panel | DC Solar Array | 1 |
| **Solar Controller** | 12V 10A Solar Charge PWM/MPPT | Automated 3-stage charge management, low-voltage disconnect (LVD) @ $10.8\text{V}$ | 12V Battery / Solar I/O | 1 |
| **Battery Storage** | 12V 7Ah Sealed Lead-Acid (SLA) | Deep-cycle SLA or 3S LiFePO4 battery; provides surge current required by 110dB siren | 12V Main Rail | 1 |
| **Buck Converter** | LM2596 / MP1584 Buck Module | High-efficiency DC-DC step-down ($12\text{V} \to 5\text{V} @ 3\text{A}$) for ESP32-S3 and relays | 5V Power Rail | 1 |
| **Mounting & Housing** | IP65 Roadside Mast Enclosure | Hinged weatherproof outdoor electrical box with mast-mount U-bolts and sun hood | Mechanical mast | 1 |

---

### 1.3 Device 3: Train Driver Cabin Unit (`device-3-cabin`)

Console-mounted inside the locomotive operator cab. Displays real-time crossing warnings and enforces the 4.0-second fail-safe packet timeout rule.

| Component | Part / Model | Specification / Description | Interface / Pinout | Qty |
|---|---|---|---|---|
| **Microcontroller** | ESP32-S3 DevKit / NodeMCU | Dual-Core 32-bit MCU (Xtensa LX7/LX6), 512 KB SRAM, 4 MB / 8 MB Flash | I2C, SPI, GPIO | 1 |
| **Primary Display** | 1602 / 2004 Character LCD | HD44780-compatible LCD with PCF8574 I2C backpack (or optional 1.3" I2C OLED display) | I2C (SDA, SCL, 5V) | 1 |
| **Status Indicators** | 5mm / 8mm Discrete LEDs | High-brightness LEDs: Solid Green (Clear), Flashing Red (Hazard), Amber (Timeout) | GPIO via 330Ω resistors | 3 |
| **Audible Alarm** | 5V Active Piezo Buzzer | Continuous / pulsed acoustic beeper ($\ge 85\text{ dB}$ at operator console) | GPIO / NPN Driver | 1 |
| **LoRa Transceiver** | Semtech SX1278 (Ra-02) | 433 MHz transceiver module, SPI interface | SPI Bus | 1 |
| **LoRa Antenna** | 433 MHz Magnetic Roof Antenna | External magnetic base antenna with coaxial feeder to cab window / roof | SMA connector | 1 |
| **Power Interface** | 5V USB / 12V-24V Buck | Powered via standard 5V USB terminal or locomotive cabin auxiliary supply | DC Power Input | 1 |
| **Backup Battery** | 3.7V 18650 Li-Ion Cell | $2600\text{ mAh}$ internal cell with TP4056 charge/boost module (uninterrupted power) | Internal DC Bus | 1 |
| **Enclosure** | 3D-Printed ABS / Dashboard Box | Compact dashboard console enclosure with non-slip mounting bracket | Operator console | 1 |

---

## 2. Microcontroller Pinout & Wiring Tables

### 2.1 Device 1 (ESP32-S3 DevKit)
```text
                  +--------------------------------+
                  |         ESP32-S3 DevKit        |
                  |                                |
  [INMP441 BCLK] -| GPIO 14 (I2S SCK)              |
    [INMP441 WS] -| GPIO 15 (I2S WS)               |
    [INMP441 SD] -| GPIO 16 (I2S SD)               |
                  |                                |
   [MPU6050 SDA] -| GPIO 8  (I2C SDA)              |
   [MPU6050 SCL] -| GPIO 9  (I2C SCL)              |
                  |                                |
  [SX1278 MOSI]  -| GPIO 11 (SPI MOSI)             |
  [SX1278 MISO]  -| GPIO 13 (SPI MISO)             |
   [SX1278 SCK]  -| GPIO 12 (SPI SCK)              |
   [SX1278 NSS]  -| GPIO 10 (SPI CS)               |
  [SX1278 DIO0]  -| GPIO 4  (Interrupt)            |
  [SX1278 RST]   -| GPIO 5  (Reset)                |
                  +--------------------------------+
```

### 2.2 Device 2 (ESP32-S3-CAM)
```text
                  +--------------------------------+
                  |         ESP32-S3-CAM           |
                  |                                |
   [OV2640 DVP]  -| Dedicated Camera Header Pins   |
                  |                                |
 [LD2410 Radar TX]-| GPIO 43 (UART RX)             |
 [LD2410 Radar RX]-| GPIO 44 (UART TX)             |
                  |                                |
  [Relay 1: Siren]-| GPIO 1  (Active LOW)           |
  [Relay 2: Strobe-| GPIO 2  (Active LOW)           |
   [IR Illuminator]-| GPIO 42 (Switched MOSFET Gate)|
                  |                                |
  [SX1278 MOSI]  -| GPIO 11 (SPI MOSI)             |
  [SX1278 MISO]  -| GPIO 13 (SPI MISO)             |
   [SX1278 SCK]  -| GPIO 12 (SPI SCK)              |
   [SX1278 NSS]  -| GPIO 10 (SPI CS)               |
  [SX1278 DIO0]  -| GPIO 21 (Interrupt)            |
  [SX1278 RST]   -| GPIO 47 (Reset)                |
                  +--------------------------------+
```

### 2.3 Device 3 (In-Cabin Display Unit)
```text
                  +--------------------------------+
                  |         ESP32-S3 Unit          |
                  |                                |
  [LCD PCF8574 SDA]-| GPIO 8  (I2C SDA)            |
  [LCD PCF8574 SCL]-| GPIO 9  (I2C SCL)            |
                  |                                |
  [Green LED]    -| GPIO 1  (Active HIGH)          |
  [Red LED]      -| GPIO 2  (Active HIGH)          |
  [Piezo Buzzer] -| GPIO 4  (PWM / High-Output)    |
                  |                                |
  [SX1278 MOSI]  -| GPIO 11 (SPI MOSI)             |
  [SX1278 MISO]  -| GPIO 13 (SPI MISO)             |
   [SX1278 SCK]  -| GPIO 12 (SPI SCK)              |
   [SX1278 NSS]  -| GPIO 10 (SPI CS)               |
  [SX1278 DIO0]  -| GPIO 14 (Interrupt)            |
  [SX1278 RST]   -| GPIO 15 (Reset)                |
                  +--------------------------------+
```

---

## 3. Power Consumption & Off-Grid Autonomy Analysis

### 3.1 Device 1 Power Profile (3.7V Li-Ion, 5200 mAh = 19.24 Wh)
* **Idle Sampling State (Acoustic/Vibration DMA):** $42\text{ mA} @ 3.3\text{V} = 138.6\text{ mW}$
* **CNN Inference Burst (CustomNet, 1.2s):** $110\text{ mA} @ 3.3\text{V} = 363\text{ mW}$
* **LoRa Transmission Burst (85ms):** $120\text{ mA} @ 3.3\text{V} = 396\text{ mW}$
* **Average Consumption (assuming 1 train approach per hour):** $\sim 45\text{ mA}$ continuous ($166.5\text{ mW}$)
* **Autonomy without Solar:**
  $$\text{Runtime} = \frac{5200\text{ mAh}}{45\text{ mA}} \approx 115\text{ hours (} \approx 4.8\text{ days)}$$
* **Solar Recovery (10W Panel, 4 Peak Sun Hours/day):** Produces $\approx 25\text{ Wh/day}$, exceeding daily consumption ($4.0\text{ Wh/day}$) by $> 6\times$.

### 3.2 Device 2 Power Profile (12V 7Ah SLA = 84 Wh)
* **Standby LoRa Listen Mode:** $18\text{ mA} @ 12\text{V} = 216\text{ mW}$
* **Active Wake-Up (Camera + mmWave Radar + MSL-YOLO):** $280\text{ mA} @ 12\text{V} = 3.36\text{ W}$
* **Nighttime Active Mode (+ 850nm IR Illuminator):** $680\text{ mA} @ 12\text{V} = 8.16\text{ W}$
* **Full Obstruction Alarm Active (+ 110dB Siren + Strobe Beacon):** $1.65\text{ A} @ 12\text{V} = 19.8\text{ W}$
* **Duty Cycle Model:**
  * Standby: $23.5\text{ hours/day}$ ($5.07\text{ Wh}$)
  * Active monitoring (15 train approaches/day, 2 mins each = 30 mins): $1.68\text{ Wh}$
  * Total Daily Consumption (nominal without alarms): $\approx 6.75\text{ Wh/day}$
* **Autonomy without Solar (Overcast / Rainy Days):**
  $$\text{Runtime} = \frac{84\text{ Wh} \times 0.70\text{ DOD}}{6.75\text{ Wh/day}} \approx 8.7\text{ days of autonomous reserve}$$
* **Solar Recovery (20W Panel, 4 Peak Sun Hours/day):** Produces $\approx 50\text{ Wh/day}$, providing ample power to run alarms and charge depleted batteries.

---

## 4. Hardware Sourcing & Procurement Directives

1. **Camera Module:** Ensure the OV2640 module is purchased **without an IR-cut filter** (or manually de-lensed to excise the filter glass) to ensure sensitivity under $850\text{ nm}$ infrared night illumination.
2. **LoRa Modules:** Verify that all SX1278 modules are matched for the **433 MHz** frequency band with proper $50\,\Omega$ impedance-matched omnidirectional antennas.
3. **Relay Isolation:** Optocoupled relays must share galvanic opto-isolation with separate VCC/JD-VCC jumpers to prevent back-EMF spikes from the 110 dB siren induction coil resetting the ESP32-S3 microcontroller.
