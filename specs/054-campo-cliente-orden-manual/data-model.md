# Data Model: Campo "Cliente" con valor por defecto "Consumidor final" en la creación de orden manual

Sin entidades ni campos nuevos, de backend ni de frontend (Principio VIII: N/A).

## Campo ya existente que empieza a poblarse desde esta pantalla

- **`CustomerOrder.customer_name`** (`pos-backend/app/models/customer_order.py:49`,
  `String(255)`, nullable) — ya existe, ya es opcional, ya lo acepta `OrderCreate`
  (`app/api/v1/orders/schemas.py:120`) y ya lo persiste `POST /orders`
  (`app/api/v1/orders/router.py:502-504`). Esta spec no cambia su tipo, su nulabilidad ni ninguna
  migración — solo hace que, desde esta pantalla, viaje "Consumidor final" o un nombre editado en
  vez de `null`.
- **`PosTerminalStore.customerName`** (`pos-terminal.store.ts:314`, signal de frontend ya
  existente) — ya es el signal que `createManualOrderFromDraft()` envía como `customer_name`
  (`pos-terminal.store.ts:1072-1078`). Esta spec no le agrega ningún campo nuevo, solo hace que
  `ManualOrderPageComponent` lo lea/escriba (hoy no lo hace en absoluto) y le aplique un valor por
  defecto (research.md D1).

## Sin cambios de estado ni de transición

No se agrega ningún signal nuevo al store compartido — `editandoCliente` (research.md D2) vive
como signal local de `ManualOrderPageComponent`, no del store, porque es puramente un estado de
interacción de esta pantalla (readonly vs. editable), no un dato de negocio.
