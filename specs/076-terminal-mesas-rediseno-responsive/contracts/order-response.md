# Contract: `OrderResponse` — campo nuevo `staff_user_name`

**Endpoint(s) afectados**: todos los que ya devuelven `OrderResponse`
(`app/api/v1/orders/schemas.py:197-224`) — el listado/consulta de órdenes que hoy consume
`pos-terminal.store.ts` para construir `tablesView()`/`ordersByType()`. No se agrega ningún
endpoint nuevo; se extiende el schema de respuesta ya existente.

## Cambio

Se agrega un campo opcional, computado (no columna), siguiendo el mismo patrón ya usado por `paid`
en el mismo schema (líneas 218-224, "El router lo asigna antes de serializar"):

```text
OrderResponse
  ...campos ya existentes, sin cambios...
  staff_user_name: str | None = None   # NUEVO
```

**Regla de valor** (FR-021, FR-023, ver data-model.md):
- `None` si `DiningOrder.user_id` es `None` (pedido enviado por el cliente vía QR).
- En otro caso, el nombre para mostrar del usuario referenciado por `user_id` — sin filtrar por rol
  (Cajero y Mesero se resuelven igual, Clarifications de spec.md).
- Si el usuario referenciado ya no existe o no puede resolverse (edge case de spec.md), el campo se
  comporta igual que ya lo hace hoy `closed_by_user_name` en un caso equivalente (mejor referencia
  disponible, sin lanzar error).

## Compatibilidad

Campo aditivo y opcional: cualquier consumidor existente de `OrderResponse` que no lo lea sigue
funcionando exactamente igual (no rompe ningún contrato ya consumido por otras pantallas que también
usan este mismo schema, por ejemplo Órdenes/spec 075).

## Contratos reutilizados sin cambios (Historia 3 — turno de caja)

No se crea ningún contrato nuevo para esta historia — se reutilizan tal cual, sin modificar su
forma de request/response:

- `GET /api/v1/cash/shifts/current` (`cash.service.ts:117`) — ya devuelve `CashShift` completo
  (incluye `status`, `user_name`, `opened_at`); la Terminal de Mesas solo pasa a **leerlo**, algo
  que hoy ya hace indirectamente vía `cashShiftId()` sin presentarlo.
- `POST /api/v1/cash/shifts/open` (`cash.service.ts:107`) — invocado por el mismo flujo ya
  implementado en `CashSessionStore.openShift()`; el atajo `F1`/botón de la nueva barra superior
  dispara ese mismo flujo (o navega a él), sin llamarlo por una ruta distinta.
