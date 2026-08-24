# Fase 1 — Data Model: Catálogo de Métodos de Pago Administrado por el Super Admin

Ver decisiones técnicas detalladas en [research.md](./research.md). Este documento describe las
entidades, campos, validaciones y migraciones — no el DDL completo (eso pertenece a la
implementación).

## Entidades

### `PaymentMethodCatalog` (nueva — esquema `shared`, tabla `payment_method_catalog`)

Corresponde a "Método de Pago (Catálogo)" en `spec.md`. Administrada exclusivamente por el Super
Admin (FR-001/FR-002/FR-003).

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID (PK) | |
| `name` | `String(100)`, único | Ej. "Efectivo", "Nequi", "Transferencia Bancolombia", "Daviplata" |
| `type` | `String(20)`, CHECK `IN ('cash','card','transfer','other')` | Reutiliza la misma clasificación que ya usa el arqueo (`app/models/payment.py`) — se copia al `PaymentMethod` del tenant al activarse (research.md Decisión 5) |
| `active` | `bool`, default `true` | Activación a nivel plataforma (FR-003, RF-1). Nunca se borra la fila — desactivar es el único mecanismo (Assumptions: sin hard delete) |
| `fields` | `JSONB` | Lista de definiciones de campo de integración, ver forma abajo (FR-004) |
| `created_at` / `updated_at` | `DateTime` | Timestamps estándar del proyecto |

**Forma de `fields`** (validada por Pydantic al crear/editar, no por constraint de Postgres —
research.md Decisión 2):

```json
[
  {"key": "celular", "label": "Número de celular", "required": true,  "format": "numeric", "length": 10},
  {"key": "qr",       "label": "Código QR",         "required": false, "format": "image"}
]
```

`format` ∈ `"text" | "numeric" | "image"`. `length` solo aplica a `"text"`/`"numeric"` (longitud
exacta esperada, ej. FR-004 "número de celular a 10 dígitos"). Lista vacía = método sin campos
adicionales (ej. Efectivo).

**Reglas**:
- `name` único a nivel de toda la plataforma (una sola fila por nombre, sin distinción por tenant —
  es exactamente el catálogo compartido que el spec exige, RF-7).
- No existe operación de borrado físico — solo `active` (Assumptions de `spec.md`).
- Editar `fields` no recalcula la completitud de configuraciones de tenant ya guardadas
  (research.md Decisión 4 / edge case de `spec.md`).

### `PaymentMethod` (modificada — esquema `tenant`, tabla `payment_methods`, ya existente)

Corresponde a "Configuración de Método de Pago por Tenant" en `spec.md`. Columnas nuevas resaltadas.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID (PK) | Sin cambios |
| `name` | `String(100)` | Sin cambios de tipo — ahora se **copia** de `catalog.name` al activar (no editable libremente, research.md Decisión 5) |
| `type` / `is_cash` | `String(20)` / `bool` | Sin cambios de tipo — ahora se **copian** de `catalog.type` al activar |
| `active` | `bool` | Sin cambios de tipo — ahora también se apaga automáticamente cuando `catalog.active` pasa a `false` (FR-013; ver Transiciones) |
| `payment_info` | `JSONB` (`dict[str, str]`, nullable) | Sin cambios de tipo — ahora **validado** contra `catalog.fields` al guardarse (FR-009) |
| **`catalog_id`** | UUID (FK → `shared.payment_method_catalog.id`), **nullable** | Nuevo. Nullable por la estrategia de backfill (research.md Decisión 3) — la capa de aplicación exige valor para filas nuevas |
| **`is_complete`** | `bool`, default `true` | Nuevo. Recalculado al guardar `payment_info` (research.md Decisión 4). Default `true`, no `false`: descubierto durante la implementación de US3 — con `false`, aplicar la migración de esquema (antes de correr el backfill) habría vaciado el checkout de **todos** los tenants de golpe (ninguna fila preexistente pasaría el filtro `is_complete=true`), violando FR-016. El default `true` preserva la disponibilidad de las filas ya existentes hasta que el backfill recalcule el valor real de cada una. |

**Reglas**:
- `UNIQUE (catalog_id)` (restricción normal, no parcial, dentro del esquema del tenant) — a lo sumo
  **una fila por tenant por método de catálogo, para siempre** (FR-017; research.md Decisión 9).
  Activar de nuevo un método ya desactivado no crea una fila nueva: se reactiva/edita la misma vía
  `PATCH`. Postgres no considera iguales dos `NULL`, así que esta restricción no bloquea las filas
  todavía sin `catalog_id` durante la ventana de backfill.
- `name` sigue siendo único dentro del tenant (constraint ya existente, sin cambios) — se mantiene
  porque `name` sigue siendo una columna real, solo que ahora su valor viene del catálogo; la
  restricción única de `catalog_id` (arriba) es justamente lo que evita que dos filas terminen
  copiando el mismo `catalog.name` y choquen entre sí.
- Nunca se borra físicamente (ya es así hoy) — preserva `Payment.payment_method_id` (Principio VII).

### `Payment` / `Sale` (sin cambios)

No se modifican. `Payment.payment_method_id` sigue apuntando a la misma fila de
`tenant.payment_methods` de siempre — su integridad histórica no depende de `catalog_id` ni de
`is_complete`.

## Relaciones

```
shared.payment_method_catalog (1) ──< (0..N) tenant.payment_methods (por cada esquema de tenant)
tenant.payment_methods (1) ──< (0..N) tenant.payments   [SIN CAMBIOS, ya existente]
```

Un `PaymentMethodCatalog` puede tener cero o varias filas de `PaymentMethod` referenciándolo — como
máximo **una por cada tenant** que lo activó alguna vez (activa o no, `UNIQUE (catalog_id)` dentro
de cada esquema de tenant). Un `PaymentMethod` referencia como máximo un `PaymentMethodCatalog` (o
ninguno, durante la ventana de backfill).

## Transiciones de estado

### `PaymentMethodCatalog.active`

```
[creado] --activar (default)--> active=true --desactivar (Super Admin)--> active=false
                                      ^                                          |
                                      └──────────── reactivar (Super Admin) ─────┘
```

Efecto de `active=false` sobre filas dependientes: **no se toca `PaymentMethod.active` por fila
individualmente** (evita un `UPDATE` masivo innecesario) — el filtro de disponibilidad para
checkout (`GET /sales/payment-methods?available=true`, ver
[contracts/checkout-payment-methods.md](./contracts/checkout-payment-methods.md)) exige
`catalog.active=true` además de `PaymentMethod.active=true`, así que basta con que el catálogo esté
inactivo para que ningún tenant lo vea disponible (FR-013), sin perder el estado `active` que el
tenant tenía configurado (para que, si el Super Admin reactiva el catálogo, el tenant lo recupere
sin volver a activarlo, edge case de `spec.md`).

### `PaymentMethod` (por tenant)

```
[no existe]
    │ Tenant Admin activa un catalog_id (FR-007)
    ▼
active=true, is_complete=false  (si catalog.fields tiene campos obligatorios sin diligenciar)
    │ Tenant Admin completa payment_info válido (FR-008/FR-009)
    ▼
active=true, is_complete=true  ◄────────────────────────┐
    │ Tenant Admin desactiva (FR-010)                     │ Tenant Admin reactiva
    ▼                                                      │ (PATCH sobre la MISMA fila —
active=false, is_complete=true  ───────────────────────────┘  `catalog_id` es único por tenant,
                                                                FR-017 — nunca una fila nueva)
```

`is_complete` nunca se resetea a `false` por una acción del Super Admin (editar/desactivar el
catálogo) — solo cambia cuando el propio tenant edita `payment_info` (research.md Decisión 4).

## Disponibilidad para checkout (vista derivada, no una tabla)

Un método de pago es "disponible en caja" (FR-012) si y solo si:

```
PaymentMethod.active = true
AND PaymentMethod.is_complete = true
AND (PaymentMethod.catalog_id IS NULL OR PaymentMethodCatalog.active = true)
```

El `OR PaymentMethod.catalog_id IS NULL` existe solo por la ventana de backfill (`outerjoin`, no
`join`, en `service.list_available_payment_methods`): una fila que todavía no tiene `catalog_id`
asignado no debe excluirse por "no tener catálogo" — se excluye únicamente cuando **sí** tiene
`catalog_id` y ese catálogo específico está inactivo. Esta combinación se resuelve en la consulta
de `GET /sales/payment-methods?available=true` (ver contrato), no se persiste como columna — evita
que un cambio en el catálogo requiera propagar un `UPDATE` a cada tenant.

## Migración de datos existentes (FR-015/FR-015a)

1. **Seed inicial del catálogo** (migración Alembic, esquema `shared`): tres filas —
   `("Efectivo", type=cash, fields=[])`, `("Nequi", type=transfer, fields=[celular obligatorio
   numeric(10), qr opcional image])`, `("Transferencia Bancolombia", type=transfer, fields=[cuenta
   obligatorio text, tipo_cuenta obligatorio text, qr opcional image])`.
2. **Columnas nuevas** (migración Alembic, `@for_each_tenant_schema`): `catalog_id` (nullable),
   `is_complete` (default `false`) en `tenant.payment_methods`.
3. **Reporte de discrepancias** (script, no migración): lista, por tenant, los `payment_methods`
   cuyo `name` normalizado no matchea ninguna fila del catálogo — para que el Super Admin decida
   cuáles agregar al catálogo (FR-015a) antes de continuar.
4. **Backfill** (script, reejecutable): para cada `payment_methods` cuyo nombre normalizado sí
   matchea (incluyendo los que el Super Admin acabe de agregar en el paso 3), setea `catalog_id` y
   recalcula `is_complete` contra `catalog.fields` vigente en ese momento — preservando el
   `payment_info` ya capturado por el tenant sin pedir que lo vuelva a diligenciar (FR-015).
5. **Verificación** (parte de quickstart.md): cada tenant ve, después de la migración, exactamente
   los mismos métodos que veía antes (FR-016/SC-004) — cero filas con `catalog_id IS NULL` que
   estuvieran `active=true` antes de migrar.
