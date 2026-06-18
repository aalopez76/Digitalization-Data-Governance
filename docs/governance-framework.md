# Governance Framework — Business Rules Enforced at the Database Level

## Philosophy

Business rules that protect food safety and financial integrity should not live
in application code. A new frontend, a direct SQL query, or a future API version
could all bypass application-level checks. At La Ferme, the five core governance
rules are implemented as **PostgreSQL triggers** — they fire regardless of how
data enters the system.

This approach aligns with DCAM's Data Quality Management pillar: rules are
declared once, at the source of truth, and apply universally.

---

## The Five Gatekeepers

### 1. Batch Consumption — Segregation of Functions

**Rule:** A raw cream batch can only enter production if it is in `Released` status
AND has a conforming lab analysis signed by someone with the `Quality_Assurance` role.

**Why it matters:** An operator cannot analyze their own batch and approve it for
production. The QA analyst who signs the release must be a different person from
the one who received the cream. This mirrors Mexico's NOM food safety requirements
for chain of custody.

**What the trigger blocks:**
- Consuming a batch that is still `In_Analysis` or `Rejected`
- Consuming a batch whose analysis was signed by an `Operator` role
- The operator approving their own batch (same `employee_id`)

**Pattern:**
```sql
-- Before inserting into consumos_orden, verify:
-- 1. lote.estado = 'Liberado'
-- 2. EXISTS analysis WHERE lote_id = NEW.lote_id
--      AND resultado = 'Conforme'
--      AND firmado_por.rol = 'Calidad_Aseguramiento'
--      AND firmado_por.id != lote.recibido_por.id
-- RAISE EXCEPTION if any condition fails
```

---

### 2. CIP Validation — Equipment Cleanliness Before Production

**Rule:** A production asset (pasteurizer, separator, storage tank) can only
start a new production order if its last recorded status in the CIP log is
`Clean_Available`.

**Why it matters:** CIP (Cleaning in Place) is mandatory between production runs
under HACCP plans. An asset that was not cleaned cannot be cleared for food contact.
Enforcing this in the database means no production order can be created for a
dirty asset — even if the ERP screen does not warn the user.

**What the trigger blocks:**
- Opening an order on an asset with last CIP status `In_Cleaning`, `Pending`, or no log entry

**Pattern:**
```sql
-- Before inserting into orden_produccion:
-- SELECT estado FROM cip_log WHERE activo_id = NEW.activo_id
--   ORDER BY fecha DESC LIMIT 1
-- IF estado != 'Limpio_Disponible' THEN RAISE EXCEPTION
```

---

### 3. Dispatch — No Expired or Unreleased Batches Leave the Plant

**Rule:** A dispatch record can only be created for a finished product batch
that is in `Released` status and has not passed its expiry date.

**Why it matters:** Food safety liability. If a recall is ever needed, the database
must show that no expired or uncleared product was intentionally shipped. The trigger
creates an immutable record that the check was performed at dispatch time.

**Pattern:**
```sql
-- Before inserting into despacho:
-- IF producto.lote.estado != 'Liberado'
--   OR producto.fecha_vencimiento < CURRENT_DATE
-- THEN RAISE EXCEPTION
```

---

### 4. Invoicing — Only Delivered Orders Can Be Billed

**Rule:** An invoice can only be created for a sales order in `Delivered` status.

**Why it matters:** Cash flow accuracy. Billing an undelivered order inflates
revenue and creates customer disputes. The trigger ensures that the accounting
entry for revenue recognition only fires when the physical delivery is confirmed.

**Pattern:**
```sql
-- Before inserting into factura:
-- IF pedido.estado != 'Entregado' THEN RAISE EXCEPTION
```

---

### 5. Double-Entry Accounting — Balanced Books, Open Period Only

**Rule:** An accounting entry (journal entry) can only be posted if:
- The sum of all debit lines equals the sum of all credit lines (Debe = Haber)
- The accounting period is not closed

**Why it matters:** Mexican CFDI and SAT compliance requires balanced books.
A trigger that enforces double-entry at the row level means no unbalanced entry
can ever reach the ledger, even from a direct INSERT.

**Pattern:**
```sql
-- Before inserting into asiento_contable:
-- IF SUM(detalle.debe) != SUM(detalle.haber) THEN RAISE EXCEPTION
-- IF periodo.estado = 'Cerrado' THEN RAISE EXCEPTION
```

---

## Governance at Scale

The five triggers above cover the highest-risk operations. The full schema
also enforces governance through:

- **CHECK constraints** on all status columns (only valid states allowed)
- **NOT NULL + FK constraints** ensuring referential integrity across all 49 tables
- **Generated columns** for derived values (e.g., `costo_total = cantidad_kg * costo_por_kg`)
  that cannot be manually overridden
- **Views** that expose pre-aggregated data for reporting without granting direct table access
- **Role-based JWT authentication** limiting which API endpoints each user type can call

Together, these layers implement a lightweight but effective data governance framework
appropriate for a mid-size food manufacturing company operating under Mexican NOM and
HACCP standards.

---

## Alignment with DCAM

| DCAM Capability | Implementation |
|-----------------|---------------|
| Data Quality Management | Triggers, CHECK constraints, Pydantic v2 validation |
| Data Governance | Role-based access, segregation of duties trigger |
| Data Architecture | Normalized schema, `lote_maestro` as canonical entity |
| Metadata Management | Data dictionary, Alembic migration history |
| Data Security | bcrypt PIN hashing, JWT expiry, no wildcard CORS |
| Data Lineage & Traceability | Folio propagation from batch to invoice |
| Reference & Master Data | `empleado`, `proveedor`, `cliente`, `cuenta_contable` as master entities |
