# System Architecture: Decentralized Edge-Interlocking Railway Crossing Safety System

**Project:** Final-Year Software Engineering Prototype (Group 42)  
**Target Environment:** Unprotected rural railway level crossings in Sri Lanka  
**Design Paradigm:** Fully decentralized, edge-native AI, off-grid solar/battery operation, wireless LoRa ad-hoc telemetry (zero cloud/cellular reliance, zero trackside cabling)

---

## 1. System Overview

The **Decentralized Edge-Interlocking System** addresses catastrophic collisions at unprotected railway level crossings through a distributed 3-node edge architecture. Rural level crossings often lack electrical grid connections and cellular data coverage, rendering cloud-dependent or wired track-circuit safety solutions impractical.

This system replaces centralized interlocking with an autonomous, multi-sensor, multi-node edge computing pipeline running local INT8-quantized neural networks on Espressif ESP32-S3 microcontrollers, communicating via point-to-point Semtech SX1278 LoRa transceivers at 433 MHz.

```mermaid
flowchart TD
    subgraph Device1["Device 1: Upstream Train Detection Node (500m - 1km Upstream)"]
        D1_Sensors["Sensors:<br/>• INMP441 I2S Mic (16 kHz)<br/>• MPU6050 Accelerometer (≥100 Hz)"]
        D1_AI["Edge AI Pipeline:<br/>• Mel-Spectrogram STFT Preprocessing<br/>• INT8 Quantized Compact 2D-CNN (CustomNet)<br/>• Arrival Time (ETA) Regressor"]
        D1_LoRa["SX1278 LoRa 433MHz Transceiver"]
        D1_Power["10W-20W Solar + Li-ion / SLA Battery"]
        D1_Sensors --> D1_AI --> D1_LoRa
    end

    subgraph Device2["Device 2: Crossing Danger-Zone Monitoring Node (At Crossing)"]
        D2_Power["20W Solar + 12V 7Ah SLA Battery"]
        D2_LoRa["SX1278 LoRa 433MHz Transceiver"]
        D2_AI["Edge AI & Fusion Pipeline:<br/>• INT8 Quantized MSL-YOLO (≥5 FPS)<br/>• Danger-Zone ROI Bounding-Box Tracker<br/>• Stationary Dwell-Time Filter (4.0s)<br/>• mmWave Radar Multi-Modal Fusion"]
        D2_Sensors["Perception Suite:<br/>• ESP32-S3-CAM + OV2640 (No IR-Cut)<br/>• 850nm High-Power IR Illuminator<br/>• HLK-LD2410 24GHz mmWave Radar"]
        D2_Actuators["Local Warning Actuators:<br/>• Dual 5V Relays<br/>• 12V 110dB Siren<br/>• 12V Red Strobe Beacon"]
        D2_Sensors --> D2_AI
        D2_LoRa -->|Wake-up Signal| D2_AI
        D2_AI --> D2_Actuators
        D2_AI --> D2_LoRa
    end

    subgraph Device3["Device 3: Train Driver Cabin Unit (Inside Locomotive)"]
        D3_LoRa["SX1278 LoRa 433MHz Transceiver"]
        D3_Core["ESP32-S3 Controller & Watchdog:<br/>• LoRa Packet Parser<br/>• 4.0s Fail-Safe Heartbeat Watchdog"]
        D3_HMI["Driver Interface (HMI):<br/>• 1602/2004 I2C LCD / OLED Screen<br/>• Discrete Status LEDs (Green/Red)<br/>• Active Piezo Warning Buzzer (≥85dB)"]
        D3_LoRa --> D3_Core --> D3_HMI
    end

    subgraph RoadUsers["Road Users at Crossing"]
        LocalWarning["Acoustic & Optical Warnings:<br/>Siren Sound + Strobe Flash"]
    end

    subgraph TrainDriver["Locomotive Driver"]
        InCabinAlert["Situational Awareness:<br/>ETA + CLEAR / OBSTRUCTED Status"]
    end

    D1_LoRa -.->|"LoRa Broadcast: MSG_TRAIN_DETECTED (ETA)"| D2_LoRa
    D1_LoRa -.->|"LoRa Broadcast: Telemetry"| D3_LoRa
    D2_LoRa -.->|"LoRa Point-to-Point: MSG_CROSSING_STATUS"| D3_LoRa
    D2_Actuators --> LocalWarning --> RoadUsers
    D3_HMI --> InCabinAlert --> TrainDriver
```

---

## 2. Node Responsibilities & Internal Pipelines

### 2.1 Device 1: Upstream Train Detection Node (`device-1-upstream`)
* **Physical Placement:** Mounted rigidly on the railway track rail web/sleeper approximately 500 m to 1,000 m ahead of the level crossing.
* **Sensing Modalities:**
  * **Acoustics:** INMP441 omnidirectional MEMS microphone operating over I2S digital audio at 16 kHz sampling rate (captures wheel-rail screech, engine roar, horn).
  * **Vibrations:** MPU6050 6-axis IMU accelerometer sampled via I2C at $\ge 100\text{ Hz}$ (captures structural P-waves and surface rail acoustic emissions).
* **Edge Inference (CustomNet):**
  * Time-domain signals undergo fixed-point FFT/STFT windowing to produce logarithmic Mel-spectrograms.
  * An INT8-quantized compact 2D-CNN (**CustomNet**, ~1.63 MB footprint) executes inference within $\le 1.2\text{ s}$ to classify train acoustic signatures vs. environmental noise (heavy road traffic, wind, thunderstorms).
  * Feature growth dynamics estimate the Train Arrival Time (**ETA**).
* **Communication:** Assembles and transmits a binary LoRa packet (`MSG_TRAIN_DETECTED`, `ETA`, `Battery Voltage`) at 433 MHz. Continues transmitting 1.0 Hz periodic updates while the train approaches.

### 2.2 Device 2: Crossing Danger-Zone Monitoring Node (`device-2-crossing`)
* **Physical Placement:** Stationed roadside at the level crossing on a rigid mast with a direct field of view over the railway-road intersection danger zone.
* **Power Management:** Remains in low-power LoRa listening sleep mode until an incoming wake-up packet from Device 1 is received, avoiding continuous battery drain.
* **Perception Modalities:**
  * **Vision:** ESP32-S3-CAM with OV2640 sensor (IR-cut filter removed) paired with an 850 nm high-power infrared illuminator for zero-lux night vision.
  * **mmWave Radar:** HLK-LD2410 24 GHz FMCW radar communicating over UART at 3.3V logic. Provides range, speed, and energy detection immune to dense fog, monsoon downpours, smoke, or optical glare.
* **Edge Vision & Fusion Pipeline:**
  * **MSL-YOLO Inference:** Runs an INT8-quantized MSL-YOLO object detection model processing frames at $\ge 5\text{ FPS}$ to identify vehicles, motorcycles, tuk-tuks, carts, and pedestrians within a calibrated danger-zone Region of Interest (ROI).
  * **Stationary Dwell-Time Filter:** A temporal bounding-box tracker measures dwell time. Transient objects traversing the crossing normally (dwell $< 4.0\text{ s}$) are ignored. If an object remains stationary in the danger zone for $\ge 4.0\text{ s}$ (stalled vehicle, trapped cart), an obstruction event is confirmed.
  * **Radar Cross-Validation:** Fuses vision bounding boxes with radar range bins. If vision is occluded by mud or extreme weather, radar target presence confirms the obstacle.
* **Actuation:**
  * Energizes local 5V optocoupled relays to fire a 12V 110 dB siren and red flashing strobe beacon to warn crossing road users.
  * Dispatches `MSG_CROSSING_STATUS` (`CLEAR` vs. `OBSTRUCTED`) via LoRa to Device 3.

### 2.3 Device 3: Train Driver Cabin Unit (`device-3-cabin`)
* **Physical Placement:** Installed in the locomotive operator cab, powered by cabin auxiliary power with internal battery backup.
* **HMI Indicators:**
  * **Visual Screen:** 1602/2004 character I2C LCD (or 1.3" I2C OLED display) showing real-time text: approaching train status, remaining ETA countdown, and crossing state.
  * **Discrete Status LEDs:** Solid Green = Crossing Clear; Flashing Red = Crossing Obstructed / Hazard.
  * **Acoustic Warning:** High-output 5V piezo buzzer emitting continuous or pulsed warning tones.
* **Fail-Safe Watchdog (4.0s Packet Timeout):**
  * During an active train approach, Device 3 expects periodic telemetry packets every 1.0 s.
  * If no packet is received for $> 4.0\text{ s}$, Device 3 triggers a **Fail-Safe Caution Alert** (pulsing buzzer, warning indicator, LCD communication lost warning) instructing the driver to prepare for potential stop/caution.

---

## 3. End-to-End Operational Sequences

### 3.1 Nominal Sequence: Clear Crossing
```mermaid
sequenceDiagram
    autonumber
    participant D1 as Device 1 (Upstream)
    participant D2 as Device 2 (Crossing)
    participant D3 as Device 3 (Cabin)
    participant HW as Relays & Local Sirens

    Note over D1: Continuous Low-Power Acoustic & Track Vibration Sampling
    Note over D2: Low-Power LoRa Sleep Mode
    Note over D3: Standby Mode (Display Idle)

    Note over D1: Train Signature Detected by CustomNet (ETA = 45s)
    D1->>D2: LoRa Broadcast: MSG_TRAIN_DETECTED (ETA=45s, Seq=1)
    D1->>D3: LoRa Broadcast: MSG_TRAIN_DETECTED (ETA=45s, Seq=1)

    Note over D3: Display Updates: ETA 45s | Status: AWAITING CROSSING
    Note over D2: Wake up CAM, IR Illuminator & mmWave Radar
    Note over D2: MSL-YOLO Inference (≥5 FPS) + Radar Scan
    Note over D2: Danger Zone ROI Evaluated: No Stationary Obstacle

    D2->>D3: LoRa P2P: MSG_CROSSING_STATUS (State=CLEAR, ETA=43s, Seq=2)
    Note over D3: Resets 4.0s Watchdog
    Note over D3: Display: CLEAR | Solid Green LED ON | Buzzer OFF

    loop Every 1.0s Until Train Passes
        D1->>D2: Telemetry Heartbeat (ETA update)
        D2->>D3: MSG_CROSSING_STATUS (State=CLEAR)
        Note over D3: Resets 4.0s Watchdog | Updates Display ETA
    end

    Note over D1: Train Cleared Upstream Node
    D1->>D2: LoRa Broadcast: MSG_TRAIN_CLEARED
    Note over D2: Power Down Vision Sensors -> Return to Sleep
    D2->>D3: MSG_CROSSING_STATUS (State=IDLE)
    Note over D3: Reset to Standby
```

### 3.2 Hazard Sequence: Trapped Vehicle in Danger Zone
```mermaid
sequenceDiagram
    autonumber
    participant D1 as Device 1 (Upstream)
    participant D2 as Device 2 (Crossing)
    participant D3 as Device 3 (Cabin)
    participant HW as Relays, Siren & Strobe

    D1->>D2: LoRa Broadcast: MSG_TRAIN_DETECTED (ETA=40s)
    D1->>D3: LoRa Broadcast: MSG_TRAIN_DETECTED (ETA=40s)

    Note over D2: Wake Up: Camera & mmWave Radar Active
    Note over D2: MSL-YOLO Detects Vehicle in Danger Zone ROI
    Note over D2: Vehicle Dwell Time Counter Starts (t=0s)

    loop Dwell Tracking
        Note over D2: Frame analysis confirms vehicle stationary: t ≥ 4.0s
    end

    Note over D2: OBSTRUCTION CONFIRMED (Vision + Radar Fusion)
    D2->>HW: Energize Relays (≤100ms delay)
    Note over HW: 110dB Siren Sounds + Red Strobe Beacon Flashes
    
    D2->>D3: LoRa Immediate: MSG_CROSSING_STATUS (State=OBSTRUCTED, ETA=34s)
    Note over D3: Resets 4.0s Watchdog
    Note over D3: FLASHING RED LED ON | Active Piezo Buzzer Continuous Alarm
    Note over D3: Display: !! HAZARD: OBSTRUCTED !! ETA 34s
    Note over D3: Train Operator Initiates Emergency Braking / Horn
```

### 3.3 Fail-Safe Sequence: LoRa Packet Loss / RF Interference
```mermaid
sequenceDiagram
    autonumber
    participant D1 as Device 1 (Upstream)
    participant D2 as Device 2 (Crossing)
    participant D3 as Device 3 (Cabin)

    D1->>D3: LoRa Broadcast: MSG_TRAIN_DETECTED (ETA=50s)
    Note over D3: Active Approach Started. Arm 4.0s Fail-Safe Watchdog.
    D2->>D3: LoRa: MSG_CROSSING_STATUS (State=CLEAR, Seq=1)
    Note over D3: Watchdog Reset to 4.0s

    Note over D2: Device 2 transmits, but packet is blocked by RF obstruction/interference
    D2-x D3: LoRa: MSG_CROSSING_STATUS (Dropped)
    Note over D3: Watchdog at 2.0s...

    D2-x D3: LoRa: MSG_CROSSING_STATUS (Dropped)
    Note over D3: Watchdog at 3.0s...

    Note over D3: WATCHDOG TIMER EXPIRES (t ≥ 4.0s without valid packet)
    Note over D3: Trigger FAIL-SAFE CAUTION STATE:
    Note over D3: • Amber / Flashing Red Warning LED<br/>• Intermittent Warning Buzzer Cadence<br/>• Display: "COMM TIMEOUT / CAUTION"
    Note over D3: Driver alerted to approach with caution due to unconfirmed crossing status
```

---

## 4. Node Finite State Machines (FSM)

### 4.1 Device 1 State Machine
```mermaid
stateDiagram-v2
    [*] --> STATE_IDLE_MONITORING
    STATE_IDLE_MONITORING --> STATE_FEATURE_EXTRACTION: Energy Threshold Triggered
    STATE_FEATURE_EXTRACTION --> STATE_CNN_INFERENCE: STFT Mel-Spectrogram Ready
    STATE_CNN_INFERENCE --> STATE_IDLE_MONITORING: Classification < Threshold (Noise)
    STATE_CNN_INFERENCE --> STATE_TRAIN_CONFIRMED: Classification ≥ Threshold (Train)
    STATE_TRAIN_CONFIRMED --> STATE_BROADCAST_WAKEUP: Calculate Initial ETA
    STATE_BROADCAST_WAKEUP --> STATE_TELEMETRY_TRACKING: Transmit MSG_TRAIN_DETECTED
    STATE_TELEMETRY_TRACKING --> STATE_TELEMETRY_TRACKING: 1.0s Telemetry Cadence
    STATE_TELEMETRY_TRACKING --> STATE_RESET: Train Departure Signature Detected
    STATE_RESET --> STATE_IDLE_MONITORING: Broadcast MSG_TRAIN_CLEARED
```

* **`STATE_IDLE_MONITORING`:** Low-power continuous I2S and I2C circular buffer sampling. Low CPU clock.
* **`STATE_FEATURE_EXTRACTION`:** Dynamic signal energy triggers sliding-window STFT and Mel-frequency filterbank computation.
* **`STATE_CNN_INFERENCE`:** INT8 quantized CustomNet 2D-CNN evaluates spatial-spectral features ($\le 1.2\text{ s}$).
* **`STATE_TRAIN_CONFIRMED`:** Speed and distance curves yield integer ETA value (in seconds).
* **`STATE_BROADCAST_WAKEUP`:** Transmits LoRa wake-up broadcast packet to Devices 2 and 3.
* **`STATE_TELEMETRY_TRACKING`:** Periodically emits updated telemetry packets every 1.0 s.
* **`STATE_RESET`:** Confirms train exit, emits reset packet, returns to idle.

### 4.2 Device 2 State Machine
```mermaid
stateDiagram-v2
    [*] --> STATE_DEEP_SLEEP_LORA_WAIT
    STATE_DEEP_SLEEP_LORA_WAIT --> STATE_WAKEUP_INIT: LoRa Wake-Up Packet Received
    STATE_WAKEUP_INIT --> STATE_DANGER_ZONE_SCAN: Power Rails & Peripherals Online
    STATE_DANGER_ZONE_SCAN --> STATE_DWELL_EVALUATION: Object Detected in ROI
    STATE_DANGER_ZONE_SCAN --> STATE_BROADCAST_STATUS: No Object in ROI
    STATE_DWELL_EVALUATION --> STATE_OBSTRUCTION_ALARM: Dwell Time ≥ 4.0s (Or Radar Confirmed)
    STATE_DWELL_EVALUATION --> STATE_BROADCAST_STATUS: Object Cleared (Dwell < 4.0s)
    STATE_OBSTRUCTION_ALARM --> STATE_BROADCAST_STATUS: Relays Active (Siren + Strobe)
    STATE_BROADCAST_STATUS --> STATE_DANGER_ZONE_SCAN: Active Approach (ETA > 0)
    STATE_BROADCAST_STATUS --> STATE_TIMEOUT_SHUTDOWN: Train Cleared / Approach Expired
    STATE_TIMEOUT_SHUTDOWN --> STATE_DEEP_SLEEP_LORA_WAIT: Power Off Sensors & Relays
```

* **`STATE_DEEP_SLEEP_LORA_WAIT`:** SX1278 in CAD/RX mode, ESP32-S3 in light sleep, camera and illuminator unpowered.
* **`STATE_WAKEUP_INIT`:** High-side switches energize camera, 850nm IR illuminator, and HLK-LD2410 radar within $< 150\text{ ms}$.
* **`STATE_DANGER_ZONE_SCAN`:** MSL-YOLO vision model runs at $\ge 5\text{ FPS}$ on $320 \times 240$ frame stream.
* **`STATE_DWELL_EVALUATION`:** Tracks coordinates of vehicles/obstacles inside danger zone boundary; tracks dwell counter.
* **`STATE_OBSTRUCTION_ALARM`:** Fires dual relays to drive 110 dB siren and strobe beacon ($\le 100\text{ ms}$ post-verification).
* **`STATE_BROADCAST_STATUS`:** Emits `MSG_CROSSING_STATUS` to Device 3.
* **`STATE_TIMEOUT_SHUTDOWN`:** Train departure message or safety timeout shuts down high-draw peripherals and restores sleep.

### 4.3 Device 3 State Machine
```mermaid
stateDiagram-v2
    [*] --> STATE_CABIN_IDLE
    STATE_CABIN_IDLE --> STATE_ALERT_STANDBY: MSG_TRAIN_DETECTED Received
    STATE_ALERT_STANDBY --> STATE_APPROACH_MONITORING: Arm 4.0s Watchdog Timer
    STATE_APPROACH_MONITORING --> STATE_APPROACH_MONITORING: Packet Received (Watchdog Reset)
    STATE_APPROACH_MONITORING --> STATE_HAZARD_ALERT: MSG_CROSSING_STATUS == OBSTRUCTED
    STATE_APPROACH_MONITORING --> STATE_FAILSAFE_TIMEOUT: Watchdog Timer Expired (> 4.0s)
    STATE_HAZARD_ALERT --> STATE_APPROACH_MONITORING: Obstruction Cleared (Status == CLEAR)
    STATE_FAILSAFE_TIMEOUT --> STATE_APPROACH_MONITORING: Fresh Packet Received
    STATE_APPROACH_MONITORING --> STATE_CABIN_IDLE: MSG_TRAIN_CLEARED / ETA Expired
```

* **`STATE_CABIN_IDLE`:** Display shows standby status; solid low-power indicators; listening on LoRa.
* **`STATE_ALERT_STANDBY`:** Train detection registered; display shows initial ETA; arms 4.0s packet watchdog.
* **`STATE_APPROACH_MONITORING`:** Periodic packets reset the 4.0s watchdog timer; green LED remains active if crossing clear.
* **`STATE_HAZARD_ALERT`:** Crossing obstructed; flashing red LED; continuous piezo buzzer alarm; LCD warning.
* **`STATE_FAILSAFE_TIMEOUT`:** Missing telemetry during active train approach; buzzer caution pulses; amber warning.

---

## 5. Non-Functional & Performance SLAs

Derived directly from the System Requirements Specification (SRS Table 2) and field validation criteria:

| Parameter | Operational Threshold | Design Target | Failure Condition |
|---|---|---|---|
| **Upstream Detection to Alert Signal** | $\le 1.5\text{ s}$ | $1.0\text{ s}$ | $> 2.0\text{ s}$ latency |
| **LoRa Packet Airtime (SF7, 125 kHz)** | $\le 120\text{ ms}$ | $85\text{ ms}$ | $> 150\text{ ms}$ airtime |
| **Relay Trigger Delay Post-Dwell** | $\le 100\text{ ms}$ | $50\text{ ms}$ | $> 200\text{ ms}$ |
| **In-Cabin Display Refresh Rate** | $\ge 2\text{ Hz}$ | $5\text{ Hz}$ | $< 1\text{ Hz}$ |
| **Vision Inference Rate (MSL-YOLO)** | $\ge 5\text{ FPS}$ | $8\text{ FPS}$ | $< 3\text{ FPS}$ |
| **Stationary Dwell-Time Filter** | $4.0\text{ s} \pm 0.2\text{ s}$ | $4.0\text{ s}$ | False trigger on passing cars ($<2.5\text{ s}$) |
| **Packet Reception Rate (PRR)** | $\ge 95\%$ @ 500m LoS | $\ge 98\%$ @ 500m LoS | $< 90\%$ PRR |
| **Fail-Safe Watchdog Timeout** | $4.0\text{ s} \pm 0.1\text{ s}$ | $4.0\text{ s}$ | Undetected link drop ($> 4.5\text{ s}$) |
| **Off-Grid Autonomy Buffer** | $\ge 24\text{ hours}$ | $48\text{ hours}$ | Brownout during overcast weather |

---

## 6. Safety & Integrity Architecture

1. **Fail-Safe Default Philosophy:** In railway engineering, the absence of confirmed safety must default to the presence of danger. If communication drops during a confirmed train approach, Device 3 automatically triggers a fail-safe caution alert.
2. **Multi-Modal Redundancy:** Device 2 does not rely solely on optical computer vision. When optical degradation occurs (monsoon rain, dust, lens mud, complete darkness), the 24 GHz mmWave radar provides redundant spatial presence detection.
3. **Execution Integrity:** Firmware tasks are governed by hardware Task Watchdog Timers (WDT) set to 2.0 seconds to prevent software freezes or deadlock from halting safety-critical loops.
4. **Local Autonomy:** Each node possesses full authority to execute its primary safety action locally (e.g. Device 2 triggers siren/strobe relays directly without needing a cloud handshake or remote authorization).
