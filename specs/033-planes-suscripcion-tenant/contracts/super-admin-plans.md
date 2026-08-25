# Contrato: Administración de Planes y Asignación a Tenants (Super Admin)

Router nuevo `app/api/v1/super_admin/plans_router.py`, montado bajo
`app/api/v1/super_admin/router.py` (hereda `dependencies=[Depends(get_current_super_admin)]` del
router padre). Cubre **Historia de Usuario 1**. El cambio de plan de un tenant (Historia de
Usuario 2) se agrega directamente en `super_admin/router.py` (research.md Decisión 9).

## `GET /super-admin/plans`

Lista el catálogo completo de planes.

**Response 200** — `list[PlanResponse]`:

```json
[
  {
    "id": "uuid",
    "name": "Básico",
    "description": "Para negocios pequeños que están empezando.",
    "mesas_limit": 5,
    "cajas_limit": 1,
    "usuarios_limit": 3,
    "productos_limit": 50,
    "metodos_pago_activos_limit": 2,
    "inventario_access": false,
    "compras_access": false,
    "promociones_access": false,
    "precio_mensual": "35000.00",
    "precio_anual": "350000.00",
    "created_at": "2026-08-24T00:00:00Z",
    "updated_at": "2026-08-24T00:00:00Z"
  }
]
```

`mesas_limit`/etc. es `null` cuando esa característica está marcada como "ilimitado" (FR-007).
`precio_mensual`/`precio_anual` es `null` cuando ese ciclo no tiene precio definido para este plan
(FR-016) — un tenant no puede elegir ese ciclo para este plan mientras siga `null` (FR-017, ver
`PATCH /super-admin/tenants/{id}` más abajo).

## `POST /super-admin/plans`

Crea un plan (FR-001). `name` único a nivel de plataforma → `409` si ya existe.

**Request** — `PlanCreate`:

```json
{
  "name": "Pro",
  "description": "Para negocios en crecimiento.",
  "mesas_limit": 15,
  "cajas_limit": 3,
  "usuarios_limit": 10,
  "productos_limit": 300,
  "metodos_pago_activos_limit": null,
  "inventario_access": true,
  "compras_access": true,
  "promociones_access": true,
  "precio_mensual": 89000.00,
  "precio_anual": 890000.00
}
```

Todos los campos de características son opcionales en el request (FR-001: "un plan no está
obligado a definir explícitamente las 8 características"); un campo omitido se guarda con el
default de columna (`0` para límites, `false` para accesos — bloqueado, FR-002). `null` explícito
en un campo de límite se guarda como ilimitado (distinto de omitirlo). `precio_mensual`/
`precio_anual` son igualmente opcionales e independientes entre sí (FR-016) — un plan puede
definir solo uno de los dos, ambos, o ninguno (un plan sin ningún precio simplemente no puede
asignarse con ciclo mensual ni anual, solo con "sin vencimiento", FR-017/FR-021).

**Response 201** — `PlanResponse` (mismo shape que el `GET`).

**Errores**:
- `409` — `name` ya existe.
- `422` — algún `*_limit` es un entero negativo, o algún `precio_*` es negativo o tiene más de 2
  decimales.

## `PATCH /super-admin/plans/{id}`

Edita cualquier subconjunto de campos de un plan existente (FR-001).

**Request** — `PlanUpdate` (todos los campos opcionales):

```json
{ "mesas_limit": 8 }
```

**Response 200** — `PlanResponse` actualizado. El cambio aplica de inmediato a todos los tenants
con este `plan_id` — no requiere ninguna acción adicional sobre cada tenant (FR-014, SC-005: se
refleja en el siguiente intento de creación o acceso).

**Errores**:
- `404` — no existe ese plan.
- `409` — el nuevo `name` colisiona con otro plan.
- `422` — mismas validaciones de `POST`.

No existe `DELETE /super-admin/plans/{id}` mientras el plan tenga al menos un tenant asignado — la
FK `Tenant.plan_id NOT NULL` sin cascada rechaza el borrado (data-model.md); no se expone un
endpoint de borrado en el alcance de esta spec (Assumptions de `spec.md`: "el sistema simplemente
no permite eliminar planes en uso").

---

## `PATCH /super-admin/tenants/{id}` (agregado en `super_admin/router.py`)

Reasigna el plan vigente de un tenant, cambia a otro plan, **o renueva el mismo plan** — las tres
son la misma operación de datos (FR-010/FR-017/FR-018/FR-020, Historia de Usuario 2, research.md
Decisión 16): siempre recalcula `plan_iniciado_en`/`plan_vence_en` desde el momento de la
solicitud, sea o no el mismo `plan_id` que el tenant ya tenía.

**Request** — `TenantPlanUpdate`:

```json
{ "plan_id": "uuid-del-plan-nuevo", "ciclo_facturacion": "anual" }
```

Renovar el mismo plan por otro período: mismo `plan_id`, mismo o distinto `ciclo_facturacion`.
Asignar "sin vencimiento" (ej. un tenant de uso interno): `"ciclo_facturacion": null`. Ambos
campos son obligatorios en el request — no hay valor por defecto para `ciclo_facturacion`
(research.md Decisión 15); omitirlo es un `422`, no "deja el ciclo como estaba".

**Response 200** — `TenantResponse` (gana `plan_id`/`plan_name`/`ciclo_facturacion`/
`plan_vence_en`, pierde el campo `plan` de texto libre heredado — ver data-model.md).

**Efecto**: el cambio rige de inmediato (FR-010). Si el nuevo plan tiene un límite numérico menor
al que el tenant ya tiene en uso, los recursos existentes se conservan sin cambios — el tenant
solo queda bloqueado para crear recursos nuevos de ese tipo hasta bajar del límite (FR-011,
Acceptance Scenario 3 de Historia de Usuario 2). Si el nuevo plan no incluye un módulo que el
tenant tenía activo, ese módulo queda completamente inaccesible sin borrar sus datos (FR-012). Si
el tenant estaba bloqueado por vencimiento, esta operación lo levanta de inmediato — no requiere
ninguna acción del Tenant Admin (FR-020, Acceptance Scenario 5 de Historia de Usuario 5). Ninguno
de estos efectos requiere lógica adicional en este endpoint — todos son consecuencia directa de
que las validaciones (`enforce_plan_limit`/`require_module_access`/`ensure_plan_not_expired`)
siempre leen `Tenant`/`Plan` en el momento de cada solicitud, nunca un valor cacheado.

**Errores**:
- `404` — no existe ese tenant, o `plan_id` no referencia un plan existente (mensaje distingue
  ambos casos).
- `409` — `ciclo_facturacion: "mensual"` sobre un plan con `precio_mensual IS NULL` (o `"anual"`
  con `precio_anual IS NULL`) (FR-017, Acceptance Scenario 6 de Historia de Usuario 2).
- `422` — falta `ciclo_facturacion` en el body.

---

## `POST /admin/tenants` (modificado — `TenantCreateWithUser` gana `plan_id`/`ciclo_facturacion`)

Endpoint ya existente (`app/api/v1/admin/router.py`), fuera de `super_admin/router.py` mismo. Gana
dos campos obligatorios nuevos.

**Request** — `TenantCreateWithUser` (campos nuevos resaltados):

```json
{
  "tenant_name": "Heladería Central",
  "schema_name": "heladeria_central",
  "host": "heladeria-central",
  "name": "María Pérez",
  "email": "maria@heladeriacentral.com",
  "plan_id": "uuid-de-un-plan-existente",
  "ciclo_facturacion": "mensual"
}
```

`plan_id` y `ciclo_facturacion` son ambos obligatorios (FR-004/FR-017) — el request falla con
`422` si se omite cualquiera de los dos (validación Pydantic, antes de llegar a `tenant_create()`;
`ciclo_facturacion` acepta explícitamente `null` como "sin vencimiento", research.md Decisión 15),
y `tenant_create()` falla con rollback completo de la transacción si `plan_id` no referencia un
`Plan` existente, o si `ciclo_facturacion` no tiene precio definido en ese plan (misma familia de
error que el `schema`/`host` duplicados hoy) — nunca se persiste un tenant sin plan, ni siquiera
transitoriamente (Acceptance Scenario 1 de Historia de Usuario 2, edge case de `spec.md`).
`tenant_create()` calcula `plan_iniciado_en = utc_now()` y `plan_vence_en` a partir de
`ciclo_facturacion` (o deja ambas en `NULL` si `ciclo_facturacion` es `null`) dentro de la misma
transacción.

**Response**: sin cambios de forma (`{"status": "ok", "tenant": ...}`).

**Errores nuevos**:
- `422` — falta `plan_id` o `ciclo_facturacion`.
- `404` (o `400`, a decidir en implementación siguiendo el estilo ya usado por `tenant_create` para
  errores de referencia) — `plan_id` no corresponde a ningún plan existente.
- `409` — `ciclo_facturacion` elegido sin precio definido en ese plan (FR-017).
