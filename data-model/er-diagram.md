# Entity-Relationship Diagram — Core Data Model

## Main Traceability Flow

The central design pattern is `lote_maestro` as the single source of truth for every
batch of raw cream. All downstream records (lab analysis, production, sales, invoicing)
reference this entity, enabling end-to-end traceability.

```mermaid
erDiagram
    PROVEEDOR {
        uuid id PK
        string nombre
        string rfc
        string certificacion_calidad
    }

    RECEPCION_CREMA {
        uuid id PK
        uuid proveedor_id FK
        date fecha
        decimal temperatura_c
        decimal acidez_dornic
        decimal peso_bruto_kg
        decimal peso_neto_kg
        string folio "CB-YYYYMMDD-PROV-SEQ"
    }

    LOTE_MAESTRO {
        uuid id PK
        uuid recepcion_id FK
        string folio UK
        string estado "Recibido|En_Analisis|Liberado|Rechazado|Consumido|Vencido"
        uuid recibido_por FK
        date fecha_vencimiento
    }

    ANALISIS_LABORATORIO {
        uuid id PK
        uuid lote_id FK
        string resultado "Conforme|No_Conforme"
        boolean conforme
        uuid firmado_por FK
        timestamp fecha_firma
    }

    EMPLEADO {
        uuid id PK
        string codigo "EMP-XXX"
        string nombre
        string rol "Operador|Calidad_Aseguramiento|Produccion|Ventas|Administrador"
        string pin_hash "bcrypt"
    }

    ACTIVO {
        uuid id PK
        string codigo "ACT-XXX"
        string tipo "Pasteurizador|Separadora|Tanque"
        string estado_cip
    }

    CIP_LOG {
        uuid id PK
        uuid activo_id FK
        string estado "Limpio_Disponible|En_Limpieza|Pendiente"
        timestamp fecha
        uuid registrado_por FK
    }

    ORDEN_PRODUCCION {
        uuid id PK
        uuid activo_id FK
        string folio "OP-YYYYMMDD-ACT-SEQ"
        string estado
        timestamp fecha_inicio
        uuid operador_id FK
    }

    CONSUMOS_ORDEN {
        uuid id PK
        uuid orden_id FK
        uuid lote_id FK
        decimal cantidad_kg
    }

    RESULTADOS_LOTE {
        uuid id PK
        uuid orden_id FK
        string tipo "Producto_Principal|Co_Producto|Merma|Reproceso"
        decimal cantidad_kg
        string folio_pt "PT-YYYYMMDD-ORDEN"
    }

    CLIENTE {
        uuid id PK
        string nombre
        string rfc
        integer credito_dias
    }

    PEDIDO {
        uuid id PK
        uuid cliente_id FK
        string estado "Pendiente|Confirmado|Entregado|Cancelado"
        decimal total_neto
        date fecha_entrega
    }

    DESPACHO {
        uuid id PK
        uuid pedido_id FK
        uuid lote_pt_id FK
        decimal cantidad_kg
        timestamp fecha_despacho
    }

    FACTURA {
        uuid id PK
        uuid pedido_id FK
        string folio_fiscal
        decimal monto_total
        date fecha_timbrado
    }

    CUENTA_CONTABLE {
        string codigo PK "X.X.XX"
        string nombre
        string tipo "Activo|Pasivo|Capital|Ingreso|Gasto|Costo"
    }

    ASIENTO_CONTABLE {
        uuid id PK
        string folio "AST-TIPO-REF-FECHA"
        string estado "Borrador|Contabilizado"
        uuid periodo_id FK
    }

    PROVEEDOR ||--o{ RECEPCION_CREMA : provee
    RECEPCION_CREMA ||--|| LOTE_MAESTRO : genera
    LOTE_MAESTRO ||--o{ ANALISIS_LABORATORIO : tiene
    EMPLEADO ||--o{ ANALISIS_LABORATORIO : firma
    EMPLEADO ||--o{ LOTE_MAESTRO : recibe
    ACTIVO ||--o{ CIP_LOG : registra
    ACTIVO ||--o{ ORDEN_PRODUCCION : ejecuta
    LOTE_MAESTRO ||--o{ CONSUMOS_ORDEN : alimenta
    ORDEN_PRODUCCION ||--o{ CONSUMOS_ORDEN : consume
    ORDEN_PRODUCCION ||--o{ RESULTADOS_LOTE : produce
    CLIENTE ||--o{ PEDIDO : realiza
    PEDIDO ||--o{ DESPACHO : genera
    RESULTADOS_LOTE ||--o{ DESPACHO : incluye
    PEDIDO ||--|| FACTURA : origina
    FACTURA ||--o{ ASIENTO_CONTABLE : genera
    CUENTA_CONTABLE ||--o{ ASIENTO_CONTABLE : clasifica
```

---

## Extended Modules (360° Coverage)

Beyond the core traceability flow above, the schema includes six additional modules
that complete the 360° view of the business:

| Module | Key Entities | Purpose |
|--------|-------------|---------|
| **WMS** | `ubicacion`, `stock_actual`, `movimiento_inventario` | Warehouse locations, stock control, kardex |
| **S&OP** | `pronostico_demanda`, `plan_produccion` | Demand forecasting vs. actual production |
| **QMS/HACCP** | `punto_control_critico`, `monitoreo_pcc`, `desviacion_haccp` | HACCP critical control points and deviations |
| **RRHH** | `periodo_nomina`, `recibo_nomina` | Payroll periods and pay stubs with auto accounting |
| **IoT** | `telemetria_peso`, `telemetria_temperatura` | Real-time sensor readings via MQTT |
| **Contabilidad** | `costeo_lote`, `periodo_contable` | Cost absorption by batch, period close |

Total: **49 tables** across all modules, all in the `pasteurizadora` PostgreSQL schema.

---

## Folio System

Every entity that participates in traceability has a structured folio:

| Entity | Pattern | Example |
|--------|---------|---------|
| Raw cream batch | `CB-YYYYMMDD-PROV-SEQ` | `CB-20260601-0012-001` |
| Production order | `OP-YYYYMMDD-ACT-SEQ` | `OP-20260601-003-0042` |
| Finished product | `PT-YYYYMMDD-ORDEN` | `PT-20260601-0042` |
| Accounting entry | `AST-TIPO-REF-FECHA` | `AST-NOM-REC-20260615` |

These folios are generated by PostgreSQL functions at insert time, not by the application,
ensuring uniqueness and consistency regardless of the client.
