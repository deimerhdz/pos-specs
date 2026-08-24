# Contrato: Activación y Configuración de Métodos de Pago (Tenant Admin)

Extiende `app/api/v1/sales/router.py` (mismo prefijo `/sales`, ya usado hoy por
`payment-methods`). Cubre **Historia de Usuario 2** (`spec.md`).

## `GET /sales/payment-methods/catalog`

Nuevo. Lista el catálogo **activo a nivel plataforma** (FR-005), para que el Tenant Admin elija qué
activar. A diferencia de `GET /super-admin/payment-methods-catalog`, no incluye entradas
`active=false` — salvo que el propio tenant ya las tenga activadas (FR-006), en cuyo caso se
incluyen marcadas explícitamente para que la pantalla del tenant pueda avisar "ya no disponible".

**Auth**: `get_current_user` (cualquier usuario autenticado del tenant; el formulario de activación
en sí queda protegido en el frontend a `UserRole.ADMIN` vía `role.guard.ts`, igual que hoy la
página `payment-methods-page.component.ts`).

**Response 200** — `list[CatalogPaymentMethodOption]`:

```json
[
  {
    "id": "uuid-catalogo",
    "name": "Nequi",
    "fields": [
      {"key": "celular", "label": "Número de celular", "required": true, "format": "numeric", "length": 10},
      {"key": "qr", "label": "Código QR", "required": false, "format": "image"}
    ],
    "active": true,
    "already_activated": false
  }
]
```

`already_activated` le dice al frontend si debe ofrecer "activar" o "editar" para esa fila. `active`
es el estado del catálogo a nivel plataforma — para una fila con `already_activated: true` y
`active: false`, el frontend muestra "ya no disponible" (FR-006); el resto de filas devueltas
siempre tiene `active: true` (el listado ya excluye el catálogo inactivo salvo por esta excepción).

## `POST /sales/payment-methods`

**Modificado.** Antes: `{name, type, is_cash, payment_info}` con `name` libre. Ahora: requiere
`catalog_id`; `name`/`type`/`is_cash` se copian del catálogo (research.md Decisión 5) y ya no se
aceptan en el body (FR-007/FR-011 — un tenant no puede crear métodos fuera del catálogo).

Crea una fila **solo la primera vez** que el tenant activa ese `catalog_id`. Si ya existe una fila
para ese `catalog_id` (activa o inactiva — el tenant lo activó antes y luego lo desactivó), este
endpoint responde `409`: la reactivación y el cambio de datos de integración se hacen con `PATCH
/sales/payment-methods/{id}` sobre esa misma fila, no creando una nueva (research.md Decisión 9 —
`catalog_id` es único por tenant, para siempre; así se conserva el `payment_info` ya capturado, edge
case de `spec.md`).

**Auth**: `require_tenant_admin` (sin cambios).

**Request** — `PaymentMethodCreate`:

```json
{
  "catalog_id": "uuid-catalogo",
  "payment_info": {"celular": "3001234567"}
}
```

`payment_info` es opcional en el body (un método sin campos, como Efectivo, no manda nada); si se
manda, se valida contra `catalog.fields` del `catalog_id` indicado (formato, no solo presencia —
clarificación 2026-08-24 #3).

**Response 201** — `PaymentMethodResponse`:

```json
{
  "id": "uuid",
  "catalog_id": "uuid-catalogo",
  "name": "Nequi",
  "type": "transfer",
  "is_cash": false,
  "active": true,
  "is_complete": false,
  "payment_info": {"celular": "3001234567"}
}
```

`is_complete=false` en este ejemplo porque falta el campo obligatorio... en este caso si `celular`
ya viene completo y es el único obligatorio, `is_complete=true`; el ejemplo asume un campo
obligatorio adicional sin diligenciar para ilustrar el caso "incompleto" (FR-009).

**Errores**:
- `404` — `catalog_id` no existe.
- `409` — `catalog.active=false` (no se puede activar un método desactivado a nivel plataforma), o
  el tenant ya tiene una fila (activa o no) para ese `catalog_id` — use `PATCH` (FR-017).
- `422` — `payment_info` no cumple el formato de algún campo de `catalog.fields` (ver
  data-model.md).

## `PATCH /sales/payment-methods/{id}`

**Modificado.** Ya no acepta `name` (deja de tener sentido — el nombre viene del catálogo). Sigue
aceptando `payment_info` (recalcula `is_complete`, FR-008/FR-009) y `active` (FR-010, activar
recuperando datos previos / desactivar).

**Auth**: `require_tenant_admin` (sin cambios).

**Request** — `PaymentMethodUpdate`:

```json
{ "payment_info": {"celular": "3009876543"}, "active": true }
```

Este es el único camino para reactivar un método que el tenant había desactivado (FR-017): al ser
`catalog_id` único por tenant (research.md Decisión 9), no existe ninguna "otra" fila con la que
pueda chocar — reactivar es simplemente `PATCH {"active": true}` sobre la fila ya existente,
opcionalmente junto con `payment_info` nuevo.

**Response 200** — `PaymentMethodResponse` actualizado (mismo shape que el `POST`).

**Errores**:
- `404` — no existe esa fila para este tenant.
- `422` — `payment_info` no cumple el formato de `catalog.fields`.

Nota: el guardia existente "al menos un método activo debe quedar" (`update_payment_method`,
`sales/service.py`) se mantiene sin cambios — sigue aplicando igual con el modelo nuevo.
