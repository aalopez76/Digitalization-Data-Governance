# Digitalization & Data Governance — La Ferme

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-009688?logo=fastapi&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Capacitor](https://img.shields.io/badge/Capacitor-8-119EFF?logo=capacitor&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

End-to-end digitalization and data governance initiative for *La Ferme*, a Mexican
ultra-pasteurization dairy company operating since 1980.

---

## Project Outcomes

The implemented system covers the full production lifecycle — from raw cream reception
through pasteurization, finished product, dispatch, invoicing, and accounting:

| Area | Result |
|------|--------|
| Database | 49 tables · 14 views · 35+ functions · **23 business-rule triggers** |
| API | FastAPI — 43 tests · 75% coverage · JWT + IoT API Key auth |
| Frontend | React 18 PWA (installable on iOS) + Android APK via Capacitor 8 |
| Infrastructure | 6 Docker services · GitHub Actions CI (4 jobs) · Gitleaks secret scanning |
| Modules | LIMS · MES · WMS · S&OP · QMS/HACCP · RRHH · IoT · Accounting (8 total) |

---

## Project Background

*La Ferme* is a Mexican company producing artisanal pasteurized cow cream since 1980.
For four decades, operations ran entirely on paper. In 2020, the company began a digital
transformation to improve traceability, operational efficiency, and data quality.

**The core challenge:** connect every cream batch from supplier to shelf, with food safety
rules enforced automatically and auditably at the database level.

Read the full project narrative: [Case Study](docs/case-study.md)

---

## Key Business Metrics

Six KPIs now tracked automatically from operational data:

| Metric | Description |
|--------|-------------|
| Production volume | Kilograms of finished product per month, by output type |
| Quality rejection rate | % of batches rejected by lab, by supplier |
| Average distribution time | Days from order placement to dispatch |
| Monthly sales volume | Net revenue per customer and channel |
| Net income | Ledger-sourced P&L from double-entry accounting |
| Cost per liter | Absorption cost by batch: raw material + labor + overhead |

SQL queries for all six KPIs: [Business Queries](scripts/business_queries.md)

---

## Key Areas: Insights and Recommendations

### 1. Traceability & Governance
- Every batch gets a structured folio (`CB-YYYYMMDD-PROV-SEQ`) that propagates to production, dispatch, and invoice
- 5 business rules enforced as PostgreSQL triggers (gatekeepers): see [Governance Framework](docs/governance-framework.md)
- Data catalog: [Data Dictionary](data-model/data-dictionary.md)

### 2. Production & Quality
- IoT sensors (weight, temperature) stream via MQTT → worker → PostgreSQL in real time
- HACCP critical control points monitored digitally with deviation tracking and corrective actions
- Predictive quality control: rejection rate trend by supplier (available via [Business Queries](scripts/business_queries.md))

### 3. Sales & Channels
- Demand forecasting per channel (S&OP module: plan vs. actual)
- Sales contribution analysis by customer and product type

### 4. Efficiency & Cost
- Cost absorption by batch — cost per kg available at every production run
- Route and fulfillment optimization tracked via distribution time KPI

---

## Technical Resources

- [SQL Cleaning & Validation Scripts](scripts/sql_cleaning.md) ✅
- [Business KPI Queries](scripts/business_queries.md) ✅
- [Entity-Relationship Diagram](data-model/er-diagram.md) ✅
- [Data Dictionary](data-model/data-dictionary.md) ✅
- [System Architecture](docs/architecture.md) ✅
- [Governance Framework](docs/governance-framework.md) ✅
- [Case Study](docs/case-study.md) ✅
- Streamlit / BI Dashboard — *roadmap*

---

## Documentation

| Document | Description |
|----------|-------------|
| [Case Study](docs/case-study.md) | Full project narrative: problem, solution, decisions, results |
| [Architecture](docs/architecture.md) | System design, Docker services, CI/CD, network topology |
| [Governance Framework](docs/governance-framework.md) | 5 business rules enforced as DB triggers, DCAM alignment |
| [ER Diagram](data-model/er-diagram.md) | Core entity relationships (Mermaid) |
| [Data Dictionary](data-model/data-dictionary.md) | 15 key entities with fields and constraints |
| [SQL Validation](scripts/sql_cleaning.md) | CHECK constraints, triggers, cleaning queries |
| [KPI Queries](scripts/business_queries.md) | SQL for all 6 business metrics |

---

## More Info

[Full project narrative and strategic framework](https://aalopez76.github.io/projects/D&G_project/)
