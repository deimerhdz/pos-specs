# Data Model: Pagos de Órdenes en Mesa (Skeilopos)

Todas las tablas nuevas o modificadas viven en el esquema `tenant` (schema-per-tenant), igual que
`payment_methods`/`customer_orders` hoy. Las decisiones de diseño detrás de cada elección están en
[research.md](./research.md); este documento se limita a las columnas, restricciones y transiciones.

## PaymentMethod (`payment_methods`) — MODIFICADA

Entidad ya existente (`app/models/payment.py`). Columna nueva:

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `payment_info` | `JSONB` | Sí | Diccionario libre con los datos de pago que el comensal necesita ver (ej. `{"cuenta": "3001234567", "titular": "Heladería La 14"}`). `NULL` para efectivo y para métodos existentes hasta que se editen (research.md Decisión 2). |

Columnas sin cambio: `id`, `name` (único por tenant), `is_cash`, `type`
(`cash|card|transfer|other`), `active`.

**Reglas de validación nuevas** (a nivel de servicio, no de `CHECK`):
- FR-003: no se permite `active = false` en un `PATCH` si, contando el resto de métodos del
  tenant, quedarían 0 métodos con `active = true` (research.md Decisión 10).
- `payment_info` solo tiene sentido cuando `type != 'cash'`; no se valida su forma interna (el spec
  no fija un esquema de campos — "según el método").
- Editar `payment_info`/`name`/`active` de un método **no** altera ningún `OrderPaymentAttempt` ya
  creado con ese método (Acceptance Scenario 4, US1) — `OrderPaymentAttempt.payment_method_id` es
  una referencia estable; no se copian los datos de pago dentro del intento.

## OrderPaymentAttempt (`order_payment_attempts`) — NUEVA

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id` | UUID (PK) | No | `UUIDPrimaryKeyMixin`, igual que el resto de entidades del repo. |
| `order_id` | UUID (FK → `customer_orders.id`) | No | Indexado. `ON DELETE` no aplica (una orden nunca se borra, solo cambia de estado). |
| `payment_method_id` | UUID (FK → `payment_methods.id`) | No | Método elegido para este intento. |
| `status` | `String(12)` | No | `pendiente` (default) / `confirmado` / `rechazado`. |
| `amount_received` | `Numeric(12,2)` | Sí | Solo efectivo (FR-009). `NULL` en intentos de transferencia. |
| `change_amount` | `Numeric(12,2)` | Sí | Solo efectivo, calculado al confirmar (`amount_received - total_orden`, FR-010). `NULL` hasta la confirmación. |
| `receipt_file_url` | `String(500)` | Sí | Solo transferencia (research.md Decisión 3). `NULL` hasta que el comensal suba el comprobante. |
| `rejection_reason` | `String(500)` | Sí | Obligatorio cuando `status = 'rechazado'` (FR-014). Visible solo para cajero/back-office — el endpoint del comensal nunca lo serializa (Clarification 3). |
| `resolved_by_user_id` | UUID | Sí | Referencia blanda a `shared.users.id` (mismo patrón que `CustomerOrder.user_id`) — quién aprobó/rechazó/confirmó. `NULL` mientras `pendiente`. |
| `resolved_at` | `DateTime` | Sí | Cuándo se resolvió (aprobado/rechazado/confirmado). `NULL` mientras `pendiente`. |
| `version` | `Integer` | No | `server_default 0`. Lock optimista, mismo patrón que `CustomerOrder.version` (research.md Decisión 9). |
| `created_at` | `DateTime` | No | `server_default now()`. |

**Restricciones** (`__table_args__`, `{"schema": "tenant"}`):
- `CheckConstraint("status IN ('pendiente', 'confirmado', 'rechazado')")`.
- `CheckConstraint("status != 'rechazado' OR rejection_reason IS NOT NULL")` — FR-014 a nivel de
  base de datos, no solo de aplicación.
- Índice único parcial: `Index(..., "order_id", unique=True, postgresql_where=text("status =
  'pendiente'"))` — garantiza FR-015a (research.md Decisión 4): a lo sumo un intento `pendiente` por
  orden en todo momento.

**Relaciones**:
- `OrderPaymentAttempt.order` → `CustomerOrder` (`back_populates="payment_attempts"`, nueva
  relación en `CustomerOrder`, `cascade` no aplica — los intentos nunca se borran).
- `OrderPaymentAttempt.payment_method` → `PaymentMethod` (sin `back_populates`, solo lectura).

## CustomerOrder (`customer_orders`) — SIN CAMBIO DE ESQUEMA

No se agrega ninguna columna ni se toca el `CHECK` de `status` (research.md Decisión 1). Cambia
únicamente su **comportamiento derivado**:

| Concepto del spec | Cómo se deriva |
|---|---|
| "Pendiente de pago" (Key Entity `Orden`) | `status == "recibida"` (sin importar si ya tiene intentos rechazados en el historial). |
| Elegible para pasar a comanda (FR-017) | `status == "recibida"` **y** existe algún `OrderPaymentAttempt` con `status == "confirmado"` — verificado dentro de `confirm_order`, no antes. |
| "Finalizada" (User Story 6) | `status IN ("pagada", "cancelada")` (research.md Decisión 8). |

Se agrega la relación inversa `CustomerOrder.payment_attempts: list[OrderPaymentAttempt]` (solo
lectura, para serializar el historial completo en `GET /orders/{id}/payment-attempts`, FR-016).

## Transiciones de estado de `OrderPaymentAttempt`

```text
                    ┌──────────────┐
   (creación) ────► │  pendiente   │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │                         │
      aprobar comprobante        rechazar (motivo
      / confirmar efectivo        obligatorio)
      (monto >= total)                  │
              │                         │
              ▼                         ▼
        ┌───────────┐            ┌────────────┐
        │ confirmado│            │  rechazado │ ── (terminal para ESTE intento)
        └───────────┘                  │
         (terminal)                    │
                                        ▼
                          participante crea un intento NUEVO
                          (fila nueva, mismo order_id) — el
                          rechazado permanece en el historial
```

- Una orden puede tener **N** filas de `OrderPaymentAttempt` a lo largo del tiempo (0 o más
  `rechazado`, seguidos de a lo sumo un `pendiente` o un `confirmado` — nunca dos `pendiente`
  simultáneos, garantizado por el índice único parcial).
- Una vez un intento es `confirmado` o `rechazado`, es terminal — nunca vuelve a `pendiente` ni
  cambia de método (FR-018: aprobar/rechazar/confirmar-efectivo solo opera sobre filas `pendiente`,
  verificado con `WITH FOR UPDATE`, research.md Decisión 9).
- No hay transición directa `pendiente → pendiente` (cambiar de método sobre el mismo intento): un
  cambio de método siempre crea una fila nueva, y solo después de que la anterior — si existe y está
  `pendiente` — sea resuelta (FR-015a).

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Al menos un `PaymentMethod` activo por tenant | `PATCH /sales/payment-methods/{id}` (servicio) | US1 |
| Comensal solo ve métodos con `active = true` | `GET /cart/payment-methods` (query `WHERE active`) | US2 |
| Un solo intento `pendiente` por orden | Índice único parcial + verificación de servicio | US2, US5 |
| Rechazo exige motivo | `CHECK` de BD + validación de request (`reason` requerido) | US2 |
| Motivo de rechazo no visible para el comensal | Serialización — el schema de respuesta del comensal (`cart` router) omite `rejection_reason`; el del staff (`orders` router) lo incluye | US2 |
| `amount_received >= total_orden` | Servicio, antes de marcar `confirmado` (join contra líneas de la orden) | US3 |
| `confirm_order` exige intento `confirmado` | `checkout.confirm_order`, antes del `with_for_update` existente | US4 |
| Doble aprobación/rechazo/confirmación sin efecto | `WITH FOR UPDATE` sobre `status = 'pendiente'` | US4 |
| Reintento conserva los ítems de la orden | No aplica — el intento de pago no toca `OrderItem`; la orden y sus líneas nunca se recrean | US5 |
| Una orden activa por comensal | `submit_cart`, `WHERE participant_id = :id AND status NOT IN ('pagada','cancelada')` | US6 |
