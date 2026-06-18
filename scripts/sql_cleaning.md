# SQL Cleaning & Validation Scripts

Representative examples of the data validation rules implemented in the system.
These patterns are enforced at the PostgreSQL level — they apply to every write
operation regardless of the client.

---

## 1. Domain Validation via CHECK Constraints

**Purpose:** Prevent invalid state values from ever entering the database.
The status columns in `lote_maestro` and `orden_produccion` only accept
a defined set of values.

```sql
-- Batch status: only valid states allowed
ALTER TABLE pasteurizadora.lote_maestro
  ADD CONSTRAINT chk_lote_estado
  CHECK (estado IN (
    'Recibido',
    'En_Analisis',
    'Liberado',
    'Rechazado',
    'Consumido',
    'Vencido'
  ));

-- Production order status
ALTER TABLE pasteurizadora.orden_produccion
  ADD CONSTRAINT chk_orden_estado
  CHECK (estado IN (
    'Planificada',
    'En_Proceso',
    'Completada',
    'Cancelada'
  ));
```

**Result:** Any INSERT or UPDATE with an unlisted value raises:
`ERROR: new row for relation "lote_maestro" violates check constraint "chk_lote_estado"`

---

## 2. Physical Quality Range Validation (HACCP)

**Purpose:** Enforce HACCP critical limits at the moment of data entry.
A reception record with temperature or acidity outside safe bounds is rejected.

```sql
-- Temperature and acidity ranges from NOM-155-SCFI-2012 (cream)
ALTER TABLE pasteurizadora.recepcion_crema
  ADD CONSTRAINT chk_temperatura_recepcion
  CHECK (temperatura_c BETWEEN 1.0 AND 8.0),  -- cold chain limit

  ADD CONSTRAINT chk_acidez_recepcion
  CHECK (acidez_dornic BETWEEN 13.0 AND 22.0);  -- acceptable Dornic range
```

**Result:** A truck arriving with cream at 12°C cannot be logged — the INSERT fails
before any business logic runs. No application bug can create an invalid reception record.

---

## 3. Folio Format Validation

**Purpose:** Ensure all batch folios follow the traceability naming convention.
A folio that does not match `CB-YYYYMMDD-NNNN-NNN` cannot serve as a search key
in recall scenarios.

```sql
-- Validate folio format on lote_maestro
ALTER TABLE pasteurizadora.lote_maestro
  ADD CONSTRAINT chk_folio_formato
  CHECK (folio ~ '^CB-[0-9]{8}-[0-9]{4}-[0-9]{3}$');

-- Validate production order folio format
ALTER TABLE pasteurizadora.orden_produccion
  ADD CONSTRAINT chk_folio_op_formato
  CHECK (folio ~ '^OP-[0-9]{8}-[0-9]{3}-[0-9]{4}$');
```

---

## 4. Segregation of Functions — Trigger Pattern

**Purpose:** Prevent a person from approving their own work (a common food safety
audit finding). The analyst who signs a lab result must be different from the
operator who received the batch.

```sql
CREATE OR REPLACE FUNCTION pasteurizadora.check_segregacion_funciones()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
  v_recibido_por UUID;
BEGIN
  -- Get who received this batch
  SELECT recibido_por
    INTO v_recibido_por
    FROM pasteurizadora.lote_maestro
   WHERE id = NEW.lote_id;

  -- Block self-approval
  IF NEW.firmado_por = v_recibido_por THEN
    RAISE EXCEPTION
      'Segregation of functions violation: the analyst (%) cannot sign '
      'the analysis for a batch they received.',
      NEW.firmado_por
      USING ERRCODE = 'integrity_constraint_violation';
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_segregacion_funciones
  BEFORE INSERT ON pasteurizadora.analisis_laboratorio
  FOR EACH ROW EXECUTE FUNCTION pasteurizadora.check_segregacion_funciones();
```

**Result:** Attempting to sign your own batch raises HTTP 409 Conflict at the API level
(the trigger fires before the row is committed, the exception propagates to SQLAlchemy,
which the API converts to a structured error response).

---

## 5. Double-Entry Bookkeeping Validation

**Purpose:** No accounting entry can be posted unless debits equal credits.
This is enforced at the database level so it applies to API calls, scheduled jobs,
and any future integration.

```sql
CREATE OR REPLACE FUNCTION pasteurizadora.check_partida_doble()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
  v_total_debe  NUMERIC;
  v_total_haber NUMERIC;
BEGIN
  -- Only validate when transitioning to 'Contabilizado'
  IF NEW.estado = 'Contabilizado' AND OLD.estado = 'Borrador' THEN

    SELECT COALESCE(SUM(debe), 0), COALESCE(SUM(haber), 0)
      INTO v_total_debe, v_total_haber
      FROM pasteurizadora.detalle_asiento
     WHERE asiento_id = NEW.id;

    IF v_total_debe != v_total_haber THEN
      RAISE EXCEPTION
        'Double-entry violation: debit (%) != credit (%) for entry %.',
        v_total_debe, v_total_haber, NEW.folio
        USING ERRCODE = 'integrity_constraint_violation';
    END IF;

  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_partida_doble
  BEFORE UPDATE ON pasteurizadora.asiento_contable
  FOR EACH ROW EXECUTE FUNCTION pasteurizadora.check_partida_doble();
```

---

## 6. Cleaning Utility Queries

**Purpose:** Ad-hoc queries to identify data quality issues during initial
digitization of historical records.

```sql
-- Identify batches with no lab analysis (orphaned batches)
SELECT lm.folio, lm.fecha_recepcion, lm.estado
  FROM pasteurizadora.lote_maestro lm
 WHERE NOT EXISTS (
   SELECT 1 FROM pasteurizadora.analisis_laboratorio al
    WHERE al.lote_id = lm.id
 )
   AND lm.estado NOT IN ('Recibido')  -- expected to have no analysis yet
ORDER BY lm.fecha_recepcion DESC;

-- Find production orders with unbalanced mass (output > input by > 2%)
SELECT op.folio,
       SUM(co.cantidad_kg) AS input_kg,
       SUM(rl.cantidad_kg) AS output_kg,
       ROUND(
         (SUM(rl.cantidad_kg) - SUM(co.cantidad_kg)) / SUM(co.cantidad_kg) * 100,
         2
       ) AS balance_pct
  FROM pasteurizadora.orden_produccion op
  JOIN pasteurizadora.consumos_orden co ON co.orden_id = op.id
  JOIN pasteurizadora.resultados_lote rl ON rl.orden_id = op.id
 WHERE op.estado = 'Completada'
 GROUP BY op.id, op.folio
HAVING ABS(
         (SUM(rl.cantidad_kg) - SUM(co.cantidad_kg)) / SUM(co.cantidad_kg)
       ) > 0.02
ORDER BY balance_pct DESC;
```
