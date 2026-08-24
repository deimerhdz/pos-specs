# Contrato: `date_from`/`date_to` en `GET /sales` y `GET /reports/*`

Endpoints existentes, **sin cambio de forma** — mismos parámetros de consulta
(`date_from: date | None`, `date_to: date | None`, formato `YYYY-MM-DD`), mismos códigos de estado.
Lo único que cambia es **cómo el backend interpreta esos dos valores** al construir el filtro.

## Request — sin cambios

```
GET /sales?date_from=2026-08-24&date_to=2026-08-24
GET /reports/sales?date_from=2026-08-24&date_to=2026-08-24
```

`date_from`/`date_to` siguen siendo fechas puras (`YYYY-MM-DD`), sin componente de hora ni de zona —
el frontend no cambia qué envía (research.md Decisión 5; los `input[type=date]` de
`sales-page.component.ts` y `reports-page.component.ts` no se tocan).

## Interpretación — antes de esta spec

```python
if date_from:
    stmt = stmt.where(Sale.sold_at >= date_from)          # medianoche UTC implícita
if date_to:
    stmt = stmt.where(Sale.sold_at < date_to + timedelta(days=1))  # idem
```

`date_from`/`date_to` se comparan contra `Sale.sold_at` como si fueran medianoche **UTC** — una venta
hecha a las 23:30 hora de Bogotá del día D (04:30 UTC del día D+1) cae fuera del filtro `date_to=D`;
una venta a las 00:15 hora de Bogotá del día D+1 (05:15 UTC del mismo día D+1) puede colarse en el
filtro `date_to=D` si D+1 aún no alcanzó medianoche UTC. Historia 3 de spec.md, Escenarios 1 y 2.

## Interpretación — después de esta spec

```python
tz = resolve_timezone(tenant)
if date_from:
    start, _ = local_day_bounds_utc(date_from, tz)
    stmt = stmt.where(Sale.sold_at >= start)
if date_to:
    _, end = local_day_bounds_utc(date_to, tz)
    stmt = stmt.where(Sale.sold_at < end)
```

`date_from`/`date_to` se interpretan como días calendario en la zona horaria del **tenant**
(`America/Bogota` por defecto). El bucketing por día/mes de `reports/service.py` (`func.date_trunc`,
`func.date`) aplica el mismo criterio vía `AT TIME ZONE` doble en SQL (research.md Decisión 5), para
que los totales agrupados por día también respeten la medianoche del negocio, no la de UTC.

## Casos de prueba obligatorios (`FR-010`)

| Caso | `date_from`/`date_to` | Venta creada | Resultado esperado |
|---|---|---|---|
| Medianoche exacta, dentro | `date_to=D` | 23:59 hora de Bogotá del día D | Incluida (antes: podía excluirse) |
| Medianoche exacta, fuera | `date_to=D` | 00:01 hora de Bogotá del día D+1 | Excluida (antes: podía incluirse) |
| Rango de un día | `date_from=D`, `date_to=D` | Cualquier venta entre 00:00:00 y 23:59:59 hora de Bogotá de D | Incluida exactamente ese rango |

## Compatibilidad

Ningún parámetro cambia de nombre, tipo ni formato — un cliente que ya construye
`date_from`/`date_to` como `YYYY-MM-DD` (el único formato que estos parámetros aceptaron siempre)
sigue funcionando sin cambios. El resultado devuelto para un mismo par de fechas puede diferir de antes
de esta spec **por diseño** — es exactamente el defecto que Historia 3 corrige, no una ruptura de
compatibilidad.
