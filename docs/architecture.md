# System Architecture — Pasteurizadora Digital 360°

## Overview

The system is a three-tier, on-premise platform deployed on the plant's local network.
All traffic flows through a single entrypoint (Caddy reverse proxy) to either the
static React SPA or the FastAPI backend, which in turn persists to PostgreSQL.

---

## Architecture Diagram

```
                    ┌──────────────────────────────┐
                    │       CLIENT DEVICES          │
                    │  iOS Safari  │  Android APK   │
                    │  Desktop browser              │
                    └──────────────┬───────────────┘
                                   │  HTTP/HTTPS (LAN only)
                    ┌──────────────▼───────────────┐
                    │       CADDY (ports 80/443)    │
                    │   Reverse proxy + TLS (CA)    │
                    │   Serves React SPA (dist/)    │
                    └──────┬────────────┬──────────┘
                           │ /api/*     │ /*
              ┌────────────▼──┐   ┌────▼─────────────┐
              │  FastAPI :8000│   │  Static files     │
              │  JWT + API Key│   │  (React PWA)      │
              └──────┬────────┘   └──────────────────┘
                     │
          ┌──────────▼──────────┐
          │   PostgreSQL 16     │
          │   Schema: pasteur.  │
          │   49 tables         │
          │   23 triggers       │
          └─────────────────────┘

   IoT Sensors (weight/temp)
          │ MQTT
   ┌──────▼──────────┐
   │  Mosquitto :1883│
   └──────┬──────────┘
          │ subscribe
   ┌──────▼──────────┐
   │  MQTT Worker    │──────► PostgreSQL
   │  (Python)       │
   └─────────────────┘
```

---

## Docker Services

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `db` | postgres:16 | host:5434 | Primary database; 17 init scripts run in alphabetical order |
| `api` | custom (FastAPI) | 8000 | REST API; dual auth (JWT + IoT API Key) |
| `mqtt` | eclipse-mosquitto:2 | 1883 | MQTT broker for IoT sensor telemetry |
| `worker` | custom (same as api) | — | Subscribes to MQTT topics, persists telemetry to DB |
| `caddy` | caddy:2-alpine | 80, 443 | Reverse proxy; serves frontend static files; TLS termination |

All services communicate over an internal Docker network by service name.
PostgreSQL is mapped to host port 5434 to avoid conflicts with native Windows installations.

---

## Authentication

The system uses two authentication mechanisms:

```
Human users (plant operators, QA, managers)
    │
    ├─ POST /api/v1/auth/token  (username + PIN)
    │         │
    │    bcrypt verification (no passlib)
    │         │
    └────► JWT Bearer token (8h expiry)
              │
         Decoded on every protected request
         Role extracted: Operador / Calidad_Aseguramiento /
                         Produccion / Ventas / Administrador

IoT Sensors (weight scale, temperature probes)
    │
    └─ X-API-Key header
         │
    HMAC key lookup (JSON map in env var)
         │
    ─► POST /api/iot/telemetria/*
```

This separation means sensors can never access user-protected endpoints,
and users cannot access IoT-only endpoints with their JWT.

---

## CI/CD Pipeline (GitHub Actions)

Four jobs run on every push to `main`:

```
push → main
  │
  ├─ job: db          Docker pulls postgres:16, runs all 17 sql/init/ scripts,
  │                   executes sanity_check.sql (table/view/function counts)
  │
  ├─ job: api         Ruff lint + format check → pytest --cov=app
  │                   (testcontainers launches ephemeral Postgres; never uses dev DB)
  │                   Coverage threshold: 60%
  │
  ├─ job: web         ESLint → Prettier format check → Vitest → npm run build
  │
  └─ job: gitleaks    Scans full git history for leaked secrets
```

---

## Database Initialization

PostgreSQL executes all `*.sql` files in `sql/init/` alphabetically on first startup.
File names encode dependency order:

```
01_extensions.sql          → pgcrypto, uuid-ossp
02_schema.sql              → CREATE SCHEMA pasteurizadora
03_ddl_core.sql            → lote_maestro, empleado, proveedor, activo...
04_ddl_lims.sql            → recepcion_crema, analisis_laboratorio...
05_ddl_mes.sql             → orden_produccion, consumos_orden, resultados_lote...
06_ddl_ventas.sql          → cliente, pedido, despacho, factura...
07_ddl_contabilidad.sql    → cuenta_contable, asiento_contable, costeo_lote...
08_ddl_areas360.sql        → wms, sop, qms, rrhh, iot tables...
09_gatekeepers.sql         → all 23 business-rule triggers
10_views.sql               → 14 analytical views
11_functions.sql           → 35+ utility and analytical functions
...
```

Schema changes follow a dual-write convention:
1. Edit `sql/init/` file (for fresh installs and CI)
2. Create an Alembic SQL migration (for already-running development databases)

---

## Network Topology (Local LAN)

```
Router (192.168.1.1)
  │
  └─ Server PC (192.168.1.11)
        │  port 80/443 → Caddy
        │  port 5434   → PostgreSQL (internal access)
        │  port 1883   → MQTT broker
        │
  └─ iPhone / Android (same LAN)
        └─ http://192.168.1.11    (HTTP, no TLS — testing only)
        └─ https://laferme.local  (HTTPS via router DNS + Caddy internal CA)
```

For HTTPS on iOS, Caddy generates an internal CA certificate (`root.crt`) that must be
installed as a trusted root on each device. Once trusted, Safari can install the app
as a PWA via "Add to Home Screen".
