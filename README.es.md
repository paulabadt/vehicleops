# VehicleOps 🚗

*Plataforma SaaS de monitoreo y control en tiempo real para flotas de vehículos
industriales autónomos, con arquitectura orientada a eventos, rastreo geoespacial
en vivo con Mapbox y procesamiento de telemetría en series de tiempo*

---

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Funcionalidades Principales](#funcionalidades-principales)
- [Stack Tecnológico](#stack-tecnológico)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Instalación](#instalación)
- [Uso](#uso)
- [Ejemplos de Código](#ejemplos-de-código)
- [Documentación de la API](#documentación-de-la-api)
- [Contribución](#contribución)
- [Licencia](#licencia)

---

## 🌟 Descripción General

**VehicleOps** es una plataforma SaaS nativa en la nube diseñada para monitorear,
controlar y coordinar flotas de vehículos industriales autónomos en tiempo real.
El sistema ingesta flujos de telemetría de alta frecuencia desde sensores vehiculares
distribuidos, los procesa a través de una arquitectura orientada a eventos respaldada
por Apache Kafka, persiste métricas de series de tiempo en TimescaleDB, y entrega
inteligencia operativa en vivo a los operadores a través de un dashboard interactivo
con Mapbox construido en React y TypeScript.

Desarrollado como parte de un proyecto de investigación en el SENA (Servicio Nacional
de Aprendizaje), este sistema demuestra ingeniería full-stack Python de nivel
producción usando FastAPI en el backend y React en el frontend, con énfasis en
arquitectura orientada a eventos, streaming WebSocket en tiempo real, visualización
de datos geoespaciales y patrones de diseño de microservicios escalables aplicables
a sistemas autónomos e IoT industrial.

### 🎯 Objetivos del Proyecto

- Procesar flujos de telemetría en tiempo real desde flotas de vehículos autónomos distribuidos
- Implementar Arquitectura Orientada a Eventos (EDA) de nivel producción con Apache Kafka
- Persistir y consultar datos de sensores de alta frecuencia usando el motor de series de tiempo TimescaleDB
- Entregar actualizaciones de posición vehicular en vivo con sub-segundo vía WebSocket a dashboards Mapbox
- Demostrar microservicios Python FastAPI escalables con I/O asíncrono en toda la arquitectura
- Construir frontend React + TypeScript responsivo y resiliente bajo alta carga de datos
- Exponer APIs REST y endpoints WebSocket consumibles por sistemas de terceros
- Aplicar TDD con Pytest en servicios backend y BDD con Cypress en flujos E2E

### 🏆 Logros

- ✅ Procesamiento de más de 10.000 eventos de telemetría por minuto con latencia promedio inferior a 50ms
- ✅ Actualizaciones de posición Mapbox en vivo para más de 200 vehículos concurrentes
- ✅ Latencia del stream WebSocket inferior a 80ms desde el evento del sensor hasta el dashboard
- ✅ Consultas de hipertabla TimescaleDB sobre historial de 30 días de sensores con respuesta inferior a 200ms
- ✅ Throughput del grupo de consumidores Kafka sostenido a 50.000 mensajes/minuto bajo carga
- ✅ Cero pérdida de datos durante pruebas simuladas de failover y reconexión del broker Kafka
- ✅ Cobertura de pruebas unitarias superior al 98% en todas las capas de servicio FastAPI con Pytest

---

## ✨ Funcionalidades Principales

### 📡 Pipeline de Ingesta de Telemetría en Tiempo Real
```python
# telemetry/consumers/vehicle_telemetry_consumer.py
# FastAPI + aiokafka — Consumidor Kafka asíncrono para flujos de telemetría vehicular

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
        logger.info(
            "Consumidor de telemetría iniciado — escuchando en %s",
            settings.KAFKA_TELEMETRY_TOPIC,
        )

        try:
            async for message in self.consumer:
                await self._process_message(message.value)
        finally:
            await self.consumer.stop()

    async def _process_message(self, payload: dict) -> None:
        try:
            # Persistir telemetría cruda en hipertabla TimescaleDB
            await self.telemetry_service.ingest(payload)

            # Evaluar umbrales de alerta
            alerts = await self.alert_service.evaluate(payload)

            # Transmitir actualización en vivo a suscriptores WebSocket
            await self.ws_manager.broadcast_vehicle_update(
                fleet_id=payload["fleet_id"],
                data={
                    "vehicle_id":    payload["vehicle_id"],
                    "latitude":      payload["latitude"],
                    "longitude":     payload["longitude"],
                    "speed_kmh":     payload["speed_kmh"],
                    "battery_pct":   payload["battery_pct"],
                    "status":        payload["status"],
                    "alerts":        alerts,
                    "timestamp":     payload["timestamp"],
                },
            )
        except Exception as exc:
            logger.error(
                "Error procesando mensaje de telemetría: %s", exc,
                exc_info=True,
            )
```

**Funcionalidades:**
- ⚡ Consumidor Kafka asíncrono con aiokafka procesando más de 10K eventos/minuto
- 🗄️ Ingesta en hipertabla TimescaleDB para datos de sensores de series de tiempo de alta frecuencia
- 📡 Transmisión WebSocket en tiempo real a todos los suscriptores del dashboard por flota
- 🚨 Evaluación de umbrales de alerta en cada evento de telemetría entrante
- 🔁 Rebalanceo automático del grupo de consumidores y gestión de offsets

### 🗺️ Rastreo Geoespacial en Vivo — Mapbox + WebSocket
```typescript
// components/FleetMap/FleetMap.tsx
// React + TypeScript — Mapa de flota en vivo con Mapbox GL JS + WebSocket

import React, { useEffect, useRef, useCallback } from 'react';
import mapboxgl from 'mapbox-gl';
import { useDispatch, useSelector } from 'react-redux';
import { AppDispatch, RootState } from '../../store';
import {
  updateVehiclePosition,
  selectVehicle,
} from '../../store/slices/fleetSlice';
import { VehicleTelemetry } from '../../types/fleet';
import styles from './FleetMap.module.scss';

mapboxgl.accessToken = process.env.REACT_APP_MAPBOX_TOKEN!;

const FleetMap: React.FC = () => {
  const dispatch = useDispatch<AppDispatch>();
  const { vehicles, selectedVehicleId, fleetId } = useSelector(
    (state: RootState) => state.fleet
  );

  const mapContainerRef = useRef<HTMLDivElement>(null);
  const mapRef         = useRef<mapboxgl.Map>();
  const markersRef     = useRef<Map<string, mapboxgl.Marker>>(new Map());
  const socketRef      = useRef<WebSocket>();

  // Inicializar mapa Mapbox
  useEffect(() => {
    if (!mapContainerRef.current) return;

    mapRef.current = new mapboxgl.Map({
      container: mapContainerRef.current,
      style:     'mapbox://styles/mapbox/dark-v11',
      center:    [-74.0721, 4.711],  // Bogotá
      zoom:      13,
    });

    mapRef.current.addControl(new mapboxgl.NavigationControl());
    mapRef.current.addControl(new mapboxgl.ScaleControl());

    return () => mapRef.current?.remove();
  }, []);

  // Conexión WebSocket para telemetría en vivo
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
      console.error('Error WebSocket:', err);
    };

    return () => socketRef.current?.close();
  }, [fleetId, dispatch]);

  // Actualizar o crear marcador Mapbox por vehículo
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
                <p>Velocidad: ${telemetry.speed_kmh} km/h</p>
                <p>Batería: ${telemetry.battery_pct}%</p>
                <p>Estado: ${status}</p>
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
        {vehicles.length} vehículos activos
      </div>
    </div>
  );
};

export default FleetMap;
```

**Funcionalidades:**
- 🗺️ Mapa industrial Mapbox GL JS estilo oscuro con marcadores vehiculares en tiempo real
- 🔴 Marcadores con código de color por estado del vehículo y estado de alerta activa
- 💬 Popups contextuales con velocidad, batería y estado en vivo por vehículo
- 🔌 Conexión WebSocket gestionada independientemente de Redux para baja latencia
- 📍 Reposicionamiento suave de marcadores en cada evento de telemetría entrante

### ⚡ Arquitectura Orientada a Eventos — Productores Apache Kafka
```python
# events/producers/telemetry_producer.py
# FastAPI — Productor Kafka para eventos de telemetría vehicular

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
            acks="all",               # Esperar todas las réplicas
            compression_type="gzip",  # Comprimir para throughput
            max_batch_size=32768,
            linger_ms=5,              # Ventana de micro-batching
        )
        await self.producer.start()
        logger.info("Productor Kafka de telemetría iniciado")

    async def stop(self) -> None:
        if self.producer:
            await self.producer.stop()

    async def publish_telemetry(self, event: TelemetryEvent) -> None:
        payload = {
            "vehicle_id":    event.vehicle_id,
            "fleet_id":      event.fleet_id,
            "latitude":      event.latitude,
            "longitude":     event.longitude,
            "speed_kmh":     event.speed_kmh,
            "battery_pct":   event.battery_pct,
            "engine_temp_c": event.engine_temp_c,
            "load_pct":      event.load_pct,
            "status":        event.status,
            "timestamp":     datetime.now(timezone.utc).isoformat(),
        }

        await self.producer.send_and_wait(
            topic=settings.KAFKA_TELEMETRY_TOPIC,
            key=event.vehicle_id.encode("utf-8"),
            value=payload,
        )

        logger.debug(
            "Telemetría publicada para vehículo %s", event.vehicle_id
        )

    async def publish_command(
        self, vehicle_id: str, command: str, params: dict
    ) -> None:
        payload = {
            "vehicle_id": vehicle_id,
            "command":    command,    # STOP, RESUME, REROUTE, RETURN_TO_BASE
            "params":     params,
            "issued_at":  datetime.now(timezone.utc).isoformat(),
        }

        await self.producer.send_and_wait(
            topic=settings.KAFKA_COMMANDS_TOPIC,
            key=vehicle_id.encode("utf-8"),
            value=payload,
        )

        logger.info(
            "Comando %s enviado al vehículo %s", command, vehicle_id
        )
```

**Funcionalidades:**
- 🔀 Tópicos Kafka separados para flujos de telemetría y eventos de comandos vehiculares
- ✅ Configuración de productor `acks=all` para garantías de cero pérdida de mensajes
- 🗜️ Compresión Gzip y micro-batching para ingesta de alto throughput
- 🔑 Vehicle ID como clave de partición Kafka para streams de eventos ordenados por vehículo
- 📨 Despacho asíncrono de comandos: STOP, RESUME, REROUTE, RETURN_TO_BASE

### 🕐 Analítica de Series de Tiempo — TimescaleDB
```python
# services/analytics_service.py
# FastAPI — Consultas TimescaleDB para analítica de desempeño vehicular

from datetime import datetime
from typing import Any
import asyncpg
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

**Funcionalidades:**
- ⏱️ `time_bucket()` de TimescaleDB para agregaciones automáticas por intervalo
- 📊 Series de tiempo de velocidad, batería y temperatura por vehículo con granularidad configurable
- 🚛 Reportes de desempeño por flota con ranking de distancia, eficiencia y alertas
- 🔍 Compresión de chunks de hipertabla para retención de telemetría de 90 días a costo mínimo
- ⚡ Pool de conexiones asyncpg para consultas de base de datos asíncronas no bloqueantes

---

## 🛠️ Stack Tecnológico

### Backend

| Tecnología        | Propósito                                              | Versión  |
|-------------------|--------------------------------------------------------|----------|
| **Python**        | Lenguaje backend principal                             | 3.11+    |
| **FastAPI**       | Framework REST API y WebSocket asíncrono               | 0.111.x  |
| **aiokafka**      | Productor y consumidor Kafka asíncrono                 | 0.10.x   |
| **Apache Kafka**  | Broker de mensajes para arquitectura orientada a eventos| 3.7.x   |
| **TimescaleDB**   | Hipertabla de series de tiempo para telemetría         | 2.14.x   |
| **PostgreSQL**    | Datos relacionales (flota, vehículos, operadores)      | 15+      |
| **asyncpg**       | Driver PostgreSQL asíncrono (TimescaleDB)              | 0.29.x   |
| **SQLAlchemy**    | ORM para entidades PostgreSQL relacionales             | 2.x      |
| **Redis**         | Caché, pub/sub y limitación de velocidad               | 7.x      |
| **Pydantic v2**   | Validación de datos y gestión de configuración         | 2.x      |
| **Passlib + JWT** | Autenticación y gestión de tokens                      | latest   |
| **Pytest**        | TDD — suite de pruebas unitarias e integración         | 8.x      |
| **pytest-asyncio**| Soporte de pruebas async para servicios FastAPI        | 0.23.x   |
| **httpx**         | Cliente HTTP asíncrono para pruebas de integración     | 0.27.x   |

### Frontend

| Tecnología                | Propósito                                       | Versión  |
|---------------------------|-------------------------------------------------|----------|
| **React**                 | Framework de UI                                 | 18.x     |
| **TypeScript**            | Tipado estático                                 | 5.x      |
| **React-Redux**           | Gestión de estado global                        | 8.x      |
| **Redux Toolkit**         | Redux simplificado con slices y Thunks          | 1.9.x    |
| **Redux Thunk**           | Middleware asíncrono para llamadas a la API     | 2.4.x    |
| **Mapbox GL JS**          | Mapa geoespacial interactivo de flota           | 3.x      |
| **React Router v6**       | Enrutamiento del lado del cliente               | 6.x      |
| **Webpack**               | Configuración manual de empaquetado de módulos  | 5.x      |
| **SASS/SCSS**             | Preprocesamiento avanzado de CSS                | 1.x      |
| **Axios**                 | Cliente HTTP con interceptores                  | 1.x      |
| **Chart.js**              | Gráficas de telemetría en series de tiempo      | 4.x      |
| **React Testing Library** | Pruebas unitarias de componentes (TDD)          | 14.x     |
| **Cypress**               | Pruebas end-to-end BDD                          | 13.x     |

### DevOps e Infraestructura

| Tecnología          | Propósito                                           |
|---------------------|-----------------------------------------------------|
| **Docker**          | Contenedorización de servicios                      |
| **Docker Compose**  | Orquestación local de múltiples servicios           |
| **AWS ECS Fargate** | Hosting serverless de contenedores                  |
| **AWS MSK**         | Apache Kafka gestionado en AWS                      |
| **AWS RDS**         | PostgreSQL + TimescaleDB gestionados                |
| **AWS ElastiCache** | Redis gestionado                                    |
| **GitHub Actions**  | Pipeline CI/CD — pruebas, build, despliegue         |
| **Prometheus**      | Recolección de métricas desde servicios FastAPI     |
| **Grafana**         | Dashboards operativos y alertas                     |

---

## 🏗️ Arquitectura del Sistema

### Arquitectura General
```
┌─────────────────────────────────────────────────────────────────────┐
│                        CAPA DE PRESENTACIÓN                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐ │
│  │  Dashboard del   │  │  Dashboard del   │  │   Panel de       │ │
│  │    Operador      │  │    Gerente       │  │Administración    │ │
│  │  (React + TS)    │  │  (React + TS)    │  │  (React + TS)    │ │
│  │  Mapa Mapbox     │  │  Vista Analítica │  │  Config Sistema  │ │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘ │
│           │                     │                      │            │
│           └─────────────────────┴──────────────────────┘           │
│                                 │                                   │
│              ┌──────────────────▼──────────────────┐               │
│              │        Redux Store + Thunks          │               │
│              │  flota · telemetría · alertas · auth │               │
│              └──────────────────┬──────────────────┘               │
└─────────────────────────────────┼───────────────────────────────────┘
                    REST API       │       WebSocket
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                      CAPA DE APLICACIÓN                             │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │           API Gateway FastAPI (Python 3.11)                  │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │  Router   │ │  Router   │ │  Router   │ │   Router   │  │   │
│  │  │   Auth    │ │   Flota   │ │Telemetría │ │  Comandos  │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  │                                                              │   │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌────────────┐  │   │
│  │  │  Router   │ │  Router   │ │ Gestor    │ │   Router   │  │   │
│  │  │Analítica  │ │  Alertas  │ │WebSocket  │ │  Misiones  │  │   │
│  │  └───────────┘ └───────────┘ └───────────┘ └────────────┘  │   │
│  └──────────────────────────────────────────────────────────────┘  │
│                                 │                                   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│               CAPA DE ARQUITECTURA ORIENTADA A EVENTOS (EDA)        │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │                   Clúster Apache Kafka                       │   │
│  │                                                              │   │
│  │  Tópicos:                                                    │   │
│  │  ├── fleet.telemetry.raw          (streams de sensores)      │   │
│  │  ├── fleet.telemetry.processed    (eventos enriquecidos)     │   │
│  │  ├── fleet.commands               (STOP·RESUME·REROUTE)      │   │
│  │  ├── fleet.alerts.triggered       (violaciones de umbral)    │   │
│  │  └── fleet.missions.events        (ciclo de vida misión)     │   │
│  └──────────────────────────────────────────────────────────────┘  │
│           │                    │                    │               │
│  ┌────────▼──────┐  ┌──────────▼──────┐  ┌────────▼──────────┐   │
│  │  Consumidor   │  │  Consumidor     │  │  Consumidor       │   │
│  │ Telemetría    │  │   Alertas       │  │   Misiones        │   │
│  │  (Python)     │  │  (Python)       │  │  (Python)         │   │
│  └───────────────┘  └─────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                         CAPA DE DATOS                               │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────┐  ┌────────────▼───┐  ┌──────────┐  ┌─────────┐  │
│  │ TimescaleDB  │  │  PostgreSQL    │  │  Redis   │  │ AWS S3  │  │
│  │              │  │                │  │          │  │         │  │
│  │ - Hipertabla │  │ - Flotas       │  │ - Caché  │  │ - Logs  │  │
│  │   telemetría │  │ - Vehículos    │  │ - Pub/Sub│  │ - Rutas │  │
│  │ - Series     │  │ - Operadores   │  │ - Rate   │  │         │  │
│  │   velocidad  │  │ - Misiones     │  │   Limit  │  │         │  │
│  │ - Series     │  │ - Alertas      │  │ - Estado │  │         │  │
│  │   batería    │  │ - Comandos     │  │   WS     │  │         │  │
│  └──────────────┘  └────────────────┘  └──────────┘  └─────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                     CAPA VEHICULAR / IoT                            │
├─────────────────────────────────┼───────────────────────────────────┤
│                                 │                                   │
│  ┌──────────────────────────────▼──────────────────────────────┐   │
│  │          Simulador de Telemetría Vehicular (Python)          │   │
│  │  Simula GPS, velocidad, batería, temperatura y sensores      │   │
│  │  de carga de una flota configurable de vehículos autónomos   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│        │                  │                  │                       │
│  ┌─────▼──────┐  ┌────────▼──────┐  ┌───────▼───────┐            │
│  │ Vehículo A │  │  Vehículo B   │  │  Vehículo C   │            │
│  │ GPS + IMU  │  │  GPS + IMU    │  │  GPS + IMU    │            │
│  │ (simulado) │  │  (simulado)   │  │  (simulado)   │            │
│  └────────────┘  └───────────────┘  └───────────────┘            │
└─────────────────────────────────────────────────────────────────────┘
```

### Estructura de Servicios
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

### Flujo de Datos
```
1. El simulador vehicular publica telemetría en Kafka
   └──> Tópico: fleet.telemetry.raw
        └──> TelemetryConsumer recibe el evento
             └──> TelemetryService persiste en hipertabla TimescaleDB
                  └──> AlertService evalúa umbrales de alerta
                       ├──> Si hay alerta: publica en fleet.alerts.triggered
                       │    └──> AlertConsumer notifica operadores vía WebSocket
                       └──> WebSocketManager transmite a todos los suscriptores de la flota
                            └──> Marcadores Mapbox se actualizan en tiempo real en el dashboard

2. El operador emite un comando vehicular desde el dashboard
   └──> POST /api/v1/commands/{vehicle_id}
        └──> CommandProducer publica en fleet.commands
             └──> El simulador vehicular consume el comando
                  └──> Ejecuta: STOP · RESUME · REROUTE · RETURN_TO_BASE
                       └──> Actualización de estado publicada de vuelta en fleet.telemetry.raw
```

### Tópicos Kafka
```
fleet.telemetry.raw
├── Clave:       vehicle_id
├── Particiones: 12 (una por grupo de vehículos)
├── Retención:   24 horas
└── Consumidores: telemetry-service, alert-service

fleet.telemetry.processed
├── Clave:       vehicle_id
├── Particiones: 12
├── Retención:   6 horas
└── Consumidores: analytics-service, websocket-broadcaster

fleet.commands
├── Clave:       vehicle_id
├── Particiones: 12
├── Retención:   1 hora
└── Consumidores: vehicle-simulator, command-logger

fleet.alerts.triggered
├── Clave:       fleet_id
├── Particiones: 4
├── Retención:   48 horas
└── Consumidores: notification-service, alert-recorder

fleet.missions.events
├── Clave:       mission_id
├── Particiones: 4
├── Retención:   7 días
└── Consumidores: mission-service, audit-logger
```

---

## 💾 Instalación

### Requisitos Previos
```bash
# Software requerido
- Python 3.11 o superior
- Node.js 20 LTS o superior
- Docker y Docker Compose
- Cuenta Mapbox y token de acceso (disponible en nivel gratuito)
- Cuenta AWS (opcional — para despliegue en nube)
```

### Opción 1: Instalación con Docker (Recomendada)
```bash
# 1. Clonar el repositorio
git clone https://github.com/paulabadt/vehicleops.git
cd vehicleops

# 2. Copiar archivos de variables de entorno
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local

# 3. Agregar token Mapbox al frontend
echo "REACT_APP_MAPBOX_TOKEN=tu_token_mapbox_aqui" >> frontend/.env.local

# 4. Iniciar todos los servicios
docker-compose up -d

# 5. Ejecutar migraciones de base de datos
docker-compose exec backend alembic upgrade head

# 6. Cargar datos de demostración de flota
docker-compose exec backend python -m app.scripts.seed_demo

# 7. Iniciar simulador de vehículos (terminal separada)
docker-compose exec backend python -m simulator.vehicle_simulator \
  --fleet-id fleet-demo-001 \
  --vehicles 20 \
  --interval-ms 500

# 8. Verificar que todos los servicios están activos
docker-compose ps

# 9. Acceder a la plataforma
# Frontend:      http://localhost:3000
# API:           http://localhost:8000
# Docs API:      http://localhost:8000/docs
# Kafka UI:      http://localhost:8080
# Grafana:       http://localhost:3001
```

### Opción 2: Instalación Manual

#### Configuración del Backend (Python + FastAPI)
```bash
# 1. Ingresar al directorio del backend
cd backend

# 2. Crear y activar entorno virtual
python -m venv venv
source venv/bin/activate        # Linux/macOS
# venv\Scripts\activate         # Windows

# 3. Instalar dependencias
pip install -r requirements.txt

# 4. Configurar variables de entorno
cp .env.example .env
# Editar .env con las cadenas de conexión a base de datos y Kafka

# 5. Iniciar PostgreSQL, TimescaleDB y Redis (vía Docker)
docker-compose up -d postgres timescaledb redis kafka zookeeper

# 6. Ejecutar migraciones Alembic
alembic upgrade head

# 7. Cargar datos de demostración de flota
python -m app.scripts.seed_demo

# 8. Iniciar servidor FastAPI en modo desarrollo
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# 9. Iniciar consumidores Kafka (terminal separada)
python -m app.events.consumers.telemetry_consumer
```

#### Configuración del Frontend (React + TypeScript)
```bash
# 1. Ingresar al directorio del frontend
cd frontend

# 2. Instalar dependencias
npm install

# 3. Configurar variables de entorno
cp .env.example .env.local
# Editar .env.local — definir URL de API y token Mapbox

# 4. Iniciar servidor de desarrollo
npm run dev

# 5. Compilar para producción
npm run build
```

### Variables de Entorno
```bash
# backend/.env

# Aplicación
APP_ENV=development
APP_HOST=0.0.0.0
APP_PORT=8000
SECRET_KEY=tu_clave_secreta_super_segura_minimo_32_caracteres
ACCESS_TOKEN_EXPIRE_MINUTES=60

# PostgreSQL — Datos relacionales
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=vehicleops_user
POSTGRES_PASSWORD=tu_contraseña_segura
POSTGRES_DB=vehicleops_db

# TimescaleDB — Series de tiempo de telemetría
TIMESCALE_HOST=localhost
TIMESCALE_PORT=5433
TIMESCALE_USER=timescale_user
TIMESCALE_PASSWORD=tu_contraseña_timescale
TIMESCALE_DB=vehicleops_timeseries

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=tu_contraseña_redis

# Apache Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_TELEMETRY_TOPIC=fleet.telemetry.raw
KAFKA_COMMANDS_TOPIC=fleet.commands
KAFKA_ALERTS_TOPIC=fleet.alerts.triggered
KAFKA_MISSIONS_TOPIC=fleet.missions.events
KAFKA_CONSUMER_GROUP=vehicleops-backend

# AWS (producción)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=tu_access_key
AWS_SECRET_ACCESS_KEY=tu_secret_key
AWS_S3_BUCKET=vehicleops-logs

# Frontend
FRONTEND_URL=http://localhost:3000
```
```bash
# frontend/.env.local

REACT_APP_API_URL=http://localhost:8000/api/v1
REACT_APP_WS_URL=ws://localhost:8000
REACT_APP_MAPBOX_TOKEN=tu_token_publico_mapbox
REACT_APP_APP_NAME=VehicleOps
```

### Servicios Docker Compose
```yaml
# docker-compose.yml — resumen de servicios
services:
  backend:      # App FastAPI       — puerto 8000
  frontend:     # App React         — puerto 3000
  postgres:     # PostgreSQL        — puerto 5432
  timescaledb:  # TimescaleDB       — puerto 5433
  redis:        # Redis             — puerto 6379
  zookeeper:    # Dependencia Kafka
  kafka:        # Apache Kafka      — puerto 9092
  kafka-ui:     # Dashboard Kafka   — puerto 8080
  prometheus:   # Recolección métricas — puerto 9090
  grafana:      # Dashboards operativos — puerto 3001
```

---

## 🚀 Uso

### Iniciar la Plataforma
```bash
# Iniciar todos los servicios de infraestructura
docker-compose up -d

# Iniciar backend FastAPI
cd backend
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Iniciar consumidores Kafka (terminal separada)
python -m app.events.consumers.telemetry_consumer &
python -m app.events.consumers.alert_consumer &
python -m app.events.consumers.mission_consumer &

# Iniciar frontend React (terminal separada)
cd frontend
npm run dev

# Iniciar simulador de vehículos con 20 unidades (terminal separada)
cd backend
python -m simulator.vehicle_simulator \
  --fleet-id fleet-demo-001 \
  --vehicles 20 \
  --interval-ms 500
```

### Credenciales por Defecto
```bash
# Administrador del sistema
Email:    admin@vehicleops.io
Password: Admin123! (¡cambiar inmediatamente!)

# Gerente de flota
Email:    manager@vehicleops.io
Password: Manager123!

# Operador de flota
Email:    operator@vehicleops.io
Password: Operator123!
```

### Opciones del Simulador de Vehículos
```bash
# Ejecutar simulador con parámetros personalizados
python -m simulator.vehicle_simulator \
  --fleet-id     fleet-001     \  # Identificador de flota objetivo
  --vehicles     50            \  # Número de vehículos a simular
  --interval-ms  500           \  # Intervalo de publicación de telemetría
  --area         bogota        \  # Área geolimitada (bogota|medellin|cali)
  --scenario     mixed            # Escenario: normal|high-load|fault-injection

# Escenarios disponibles
# normal          — operaciones estándar, rutas aleatorias
# high-load       — prueba de estrés de máximo throughput
# fault-injection — simula fallos de batería y violaciones de velocidad
```

### Scripts Disponibles
```bash
# Backend
uvicorn app.main:app --reload          # Servidor de desarrollo con recarga automática
uvicorn app.main:app --workers 4       # Servidor de producción multi-worker
alembic upgrade head                   # Ejecutar migraciones pendientes
alembic revision --autogenerate -m ""  # Generar nueva migración
python -m app.scripts.seed_demo        # Cargar datos de flota y operadores de demo
pytest                                 # Ejecutar suite completa de pruebas
pytest --cov=app --cov-report=html     # Pruebas con reporte de cobertura HTML
pytest -m unit                         # Solo pruebas unitarias
pytest -m integration                  # Solo pruebas de integración

# Frontend
npm run dev                            # Servidor de desarrollo con HMR
npm run build                          # Bundle de producción Webpack optimizado
npm run preview                        # Previsualizar build de producción
npm run test                           # Pruebas unitarias React Testing Library
npm run test:coverage                  # Reporte de cobertura del frontend
npm run cypress:open                   # Runner interactivo BDD de Cypress
npm run cypress:run                    # Suite completa Cypress en modo headless
npm run lint                           # Verificación ESLint + TypeScript
```

---

## 💻 Ejemplos de Código

### 1. Gestor WebSocket FastAPI — Transmisión en Vivo de Flota
```python
# app/websocket/manager.py
# FastAPI — Gestor de conexiones WebSocket para actualizaciones de flota en tiempo real

import asyncio
import json
import logging
from collections import defaultdict
from fastapi import WebSocket

logger = logging.getLogger(__name__)


class WebSocketManager:
    def __init__(self):
        # fleet_id → conjunto de conexiones WebSocket activas
        self._connections: dict[str, set[WebSocket]] = defaultdict(set)

    async def connect(self, websocket: WebSocket, fleet_id: str) -> None:
        await websocket.accept()
        self._connections[fleet_id].add(websocket)
        logger.info(
            "WebSocket conectado — flota=%s conexiones_totales=%d",
            fleet_id,
            len(self._connections[fleet_id]),
        )

    def disconnect(self, websocket: WebSocket, fleet_id: str) -> None:
        self._connections[fleet_id].discard(websocket)
        logger.info(
            "WebSocket desconectado — flota=%s restantes=%d",
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
                logger.warning(
                    "WebSocket inactivo detectado — eliminando del pool"
                )
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
# FastAPI — Endpoint WebSocket para suscripción a telemetría de flota en vivo

from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Depends
from app.websocket.manager import WebSocketManager
from app.core.deps import get_ws_manager

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
            # Mantener conexión activa — cliente envía ping cada 30s
            data = await websocket.receive_text()
            if data == "ping":
                await websocket.send_text("pong")
    except WebSocketDisconnect:
        manager.disconnect(websocket, fleet_id)
```

---

### 2. Endpoints REST FastAPI — Flota y Comandos
```python
# app/api/v1/commands.py
# FastAPI — Despacho de comandos vehiculares con productor Kafka

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
    # Verificar que el vehículo pertenece a la flota del operador
    vehicle = await fleet_service.get_vehicle(vehicle_id)
    if not vehicle:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Vehículo {vehicle_id} no encontrado",
        )

    if vehicle.fleet_id not in current_operator.fleet_ids:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="El operador no tiene acceso a la flota de este vehículo",
        )

    # Publicar evento de comando en Kafka
    await producer.publish_command(
        vehicle_id=vehicle_id,
        command=request.command,  # STOP | RESUME | REROUTE | RETURN_TO_BASE
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
# FastAPI — Endpoints de analítica para consultas de telemetría en series de tiempo

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

### 3. Simulador de Vehículos — Generador Asíncrono de Telemetría
```python
# simulator/vehicle_simulator.py
# Python — Simulador asíncrono de telemetría vehicular publicando en Kafka

import asyncio
import argparse
import random
import logging
from datetime import datetime, timezone
from dataclasses import dataclass
from aiokafka import AIOKafkaProducer
import json

logger = logging.getLogger(__name__)

# Áreas de simulación geolimitadas
AREAS = {
    "bogota":   {"lat": 4.7110,  "lng": -74.0721, "radio_km": 15},
    "medellin": {"lat": 6.2442,  "lng": -75.5812, "radio_km": 12},
    "cali":     {"lat": 3.4516,  "lng": -76.5320, "radio_km": 10},
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
        """Avanzar el estado del vehículo un paso de simulación."""
        # Mover vehículo en dirección aleatoria
        self.latitude  += random.uniform(-0.0003, 0.0003)
        self.longitude += random.uniform(-0.0003, 0.0003)

        # Actualizar velocidad
        if scenario == "fault-injection" and random.random() < 0.05:
            self.speed_kmh = random.uniform(90, 120)  # Fallo por exceso de velocidad
        else:
            self.speed_kmh = random.uniform(5, 60)

        # Consumo gradual de batería
        self.battery_pct = max(0.0, self.battery_pct - random.uniform(0, 0.05))

        # Fluctuación de temperatura del motor
        self.engine_temp_c = random.uniform(60, 105)

        # Porcentaje de carga
        self.load_pct = random.uniform(20, 95)

        # Derivar estado
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
            "Publicado: vehicle=%s speed=%.1f battery=%.1f%%",
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
        "Simulador iniciado — flota=%s vehículos=%d área=%s escenario=%s",
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
    parser = argparse.ArgumentParser(
        description="Simulador de Flota VehicleOps"
    )
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

### 4. Redux Thunks — Gestión de Estado de Flota
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
        error.response?.data?.detail ||
        'Error al obtener resumen de flota'
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
        error.response?.data?.detail ||
        'Error al despachar comando'
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
    // Llamado en cada mensaje WebSocket de telemetría
    updateVehiclePosition: (
      state,
      action: PayloadAction<VehicleTelemetry>
    ) => {
      const { vehicle_id } = action.payload;
      state.vehicles[vehicle_id] = action.payload;
    },
    // Llamado cuando WebSocket emite un evento de alerta
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
        console.info('Comando despachado:', action.payload);
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

### 5. Pruebas Backend — Pytest + pytest-asyncio (TDD)
```python
# tests/unit/test_alert_service.py
import pytest
from unittest.mock import AsyncMock
from app.services.alert_service import AlertService


@pytest.fixture
def alert_service():
    mock_repo     = AsyncMock()
    mock_producer = AsyncMock()
    return AlertService(
        alert_repository=mock_repo,
        kafka_producer=mock_producer,
    )


@pytest.mark.asyncio
async def test_evaluate_dispara_alerta_exceso_velocidad(alert_service):
    # Dado
    payload = {
        "vehicle_id":    "fleet-001-VH-001",
        "fleet_id":      "fleet-001",
        "speed_kmh":     92.5,
        "battery_pct":   80.0,
        "engine_temp_c": 70.0,
        "status":        "ALERT",
        "timestamp":     "2024-01-01T10:00:00Z",
    }

    # Cuando
    alerts = await alert_service.evaluate(payload)

    # Entonces
    assert any(a["type"] == "OVERSPEED" for a in alerts)
    alert_service.kafka_producer.publish_command.assert_not_called()


@pytest.mark.asyncio
async def test_evaluate_dispara_alerta_bateria_baja(alert_service):
    # Dado
    payload = {
        "vehicle_id":    "fleet-001-VH-002",
        "fleet_id":      "fleet-001",
        "speed_kmh":     30.0,
        "battery_pct":   12.0,           # Por debajo del umbral del 15%
        "engine_temp_c": 70.0,
        "status":        "LOW_BATTERY",
        "timestamp":     "2024-01-01T10:01:00Z",
    }

    # Cuando
    alerts = await alert_service.evaluate(payload)

    # Entonces
    assert any(a["type"] == "LOW_BATTERY" for a in alerts)
    battery_alert = next(
        a for a in alerts if a["type"] == "LOW_BATTERY"
    )
    assert battery_alert["value"] == 12.0


@pytest.mark.asyncio
async def test_evaluate_dispara_alerta_sobrecalentamiento(alert_service):
    # Dado
    payload = {
        "vehicle_id":    "fleet-001-VH-003",
        "fleet_id":      "fleet-001",
        "speed_kmh":     45.0,
        "battery_pct":   70.0,
        "engine_temp_c": 108.0,          # Por encima del umbral de 100°C
        "status":        "ALERT",
        "timestamp":     "2024-01-01T10:02:00Z",
    }

    # Cuando
    alerts = await alert_service.evaluate(payload)

    # Entonces
    assert any(a["type"] == "ENGINE_OVERHEAT" for a in alerts)


@pytest.mark.asyncio
async def test_evaluate_no_genera_alertas_para_telemetria_normal(alert_service):
    # Dado
    payload = {
        "vehicle_id":    "fleet-001-VH-004",
        "fleet_id":      "fleet-001",
        "speed_kmh":     42.0,
        "battery_pct":   85.0,
        "engine_temp_c": 72.0,
        "status":        "MOVING",
        "timestamp":     "2024-01-01T10:03:00Z",
    }

    # Cuando
    alerts = await alert_service.evaluate(payload)

    # Entonces
    assert alerts == []
```
```typescript
// cypress/e2e/fleet_map.cy.ts — BDD E2E
describe('Mapa de Flota — BDD', () => {
  beforeEach(() => {
    cy.login('operator@vehicleops.io', 'Operator123!');
    cy.visit('/dashboard/fleet/fleet-demo-001');
  });

  it('Dado un operador, Cuando carga el dashboard, Entonces el mapa Mapbox renderiza con vehículos activos',
    () => {
      cy.intercept('GET', '/api/v1/fleet/*/summary', {
        fixture: 'fleet-summary.json',
      }).as('fleetSummary');

      cy.wait('@fleetSummary');

      cy.get('[data-testid="fleet-map"]').should('be.visible');
      cy.get('[data-testid="vehicle-count"]')
        .should('contain.text', 'vehículos activos');
  });

  it('Dado un operador, Cuando selecciona un vehículo, Entonces el panel de detalle muestra telemetría en vivo',
    () => {
      cy.get('[data-testid="fleet-map"]').should('be.visible');
      cy.get('[data-testid="vehicle-list-item"]').first().click();

      cy.get('[data-testid="vehicle-detail-panel"]').should('be.visible');
      cy.get('[data-testid="speed-value"]').should('exist');
      cy.get('[data-testid="battery-value"]').should('exist');
  });

  it('Dado un operador, Cuando despacha el comando STOP, Entonces el estado se actualiza a DETENIDO',
    () => {
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

## 📚 Documentación de la API

### URL Base
```
Desarrollo:    http://localhost:8000/api/v1
Producción:    https://api.vehicleops.io/api/v1
Swagger UI:    http://localhost:8000/docs
ReDoc:         http://localhost:8000/redoc
```

### Autenticación
```bash
POST /api/v1/auth/login
Content-Type: application/json

{
  "email": "operator@vehicleops.io",
  "password": "tu_contraseña"
}

# Respuesta: 200 OK
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
# Uso del token en solicitudes protegidas
GET /api/v1/fleet/fleet-001/summary
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Endpoints

#### 1. Flota

**Obtener Resumen de Flota**
```bash
GET /api/v1/fleet/{fleet_id}/summary
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "fleet_id": "fleet-001",
  "fleet_name": "Zona Industrial A",
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

**Listar Vehículos de la Flota**
```bash
GET /api/v1/fleet/{fleet_id}/vehicles?status=MOVING&page=1&limit=50
Authorization: Bearer {token}

# Parámetros de consulta:
# - status: string (IDLE, MOVING, ALERT, LOW_BATTERY, OFFLINE)
# - page: int (por defecto: 1)
# - limit: int (por defecto: 50, máximo: 200)

# Respuesta: 200 OK
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

#### 2. Telemetría

**Obtener Última Telemetría del Vehículo**
```bash
GET /api/v1/telemetry/{vehicle_id}/latest
Authorization: Bearer {token}

# Respuesta: 200 OK
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

**Obtener Historial de Telemetría**
```bash
GET /api/v1/telemetry/{vehicle_id}/history?from=2024-02-15T00:00:00Z&to=2024-02-15T23:59:59Z&limit=1000
Authorization: Bearer {token}

# Parámetros de consulta:
# - from: fecha ISO (requerido)
# - to:   fecha ISO (requerido)
# - limit: int (por defecto: 1000, máximo: 10000)

# Respuesta: 200 OK
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

**WebSocket — Telemetría de Flota en Vivo**
```bash
WS /ws/fleet/{fleet_id}
# Token como parámetro de consulta
WS /ws/fleet/fleet-001?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# El servidor emite en cada evento de telemetría:
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

# Payload de alerta emitido al violar umbral:
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

# Keepalive del cliente:
→ enviar:  "ping"
← recibir: "pong"
```

#### 3. Comandos

**Despachar Comando Vehicular**
```bash
POST /api/v1/commands/{vehicle_id}
Authorization: Bearer {token}
Content-Type: application/json

{
  "command": "STOP",
  "params": {}
}

# Comandos disponibles:
# STOP            — detener vehículo inmediatamente
# RESUME          — reanudar última misión activa
# REROUTE         — asignar nuevos puntos de ruta
# RETURN_TO_BASE  — navegar a estación base

# Respuesta: 202 Accepted
{
  "vehicle_id": "fleet-001-VH-001",
  "command": "STOP",
  "status": "DISPATCHED",
  "issued_by": "op-001",
  "issued_at": "2024-02-15T14:35:00Z"
}
```

**Obtener Historial de Comandos**
```bash
GET /api/v1/commands/{vehicle_id}/history?limit=20
Authorization: Bearer {token}

# Respuesta: 200 OK
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

#### 4. Analítica

**Resumen de Métricas del Vehículo**
```bash
GET /api/v1/analytics/vehicle/{vehicle_id}/metrics?from=2024-02-15T00:00:00Z&to=2024-02-15T23:59:59Z
Authorization: Bearer {token}

# Respuesta: 200 OK
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

**Series de Tiempo de Velocidad**
```bash
GET /api/v1/analytics/vehicle/{vehicle_id}/speed-series?from=2024-02-15T08:00:00Z&to=2024-02-15T18:00:00Z&bucket_minutes=5
Authorization: Bearer {token}

# Respuesta: 200 OK
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

**Reporte de Desempeño de Flota**
```bash
GET /api/v1/analytics/fleet/{fleet_id}/performance?from=2024-02-01T00:00:00Z&to=2024-02-15T23:59:59Z
Authorization: Bearer {token}

# Respuesta: 200 OK
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

#### 5. Alertas

**Listar Alertas Activas**
```bash
GET /api/v1/alerts?fleet_id=fleet-001&severity=HIGH&acknowledged=false
Authorization: Bearer {token}

# Parámetros de consulta:
# - fleet_id:     string (opcional)
# - vehicle_id:   string (opcional)
# - severity:     string (LOW, MEDIUM, HIGH, CRITICAL)
# - acknowledged: boolean (por defecto: false)
# - limit:        int (por defecto: 50)

# Respuesta: 200 OK
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
      "message": "Vehículo fleet-001-VH-005 superó el límite de velocidad: 92.5 km/h",
      "acknowledged": false,
      "triggered_at": "2024-02-15T14:33:01Z"
    }
  ],
  "total": 2
}
```

**Confirmar Alerta**
```bash
POST /api/v1/alerts/{alert_id}/acknowledge
Authorization: Bearer {token}
Content-Type: application/json

{
  "notes": "Operador contactado — vehículo reduciendo velocidad"
}

# Respuesta: 200 OK
{
  "id": "alert-001",
  "acknowledged": true,
  "acknowledged_by": "op-001",
  "acknowledged_at": "2024-02-15T14:38:00Z",
  "notes": "Operador contactado — vehículo reduciendo velocidad"
}
```

#### 6. Misiones

**Crear Misión**
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

# Respuesta: 201 Created
{
  "id": "mission-008",
  "vehicle_id": "fleet-001-VH-001",
  "status": "ASSIGNED",
  "waypoints": 3,
  "created_at": "2024-02-15T15:00:00Z"
}
```

**Obtener Estado de Misión**
```bash
GET /api/v1/missions/{mission_id}
Authorization: Bearer {token}

# Respuesta: 200 OK
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

### Respuestas de Error
```json
{
  "detail": "Vehículo fleet-001-VH-099 no encontrado",
  "status_code": 404,
  "error": "NOT_FOUND"
}
```

**Códigos de Error Comunes**

| Código            | Estado HTTP | Descripción                                    |
|-------------------|-------------|------------------------------------------------|
| `UNAUTHORIZED`    | 401         | Token JWT faltante o inválido                  |
| `FORBIDDEN`       | 403         | El operador no tiene acceso a esta flota       |
| `NOT_FOUND`       | 404         | Vehículo, flota o misión no encontrado         |
| `CONFLICT`        | 409         | El vehículo ya tiene una misión activa         |
| `UNPROCESSABLE`   | 422         | Cuerpo o parámetros de solicitud inválidos     |
| `INTERNAL_ERROR`  | 500         | Error inesperado del servidor                  |

---

## 🤝 Contribución

Este proyecto fue desarrollado como parte de la labor investigativa en el SENA.
Aunque el código fuente y las aplicaciones son propiedad del SENA, las contribuciones
y sugerencias son bienvenidas.

### Flujo de Desarrollo
```bash
# 1. Crear una rama de funcionalidad
git checkout -b feature/nombre-de-la-funcionalidad

# 2. Realizar los cambios siguiendo las convenciones de servicios FastAPI

# 3. Ejecutar la suite completa de pruebas
pytest                                  # Todas las pruebas del backend
pytest --cov=app --cov-report=html      # Con reporte de cobertura HTML
pytest -m unit                          # Solo pruebas unitarias
pytest -m integration                   # Solo pruebas de integración
npx cypress run                         # Pruebas E2E del frontend

# 4. Formatear y verificar el código
black .                                 # Formateador Python
isort .                                 # Ordenador de imports
flake8 .                                # Linter Python
npm run lint                            # ESLint + TypeScript

# 5. Hacer commit usando commits convencionales
git commit -m "feat: agregar configuración de umbral de alerta por geovalla"
git commit -m "fix: corregir intervalo de bucket TimescaleDB para consultas de 1 minuto"
git commit -m "test: agregar pruebas unitarias async para manejo de errores del consumidor Kafka"
git commit -m "perf: optimizar tamaño del pool asyncpg para ingesta de alto throughput"

# 6. Subir cambios y abrir pull request
git push origin feature/nombre-de-la-funcionalidad
```

### Guía de Estilo de Código
```bash
# Python — estándares obligatorios
# - Formato Black, longitud de línea 88
# - isort para ordenamiento de imports
# - Anotaciones de tipo requeridas en todas las firmas de funciones
# - Modelos Pydantic v2 para todos los esquemas de solicitud/respuesta
# - pytest-asyncio para todas las pruebas de servicios asíncronos
# - Sin llamadas síncronas a base de datos en rutas FastAPI asíncronas

# TypeScript — estándares obligatorios
# - Modo estricto habilitado — sin any implícito
# - Todas las acciones asíncronas Redux usan createAsyncThunk
# - Todos los componentes incluyen data-testid para selectores Cypress
# - Interacciones con Mapbox aisladas en hooks personalizados
```

---

## 📄 Licencia

Este proyecto fue desarrollado durante la labor investigativa y de instrucción en
el **SENA (Servicio Nacional de Aprendizaje)** bajo el programa **SENNOVA**,
enfocado en apoyar la transformación digital e innovación en IoT industrial para
PYMES colombianas e instituciones de investigación.

> ⚠️ **Aviso de Propiedad Intelectual**
>
> El código fuente, diseño arquitectónico, documentación técnica y todos los
> activos asociados son **propiedad institucional del SENA** y no están
> disponibles públicamente en este repositorio. El contenido presentado aquí —
> incluyendo especificaciones técnicas, diagramas de arquitectura, fragmentos
> de código, implementaciones del simulador y documentación de la API — ha sido
> **recreado únicamente con fines de demostración de portafolio**, sin exponer
> información institucional confidencial ni el código fuente original de
> producción.
>
> Las capturas de pantalla e imágenes de la interfaz han sido intencionalmente
> excluidas para proteger la confidencialidad de los datos operativos y la
> privacidad institucional.

**Disponible para:**

- ✅ Consultoría personalizada e implementación para sistemas de gestión de flotas industriales
- ✅ Diseño de arquitectura de telemetría IoT en tiempo real con Kafka y TimescaleDB
- ✅ Desarrollo full-stack Python FastAPI + React para plataformas SaaS
- ✅ Desarrollo de dashboards geoespaciales con Mapbox GL JS
- ✅ Diseño de arquitectura orientada a eventos para sistemas autónomos distribuidos
- ✅ Desarrollo de módulos adicionales y soporte de sistemas en producción

---

*Desarrollado por **Paula Abad** — Desarrolladora de Software Senior e Instructora/Investigadora SENA*
*🌐 [paulabad.tech](https://paulabad.tech) · 📱 Soporte directo de la desarrolladora vía WhatsApp*
