# Contrato: Extensión del Bloqueo por Módulo Inventario a Unidades de Medida y Reportes

Cubre las tres historias de usuario de esta spec. No introduce ningún endpoint, request ni
response nuevo — extiende el mismo mecanismo `require_module_access("inventario")` de spec 033
(contrato `tenant-plan-enforcement.md` de esa spec, sección "Bloqueo por acceso a módulo") a
endpoints que hoy no lo tienen. La forma de respuesta exitosa de cada endpoint no cambia en
absoluto; solo se agrega la posibilidad de una respuesta `403` nueva, con el mismo texto ya
definido por spec 033.

## Unidades de Medida (`GET/POST/PATCH/DELETE /unit-measures*`) — Historia de Usuario 1

Aplica a los 5 endpoints existentes de `unit_measures/router.py` (`GET ""`, `GET "/{id}"`,
`POST ""`, `PATCH "/{id}"`, `DELETE "/{id}"`), sin excepción — incluyendo los de solo lectura,
mismo criterio que Inventario/Promociones en spec 033 (el módulo queda completamente
inaccesible, no en modo de solo lectura).

**Response 403** (nueva, cuando el plan vigente del tenant no incluye `inventario_access`, o está
vencido):

```json
{ "detail": "Tu plan actual no incluye el módulo de inventario." }
```

(o, si el tenant está vencido: `{ "detail": "Tu plan venció. Debe renovarse para seguir usando
el sistema." }` — mismo texto exacto que spec 033, `ensure_plan_not_expired`, evaluado antes que
el acceso al módulo específico).

**Sin bloqueo** cuando `Plan.inventario_access = true` y el tenant no está vencido — respuesta
idéntica a la actual, sin ningún campo nuevo (FR-004 de esta spec: las unidades de medida ya
creadas siguen intactas, visibles y editables).

## Reporte de insumos con stock bajo (`GET /reports/inventory`, RF-063) — Historia de Usuario 2

**Response 403** (nueva, mismo criterio y mismo texto que la sección anterior). El resto de
`reports/router.py` (`sales`, `top-products`, `products`, `categories`, `cashiers`) no cambia —
siguen respondiendo `200` sin ninguna comprobación de módulo nueva (Assumptions de `spec.md`).

**Sin bloqueo** cuando el módulo está incluido — respuesta idéntica a la actual (lista de
`InventoryRow`, sin cambios de forma).

## Reporte de rentabilidad (`GET /reports/profitability`, RF-065) — Historia de Usuario 3

**Response 403** (nueva, mismo criterio y mismo texto). No se introduce un campo ni un valor
especial para "sin datos de inventario" (ej. `margin: null`) — la decisión tomada en `spec.md`
(FR-007, Clarifications de `/speckit-specify`) es ocultar la tarjeta por completo en el frontend,
así que el backend simplemente no permite la petición en absoluto para un tenant sin el módulo;
no hay ninguna respuesta parcial o degradada que documentar.

**Sin bloqueo** cuando el módulo está incluido — respuesta idéntica a la actual
(`ProfitabilityReport`, incluyendo `by_category`, sin cambios de forma).

## Contrato del lado del cliente (frontend) — no HTTP, pero parte del comportamiento observable

Para las tres historias, el frontend **no debe** producir el 403 de arriba como efecto observable
normal — es una barrera de defensa, no el mecanismo primario de ocultamiento:

- Unidades de Medida: la ruta `ajustes/unidades` deniega la navegación antes de que el componente
  se monte y dispare ninguna petición (`planModuleGuard('inventario')`, redirección a
  `/dashboard` — Clarifications de esta spec).
- Insumos con stock bajo / Margen: `ReportsService` no dispara `GET /reports/inventory` ni
  `GET /reports/profitability` en absoluto para un tenant sin el módulo (`enabled:
  inventarioIncluido()`, research.md Decisión 4) — el 403 de esta sección solo se observaría si
  alguien llama a esos endpoints por fuera de la aplicación (ej. directamente por HTTP), que es
  exactamente el caso que este contrato existe para cubrir (FR-006).
