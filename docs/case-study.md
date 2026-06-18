# Case Study: Digitalization of La Ferme — Pasteurizadora Digital 360°

## Background

La Ferme is a Mexican artisanal dairy company producing ultra-pasteurized cow cream since
1980. For four decades, operations ran entirely on paper: production batches were tracked
in handwritten notebooks, quality results in physical binders, and sales in Excel
spreadsheets. This produced data that was fragmented, inconsistent, and impossible to
audit across departments.

In 2020, the company decided to modernize — not just digitize — its operations. The goal
was end-to-end traceability: from raw cream reception through pasteurization, finished
product, dispatch, and invoicing, all governed by food safety rules that could be
enforced automatically and audited at any time.

---

## Diagnosis

An initial audit of the existing data landscape revealed a strong conceptual foundation
(the team understood their processes well) but severe technical debt:

| Area | Finding |
|------|---------|
| **Data storage** | Excel files and handwritten notebooks with no relational structure |
| **Traceability** | Impossible end-to-end; no link between supplier batch and dispatch |
| **Quality records** | Physical signatures only; no digital chain of custody |
| **Versioning** | No version control; changes were irreversible overwrites |
| **Reproducibility** | Docker Compose referenced absolute paths that did not exist |
| **Secrets** | Credentials hardcoded in configuration files |
| **Testing** | 2 test files with no isolation from production database |
| **CI/CD** | None |

**Verdict:** Excellent conceptual model, but technical maturity far behind. Priorities
were reproducibility, version control, and secret management before adding features.

---

## Solution Design

### Architecture

A three-tier system built for on-premise deployment at the plant's local network:

```
 iOS Safari / Android APK
         │
    Caddy (TLS)  ─── Static SPA (React PWA)
         │
    FastAPI API (:8000)
         │ JWT / IoT API Key
    PostgreSQL 16
         ▲
    MQTT Broker ◄── IoT sensors (weight, temperature)
    MQTT Worker
```

All services containerized with Docker Compose. The system runs on a single server
exposed to the local network only — no public internet access required.

### Central Design Decision: Business Rules at the Database Level

All critical business rules are implemented as **PostgreSQL triggers** — not in the
API or frontend. This means:

- Rules apply regardless of which client (API, direct SQL, future integrations) accesses the data
- Rules cannot be bypassed by application bugs or intentional circumvention
- Every violation is logged and auditable
- Adding a new frontend or API does not replicate business logic

This pattern — called "gatekeepers" — enforces five core rules at the database level
(see [Governance Framework](governance-framework.md)).

### Traceability Pattern

Every batch of raw cream receives a unique folio: `CB-YYYYMMDD-PROV-SEQ`.
This folio propagates through every downstream record:

```
CB-20260601-0012-001  ← raw cream batch (lote_maestro)
  └─ LA-20260601-001  ← lab analysis (LIMS)
       └─ OP-20260601-003-0042  ← production order (MES)
            └─ PT-20260601-0042  ← finished product
                 └─ PED-2026-00234  ← sales order
                      └─ FAC-2026-00234  ← invoice
```

A quality manager can trace any commercial complaint back to the exact batch of raw
cream, its supplier, its analysis results, the operator who processed it, and the
equipment used — in seconds.

---

## Key Technical Decisions

| Decision | Rationale |
|----------|-----------|
| PostgreSQL 16 | Best-in-class for relational integrity, triggers, and HACCP-level auditability |
| FastAPI + Pydantic v2 | Type safety end-to-end; automatic OpenAPI docs for field teams |
| React 18 + Vite PWA | Single codebase for web and mobile (iOS "Add to Home Screen") |
| Capacitor 8 (Android) | Wraps the same React app in a native APK; no separate mobile codebase |
| MQTT + Mosquitto | Lightweight IoT protocol; works on plant-floor hardware with no SDK |
| Caddy reverse proxy | Automatic TLS (internal CA); single entrypoint for API + static files |
| Testcontainers (pytest) | Ephemeral PostgreSQL per test run; CI never touches the dev database |
| bcrypt (direct, no passlib) | Eliminated fragile dependency; consistent with bcrypt ≥ 4.1 |
| Alembic SQL-first migrations | Schema changes tracked in both `sql/init/` and migration files |

---

## Results

After full implementation, the system covers **8 operational modules**:

| Module | Description |
|--------|-------------|
| **LIMS** | Lab analysis with digital QA signature and segregation of duties |
| **MES** | Production orders linked to raw cream batches |
| **WMS** | Warehouse locations, inventory movements, stock kardex |
| **S&OP** | Demand forecasting and production planning vs. actual |
| **QMS/HACCP** | Critical control points monitoring, deviations, corrective actions |
| **RRHH** | Payroll periods, pay stubs, automatic accounting entries |
| **IoT** | Real-time MQTT telemetry (weight, temperature) → PostgreSQL |
| **Accounting** | Double-entry bookkeeping enforced at DB level, cost absorption by batch |

### Technical Metrics

| Metric | Value |
|--------|-------|
| Database tables | 49 |
| Database views | 14 |
| PostgreSQL functions | 35+ |
| Business-rule triggers | 23 |
| API test suite | 43 tests |
| Test coverage | 75% |
| Frontend smoke tests | 10 |
| Docker services | 6 (db, api, mqtt, worker, caddy + frontend static) |
| CI jobs | 4 (db-init, api-pytest, web-vitest, gitleaks) |
| Mobile deliverables | Android APK (3.2 MB release, RSA 2048) + iOS PWA |

---

## Technologies Used

| Layer | Technology | Version |
|-------|-----------|---------|
| Database | PostgreSQL | 16 |
| ORM | SQLAlchemy | 2.x |
| Validation | Pydantic | v2 |
| API framework | FastAPI | latest |
| Auth | JWT (users) + HMAC API Key (IoT) | — |
| Frontend | React + Vite + TypeScript + Tailwind | 18 / 5 |
| Mobile | Capacitor | 8 |
| IoT broker | Eclipse Mosquitto | 2 |
| Reverse proxy | Caddy | 2 |
| Containerization | Docker Compose | v2 |
| Testing (API) | pytest + testcontainers | — |
| Testing (web) | Vitest + Testing Library | — |
| Linting | Ruff (Python) + ESLint (TS) | — |
| CI/CD | GitHub Actions | — |
| Secret scanning | Gitleaks | — |
