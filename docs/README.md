# 📊 ESP32 IoT Monitoring System

[![C](https://img.shields.io/badge/Language-C-00599C?style=flat-square&logo=c)](https://github.com/JoseManuelEnriquez/monitoring-system-esp32)
[![ESP32](https://img.shields.io/badge/Hardware-ESP32-E7352C?style=flat-square&logo=espressif)](https://github.com/JoseManuelEnriquez/monitoring-system-esp32)
[![ESP-IDF](https://img.shields.io/badge/Framework-ESP--IDF-blue?style=flat-square)](https://github.com/JoseManuelEnriquez/monitoring-system-esp32)
[![FreeRTOS](https://img.shields.io/badge/RTOS-FreeRTOS-green?style=flat-square)](https://github.com/JoseManuelEnriquez/monitoring-system-esp32)
[![MQTT](https://img.shields.io/badge/Protocol-MQTT-3C3C3C?style=flat-square&logo=mqtt&logoColor=white)](https://github.com/JoseManuelEnriquez/monitoring-system-esp32)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](https://opensource.org/licenses/MIT)

A real-time IoT environmental monitoring system built on an **ESP32** microcontroller. The firmware — written in C with **ESP-IDF** and **FreeRTOS** — implements a **Finite State Machine** to manage three operating modes: active data acquisition, configuration, and deep sleep. Sensor telemetry is published over **MQTT** to a **Node-RED** dashboard.

---

## 📐 System Architecture

```
┌─────────────────────┐        MQTT (Wi-Fi)        ┌──────────────────────┐
│     ESP32 Firmware  │ ─────────────────────────► │  Mosquitto Broker    │
│                     │                             │  (Eclipse, local)    │
│  FreeRTOS Tasks     │ ◄───────────────────────── │                      │
│  └─ FSM             │      Commands / Config      └──────────┬───────────┘
│  └─ Sensor Task     │                                        │
│  └─ MQTT Task       │                             ┌──────────▼───────────┐
│                     │                             │   Node-RED Dashboard │
│  Sensors            │                             │   (Real-time charts) │
│  └─ DHT11 (temp/hum)│                             └──────────────────────┘
│  └─ LDR (light)     │
└─────────────────────┘
```

---

## 🧠 Firmware Design

### Why FreeRTOS?

A bare `while(1)` loop would block on sensor reads and MQTT I/O, making it impossible to respond to incoming commands without missing data. FreeRTOS allows each concern — sensor acquisition, MQTT publishing, and command handling — to run as an independent task, scheduled cooperatively with clear priorities. This keeps the system responsive even under poor network conditions.

### Finite State Machine (FSM)

The system operates in one of three states, managed by a FreeRTOS task:

| State | LED | Behaviour |
|---|---|---|
| ⚡ **Performance** | 🟢 Green (GPIO 21) | Reads sensors periodically and publishes telemetry via MQTT |
| ⚙️ **Configuration** | 🟡 Yellow (GPIO 22) | Accepts parameter changes (e.g. sampling delay) via MQTT |
| 💤 **Sleep** | 🔴 Red (GPIO 23) | Pauses acquisition; Wi-Fi stays alive to receive wake commands |

State transitions are triggered by physical buttons (GPIO 26 / 27) or MQTT commands from the broker.

### Why Mosquitto?

Eclipse Mosquitto was chosen as the broker for its minimal memory footprint — ideal for local deployments alongside the ESP32. Anonymous access is disabled; all connections require credentials defined in a pre-configured user list.

---

## 🔌 Hardware & Pinout

### Sensors & Controls (Inputs)

| Component | Type | GPIO | Description |
|---|---|---|---|
| DHT11 | Sensor | `18` | Temperature & relative humidity |
| LDR | Sensor | `19` | Ambient light intensity |
| Mode / Wake button | Push button | `26` | Toggle Performance ↔ Config; wake from Deep Sleep |
| Sleep button | Push button | `27` | Force immediate Deep Sleep |

### Status LEDs (Outputs)

| LED | Color | GPIO | Meaning |
|---|---|---|---|
| Performance | 🟢 Green | `21` | System active, publishing data |
| Config | 🟡 Yellow | `22` | Configuration mode active |
| Sleep | 🔴 Red | `23` | Entering / in Deep Sleep |
| Setup error | 🔴 Red | `25` | Wi-Fi or MQTT not configured |
| Connected | 🟢 Green | `17` | Wi-Fi + MQTT ready |

![Circuit diagram](img/circuito.png)

---

## ☁️ MQTT Topics

Topic pattern: `<device_type>/<device_id>/<category>/<subcategory>`

### Telemetry — ESP32 → Broker

| Metric | Topic | Payload | Unit |
|---|---|---|---|
| Temperature | `ESP32/{id}/telemetry/temperature` | `int` | °C |
| Humidity | `ESP32/{id}/telemetry/humidity` | `int` | % |
| Light | `ESP32/{id}/telemetry/light` | `bool` | — |
| Error | `ESP32/{id}/error` | `{"error": "description"}` | — |

### Commands — Broker → ESP32

| Command | Topic | Payload |
|---|---|---|
| Force Sleep | `ESP32/{id}/config/OFF` | — |
| Performance mode | `ESP32/{id}/config/ON` | — |
| Configuration mode | `ESP32/{id}/config/CONFIG` | — |
| Set sampling delay | `ESP32/{id}/config/delay` | `{"delay": <ms>}` |

> ⚠️ Minimum sampling interval: **2000 ms**

![Communication diagram](img/Comunicaciones.png)

---

## 📁 Project Structure

```
monitoring-system-esp32/
├── firmware/
│   └── esp32-circuit/       # ESP-IDF project (main firmware)
├── dashboard/               # Node-RED flow export
├── docs/
│   └── img/                 # Circuit & communication diagrams
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites

- [ESP-IDF v5.x](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/)
- [Mosquitto MQTT broker](https://mosquitto.org/download/)
- [Node-RED](https://nodered.org/docs/getting-started/)

### Firmware setup

```bash
# Clone the repository
git clone https://github.com/JoseManuelEnriquez/monitoring-system-esp32.git
cd monitoring-system-esp32/firmware/esp32-circuit

# Configure Wi-Fi credentials and MQTT broker address
idf.py menuconfig

# Build and flash
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### Broker setup

```bash
# Start Mosquitto with authentication
mosquitto -c mosquitto.conf
```

### Dashboard

Import the flow from `dashboard/` into your Node-RED instance and point the MQTT nodes to your broker address.

---

## 🎬 Demo

https://github.com/JoseManuelEnriquez/monitoring-system-esp32/assets/127325503/1468c1e0-405c-44dc-9d9b-48d370249971

---

## 🛠️ Dependencies

| Library | Purpose |
|---|---|
| ESP-IDF | Official Espressif framework |
| FreeRTOS | Task scheduling & synchronisation |
| DHT11 driver | Sensor abstraction layer |
| Frozen | Lightweight JSON serialiser for MQTT payloads |

---

## 📄 License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
