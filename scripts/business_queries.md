# Business Queries — KPI Calculations

SQL queries for the six key business metrics defined in the project scope.
All queries use the `pasteurizadora` PostgreSQL schema.

---

## 1. Production Volume (Monthly)

**KPI:** Total kilograms of finished product produced per month.

```sql
SELECT
  DATE_TRUNC('month', op.fecha_fin)::DATE AS mes,
  SUM(rl.cantidad_kg) FILTER (WHERE rl.tipo = 'Producto_Principal') AS kg_producto_principal,
  SUM(rl.cantidad_kg) FILTER (WHERE rl.tipo = 'Co_Producto')        AS kg_co_producto,
  SUM(rl.cantidad_kg) FILTER (WHERE rl.tipo = 'Merma')              AS kg_merma,
  SUM(rl.cantidad_kg)                                                AS kg_total_salida,
  SUM(co.cantidad_kg)                                                AS kg_materia_prima,
  ROUND(
    (1 - SUM(rl.cantidad_kg) FILTER (WHERE rl.tipo = 'Merma')
       / NULLIF(SUM(rl.cantidad_kg), 0)) * 100, 2
  ) AS rendimiento_pct
FROM pasteurizadora.orden_produccion op
JOIN pasteurizadora.resultados_lote  rl ON rl.orden_id = op.id
JOIN pasteurizadora.consumos_orden   co ON co.orden_id = op.id
WHERE op.estado = 'Completada'
  AND op.fecha_fin >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY 1
ORDER BY 1;
```

---

## 2. Quality Rejection Rate

**KPI:** Percentage of batches rejected by the lab — indicates raw material quality
from each supplier.

```sql
SELECT
  p.nombre AS proveedor,
  COUNT(*) AS total_lotes,
  COUNT(*) FILTER (WHERE al.resultado = 'No_Conforme') AS rechazados,
  COUNT(*) FILTER (WHERE al.resultado = 'Conforme')    AS conformes,
  ROUND(
    COUNT(*) FILTER (WHERE al.resultado = 'No_Conforme')::NUMERIC
    / NULLIF(COUNT(*), 0) * 100, 2
  ) AS tasa_rechazo_pct
FROM pasteurizadora.lote_maestro lm
JOIN pasteurizadora.recepcion_crema  rc ON rc.id = lm.recepcion_id
JOIN pasteurizadora.proveedor        p  ON p.id  = rc.proveedor_id
LEFT JOIN pasteurizadora.analisis_laboratorio al ON al.lote_id = lm.id
WHERE lm.fecha_recepcion >= CURRENT_DATE - INTERVAL '6 months'
GROUP BY p.id, p.nombre
ORDER BY tasa_rechazo_pct DESC;
```

---

## 3. Average Distribution Time

**KPI:** Average days between order placement and actual dispatch.
Measures fulfillment efficiency and cold chain responsiveness.

```sql
SELECT
  DATE_TRUNC('month', ped.fecha_pedido)::DATE AS mes,
  COUNT(d.id)                                  AS despachos,
  ROUND(AVG(
    EXTRACT(EPOCH FROM (d.fecha_despacho - ped.fecha_pedido::TIMESTAMP)) / 86400.0
  ), 1) AS dias_promedio,
  ROUND(PERCENTILE_CONT(0.9) WITHIN GROUP (
    ORDER BY EXTRACT(EPOCH FROM (d.fecha_despacho - ped.fecha_pedido::TIMESTAMP)) / 86400.0
  ), 1) AS dias_p90
FROM pasteurizadora.despacho  d
JOIN pasteurizadora.pedido   ped ON ped.id = d.pedido_id
WHERE d.fecha_despacho >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY 1
ORDER BY 1;
```

---

## 4. Monthly Sales Volume

**KPI:** Net revenue per month and per customer channel.

```sql
SELECT
  DATE_TRUNC('month', ped.fecha_pedido)::DATE AS mes,
  c.nombre                                     AS cliente,
  COUNT(ped.id)                                AS num_pedidos,
  SUM(ped.total_neto)                          AS venta_neta,
  ROUND(
    SUM(ped.total_neto) / NULLIF(COUNT(ped.id), 0), 2
  ) AS ticket_promedio
FROM pasteurizadora.pedido   ped
JOIN pasteurizadora.cliente  c ON c.id = ped.cliente_id
WHERE ped.estado IN ('Entregado', 'Confirmado')
  AND ped.fecha_pedido >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY 1, c.id, c.nombre
ORDER BY 1, venta_neta DESC;
```

---

## 5. Net Income

**KPI:** Net income from the accounting ledger — revenue accounts minus cost and
expense accounts for the current period.

```sql
WITH saldos AS (
  SELECT
    cc.tipo,
    cc.nombre,
    SUM(dd.debe  - dd.haber) AS saldo
  FROM pasteurizadora.asiento_contable  ac
  JOIN pasteurizadora.detalle_asiento   dd ON dd.asiento_id = ac.id
  JOIN pasteurizadora.cuenta_contable   cc ON cc.codigo = dd.cuenta_codigo
  JOIN pasteurizadora.periodo_contable  pc ON pc.id = ac.periodo_id
  WHERE ac.estado = 'Contabilizado'
    AND pc.anio   = EXTRACT(YEAR FROM CURRENT_DATE)
  GROUP BY cc.tipo, cc.nombre
)
SELECT
  tipo,
  nombre,
  CASE
    WHEN tipo IN ('Ingreso')       THEN -saldo  -- credit-normal accounts
    ELSE saldo
  END AS saldo_natural
FROM saldos
UNION ALL
SELECT
  'RESULTADO' AS tipo,
  'Utilidad / Pérdida Neta' AS nombre,
  SUM(CASE WHEN tipo = 'Ingreso' THEN -saldo ELSE saldo END) AS saldo_natural
FROM saldos
WHERE tipo IN ('Ingreso', 'Gasto', 'Costo')
ORDER BY tipo, nombre;
```

---

## 6. Cost per Kilogram

**KPI:** Absorption cost per kilogram of finished product, broken down by
raw material, labor, and overhead. Enables margin analysis by product type.

```sql
SELECT
  DATE_TRUNC('month', op.fecha_fin)::DATE AS mes,
  rl.tipo                                  AS tipo_producto,
  COUNT(DISTINCT op.id)                    AS ordenes,
  SUM(rl.cantidad_kg)                      AS kg_producidos,
  ROUND(AVG(cl.costo_mp_kg),          2)   AS costo_mp_promedio,
  ROUND(AVG(cl.costo_mano_obra_kg),   2)   AS costo_mo_promedio,
  ROUND(AVG(cl.costo_cif_kg),         2)   AS costo_cif_promedio,
  ROUND(AVG(cl.costo_por_kg),         2)   AS costo_total_promedio
FROM pasteurizadora.resultados_lote  rl
JOIN pasteurizadora.orden_produccion op ON op.id = rl.orden_id
JOIN pasteurizadora.costeo_lote      cl ON cl.resultado_id = rl.id
WHERE op.estado = 'Completada'
  AND op.fecha_fin >= DATE_TRUNC('year', CURRENT_DATE)
  AND rl.tipo = 'Producto_Principal'
GROUP BY 1, rl.tipo
ORDER BY 1;
```

---

## Usage Notes

- All queries target the `pasteurizadora` PostgreSQL schema.
- Replace `CURRENT_DATE` / `DATE_TRUNC('year', ...)` with a date parameter
  when embedding in a BI tool or Streamlit dashboard.
- Views in the schema (e.g., `v_estado_resultados`, `v_trazabilidad_lote`) provide
  pre-joined datasets that simplify the joins shown above.
- For high-volume production reporting, indexes on `fecha_recepcion`, `fecha_fin`,
  and `fecha_pedido` are already in place.
