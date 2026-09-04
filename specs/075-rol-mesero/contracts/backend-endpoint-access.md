# Contrato: Acceso a endpoints del backend para el rol Mesero

**Spec**: [../spec.md](../spec.md) | **Research**: [../research.md](../research.md) (D2/D3)

Este documento es la lista autoritativa de qué endpoint del backend puede
alcanzar un usuario con rol `MESERO`. Es un contrato **por diferencia**: todo
lo que no aparece explícitamente como "Permitido" en este documento queda
**denegado (403)** para Mesero, sin excepción — ese es el comportamiento
default-deny descrito en research.md, D2.

Los roles Admin y Cajero no cambian: este documento no afecta ninguna regla
de autorización que ya exista para ellos (FR-008). Las filas marcadas
"Bloqueado (ya protegido)" ya devuelven 403 a Mesero hoy mismo por
`require_tenant_admin` — no requieren ningún cambio de código, se listan
solo para dejar constancia de que quedan cubiertas.

## Permitido para Mesero

### `table-sessions` — Terminal de Mesas (router completo)

| Método | Ruta | Motivo |
|---|---|---|
| GET | `/table-sessions` | Listar sesiones de mesa |
| GET | `/table-sessions/{table_session_id}` | Detalle de sesión + comensales |
| POST | `/table-sessions/{table_session_id}/participants` | Agregar comensal (división de cuenta) |
| DELETE | `/table-sessions/{table_session_id}/participants/{participant_id}` | Quitar comensal |
| PUT | `/table-sessions/{table_session_id}/assignments` | Asignar ítems a comensales |
| GET | `/table-sessions/{table_session_id}/bill` | Cuenta de la sesión (desglose por comensal) |
| POST | `/table-sessions/{table_session_id}/close` | Cobrar y cerrar sesión |
| POST | `/table-sessions/{table_session_id}/release` | Liberar sesión ya pagada |

### `orders` — Terminal de Mesas y Órdenes (subconjunto del router)

| Método | Ruta | Motivo |
|---|---|---|
| GET | `/orders/tables` | Listar mesas (grilla de la terminal) |
| PATCH | `/orders/tables/{table_id}/status` | Cambiar estado operativo de una mesa |
| POST | `/orders/{order_id}/move` | Mover una orden de mesa |
| POST | `/orders/merge` | Fusionar mesas |
| GET | `/orders/group/{group_id}/bill` | Cuenta de un grupo de mesas fusionadas |
| POST | `/orders/{order_id}/confirm` | Confirmar orden (recibida → abierta) |
| GET | `/orders/{order_id}/payment-attempts` | Consultar intentos de pago |
| POST | `/orders/payment-attempts/{attempt_id}/approve` | Aprobar intento de pago |
| POST | `/orders/payment-attempts/{attempt_id}/reject` | Rechazar intento de pago |
| POST | `/orders/payment-attempts/{attempt_id}/confirm-cash` | Confirmar pago en efectivo |
| POST | `/orders/tables/{table_id}/consolidate` | Consolidar pedido de mesa |
| POST | `/orders/tables/{table_id}/items` | Agregar ítem al pedido de una mesa |
| PATCH | `/orders/items/{item_id}/kitchen` | Transición de estado de cocina |
| POST | `/orders/{order_id}/ready` | Marcar orden lista |
| POST | `/orders/items/{item_id}/void` | Anular ítem |
| GET | `/orders/tables/{table_id}/bill` | Cuenta de una mesa |
| GET | `/orders/{order_id}/checkout-preview` | Vista previa de cobro |
| POST | `/orders/draft-preview` | Vista previa de un pedido manual en borrador |
| POST | `/orders/{order_id}/block` | Bloquear orden para cobro |
| POST | `/orders/{order_id}/pay` | Cobrar orden |
| POST | `/orders/{order_id}/checkout-and-send` | Cobrar y enviar a cocina en un paso |
| POST | `/orders/{order_id}/cancel` | Cancelar orden |
| POST | `/orders/tables/{table_id}/release` | Liberar mesa |
| POST | `/orders` | Crear pedido manual ("orden manual") |
| GET | `/orders` | Listar órdenes — usa tanto la Terminal (filtrada a sesiones activas) como la pantalla Órdenes |
| GET | `/orders/{order_id}` | Detalle de una orden — usa tanto la Terminal como la pantalla Órdenes |

### Endpoints de solo lectura de apoyo (routers que, por lo demás, quedan bloqueados)

| Método | Ruta | Router | Motivo |
|---|---|---|---|
| GET | `/promotions` | `promotions` | Precio/promociones activas mostradas al armar un pedido |
| GET | `/sales/payment-methods` | `sales` | Métodos de pago disponibles en el modal de cobro |
| GET | `/sales/{sale_id}` | `sales` | Venta generada al cobrar, para el recibo |
| GET | `/cash/shifts/current` | `cash` | Turno de caja abierto, necesario para poder cobrar |
| GET | `/invoices` | `invoices` | Factura asociada a una orden, para el recibo (router 100% de lectura) |
| GET | `/invoices/{invoice_id}` | `invoices` | Detalle de una factura, para el recibo |
| GET | `/tenant` | `tenant` | Nombre/logo/mensaje del negocio, para el encabezado del recibo impreso |
| POST | `/realtime/ticket` | `realtime` | Ticket de un solo uso para abrir la conexión en tiempo real de la terminal |

`GET /menu`, `GET /menu/promotions` y `GET /realtime/stream` no requieren
ninguna entrada en esta lista: son públicos o se autentican con un ticket de
un solo uso, no con el JWT de staff, así que nunca pasan por la verificación
de rol de esta funcionalidad.

## Bloqueado (ya protegido hoy por `require_tenant_admin`, sin cambios)

| Método | Ruta | Router |
|---|---|---|
| POST | `/orders/tables` | `orders` (crear mesa) |
| PATCH | `/orders/tables/{table_id}` | `orders` (editar mesa) |
| GET | `/orders/tables/{table_id}/qr-token` | `orders` (token QR imprimible) |
| PATCH | `/tenant` | `tenant` (editar información del negocio) |
| POST/PATCH/DELETE | `/promotions/*` (crear, editar, forma, estado, duplicar, eliminar) | `promotions` |
| POST/PATCH | `/sales/payment-methods*` (crear, editar métodos de pago) | `sales` |
| Todos | `products`, `categories`, `inventory`, `unit_measures`, `users`, `invitations`, `reports`, `super_admin`, `plan`, `audit`, `uploads` | — |

## Bloqueado (hoy solo requiere `get_current_user`; pasa a exigir la verificación nueva de D2)

Cualquier endpoint de `cash`, `sales` u `orders` que no esté en la tabla de
"Permitido" de arriba. En particular, estos son los que hoy sí alcanza un
Cajero pero que Mesero **no** debe alcanzar:

| Método | Ruta | Router | Motivo del bloqueo |
|---|---|---|---|
| GET | `/cash/registers` | `cash` | Pantalla Caja, no Terminal de Mesas |
| POST | `/cash/shifts/open` | `cash` | Abrir turno — Caja |
| POST | `/cash/shifts/{shift_id}/close` | `cash` | Cerrar turno (arqueo) — Caja |
| GET | `/cash/shifts` | `cash` | Historial de turnos — Caja |
| GET | `/cash/shifts/{shift_id}/reconciliation` | `cash` | Arqueo — Caja |
| POST | `/cash/shifts/{shift_id}/partial-count` | `cash` | Conteo parcial — Caja |
| GET | `/cash/shifts/{shift_id}/report` | `cash` | Reporte de cierre — Caja |
| GET/POST | `/cash/shifts/{shift_id}/movements` | `cash` | Movimientos de efectivo — Caja |
| POST | `/sales` | `sales` | Checkout de mostrador — pantalla Ventas, no Terminal de Mesas |
| GET | `/sales` | `sales` | Listado de ventas — pantalla Ventas |
| GET | `/sales/payment-methods/catalog` | `sales` | Catálogo administrativo de métodos de pago — no lo usa la Terminal (verificar en implementación si algún flujo de cobro lo necesita; si es así, se agrega a la lista de permitidos) |

## Formato para la implementación

La lista de "Permitido" se implementa como pares (método HTTP, plantilla de
ruta tal como la expone FastAPI, p. ej. `/orders/{order_id}/pay`) — no como
rutas ya resueltas con valores reales, para que un solo par cubra todas las
órdenes/mesas/turnos sin importar su id.
