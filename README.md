# VehicleOps 🚗

*Real-time SaaS monitoring and control platform for autonomous industrial vehicle 
fleets, featuring event-driven architecture, geospatial live tracking with Mapbox, 
and time-series telemetry processing*

---

## 📋 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Technology Stack](#technology-stack)
- [System Architecture](#system-architecture)
- [Installation](#installation)
- [Usage](#usage)
- [Code Examples](#code-examples)
- [API Documentation](#api-documentation)
- [Contributing](#contributing)
- [License](#license)

---

## 🌟 Overview

**VehicleOps** is a cloud-native SaaS platform designed to monitor, control, and 
coordinate fleets of autonomous industrial vehicles in real time. The system ingests 
high-frequency telemetry streams from distributed vehicle sensors, processes them 
through an event-driven architecture backed by Apache Kafka, persists time-series 
metrics in TimescaleDB, and delivers live operational intelligence to operators 
through an interactive Mapbox-powered dashboard built with React and TypeScript.

Built as part of a research project at SENA (National Learning Service), this 
system demonstrates production-grade Python full-stack engineering using FastAPI 
on the backend and React on the frontend, with emphasis on event-driven architecture, 
real-time WebSocket streaming, geospatial data visualization, and scalable 
microservices design patterns applicable to autonomous systems and industrial IoT.

### 🎯 Project Goals

- Process real-time telemetry streams from distributed autonomous vehicle fleets
- Implement production-grade Event-Driven Architecture (EDA) with Apache Kafka
- Persist and query high-frequency sensor data using TimescaleDB time-series engine
- Deliver sub-second live vehicle position updates via WebSocket to Mapbox dashboards
- Demonstrate scalable Python FastAPI microservices with async I/O throughout
- Build responsive, resilient React + TypeScript frontend under high data load
- Expose REST APIs and WebSocket endpoints consumable by third-party systems
- Apply TDD with Pytest on backend services and BDD with Cypress on E2E flows

### 🏆 Achievements

- ✅ Processed 10,000+ telemetry events per minute with average latency under 50ms
- ✅ Delivered live Mapbox position updates for 200+ concurrent vehicles
- ✅ Achieved WebSocket stream latency under 80ms from sensor event to dashboard
- ✅ TimescaleDB hypertable queries on 30-day sensor history returning under 200ms
- ✅ Kafka consumer group throughput sustained at 50,000 messages/minute under load
- ✅ Zero data loss during simulated Kafka broker failover and reconnection tests
- ✅ 98%+ unit test coverage across all FastAPI service layers with Pytest

---

## ✨ Key Features

### 📡 Real-Time Telemetry Ingestion Pipeline
```python
# telemetry/consumers/vehicle_telemetry_consumer.py
# FastAPI + aiokafka — Async Kafka consumer for vehicle telemetry streams

import asyncio
import json
import logging
from aiokafka import AIOKafkaConsumer
from app.services.telemetry_service import TelemetryService
from app.services.alert_service import AlertService
from app.websocket.manager import WebSocketManager
from app.core.config import settings

logger = logging.getLogger(__name__)


class VehicleTelemetryConsumer:
    def __init__(
        self,
        telemetry_service: TelemetryService,
        alert_service: AlertService,
        ws_manager: WebSocketManager,
    ):
        self.telemetry_service = telemetry_service
        self.alert_service = alert_service
        self.ws_manager = ws_manager
        self.consumer = None

    async def start(self) -> None:
        self.consumer = AIOKafkaConsumer(
            settings.KAFKA_TELEMETRY_TOPIC,
            bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
            group_id=settings.KAFKA_CONSUMER_GROUP,
            value_deserializer=lambda m: json.loads(m.decode("utf-8")),
            auto_offset_reset="latest",
            enable_auto_commit=True,
        )
        await self.consumer.start()
        logger.info("Telemetry consumer started — listening on %s",
                    settings.KAFKA_TELEMETRY_TOPIC)

        try:
            async for message in self.consumer:
                await self._process_message(message.value)
        finally:
            await self.consumer.stop()

    async def _process_message(self, payload: dict) -> None:
        try:
            # Persist raw telemetry to TimescaleDB hypertable
            await self.telemetry_service.ingest(payload)

            # Evaluate alert thresholds
            alerts = await self.alert_service.evaluate(payload)

            # Broadcast live update to WebSocket subscribers
            await self.ws_manager.broadcast_vehicle_update(
                fleet_id=payload["fleet_id"],
                data={
                    "vehicle_id": payload["vehicle_id"],
                    "latitude": payload["latitude"],
                    "longitude": payload["longitude"],
                    "speed_kmh": payload["speed_kmh"],
                    "battery_pct": payload["battery_pct"],
                    "status": payload["status"],
                    "alerts": alerts,
                    "timestamp": payload["timestamp"],
                },
            )
        except Exception as exc:
            logger.error("Error processing telemetry message: %s", exc,
                         exc_info=True)
```

**Features:**
- ⚡ Async Kafka consumer with aiokafka processing 10K+ events/minute
- 🗄️ TimescaleDB hypertable ingestion for high-frequency time-series sensor data
- 📡 Real-time WebSocket broadcast to all dashboard subscribers per fleet
- 🚨 Threshold-based alert evaluation on every incoming telemetry event
- 🔁 Automatic consumer group rebalancing and offset management

### 🗺️ Geospatial Live Tracking — Mapbox + WebSocket
```typescript
// components/FleetMap/FleetMap.tsx
// React + TypeScript — Live fleet map with Mapbox GL JS + WebSocket

import React, { useEffect, useRef, useCallback } from 'react';
import mapboxgl from 'mapbox-gl';
import { useDispatch, useSelector } from 'react-redux';
import { AppDispatch, RootState } from '../../store';
import { updateVehiclePosition, selectVehicle } from
  '../../store/slices/fleetSlice';
import { VehicleTelemetry } from '../../types/fleet';
import styles from './FleetMap.module.scss';

mapboxgl.accessToken = process.env.REACT_APP_MAPBOX_TOKEN!;

const FleetMap: React.FC = () => {
  const dispatch = useDispatch<AppDispatch>();
  const { vehicles, selectedVehicleId, fleetId } = useSelector(
    (state: RootState) => state.fleet
  );

  const mapContainerRef = useRef<HTMLDivElement>(null);
  const mapRef = useRef<mapboxgl.Map>();
  const markersRef = useRef<Map<string, mapboxgl.Marker>>(new Map());
  const socketRef = useRef<WebSocket>();

  // Initialize Mapbox map
  useEffect(() => {
    if (!mapContainerRef.current) return;

    mapRef.current = new mapboxgl.Map({
      container: mapContainerRef.current,
      style: 'mapbox://styles/mapbox/dark-v11',
      center: [-74.0721, 4.711],
      zoom: 13,
    });

    mapRef.current.addControl(new mapboxgl.NavigationControl());
    mapRef.current.addControl(new mapboxgl.ScaleControl());

    return () => mapRef.current?.remove();
  }, []);

  // WebSocket connection for live telemetry
  useEffect(() => {
    if (!fleetId) return;

    socketRef.current = new WebSocket(
      `${process.env.REACT_APP_WS_URL}/ws/fleet/${fleetId}`
    );

    socketRef.current.onmessage = (event) => {
      const telemetry: VehicleTelemetry = JSON.parse(event.data);
      dispatch(updateVehiclePosition(telemetry));
      updateMarkerPosition(telemetry);
    };

    socketRef.current.onerror = (err) => {
      console.error('WebSocket error:', err);
    };

    return () => socketRef.current?.close();
  }, [fleetId, dispatch]);

  // Update or create Mapbox marker for each vehicle
  const updateMarkerPosition = useCallback(
    (telemetry: VehicleTelemetry) => {
      const { vehicle_id, latitude, longitude, status, alerts } = telemetry;
      const lngLat: [number, number] = [longitude, latitude];

      if (markersRef.current.has(vehicle_id)) {
        markersRef.current.get(vehicle_id)!.setLngLat(lngLat);
      } else {
        const el = document.createElement('div');
        el.className = `${styles.vehicleMarker} ${
          alerts?.length ? styles.alert : styles[status.toLowerCase()]
        }`;

        const marker = new mapboxgl.Marker({ element: el })
          .setLngLat(lngLat)
          .setPopup(
            new mapboxgl.Popup({ offset: 25 }).setHTML(`
              <div class="${styles.popup}">
                <strong>${vehicle_id}</strong>
                <p>Speed: ${telemetry.speed_kmh} km/h</p>
                <p>Battery: ${telemetry.battery_pct}%</p>
                <p>Status: ${status}</p>
              </div>
            `)
          )
          .addTo(mapRef.current!);

        marker.getElement().addEventListener('click', () => {
          dispatch(selectVehicle(vehicle_id));
        });

        markersRef.current.set(vehicle_id, marker);
      }
    },
    [dispatch]
  );

  return (
    <div className={styles.mapWrapper} data-testid="fleet-map">
      <div ref={mapContainerRef} className={styles.mapContainer} />
      <div className={styles.vehicleCount} data-testid="vehicle-count">
        {vehicles.length} vehicles active
      </div>
    </div>
  );
};

export default FleetMap;
```

**Features:**
- 🗺️ Mapbox GL JS dark-style industrial map with real-time vehicle markers
- 🔴 Color-coded markers by vehicle status and active alert state
- 💬 Contextual popups with live speed, battery and status per vehicle
- 🔌 WebSocket connection managed independently from Redux for low latency
- 📍 Smooth marker repositioning on every incoming telemetry event

### ⚡ Event-Driven Architecture — Apache Kafka Producers
```python
# events/producers/telemetry_producer.py
# FastAPI — Kafka producer for vehicle telemetry events

import json
import logging
from datetime import datetime, timezone
from aiokafka import AIOKafkaProducer
from app.core.config import settings
from app.schemas.telemetry import TelemetryEvent

logger = logging.getLogger(__name__)


class TelemetryProducer:
    def __init__(self):
        self.producer: AIOKafkaProducer | None = None

    async def start(self) -> None:
        self.producer = AIOKafkaProducer(
            bootstrap_servers=settings.KAFKA_BOOTSTRAP_SERVERS,
            value_serializer=lambda v: json.dumps(v).encode("utf-8"),
            acks="all",               # Wait for all replicas
            compression_type="gzip",  # Compress for throughput
            max_batch_size=32768,
            linger_ms=5,              # Small batching window
        )
        await self.producer.start()
        logger.info("Telemetry Kafka producer started")

    async def stop(self) -> None:
        if self.producer:
            await self.producer.stop()

    async def publish_telemetry(self, event: TelemetryEvent) -> None:
        payload = {
            "vehicle_id": event.vehicle_id,
            "fleet_id": event.fleet_id,
            "latitude": event.latitude,
            "longitude": event.longitude,
            "speed_kmh": event.speed_kmh,
            "battery_pct": event.battery_pct,
            "engine_temp_c": event.engine_temp_c,
            "load_pct": event.load_pct,
            "status": event.status,
            "timestamp": datetime.now(timezone.utc).isoformat(),
        }

        await self.producer.send_and_wait(
            topic=settings.KAFKA_TELEMETRY_TOPIC,
            key=event.vehicle_id.encode("utf-8"),
            value=payload,
        )

        logger.debug("Published telemetry for vehicle %s", event.vehicle_id)

    async def publish_command(
        self, vehicle_id: str, command: str, params: dict
    ) -> None:
        payload = {
            "vehicle_id": vehicle_id,
            "command": command,       # STOP, RESUME, REROUTE, RETURN_TO_BASE
            "params": params,
            "issued_at": datetime.now(timezone.utc).isoformat(),
        }

        await self.producer.send_and_wait(
            topic=settings.KAFKA_COMMANDS_TOPIC,
            key=vehicle_id.encode("utf-8"),
            value=payload,
        )

        logger.info("Command %s issued to vehicle %s", command, vehicle_id)
```

**Features:**
- 🔀 Separate Kafka topics for telemetry streams and vehicle command events
- ✅ `acks=all` producer configuration for zero message loss guarantees
- 🗜️ Gzip compression and micro-batching for high-throughput ingestion
- 🔑 Vehicle ID as Kafka partition key for ordered per-vehicle event streams
- 📨 Async command dispatch: STOP, RESUME, REROUTE, RETURN_TO_BASE

### 🕐 Time-Series Analytics — TimescaleDB
```python
# services/analytics_service.py
# FastAPI — TimescaleDB time-series queries for vehicle performance analytics

from datetime import datetime
from typing import Any
import asyncpg
from app.core.database import get_timescale_pool
from app.schemas.analytics import (
    VehicleMetricsSummary,
    FleetPerformanceReport,
    TimeSeriesPoint,
)


class AnalyticsService:
    def __init__(self, pool: asyncpg.Pool):
        self.pool = pool

    async def get_vehicle_metrics_summary(
        self,
        vehicle_id: str,
        from_dt: datetime,
        to_dt: datetime,
    ) -> VehicleMetricsSummary:
        async with self.pool.acquire() as conn:
            row = await conn.fetchrow(
                """
                SELECT
                    vehicle_id,
                    COUNT(*)                        AS total_readings,
                    AVG(speed_kmh)                  AS avg_speed_kmh,
                    MAX(speed_kmh)                  AS max_speed_kmh,
                    AVG(battery_pct)                AS avg_battery_pct,
                    MIN(battery_pct)                AS min_battery_pct,
                    AVG(engine_temp_c)              AS avg_engine_temp_c,
                    MAX(engine_temp_c)              AS max_engine_temp_c,
                    SUM(distance_m) / 1000.0        AS total_distance_km,
                    COUNT(*) FILTER (
                        WHERE status = 'ALERT'
                    )                               AS alert_count
                FROM vehicle_telemetry
                WHERE vehicle_id = $1
                  AND time BETWEEN $2 AND $3
                GROUP BY vehicle_id
                """,
                vehicle_id, from_dt, to_dt,
            )

        return VehicleMetricsSummary(**dict(row))

    async def get_speed_time_series(
        self,
        vehicle_id: str,
        from_dt: datetime,
        to_dt: datetime,
        bucket_minutes: int = 5,
    ) -> list[TimeSeriesPoint]:
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT
                    time_bucket($1::interval, time) AS bucket,
                    AVG(speed_kmh)                  AS avg_value,
                    MAX(speed_kmh)                  AS max_value,
                    MIN(speed_kmh)                  AS min_value
                FROM vehicle_telemetry
                WHERE vehicle_id = $2
                  AND time BETWEEN $3 AND $4
                GROUP BY bucket
                ORDER BY bucket ASC
                """,
                f"{bucket_minutes} minutes", vehicle_id, from_dt, to_dt,
            )

        return [TimeSeriesPoint(**dict(r)) for r in rows]

    async def get_fleet_performance_report(
        self,
        fleet_id: str,
        from_dt: datetime,
        to_dt: datetime,
    ) -> FleetPerformanceReport:
        async with self.pool.acquire() as conn:
            rows = await conn.fetch(
                """
                SELECT
                    vehicle_id,
                    AVG(speed_kmh)           AS avg_speed,
                    AVG(battery_pct)         AS avg_battery,
                    SUM(distance_m) / 1000.0 AS distance_km,
                    COUNT(*) FILTER (
                        WHERE status = 'ALERT'
                    )                        AS alerts
                FROM vehicle_telemetry
                WHERE fleet_id = $1
                  AND time BETWEEN $2 AND $3
                GROUP BY vehicle_id
                ORDER BY distance_km DESC
                """,
                fleet_id, from_dt, to_dt,
            )

        return FleetPerformanceReport(
            fleet_id=fleet_id,
            from_dt=from_dt,
            to_dt=to_dt,
            vehicles=[dict(r) for r in rows],
        )
```

**Features:**
- ⏱️ TimescaleDB `time_bucket()` for automatic interval-based aggregations
- 📊 Per-vehicle speed, battery and temperature time-series at configurable granularity
- 🚛 Fleet-wide performance reports with distance, efficiency and alert rankings
- 🔍 Hypertable chunk compression for 90-day telemetry retention at minimal cost
- ⚡ asyncpg connection pool for non-blocking async database queries throughout

---

## 🛠️ Technology Stack

### Backend

| Technology       | Purpose                                        | Version  |
|------------------|------------------------------------------------|----------|
| **Python**       | Primary backend language                       | 3.11+    |
| **FastAPI**      | Async REST API and WebSocket framework         | 0.111.x  |
| **aiokafka**     | Async Kafka producer and consumer              | 0.10.x   |
| **Apache Kafka** | Event-driven architecture message broker       | 3.7.x    |
| **TimescaleDB**  | Time-series hypertable for sensor telemetry   | 2.14.x   |
| **PostgreSQL**   | Relational data (fleet, vehicles, operators)   | 15+      |
| **asyncpg**      | Async PostgreSQL driver (TimescaleDB)          | 0.29.x   |
| **SQLAlchemy**   | ORM for relational PostgreSQL entities         | 2.x      |
| **Redis**        | Caching, pub/sub and rate limiting             | 7.x      |
| **Pydantic v2**  | Data validation and settings management        | 2.x      |
| **Passlib + JWT**| Authentication and token management            | latest   |
| **Pytest**       | TDD — unit and integration test suite          | 8.x      |
| **Pytest-asyncio**| Async test support for FastAPI services       | 0.23.x   |
| **httpx**        | Async HTTP client for API integration tests    | 0.27.x   |

### Frontend

| Technology              | Purpose                                 | Version  |
|-------------------------|-----------------------------------------|----------|
| **React**               | UI framework                            | 18.x     |
| **TypeScript**          | Type safety                             | 5.x      |
| **React-Redux**         | Global state management                 | 8.x      |
| **Redux Toolkit**       | Simplified Redux with slices and Thunks | 1.9.x    |
| **Redux Thunk**         | Async middleware for API calls          | 2.4.x    |
| **Mapbox GL JS**        | Interactive geospatial fleet map        | 3.x      |
| **React Router v6**     | Client-side routing                     | 6.x      |
| **Webpack**             | Manual module bundling configuration    | 5.x      |
| **SASS/SCSS**           | Advanced CSS preprocessing              | 1.x      |
| **Axios**               | HTTP client with interceptors           | 1.x      |
| **Chart.js**            | Time-series telemetry charts            | 4.x      |
| **React Testing Library** | Component unit testing (TDD)          | 14.x     |
| **Cypress**             | End-to-end BDD testing                  | 13.x     |

### DevOps & Infrastructure

| Technology          | Purpose                                      |
|---------------------|----------------------------------------------|
| **Docker**          | Service containerization                     |
| **Docker Compose**  | Local multi-service orchestration            |
| **AWS ECS Fargate** | Serverless container hosting                 |
| **AWS MSK**         | Managed Apache Kafka on AWS                  |
| **AWS RDS**         | Managed PostgreSQL + TimescaleDB             |
| **AWS ElastiCache** | Managed Redis                                |
| **GitHub Actions**  | CI/CD pipeline — test, build, deploy         |
| **Prometheus**      | Metrics collection from FastAPI services     |
| **Grafana**         | Operational dashboards and alerting          |

---

## 🏗️ System Architecture

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  Fleet Operator  │  │  Fleet Manager   │  │   Admin Panel    │ │
│  │    Dashboard     │  │    Dashboard     │  │                  │ │
│  │  (React + TS)    │  │  (React + TS)    │  │  (React + TS)    │ │
│  │  Mapbox Live Map │  │  Analytics View  │  │  System Config   │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                     │                      │            │
│           └─────────────────────┴──────────────────────┘           │
│                                 │                                   │
│              ┌──────────────────▼──────────────────┐               │
│              │      Redux Store + Thunks            │               │
│              │  fleet · telemetry · alerts · auth   │               │
│              └──────────────────┬──────────────────┘               │
└─────────────────────────────────┼───────────────────────────────────┘
                    REST API       │       WebSocket
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                      APPLICATION LAYER                              │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │              FastAPI API Gateway (Python 3.11)               │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │   Auth    │ │  Fleet    │ │ Telemetry │ │  Commands  │  │   │
│  │  │  Router   │ │  Router   │ │  Router   │ │   Router   │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │Analytics  │ │  Alerts   │ │WebSocket  │ │  Missions  │  │   │
│  │  │  Router   │ │  Router   │ │  Manager  │ │   Router   │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘  │
│                                 │                                   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                    EVENT-DRIVEN LAYER (EDA)                         │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │                  Apache Kafka Cluster                        │   │
│  │                                                              │   │
│  │  Topics:                                                     │   │
│  │  ├── fleet.telemetry.raw          (vehicle sensor streams)   │   │
│  │  ├── fleet.telemetry.processed    (enriched events)          │   │
│  │  ├── fleet.commands               (STOP·RESUME·REROUTE)      │   │
│  │  ├── fleet.alerts.triggered       (threshold violations)     │   │
│  │  └── fleet.missions.events        (mission lifecycle)        │   │
│  └──────────────────────────────────────────────────────────────┘  │
│           │                    │                    │               │
│  ┌────────▼──────┐  ┌──────────▼──────┐  ┌────────▼──────────┐   │
│  │  Telemetry    │  │  Alert          │  │  Mission          │   │
│  │  Consumer     │  │  Consumer       │  │  Consumer         │   │
│  │  (Python)     │  │  (Python)       │  │  (Python)         │   │
│  └───────────────┘  └─────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                         DATA LAYER                                  │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────┐  ┌────────────▼───┐  ┌──────────┐  ┌─────────┐  │
│  │ TimescaleDB  │  │  PostgreSQL    │  │  Redis   │  │  AWS S3 │  │
│  │              │  │                │  │          │  │         │  │
│  │ - Telemetry  │  │ - Fleets       │  │ - Cache  │  │ - Logs  │  │
│  │   hypertable │  │ - Vehicles     │  │ - Pub/Sub│  │ - Route │  │
│  │ - Speed      │  │ - Operators    │  │ - Rate   │  │   Files │  │
│  │   series     │  │ - Missions     │  │   Limit  │  │         │  │
│  │ - Battery    │  │ - Alerts       │  │ - WS     │  │         │  │
│  │   series     │  │ - Commands     │  │   State  │  │         │  │
│  └──────────────┘  └────────────────┘  └──────────┘  └─────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                    VEHICLE / IoT LAYER                              │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │              Vehicle Telemetry Simulator (Python)            │   │
│  │   Simulates GPS, speed, battery, temperature and load        │   │
│  │   sensors from a configurable fleet of autonomous vehicles   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│        │                  │                  │                       │
│  ┌─────▼──────┐  ┌────────▼──────┐  ┌───────▼───────┐            │
│  │ Vehicle A  │  │  Vehicle B    │  │  Vehicle C    │            │
│  │ GPS + IMU  │  │  GPS + IMU    │  │  GPS + IMU    │            │
│  │ (simulated)│  │  (simulated)  │  │  (simulated)  │            │
│  └────────────┘  └───────────────┘  └───────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Structure
```
vehicleops/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── auth.py
│   │   │   │   ├── fleet.py
│   │   │   │   ├── vehicles.py
│   │   │   │   ├── telemetry.py
│   │   │   │   ├── analytics.py
│   │   │   │   ├── alerts.py
│   │   │   │   ├── commands.py
│   │   │   │   └── missions.py
│   │   │   └── deps.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   ├── database.py
│   │   │   ├── security.py
│   │   │   └── logging.py
│   │   ├── events/
│   │   │   ├── producers/
│   │   │   │   ├── telemetry_producer.py
│   │   │   │   └── command_producer.py
│   │   │   └── consumers/
│   │   │       ├── telemetry_consumer.py
│   │   │       ├── alert_consumer.py
│   │   │       └── mission_consumer.py
│   │   ├── models/
│   │   │   ├── fleet.py
│   │   │   ├── vehicle.py
│   │   │   ├── mission.py
│   │   │   ├── alert.py
│   │   │   └── operator.py
│   │   ├── schemas/
│   │   │   ├── telemetry.py
│   │   │   ├── fleet.py
│   │   │   ├── analytics.py
│   │   │   └── commands.py
│   │   ├── services/
│   │   │   ├── telemetry_service.py
│   │   │   ├── analytics_service.py
│   │   │   ├── alert_service.py
│   │   │   ├── fleet_service.py
│   │   │   └── mission_service.py
│   │   ├── websocket/
│   │   │   └── manager.py
│   │   └── main.py
│   ├── simulator/
│   │   └── vehicle_simulator.py
│   └── tests/
│       ├── unit/
│       └── integration/
└── frontend/
    ├── src/
    │   ├── components/
    │   │   ├── FleetMap/
    │   │   ├── TelemetryChart/
    │   │   ├── AlertPanel/
    │   │   └── VehicleDetail/
    │   ├── store/
    │   │   ├── slices/
    │   │   │   ├── fleetSlice.ts
    │   │   │   ├── telemetrySlice.ts
    │   │   │   └── alertSlice.ts
    │   │   └── index.ts
    │   ├── services/
    │   ├── types/
    │   └── styles/
    ├── webpack.config.js
    └── cypress/
```

### Data Flow
```
1. Vehicle simulator publishes telemetry to Kafka
   └──> Topic: fleet.telemetry.raw
        └──> TelemetryConsumer receives event
             └──> TelemetryService persists to TimescaleDB hypertable
                  └──> AlertService evaluates thresholds
                       ├──> If alert: publishes to fleet.alerts.triggered
                       │    └──> AlertConsumer notifies operators via WebSocket
                       └──> WebSocketManager broadcasts to all fleet subscribers
                            └──> Mapbox markers update in real-time on dashboard

2. Operator issues vehicle command from dashboard
   └──> POST /api/v1/commands/{vehicle_id}
        └──> CommandProducer publishes to fleet.commands
             └──> Vehicle simulator consumes command
                  └──> Executes: STOP · RESUME · REROUTE · RETURN_TO_BASE
                       └──> Status update published back to fleet.telemetry.raw
```

### Kafka Topics
```
fleet.telemetry.raw
├── Key:       vehicle_id
├── Partitions: 12 (one per vehicle group)
├── Retention:  24 hours
└── Consumers:  telemetry-service, alert-service

fleet.telemetry.processed
├── Key:       vehicle_id
├── Partitions: 12
├── Retention:  6 hours
└── Consumers:  analytics-service, websocket-broadcaster

fleet.commands
├── Key:       vehicle_id
├── Partitions: 12
├── Retention:  1 hour
└── Consumers:  vehicle-simulator, command-logger

fleet.alerts.triggered
├── Key:       fleet_id
├── Partitions: 4
├── Retention:  48 hours
└── Consumers:  notification-service, alert-recorder

fleet.missions.events
├── Key:       mission_id
├── Partitions: 4
├── Retention:  7 days
└── Consumers:  mission-service, audit-logger
```

---

## 💾 Installation

### Prerequisites
```bash
# Required software
- Python 3.11 or higher
- Node.js 20 LTS or higher
- Docker and Docker Compose
- Mapbox account and access token (free tier available)
- AWS account (optional — for cloud deployment)
```

### Option 1: Docker Installation (Recommended)
```bash
# 1. Clone the repository
git clone https://github.com/paulabadt/vehicleops.git
cd vehicleops

# 2. Copy environment files
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local

# 3. Add your Mapbox token to frontend/.env.local
echo "REACT_APP_MAPBOX_TOKEN=your_mapbox_token_here" >> frontend/.env.local

# 4. Start all services
docker-compose up -d

# 5. Run database migrations
docker-compose exec backend alembic upgrade head

# 6. Seed fleet demo data
docker-compose exec backend python -m app.scripts.seed_demo

# 7. Start vehicle simulator (separate terminal)
docker-compose exec backend python -m simulator.vehicle_simulator \
  --fleet-id fleet-demo-001 \
  --vehicles 20 \
  --interval-ms 500

# 8. Check all services are healthy
docker-compose ps

# 9. Access the platform
# Frontend:   http://localhost:3000
# API:        http://localhost:8000
# API Docs:   http://localhost:8000/docs
# Kafka UI:   http://localhost:8080
# Grafana:    http://localhost:3001
```

### Option 2: Manual Installation

#### Backend Setup (Python + FastAPI)
```bash
# 1. Navigate to backend directory
cd backend

# 2. Create and activate virtual environment
python -m venv venv
source venv/bin/activate        # Linux/macOS
# venv\Scripts\activate         # Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env
# Edit .env with your database and Kafka connection strings

# 5. Start PostgreSQL and TimescaleDB (via Docker)
docker-compose up -d postgres timescaledb redis kafka zookeeper

# 6. Run Alembic database migrations
alembic upgrade head

# 7. Seed demo fleet data
python -m app.scripts.seed_demo

# 8. Start FastAPI development server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 9. Start Kafka consumers (separate terminal)
python -m app.events.consumers.telemetry_consumer
```

#### Frontend Setup (React + TypeScript)
```bash
# 1. Navigate to frontend directory
cd frontend

# 2. Install dependencies
npm install

# 3. Configure environment variables
cp .env.example .env.local
# Edit .env.local — set API URL and Mapbox token

# 4. Start development server
npm run dev

# 5. Build for production
npm run build
```

### Environment Variables
```bash
# backend/.env

# Application
APP_ENV=development
APP_HOST=0.0.0.0
APP_PORT=8000
SECRET_KEY=your_super_secret_key_minimum_32_characters
ACCESS_TOKEN_EXPIRE_MINUTES=60

# PostgreSQL — Relational data
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=vehicleops_user
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=vehicleops_db

# TimescaleDB — Time-series telemetry
TIMESCALE_HOST=localhost
TIMESCALE_PORT=5433
TIMESCALE_USER=timescale_user
TIMESCALE_PASSWORD=your_timescale_password
TIMESCALE_DB=vehicleops_timeseries

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password

# Apache Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_TELEMETRY_TOPIC=fleet.telemetry.raw
KAFKA_COMMANDS_TOPIC=fleet.commands
KAFKA_ALERTS_TOPIC=fleet.alerts.triggered
KAFKA_MISSIONS_TOPIC=fleet.missions.events
KAFKA_CONSUMER_GROUP=vehicleops-backend

# AWS (production)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
AWS_S3_BUCKET=vehicleops-logs

# Frontend
FRONTEND_URL=http://localhost:3000
```
```bash
# frontend/.env.local

REACT_APP_API_URL=http://localhost:8000/api/v1
REACT_APP_WS_URL=ws://localhost:8000
REACT_APP_MAPBOX_TOKEN=your_mapbox_public_token
REACT_APP_APP_NAME=VehicleOps
```

### Docker Compose Services
```yaml
# docker-compose.yml — service overview
services:
  backend:       # FastAPI app — port 8000
  frontend:      # React app  — port 3000
  postgres:      # PostgreSQL — port 5432
  timescaledb:   # TimescaleDB — port 5433
  redis:         # Redis       — port 6379
  zookeeper:     # Kafka dependency
  kafka:         # Apache Kafka — port 9092
  kafka-ui:      # Kafka UI dashboard — port 8080
  prometheus:    # Metrics collection — port 9090
  grafana:       # Operational dashboards — port 3001
```

---

## 🚀 Usage

### Starting the Platform
```bash
# Start all infrastructure services
docker-compose up -d

# Start FastAPI backend
cd backend
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Start Kafka consumers (separate terminal)
python -m app.events.consumers.telemetry_consumer &
python -m app.events.consumers.alert_consumer &
python -m app.events.consumers.mission_consumer &

# Start React frontend (separate terminal)
cd frontend
npm run dev

# Start vehicle simulator with 20 vehicles (separate terminal)
cd backend
python -m simulator.vehicle_simulator \
  --fleet-id fleet-demo-001 \
  --vehicles 20 \
  --interval-ms 500
```

### Default Credentials
```bash
# System Administrator
Email:    admin@vehicleops.io
Password: Admin123! (change immediately)

# Fleet Manager
Email:    manager@vehicleops.io
Password: Manager123!

# Fleet Operator
Email:    operator@vehicleops.io
Password: Operator123!
```

### Vehicle Simulator Options
```bash
# Run simulator with custom parameters
python -m simulator.vehicle_simulator \
  --fleet-id     fleet-001     \  # Target fleet identifier
  --vehicles     50            \  # Number of vehicles to simulate
  --interval-ms  500           \  # Telemetry publish interval
  --area         bogota        \  # Geofenced area (bogota|medellin|cali)
  --scenario     mixed            # Scenario: normal|high-load|fault-injection

# Available scenarios
# normal          — standard operations, random routes
# high-load       — maximum throughput stress test
# fault-injection — simulates battery failures and speed violations
```

### Available Scripts
```bash
# Backend
uvicorn app.main:app --reload          # Development server with hot reload
uvicorn app.main:app --workers 4       # Production multi-worker server
alembic upgrade head                   # Run pending database migrations
alembic revision --autogenerate -m ""  # Generate new migration
python -m app.scripts.seed_demo        # Seed demo fleet and operator data
pytest                                 # Run full test suite
pytest --cov=app --cov-report=html     # Run tests with HTML coverage report
pytest -m unit                         # Run unit tests only
pytest -m integration                  # Run integration tests only

# Frontend
npm run dev                            # Development server with HMR
npm run build                          # Production Webpack bundle
npm run preview                        # Preview production build
npm run test                           # React Testing Library unit tests
npm run test:coverage                  # Frontend coverage report
npm run cypress:open                   # Cypress interactive BDD runner
npm run cypress:run                    # Full Cypress E2E suite headless
npm run lint                           # ESLint + TypeScript checks
```

---

## 💻 Code Examples

### 1. FastAPI WebSocket Manager — Live Fleet Broadcasting
```python
# app/websocket/manager.py
# FastAPI — WebSocket connection manager for real-time fleet updates

import asyncio
import json
import logging
from collections import defaultdict
from fastapi import WebSocket

logger = logging.getLogger(__name__)


class WebSocketManager:
    def __init__(self):
        # fleet_id → set of active WebSocket connections
        self._connections: dict[str, set[WebSocket]] = defaultdict(set)

    async def connect(self, websocket: WebSocket, fleet_id: str) -> None:
        await websocket.accept()
        self._connections[fleet_id].add(websocket)
        logger.info(
            "WebSocket connected — fleet=%s total_connections=%d",
            fleet_id,
            len(self._connections[fleet_id]),
        )

    def disconnect(self, websocket: WebSocket, fleet_id: str) -> None:
        self._connections[fleet_id].discard(websocket)
        logger.info(
            "WebSocket disconnected — fleet=%s remaining=%d",
            fleet_id,
            len(self._connections[fleet_id]),
        )

    async def broadcast_vehicle_update(
        self, fleet_id: str, data: dict
    ) -> None:
        subscribers = self._connections.get(fleet_id, set())
        if not subscribers:
            return

        message = json.dumps(data)
        dead_connections: set[WebSocket] = set()

        results = await asyncio.gather(
            *[ws.send_text(message) for ws in subscribers],
            return_exceptions=True,
        )

        for ws, result in zip(subscribers, results):
            if isinstance(result, Exception):
                logger.warning("Dead WebSocket detected — removing from pool")
                dead_connections.add(ws)

        self._connections[fleet_id] -= dead_connections

    async def broadcast_alert(
        self, fleet_id: str, alert: dict
    ) -> None:
        await self.broadcast_vehicle_update(
            fleet_id,
            {"type": "ALERT", "payload": alert},
        )
```
```python
# app/api/v1/telemetry.py
# FastAPI — WebSocket endpoint for live fleet telemetry subscription

from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
from app.websocket.manager import WebSocketManager
from app.core.deps import get_ws_manager, verify_ws_token

router = APIRouter()


@router.websocket("/ws/fleet/{fleet_id}")
async def fleet_telemetry_ws(
    websocket: WebSocket,
    fleet_id: str,
    manager: WebSocketManager = Depends(get_ws_manager),
):
    await manager.connect(websocket, fleet_id)
    try:
        while True:
            # Keep connection alive — client sends ping every 30s
            data = await websocket.receive_text()
            if data == "ping":
                await websocket.send_text("pong")
    except WebSocketDisconnect:
        manager.disconnect(websocket, fleet_id)
```

---

### 2. FastAPI REST Endpoints — Fleet and Commands
```python
# app/api/v1/commands.py
# FastAPI — Vehicle command dispatch with Kafka producer

from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas.commands import CommandRequest, CommandResponse
from app.events.producers.telemetry_producer import TelemetryProducer
from app.services.fleet_service import FleetService
from app.core.deps import get_producer, get_fleet_service, get_current_operator
from app.models.operator import Operator

router = APIRouter(prefix="/commands", tags=["commands"])


@router.post(
    "/{vehicle_id}",
    response_model=CommandResponse,
    status_code=status.HTTP_202_ACCEPTED,
)
async def dispatch_command(
    vehicle_id: str,
    request: CommandRequest,
    producer: TelemetryProducer = Depends(get_producer),
    fleet_service: FleetService = Depends(get_fleet_service),
    current_operator: Operator = Depends(get_current_operator),
) -> CommandResponse:
    # Verify vehicle belongs to operator's fleet
    vehicle = await fleet_service.get_vehicle(vehicle_id)
    if not vehicle:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Vehicle {vehicle_id} not found",
        )

    if vehicle.fleet_id not in current_operator.fleet_ids:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Operator does not have access to this vehicle's fleet",
        )

    # Publish command event to Kafka
    await producer.publish_command(
        vehicle_id=vehicle_id,
        command=request.command,     # STOP | RESUME | REROUTE | RETURN_TO_BASE
        params=request.params or {},
    )

    return CommandResponse(
        vehicle_id=vehicle_id,
        command=request.command,
        status="DISPATCHED",
        issued_by=current_operator.id,
    )
```
```python
# app/api/v1/analytics.py
# FastAPI — Analytics endpoints for time-series telemetry queries

from datetime import datetime
from fastapi import APIRouter, Depends, Query
from app.services.analytics_service import AnalyticsService
from app.schemas.analytics import (
    VehicleMetricsSummary,
    FleetPerformanceReport,
    TimeSeriesPoint,
)
from app.core.deps import get_analytics_service, get_current_operator

router = APIRouter(prefix="/analytics", tags=["analytics"])


@router.get(
    "/vehicle/{vehicle_id}/metrics",
    response_model=VehicleMetricsSummary,
)
async def get_vehicle_metrics(
    vehicle_id: str,
    from_dt: datetime = Query(..., alias="from"),
    to_dt: datetime = Query(..., alias="to"),
    service: AnalyticsService = Depends(get_analytics_service),
    _: object = Depends(get_current_operator),
) -> VehicleMetricsSummary:
    return await service.get_vehicle_metrics_summary(
        vehicle_id, from_dt, to_dt
    )


@router.get(
    "/vehicle/{vehicle_id}/speed-series",
    response_model=list[TimeSeriesPoint],
)
async def get_speed_time_series(
    vehicle_id: str,
    from_dt: datetime = Query(..., alias="from"),
    to_dt: datetime = Query(..., alias="to"),
    bucket_minutes: int = Query(default=5, ge=1, le=60),
    service: AnalyticsService = Depends(get_analytics_service),
) -> list[TimeSeriesPoint]:
    return await service.get_speed_time_series(
        vehicle_id, from_dt, to_dt, bucket_minutes
    )


@router.get(
    "/fleet/{fleet_id}/performance",
    response_model=FleetPerformanceReport,
)
async def get_fleet_performance(
    fleet_id: str,
    from_dt: datetime = Query(..., alias="from"),
    to_dt: datetime = Query(..., alias="to"),
    service: AnalyticsService = Depends(get_analytics_service),
) -> FleetPerformanceReport:
    return await service.get_fleet_performance_report(
        fleet_id, from_dt, to_dt
    )
```

---

### 3. Vehicle Simulator — Async Telemetry Generator
```python
# simulator/vehicle_simulator.py
# Python — Async vehicle telemetry simulator publishing to Kafka

import asyncio
import argparse
import random
import logging
from datetime import datetime, timezone
from dataclasses import dataclass
from aiokafka import AIOKafkaProducer
import json

logger = logging.getLogger(__name__)

# Geofenced simulation areas
AREAS = {
    "bogota":   {"lat": 4.7110,  "lng": -74.0721, "radius_km": 15},
    "medellin": {"lat": 6.2442,  "lng": -75.5812, "radius_km": 12},
    "cali":     {"lat": 3.4516,  "lng": -76.5320, "radius_km": 10},
}


@dataclass
class SimulatedVehicle:
    vehicle_id: str
    fleet_id: str
    latitude: float
    longitude: float
    speed_kmh: float = 0.0
    battery_pct: float = 100.0
    engine_temp_c: float = 65.0
    load_pct: float = 0.0
    status: str = "IDLE"

    def tick(self, scenario: str) -> None:
        """Advance vehicle state by one simulation step."""
        # Move vehicle along random heading
        self.latitude  += random.uniform(-0.0003, 0.0003)
        self.longitude += random.uniform(-0.0003, 0.0003)

        # Update speed
        if scenario == "fault-injection" and random.random() < 0.05:
            self.speed_kmh = random.uniform(90, 120)   # Overspeed fault
        else:
            self.speed_kmh = random.uniform(5, 60)

        # Drain battery gradually
        self.battery_pct = max(0.0, self.battery_pct - random.uniform(0, 0.05))

        # Engine temperature fluctuation
        self.engine_temp_c = random.uniform(60, 105)

        # Load percentage
        self.load_pct = random.uniform(20, 95)

        # Derive status
        if self.speed_kmh > 80:
            self.status = "ALERT"
        elif self.battery_pct < 15:
            self.status = "LOW_BATTERY"
        elif self.speed_kmh > 0:
            self.status = "MOVING"
        else:
            self.status = "IDLE"

    def to_payload(self) -> dict:
        return {
            "vehicle_id":    self.vehicle_id,
            "fleet_id":      self.fleet_id,
            "latitude":      round(self.latitude, 6),
            "longitude":     round(self.longitude, 6),
            "speed_kmh":     round(self.speed_kmh, 2),
            "battery_pct":   round(self.battery_pct, 1),
            "engine_temp_c": round(self.engine_temp_c, 1),
            "load_pct":      round(self.load_pct, 1),
            "status":        self.status,
            "timestamp":     datetime.now(timezone.utc).isoformat(),
        }


async def simulate_vehicle(
    vehicle: SimulatedVehicle,
    producer: AIOKafkaProducer,
    interval_ms: int,
    scenario: str,
) -> None:
    while True:
        vehicle.tick(scenario)
        payload = vehicle.to_payload()

        await producer.send(
            topic="fleet.telemetry.raw",
            key=vehicle.vehicle_id.encode(),
            value=json.dumps(payload).encode(),
        )

        logger.debug(
            "Published: vehicle=%s speed=%.1f battery=%.1f%%",
            vehicle.vehicle_id,
            vehicle.speed_kmh,
            vehicle.battery_pct,
        )

        await asyncio.sleep(interval_ms / 1000)


async def run_simulator(
    fleet_id: str,
    num_vehicles: int,
    interval_ms: int,
    area: str,
    scenario: str,
) -> None:
    area_cfg = AREAS.get(area, AREAS["bogota"])

    producer = AIOKafkaProducer(
        bootstrap_servers="localhost:9092",
        value_serializer=lambda v: v,
        acks="all",
    )
    await producer.start()
    logger.info(
        "Simulator started — fleet=%s vehicles=%d area=%s scenario=%s",
        fleet_id, num_vehicles, area, scenario,
    )

    vehicles = [
        SimulatedVehicle(
            vehicle_id=f"{fleet_id}-VH-{i:03d}",
            fleet_id=fleet_id,
            latitude=area_cfg["lat"]  + random.uniform(-0.05, 0.05),
            longitude=area_cfg["lng"] + random.uniform(-0.05, 0.05),
        )
        for i in range(1, num_vehicles + 1)
    ]

    try:
        await asyncio.gather(
            *[
                simulate_vehicle(v, producer, interval_ms, scenario)
                for v in vehicles
            ]
        )
    finally:
        await producer.stop()


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="VehicleOps Fleet Simulator")
    parser.add_argument("--fleet-id",    default="fleet-demo-001")
    parser.add_argument("--vehicles",    type=int, default=20)
    parser.add_argument("--interval-ms", type=int, default=500)
    parser.add_argument("--area",        default="bogota",
                        choices=["bogota", "medellin", "cali"])
    parser.add_argument("--scenario",    default="normal",
                        choices=["normal", "high-load", "fault-injection"])
    args = parser.parse_args()

    logging.basicConfig(level=logging.INFO)
    asyncio.run(run_simulator(
        args.fleet_id, args.vehicles,
        args.interval_ms, args.area, args.scenario,
    ))
```

---

### 4. Redux Thunks — Fleet State Management
```typescript
// store/slices/fleetSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { fleetService } from '../../services/fleetService';
import { VehicleTelemetry, FleetSummary } from '../../types/fleet';

interface FleetState {
  fleetId: string | null;
  vehicles: Record<string, VehicleTelemetry>;
  selectedVehicleId: string | null;
  summary: FleetSummary | null;
  alerts: Alert[];
  loading: boolean;
  error: string | null;
}

const initialState: FleetState = {
  fleetId: null,
  vehicles: {},
  selectedVehicleId: null,
  summary: null,
  alerts: [],
  loading: false,
  error: null,
};

export const fetchFleetSummary = createAsyncThunk(
  'fleet/fetchSummary',
  async (fleetId: string, { rejectWithValue }) => {
    try {
      const response = await fleetService.getFleetSummary(fleetId);
      return response.data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Failed to fetch fleet summary'
      );
    }
  }
);

export const dispatchVehicleCommand = createAsyncThunk(
  'fleet/dispatchCommand',
  async (
    payload: { vehicleId: string; command: string; params?: object },
    { rejectWithValue }
  ) => {
    try {
      const response = await fleetService.sendCommand(payload);
      return response.data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.detail || 'Failed to dispatch command'
      );
    }
  }
);

const fleetSlice = createSlice({
  name: 'fleet',
  initialState,
  reducers: {
    setFleetId: (state, action: PayloadAction<string>) => {
      state.fleetId = action.payload;
    },
    // Called on every WebSocket telemetry message
    updateVehiclePosition: (
      state,
      action: PayloadAction<VehicleTelemetry>
    ) => {
      const { vehicle_id } = action.payload;
      state.vehicles[vehicle_id] = action.payload;
    },
    // Called when WebSocket pushes an alert event
    addAlert: (state, action: PayloadAction<Alert>) => {
      state.alerts.unshift(action.payload);
      if (state.alerts.length > 100) state.alerts.pop();
    },
    selectVehicle: (state, action: PayloadAction<string>) => {
      state.selectedVehicleId = action.payload;
    },
    clearAlerts: (state) => {
      state.alerts = [];
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchFleetSummary.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchFleetSummary.fulfilled, (state, action) => {
        state.loading = false;
        state.summary = action.payload;
      })
      .addCase(fetchFleetSummary.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      })
      .addCase(dispatchVehicleCommand.fulfilled, (_state, action) => {
        console.info('Command dispatched:', action.payload);
      });
  },
});

export const {
  setFleetId,
  updateVehiclePosition,
  addAlert,
  selectVehicle,
  clearAlerts,
} = fleetSlice.actions;

export default fleetSlice.reducer;
```

---

### 5. Backend Testing — Pytest + pytest-asyncio (TDD)
```python
# tests/unit/test_alert_service.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from app.services.alert_service import AlertService
from app.schemas.telemetry import TelemetryEvent


@pytest.fixture
def alert_service():
    mock_repo = AsyncMock()
    mock_producer = AsyncMock()
    return AlertService(
        alert_repository=mock_repo,
        kafka_producer=mock_producer,
    )


@pytest.mark.asyncio
async def test_evaluate_triggers_overspeed_alert(alert_service):
    # Given
    payload = {
        "vehicle_id": "fleet-001-VH-001",
        "fleet_id":   "fleet-001",
        "speed_kmh":  92.5,
        "battery_pct": 80.0,
        "engine_temp_c": 70.0,
        "status": "ALERT",
        "timestamp": "2024-01-01T10:00:00Z",
    }

    # When
    alerts = await alert_service.evaluate(payload)

    # Then
    assert any(a["type"] == "OVERSPEED" for a in alerts)
    alert_service.kafka_producer.publish_command.assert_not_called()


@pytest.mark.asyncio
async def test_evaluate_triggers_low_battery_alert(alert_service):
    # Given
    payload = {
        "vehicle_id":    "fleet-001-VH-002",
        "fleet_id":      "fleet-001",
        "speed_kmh":     30.0,
        "battery_pct":   12.0,          # Below 15% threshold
        "engine_temp_c": 70.0,
        "status":        "LOW_BATTERY",
        "timestamp":     "2024-01-01T10:01:00Z",
    }

    # When
    alerts = await alert_service.evaluate(payload)

    # Then
    assert any(a["type"] == "LOW_BATTERY" for a in alerts)
    battery_alert = next(
        a for a in alerts if a["type"] == "LOW_BATTERY"
    )
    assert battery_alert["value"] == 12.0


@pytest.mark.asyncio
async def test_evaluate_triggers_engine_overheat_alert(alert_service):
    # Given
    payload = {
        "vehicle_id":    "fleet-001-VH-003",
        "fleet_id":      "fleet-001",
        "speed_kmh":     45.0,
        "battery_pct":   70.0,
        "engine_temp_c": 108.0,         # Above 100°C threshold
        "status":        "ALERT",
        "timestamp":     "2024-01-01T10:02:00Z",
    }

    # When
    alerts = await alert_service.evaluate(payload)

    # Then
    assert any(a["type"] == "ENGINE_OVERHEAT" for a in alerts)


@pytest.mark.asyncio
async def test_evaluate_returns_no_alerts_for_normal_telemetry(alert_service):
    # Given
    payload = {
        "vehicle_id":    "fleet-001-VH-004",
        "fleet_id":      "fleet-001",
        "speed_kmh":     42.0,
        "battery_pct":   85.0,
        "engine_temp_c": 72.0,
        "status":        "MOVING",
        "timestamp":     "2024-01-01T10:03:00Z",
    }

    # When
    alerts = await alert_service.evaluate(payload)

    # Then
    assert alerts == []
```
```python
# tests/integration/test_telemetry_api.py
import pytest
from httpx import AsyncClient
from app.main import app


@pytest.mark.asyncio
async def test_get_vehicle_metrics_returns_summary():
    # Given
    async with AsyncClient(app=app, base_url="http://test") as client:
        login = await client.post("/api/v1/auth/login", json={
            "email": "operator@vehicleops.io",
            "password": "Operator123!",
        })
        token = login.json()["access_token"]

        # When
        response = await client.get(
            "/api/v1/analytics/vehicle/fleet-001-VH-001/metrics",
            params={
                "from": "2024-01-01T00:00:00Z",
                "to":   "2024-01-02T00:00:00Z",
            },
            headers={"Authorization": f"Bearer {token}"},
        )

    # Then
    assert response.status_code == 200
    data = response.json()
    assert "avg_speed_kmh" in data
    assert "avg_battery_pct" in data
    assert "total_distance_km" in data
    assert "alert_count" in data


@pytest.mark.asyncio
async def test_dispatch_command_returns_202():
    async with AsyncClient(app=app, base_url="http://test") as client:
        login = await client.post("/api/v1/auth/login", json={
            "email": "operator@vehicleops.io",
            "password": "Operator123!",
        })
        token = login.json()["access_token"]

        # When
        response = await client.post(
            "/api/v1/commands/fleet-001-VH-001",
            json={"command": "STOP", "params": {}},
            headers={"Authorization": f"Bearer {token}"},
        )

    # Then
    assert response.status_code == 202
    assert response.json()["status"] == "DISPATCHED"
```
```typescript
// cypress/e2e/fleet_map.cy.ts — BDD E2E
describe('Fleet Map — BDD', () => {
  beforeEach(() => {
    cy.login('operator@vehicleops.io', 'Operator123!');
    cy.visit('/dashboard/fleet/fleet-demo-001');
  });

  it('Given an operator, When dashboard loads, Then Mapbox map renders with active vehicles', () => {
    cy.intercept('GET', '/api/v1/fleet/*/summary', {
      fixture: 'fleet-summary.json',
    }).as('fleetSummary');

    cy.wait('@fleetSummary');

    cy.get('[data-testid="fleet-map"]').should('be.visible');
    cy.get('[data-testid="vehicle-count"]')
      .should('contain.text', 'vehicles active');
  });

  it('Given an operator, When a vehicle is selected, Then detail panel shows live telemetry', () => {
    cy.get('[data-testid="fleet-map"]').should('be.visible');
    cy.get('[data-testid="vehicle-list-item"]').first().click();

    cy.get('[data-testid="vehicle-detail-panel"]').should('be.visible');
    cy.get('[data-testid="speed-value"]').should('exist');
    cy.get('[data-testid="battery-value"]').should('exist');
  });

  it('Given an operator, When STOP command is dispatched, Then status updates to STOPPED', () => {
    cy.intercept('POST', '/api/v1/commands/*', {
      statusCode: 202,
      body: { status: 'DISPATCHED', command: 'STOP' },
    }).as('stopCommand');

    cy.get('[data-testid="vehicle-list-item"]').first().click();
    cy.get('[data-testid="cmd-stop-btn"]').click();
    cy.get('[data-testid="confirm-dialog-btn"]').click();

    cy.wait('@stopCommand');
    cy.get('[data-testid="command-feedback"]')
      .should('contain.text', 'DISPATCHED');
  });
});
```

---

## 📚 API Documentation

### Base URL
```
Development:  http://localhost:8000/api/v1
Production:   https://api.vehicleops.io/api/v1
Swagger UI:   http://localhost:8000/docs
ReDoc:        http://localhost:8000/redoc
```

### Authentication
```bash
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "operator@vehicleops.io",
  "password": "your_password"
}

# Response: 200 OK
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600,
  "operator": {
    "id": "op-001",
    "email": "operator@vehicleops.io",
    "role": "OPERATOR",
    "fleet_ids": ["fleet-001", "fleet-002"]
  }
}
```
```bash
# Using the token
GET /api/v1/fleet/fleet-001/summary
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Endpoints

#### 1. Fleet

**Get Fleet Summary**
```bash
GET /api/v1/fleet/{fleet_id}/summary
Authorization: Bearer {token}

# Response: 200 OK
{
  "fleet_id": "fleet-001",
  "fleet_name": "Industrial Zone A",
  "total_vehicles": 20,
  "active_vehicles": 18,
  "idle_vehicles": 2,
  "vehicles_in_alert": 1,
  "avg_battery_pct": 74.3,
  "avg_speed_kmh": 38.5,
  "active_missions": 3,
  "open_alerts": 2,
  "last_updated": "2024-02-15T14:32:00Z"
}
```

**List Fleet Vehicles**
```bash
GET /api/v1/fleet/{fleet_id}/vehicles?status=MOVING&page=1&limit=50
Authorization: Bearer {token}

# Query Parameters:
# - status: string (IDLE, MOVING, ALERT, LOW_BATTERY, OFFLINE)
# - page: int (default: 1)
# - limit: int (default: 50, max: 200)

# Response: 200 OK
{
  "data": [
    {
      "vehicle_id": "fleet-001-VH-001",
      "fleet_id": "fleet-001",
      "latitude": 4.7110,
      "longitude": -74.0721,
      "speed_kmh": 42.5,
      "battery_pct": 78.0,
      "engine_temp_c": 72.3,
      "load_pct": 65.0,
      "status": "MOVING",
      "active_mission_id": "mission-007",
      "last_seen": "2024-02-15T14:32:18Z"
    }
  ],
  "total": 18,
  "page": 1,
  "limit": 50
}
```

#### 2. Telemetry

**Get Latest Vehicle Telemetry**
```bash
GET /api/v1/telemetry/{vehicle_id}/latest
Authorization: Bearer {token}

# Response: 200 OK
{
  "vehicle_id": "fleet-001-VH-001",
  "latitude": 4.7110,
  "longitude": -74.0721,
  "speed_kmh": 42.5,
  "battery_pct": 78.0,
  "engine_temp_c": 72.3,
  "load_pct": 65.0,
  "status": "MOVING",
  "timestamp": "2024-02-15T14:32:18Z"
}
```

**Get Telemetry History**
```bash
GET /api/v1/telemetry/{vehicle_id}/history?from=2024-02-15T00:00:00Z&to=2024-02-15T23:59:59Z&limit=1000
Authorization: Bearer {token}

# Query Parameters:
# - from: ISO datetime (required)
# - to: ISO datetime (required)
# - limit: int (default: 1000, max: 10000)

# Response: 200 OK
{
  "vehicle_id": "fleet-001-VH-001",
  "from": "2024-02-15T00:00:00Z",
  "to": "2024-02-15T23:59:59Z",
  "total_records": 1728,
  "data": [
    {
      "latitude": 4.7110,
      "longitude": -74.0721,
      "speed_kmh": 42.5,
      "battery_pct": 78.0,
      "engine_temp_c": 72.3,
      "timestamp": "2024-02-15T14:32:18Z"
    }
  ]
}
```

**WebSocket — Live Fleet Telemetry**
```bash
WS /ws/fleet/{fleet_id}
# No Authorization header — pass token as query param
WS /ws/fleet/fleet-001?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Server pushes on every telemetry event:
{
  "vehicle_id": "fleet-001-VH-001",
  "latitude": 4.7115,
  "longitude": -74.0718,
  "speed_kmh": 44.2,
  "battery_pct": 77.8,
  "engine_temp_c": 73.1,
  "status": "MOVING",
  "alerts": [],
  "timestamp": "2024-02-15T14:32:19Z"
}

# Alert payload pushed when threshold violated:
{
  "type": "ALERT",
  "payload": {
    "vehicle_id": "fleet-001-VH-005",
    "alert_type": "OVERSPEED",
    "value": 92.5,
    "threshold": 80.0,
    "timestamp": "2024-02-15T14:33:01Z"
  }
}

# Client keepalive:
→ send:    "ping"
← receive: "pong"
```

#### 3. Commands

**Dispatch Vehicle Command**
```bash
POST /api/v1/commands/{vehicle_id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "command": "STOP",
  "params": {}
}

# Available commands:
# STOP            — immediately halt vehicle
# RESUME          — resume last active mission
# REROUTE         — assign new waypoints
# RETURN_TO_BASE  — navigate to home station

# Response: 202 Accepted
{
  "vehicle_id": "fleet-001-VH-001",
  "command": "STOP",
  "status": "DISPATCHED",
  "issued_by": "op-001",
  "issued_at": "2024-02-15T14:35:00Z"
}
```

**Get Command History**
```bash
GET /api/v1/commands/{vehicle_id}/history?limit=20
Authorization: Bearer {token}

# Response: 200 OK
{
  "data": [
    {
      "id": "cmd-001",
      "vehicle_id": "fleet-001-VH-001",
      "command": "STOP",
      "status": "EXECUTED",
      "issued_by": "op-001",
      "issued_at": "2024-02-15T14:35:00Z",
      "executed_at": "2024-02-15T14:35:02Z"
    }
  ],
  "total": 47
}
```

#### 4. Analytics

**Vehicle Metrics Summary**
```bash
GET /api/v1/analytics/vehicle/{vehicle_id}/metrics?from=2024-02-15T00:00:00Z&to=2024-02-15T23:59:59Z
Authorization: Bearer {token}

# Response: 200 OK
{
  "vehicle_id": "fleet-001-VH-001",
  "total_readings": 1728,
  "avg_speed_kmh": 38.4,
  "max_speed_kmh": 67.2,
  "avg_battery_pct": 72.1,
  "min_battery_pct": 41.3,
  "avg_engine_temp_c": 74.5,
  "max_engine_temp_c": 98.2,
  "total_distance_km": 124.7,
  "alert_count": 3
}
```

**Speed Time-Series**
```bash
GET /api/v1/analytics/vehicle/{vehicle_id}/speed-series?from=2024-02-15T08:00:00Z&to=2024-02-15T18:00:00Z&bucket_minutes=5
Authorization: Bearer {token}

# Response: 200 OK
[
  {
    "bucket": "2024-02-15T08:00:00Z",
    "avg_value": 35.2,
    "max_value": 58.1,
    "min_value": 12.4
  },
  {
    "bucket": "2024-02-15T08:05:00Z",
    "avg_value": 41.7,
    "max_value": 63.5,
    "min_value": 18.9
  }
]
```

**Fleet Performance Report**
```bash
GET /api/v1/analytics/fleet/{fleet_id}/performance?from=2024-02-01T00:00:00Z&to=2024-02-15T23:59:59Z
Authorization: Bearer {token}

# Response: 200 OK
{
  "fleet_id": "fleet-001",
  "from": "2024-02-01T00:00:00Z",
  "to": "2024-02-15T23:59:59Z",
  "vehicles": [
    {
      "vehicle_id": "fleet-001-VH-003",
      "avg_speed": 44.1,
      "avg_battery": 68.3,
      "distance_km": 1847.2,
      "alerts": 2
    },
    {
      "vehicle_id": "fleet-001-VH-001",
      "avg_speed": 38.4,
      "avg_battery": 72.1,
      "distance_km": 1623.5,
      "alerts": 3
    }
  ]
}
```

#### 5. Alerts

**List Active Alerts**
```bash
GET /api/v1/alerts?fleet_id=fleet-001&severity=HIGH&acknowledged=false
Authorization: Bearer {token}

# Query Parameters:
# - fleet_id: string (optional)
# - vehicle_id: string (optional)
# - severity: string (LOW, MEDIUM, HIGH, CRITICAL)
# - acknowledged: boolean (default: false)
# - limit: int (default: 50)

# Response: 200 OK
{
  "data": [
    {
      "id": "alert-001",
      "vehicle_id": "fleet-001-VH-005",
      "fleet_id": "fleet-001",
      "alert_type": "OVERSPEED",
      "severity": "HIGH",
      "value": 92.5,
      "threshold": 80.0,
      "message": "Vehicle fleet-001-VH-005 exceeded speed limit: 92.5 km/h",
      "acknowledged": false,
      "triggered_at": "2024-02-15T14:33:01Z"
    }
  ],
  "total": 2
}
```

**Acknowledge Alert**
```bash
POST /api/v1/alerts/{alert_id}/acknowledge
Authorization: Bearer {token}
Content-Type: application/json

{
  "notes": "Operator contacted — vehicle slowing down"
}

# Response: 200 OK
{
  "id": "alert-001",
  "acknowledged": true,
  "acknowledged_by": "op-001",
  "acknowledged_at": "2024-02-15T14:38:00Z",
  "notes": "Operator contacted — vehicle slowing down"
}
```

#### 6. Missions

**Create Mission**
```bash
POST /api/v1/missions
Authorization: Bearer {token}
Content-Type: application/json

{
  "fleet_id": "fleet-001",
  "vehicle_id": "fleet-001-VH-001",
  "mission_type": "DELIVERY",
  "waypoints": [
    { "latitude": 4.7120, "longitude": -74.0730, "order": 1 },
    { "latitude": 4.7205, "longitude": -74.0650, "order": 2 },
    { "latitude": 4.7310, "longitude": -74.0580, "order": 3 }
  ],
  "priority": "NORMAL"
}

# Response: 201 Created
{
  "id": "mission-008",
  "vehicle_id": "fleet-001-VH-001",
  "status": "ASSIGNED",
  "waypoints": 3,
  "created_at": "2024-02-15T15:00:00Z"
}
```

**Get Mission Status**
```bash
GET /api/v1/missions/{mission_id}
Authorization: Bearer {token}

# Response: 200 OK
{
  "id": "mission-008",
  "vehicle_id": "fleet-001-VH-001",
  "mission_type": "DELIVERY",
  "status": "IN_PROGRESS",
  "completed_waypoints": 1,
  "total_waypoints": 3,
  "completion_pct": 33.3,
  "started_at": "2024-02-15T15:01:00Z",
  "estimated_completion": "2024-02-15T15:45:00Z"
}
```

### Error Responses
```json
{
  "detail": "Vehicle fleet-001-VH-099 not found",
  "status_code": 404,
  "error": "NOT_FOUND"
}
```

**Common Error Codes**

| Code                  | HTTP Status | Description                              |
|-----------------------|-------------|------------------------------------------|
| `UNAUTHORIZED`        | 401         | Missing or invalid JWT token             |
| `FORBIDDEN`           | 403         | Operator lacks access to this fleet      |
| `NOT_FOUND`           | 404         | Vehicle, fleet or mission not found      |
| `CONFLICT`            | 409         | Vehicle already has an active mission    |
| `UNPROCESSABLE`       | 422         | Invalid request body or parameters       |
| `INTERNAL_ERROR`      | 500         | Unexpected server error                  |

---

## 🤝 Contributing

This project was developed as part of research at SENA. While the source code
and applications are property of SENA, contributions and suggestions are welcome.

### Development Workflow
```bash
# 1. Create a feature branch
git checkout -b feature/your-feature-name

# 2. Make your changes following FastAPI service conventions

# 3. Run the full test suite
pytest                                  # All backend tests
pytest --cov=app --cov-report=html      # With HTML coverage report
pytest -m unit                          # Unit tests only
pytest -m integration                   # Integration tests only
npx cypress run                         # Frontend E2E tests

# 4. Format and lint
black .                                 # Python formatter
isort .                                 # Import sorter
flake8 .                                # Python linter
npm run lint                            # ESLint + TypeScript

# 5. Commit using conventional commits
git commit -m "feat: add fleet geofencing alert threshold configuration"
git commit -m "fix: correct TimescaleDB bucket interval for 1-minute queries"
git commit -m "test: add async unit tests for Kafka consumer error handling"
git commit -m "perf: optimize asyncpg pool size for high-throughput ingestion"

# 6. Push and open pull request
git push origin feature/your-feature-name
```

### Code Style Guidelines
```bash
# Python — enforced standards
# - Black formatting, line length 88
# - isort for import ordering
# - Type annotations required on all function signatures
# - Pydantic v2 models for all request/response schemas
# - pytest-asyncio for all async service tests
# - No synchronous database calls in async FastAPI routes

# TypeScript — enforced standards
# - Strict mode enabled — no implicit any
# - All Redux async actions use createAsyncThunk
# - All components include data-testid for Cypress selectors
# - Mapbox interactions isolated in custom hooks
```

---

## 📄 License

This project was developed during research and instructional work at
**SENA (Servicio Nacional de Aprendizaje)** under the **SENNOVA** program,
focused on supporting digital transformation and industrial IoT innovation
for Colombian SMEs and research institutions.

> ⚠️ **Intellectual Property Notice**
>
> The source code, architecture design, technical documentation, and all
> associated assets are **institutional property of SENA** and are not
> publicly available in this repository. The content presented here —
> including technical specifications, architecture diagrams, code samples,
> simulator implementations, and API documentation — has been **recreated
> for portfolio demonstration purposes only**, without exposing confidential
> institutional information or the original production codebase.
>
> Screenshots and UI captures have been intentionally excluded to protect
> operational data confidentiality and institutional privacy.

**Available for:**

- ✅ Custom consulting and implementation for industrial fleet management systems
- ✅ Real-time IoT telemetry architecture design with Kafka and TimescaleDB
- ✅ Python FastAPI + React full-stack development for SaaS platforms
- ✅ Geospatial dashboard development with Mapbox GL JS
- ✅ Event-driven architecture design for distributed autonomous systems
- ✅ Additional module development and production system support

---

*Developed by **Paula Abad** — Senior Software Developer & SENA Instructor/Researcher*
*🌐 [paulabad.tech](https://paulabad.tech) · 📱 Direct developer support via WhatsApp*
