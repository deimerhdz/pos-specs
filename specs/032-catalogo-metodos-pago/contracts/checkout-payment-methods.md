# Contrato: Métodos de Pago Disponibles en Caja (Cajero)

Extiende `GET /sales/payment-methods` (`app/api/v1/sales/router.py`) con un query param nuevo, en
vez de crear un endpoint aparte (research.md Decisión 6). Cubre **Historia de Usuario 3**
(`spec.md`).

## `GET /sales/payment-methods?available=true`

**Auth**: `get_current_user` (sin cambios — el cajero ya podía leer este endpoint hoy).

**Comportamiento nuevo** (solo cuando `available=true`):
- Filtra a `PaymentMethod.active = true AND PaymentMethod.is_complete = true AND
  (PaymentMethod.catalog_id IS NULL OR PaymentMethodCatalog.active = true)` (FR-012, ver
  data-model.md "Disponibilidad para checkout" — el `OR catalog_id IS NULL` es solo para la
  ventana de backfill, no afecta el comportamiento en régimen).
- Responde con un schema **reducido**, `PaymentMethodCheckoutOption`, que **no incluye**
  `payment_info` (FR-012a / clarificación 2026-08-24 #1) — los "datos de integración" (cuenta,
  celular, QR) que esa clarificación reserva al Tenant Admin. **Sí incluye** `is_cash`: no es un
  dato de integración, es la clasificación operativa que
  `payment-input.component.ts`/`payment-draft.util.ts` ya necesitaban antes de esta spec para
  decidir si calculan vuelto — omitirlo habría roto esa funcionalidad existente (Principio V).

**Response 200** — `list[PaymentMethodCheckoutOption]`:

```json
[
  {"id": "uuid-nequi-tenant", "name": "Nequi", "is_cash": false},
  {"id": "uuid-efectivo-tenant", "name": "Efectivo", "is_cash": true}
]
```

Sin `available` (o `available=false`, default): comportamiento **sin cambios** respecto a hoy —
devuelve todas las filas del tenant con `PaymentMethodResponse` completo (uso administrativo,
`payment-methods-page.component.ts` sigue consumiéndolo así).

## Consumo en frontend (`pos-heladeria`)

`pos-terminal.store.ts` (`src/app/modules/tables/services/`) gana `paymentMethodsAvailable =
paymentMethodService.checkoutOptions` **junto a** (no en reemplazo de) `paymentMethods =
paymentMethodService.methods`: el listado completo se sigue necesitando para `methodName()`
(resolver el nombre de un método ya usado en una venta al imprimir un recibo, incluso si ya no
está disponible para cobros nuevos — Principio VII, el histórico no cambia). Solo
`pos-checkout-panel.component.ts` (el picker de cobro) y sus hijos
(`session-bill-panel.component.ts`, `payment-input.component.ts`, `payment-draft.util.ts`) pasan a
recibir `paymentMethodsAvailable()`, tipado `PaymentMethodCheckoutOption[]` en vez de
`PaymentMethod[]`.

**Efecto observable** (SC-006 / Acceptance Scenario US3 #3): si el Super Admin desactiva un método
de catálogo que un tenant tenía activo, la próxima vez que el cajero abra la pantalla de cobro
(recarga de `paymentMethods`) ese método deja de aparecer — sin que el tenant haya hecho nada, y
sin afectar `Payment`/`Sale` ya registrados con ese método (Principio VII).
