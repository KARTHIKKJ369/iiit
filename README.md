<div align="center">

# IoT-Based Intelligent Backwater Boat Collision Avoidance System

*Edge AI trajectory forecasting, cooperative avoidance maneuvers, and real-time geospatial risk assessment for narrow inland waterways.*

[![Python](https://img.shields.io/badge/Python-3.10%20%7C%203.11-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.111-009688?style=flat-square&logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-19-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.8-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![TensorFlow Lite](https://img.shields.io/badge/TFLite-Edge%20Inference-FF6F00?style=flat-square&logo=tensorflow&logoColor=white)](https://www.tensorflow.org/lite)
[![MQTT](https://img.shields.io/badge/MQTT-Mosquitto%20v2-660066?style=flat-square&logo=eclipsemosquitto&logoColor=white)](https://mosquitto.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)

[Overview](#overview) • [Key Features](#key-features) • [System Architecture](#system-architecture) • [Quick Start](#quick-start) • [Local Development](#local-development) • [Simulation Scenarios](#simulation-scenarios) • [Cooperative Avoidance](#cooperative-avoidance) • [API & MQTT Reference](#api--mqtt-reference) • [Edge AI & Model Pipeline](#edge-ai--model-pipeline) • [Evaluation & Benchmarks](#evaluation--benchmarks)

</div>

---

## Overview

Navigating narrow inland waterways, blind canal bends, and heavily trafficked backwaters presents severe collision risks due to restricted maneuvering space, obstructed lines of sight, and dynamic weather conditions. Traditional marine radar and AIS systems are often cost-prohibitive or ineffective for small-to-medium vessels in tight waterways.

This project provides an end-to-end, software-first and hardware-ready collision avoidance platform designed for backwater vessels. It integrates high-frequency sensor telemetry (GPS, 6-DOF IMU, and obstacle flags), LoRa/MQTT communications, multi-factor kinematic risk scoring, quantized LSTM trajectory prediction on edge hardware (Raspberry Pi), cooperative collision avoidance recommendations, and a real-time geospatial React dashboard.

> [!NOTE]
> The platform includes a self-contained multi-scenario engine and standalone evaluation suite, allowing complete end-to-end simulation, model inference, and metric analysis without requiring physical boat hardware or external network access.

---

## Key Features

- **Multi-Sensor Telemetry Pipeline**: Ingests GPS coordinates, speed, heading, 6-DOF IMU (accelerometer $a_x, a_y, a_z$ and gyroscope $g_x, g_y, g_z$), and computer-vision/sensor obstacle alerts via MQTT.
- **Dynamic Multi-Factor Risk Engine**: Computes spatial proximity (Haversine), relative closing velocity, heading divergence, time-to-collision (TTC), and dedicated same-direction sudden-stop / overtaking risk.
- **Weather-Aware Adaptive Thresholds**: Integrates live OpenWeatherMap data (visibility, wind speed, precipitation) to dynamically scale safety radii, danger zones, and vessel reaction time buffers ($4\,\text{s} \to 8\,\text{s}$).
- **Edge AI Trajectory Forecasting**: 11-channel multi-step LSTM model predicting 15 seconds of future vessel coordinates ($(\text{lat}, \text{lon})$) with confidence scoring, optimized as a quantized TFLite model ($\approx 2.4\,\text{ms}$ inference latency) with automatic fallback to kinematic dead-reckoning.
- **Predictive Collision Detection**: Compares future trajectory point pairs step-by-step to identify minimum projected separation and classify states into `SAFE`, `WARNING`, and `DANGER` with debounced state-transition alerts.
- **Cooperative Avoidance & Recommendations**: COLREGS-informed emergency and early-warning maneuver recommendations (`TURN_RIGHT`, `HARD_LEFT`, `SLOW_DOWN`, `MAINTAIN`) dispatched over MQTT with two-way operator acknowledgment tracking.
- **Geospatial Real-Time Dashboard**: Modern interactive interface built with React 19, Leaflet, TanStack Router, and Tailwind CSS displaying live vessel tracks, predicted trajectories, collision cones, telemetry logs, weather widgets, and scenario controls.

---

## System Architecture

```text
  +-------------------------------------------------------------------------+
  |                              Vessel Nodes                               |
  |  ESP32 / Virtual Sensor Node (GPS + 6-DOF IMU + Obstacle Camera + LoRa) |
  +------------------------------------+------------------------------------+
                                       | LoRa (SX1278 / 868-915 MHz)
                                       v
  +-------------------------------------------------------------------------+
  |                          Shore / Gateway Node                           |
  |             LoRa-to-MQTT Bridge & Mosquitto Message Broker              |
  +------------------------------------+------------------------------------+
                                       | MQTT Topics (boats/{id}/sensor)
                                       v
  +-------------------------------------------------------------------------+
  |                           FastAPI Backend Core                          |
  |                                                                         |
  |  +---------------------+   +--------------------+   +----------------+  |
  |  | Ingestion & Storage |-->| Dynamic Risk & TTC |-->| Prediction     |  |
  |  | (SQLite DB)         |   | Engine + Weather   |   | Gating & Trigger| |
  |  +---------------------+   +--------------------+   +-------+--------+  |
  |                                                             |           |
  |  +---------------------+   +--------------------+           v           |
  |  | Avoidance Maneuver  |<--| Future Trajectory  |<--+----------------+  |
  |  | Decision Engine     |   | Intersection Check |   | Edge AI / LSTM |  |
  |  +----------+----------+   +--------------------+   | (TFLite / Pi)  |  |
  +-------------|---------------------------------------+----------------+--+
                |                                               |
                | REST / WebSocket                              | MQTT
                v                                               v
  +-----------------------------+               +-------------------------------+
  |   React Leaflet Dashboard   |               | Vessel Autopilot / Alert Unit |
  | (Live Map, Metrics, History)|               | (Audio/Visual Maneuver Direct)|
  +-----------------------------+               +-------------------------------+
```

### Module Breakdown

| Module | Location | Description |
|---|---|---|
| **Backend API** | `backwater-boat/backend/api/` | FastAPI service handling ingestion, database access, scenario orchestration, and metrics. |
| **Risk Engine** | `backwater-boat/backend/risk_engine/` | Kinematic spatial analysis, closing speed, heading disparity, TTC calculation, and weather scaling. |
| **Avoidance Engine** | `backwater-boat/backend/avoidance/` | Decision logic generating proactive and emergency collision avoidance maneuvers. |
| **Edge ML** | `backwater-boat/ml/` | Synthetic dataset generation, 11-feature LSTM model training, and quantized TFLite inference engine. |
| **Web Dashboard** | `backwater-boat/dashboard/` | High-performance React 19 + Leaflet GIS frontend with live scenario controls and analytics. |
| **Simulator** | `backwater-boat/simulator/` | Multi-vessel simulation environment generating realistic physics, IMU channels, and waterway hazards. |
| **Firmware** | `backwater-boat/firmware/` | ESP32 sensor acquisition sketches and Raspberry Pi LoRa-to-MQTT bridge specifications. |

---

## Quick Start

The entire stack can be launched with Docker Compose:

```bash
cd backwater-boat
docker compose up --build
```

> [!TIP]
> To launch the standalone simulation container simultaneously with the core stack, pass the `dev` profile:
> ```bash
> docker compose --profile dev up --build
> ```

### Services & Port Mappings

| Service | Endpoint / Port | Description |
|---|---|---|
| **Web Dashboard** | [http://localhost:8080](http://localhost:8080) | Interactive GIS navigation & monitoring dashboard |
| **Backend REST API** | [http://localhost:8000](http://localhost:8000) | Core application API |
| **Interactive API Docs** | [http://localhost:8000/docs](http://localhost:8000/docs) | Swagger / OpenAPI specification |
| **Mosquitto MQTT** | `localhost:1883` | Telemetry & maneuver message broker |

---

## Local Development

### 1. Backend Service

```bash
cd backwater-boat/backend
pip install -r requirements.txt
PYTHONPATH=.. uvicorn backend.api.main:app --reload --port 8000
```

### 2. Web Dashboard

```bash
cd backwater-boat/dashboard
npm install
npm run dev
```
Dashboard dev server will start at `http://localhost:5173`.

### 3. Simulation Generator

Run a specific collision scenario against the local backend and MQTT broker:

```bash
cd backwater-boat/simulator
pip install -r requirements.txt
MQTT_HOST=localhost python boat_sim.py --scenario HEAD_ON
```

### 4. Unit Tests

```bash
cd backwater-boat
PYTHONPATH=. python -m unittest discover backend/tests
```

---

## Simulation Scenarios

The system includes pre-calibrated navigation scenarios representing common high-risk maritime encounters in backwaters:

```text
HEAD-ON COLLISION                   CROSSING ENCOUNTER
Boat 1 --->   <--- Boat 2           Boat 1 --->
                                                |
                                                v Boat 2

BLIND CANAL BEND                    SUDDEN STOP / OVERTAKING
   \       /                        Boat 1 ---> [ Boat 2 Braking ]
    \ B1  /
     \ ^ /
      | | 
     / v \
    /  B2 \
```

- **`HEAD_ON`**: Two vessels approaching each other at maximum speed ($8.0\,\text{m/s}$) along the same canal axis. Triggers `DANGER` alert state within 10 seconds.
- **`CROSSING`**: Vessels on perpendicular intersecting trajectories. Evaluates give-way vessel turning responses.
- **`BLIND_TURN`**: Both vessels navigating around an obstructed canal bend with reduced visibility and obstacle flags active.
- **`SUDDEN_STOP`**: In-line vessel transit where the lead boat abruptly decelerates from $6\,\text{m/s}$ to $0\,\text{m/s}$, verifying the overtake and longitudinal closure engine.

Scenarios can be triggered directly from the Web Dashboard UI or via API:

```bash
curl -X POST http://localhost:8000/scenarios/HEAD_ON/run
```

---

## Cooperative Avoidance

When a collision risk is predicted, the backend issues maneuver directives according to the encounter geometry:

```text
Telemetry Received
       │
       ▼
Risk & TTC Evaluation ──[ Risk ≥ 0.3 or Dist < 150m ]──► LSTM Trajectory Inference
                                                                  │
                                                                  ▼
Alert Dispatched ◄───[ State: WARNING / DANGER ]─── Predictive Separation Check
       │
       ▼
Avoidance Maneuver Generated
       │
       ├─► MQTT: boats/{id}/recommendation
       └─► Dashboard Alert Banner
```

### Maneuver Decision Logic

- **Head-On ($\Delta\text{Heading} \ge 150^\circ$)**: Issues `TURN_RIGHT` (or `HARD_RIGHT` if $\text{TTC} < 5\,\text{s}$) to pass port-to-port.
- **Crossing ($20^\circ \le \Delta\text{Heading} < 150^\circ$)**: Issues `TURN_LEFT` / `HARD_LEFT` to open lateral separation.
- **Overtaking / Sudden Stop ($\Delta\text{Heading} < 20^\circ$)**: Issues `SLOW_DOWN` to avoid rear-end collisions without hazardous lateral steering in narrow channels.

#### Sample Recommendation Payload (`boats/{id}/recommendation`)

```json
{
  "boat_id": "B01",
  "action": "HARD_RIGHT",
  "alert_state": "DANGER",
  "accepted": false
}
```

---

## API & MQTT Reference

### Key REST Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/health` | Backend service health check |
| `GET` | `/boats` | List all registered vessels |
| `GET` | `/telemetry/latest` | Retrieve current position, speed, and heading for all active vessels |
| `POST` | `/telemetry` | Ingest vessel telemetry payload directly (HTTP fallback) |
| `GET` | `/alerts?limit=100` | Fetch chronological warning and danger alerts |
| `GET` | `/predictions` | Retrieve latest 15-second future coordinate paths |
| `POST` | `/predict` | Manually trigger trajectory prediction for a vessel |
| `GET` | `/recommendations` | Get active cooperative avoidance directives |
| `POST` | `/boats/{id}/ack` | Acknowledge receipt and execution of maneuver advice |
| `GET` | `/scenarios` | List available built-in simulation scenarios |
| `POST` | `/scenarios/{name}/run` | Execute scenario background simulation |
| `GET` | `/weather?lat={lat}&lon={lon}` | Get live weather factor and atmospheric data |
| `GET` | `/evaluation?scenario={name}` | Compute precision, recall, F1, and TTC evaluation metrics |

### MQTT Topic Taxonomy

| Topic | Direction | Description |
|---|---|---|
| `boats/{id}/sensor` | Publish (Vessel $\to$ Broker) | Real-time GPS, IMU, speed, heading, and obstacle data |
| `boats/{id}/predict` | Publish (Broker $\to$ Vessel) | Broadcasted 15-second future waypoint sequence |
| `boats/{id}/alert` | Publish (Broker $\to$ Vessel) | Alert transitions (`SAFE`, `WARNING`, `DANGER`) |
| `boats/{id}/recommendation` | Publish (Broker $\to$ Vessel) | Maneuver advice (`TURN_RIGHT`, `SLOW_DOWN`, etc.) |
| `boats/{id}/ack` | Publish (Vessel $\to$ Broker) | Operator maneuver acceptance confirmation |

---

## Edge AI & Model Pipeline

### 11-Channel Feature Representation

The trajectory prediction model takes a sliding history window of 10 timesteps ($10\,\text{s}$) across 11 normalized kinematic and inertial channels:

$$\mathbf{x}_t = \big[ \text{lat}_n,\, \text{lon}_n,\, \frac{\text{speed}}{10},\, \sin(\theta),\, \cos(\theta),\, a_{x,n},\, a_{y,n},\, a_{z,n},\, g_{x,n},\, g_{y,n},\, g_{z,n} \big]$$

### Model Training & Quantization

To train a new LSTM model and export standard Keras (`.h5`) and quantized TensorFlow Lite (`.tflite`) weights:

```bash
cd backwater-boat
pip install -r ml/requirements.txt
python ml/training/train_lstm.py
```

Output artifacts generated in `ml/`:
- `model.tflite`: Quantized model for Raspberry Pi deployment.
- `model.h5`: Full-precision Keras reference model.
- `norm_params.json`: Feature normalization constants required during inference.

### Raspberry Pi Edge Deployment

Only `model.tflite` and `norm_params.json` are required on the vessel edge processor:

```bash
# On Raspberry Pi (Edge Gateway)
pip install tflite-runtime numpy
```

> [!IMPORTANT]
> The edge inference runner automatically detects the execution environment. It prioritizes `tflite-runtime` for lightweight sub-3ms edge execution, falls back to full TensorFlow if installed, and uses a kinematic dead-reckoning estimator if no model runtime is present.

---

## Evaluation & Benchmarks

The framework includes a standalone offline evaluation suite that calculates collision detection performance against mathematical ground-truth trajectory intersections.

### Run Standalone Evaluation

```bash
cd backwater-boat
python evaluate_standalone.py
```

### Benchmark Summary Across Scenarios

| Metric | Overall Benchmark |
|---|---|
| **Evaluated Scenarios** | 4 (`HEAD_ON`, `CROSSING`, `BLIND_TURN`, `SUDDEN_STOP`) |
| **Mean Inference Latency** | **$2.42\,\text{ms}$** |
| **Mean Trajectory Prediction Error** | **$5.57\,\text{m}$** (at 15 s forecast horizon) |
| **Average Time-to-Collision (TTC) Lead Time** | **$50.89\,\text{s}$** |
| **Head-On Collision Recall** | **$100.0\%$** ($0$ false negatives) |
| **Sudden Stop Collision Recall** | **$35.7\%$** ($5$ True Positives / $0$ Missed Impacts) |

Detailed per-scenario logs are exported to `results/summary.json` and `results/{scenario}.csv`.
