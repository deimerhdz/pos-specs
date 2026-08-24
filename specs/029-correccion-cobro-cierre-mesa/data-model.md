# Data Model: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Consistente con la sección **Key Entities** del spec: esta funcionalidad **no agrega entidades ni
columnas nuevas** a la base de datos. Reutiliza el modelo ya existente en `pos-backend`. El único
campo nuevo es un campo **computado** en una respuesta de API (sin columna, sin migración) y un
ajuste de restricción sobre un campo de request ya existente.

## Entidades reutilizadas (sin cambios de esquema)

### `CustomerOrder` (`app/models/customer_order.py`)

| Campo | Relevancia para esta spec |
|---|---|
| `status` | `recibida → abierta → bloqueada → pagada` (+ `cancelada`). **No** es, por sí solo, la señal de "ya se pagó" para esta spec — ver `Sale.customer_order_id` más abajo: los caminos QR y de mostrador (`checkout_and_send`) dejan la orden en `abierta` con una `Sale` ya emitida, sin pasar nunca por `pagada`. Solo el camino legado (`block_order`→`pay_order`) alcanza `pagada`. |
| `version` | sin cambios — sigue siendo el lock optimista de `checkout_and_send` (spec 028). |

### `OrderItem` (`app/models/order_item.py`)

| Campo | Relevancia |
|---|---|
| `estado_cocina` | `pendiente \| en_preparacion \| listo \| anulado` — sin cambios de valores ni de transiciones (`_ALLOWED` en `kitchen.py`). Sigue siendo independiente del pago para `transition_kitchen`/`mark_order_ready` (ver D3 de research.md); deja de ser, por sí solo, suficiente para decidir la insignia "Listo" visible al personal (ver más abajo). |

### `Sale` (`app/models/sale.py`)

| Campo | Relevancia |
|---|---|
| `customer_order_id` | FK opcional, indexada, ya poblada tanto por `pay_order` como por `checkout_and_send`. **Es la señal única y confiable de "esta orden ya tiene un pago registrado"** que introduce esta spec (D2/D3 de research.md) — se usa para el nuevo campo `paid` de `OrderResponse` y para el nuevo guard de `void_item`. Sin cambios al modelo: se consulta con una subconsulta `EXISTS`, no se agrega ninguna columna. |

### `OrderItemVoidLog` (`app/models/order_item_void_log.py`)

Sin cambios. Sigue registrando cada anulación exitosa (`motivo`, `user_id`, `user_name`); un
intento de anulación **rechazado** por el nuevo guard de `void_item` (D3) no llega a este registro
porque la función retorna antes de mutar nada — no hace falta un registro nuevo para intentos
bloqueados (fuera de alcance del spec, ver Out of Scope).

## Campo nuevo (computado, sin migración)

### `OrderResponse.paid: bool` (`app/api/v1/orders/schemas.py`)

```python
class OrderResponse(BaseModel):
    ...
    paid: bool  # NUEVO — computado, no persistido
```

**Cómputo**: verdadero si existe al menos una fila en `sales` con `customer_order_id` igual al id
de la orden — mismo patrón de subconsulta que ya usa, probado en producción,
`has_billable_orders` (`table_sessions/service.py:65-83`, ver research.md D2). Se calcula al armar
la respuesta de cada endpoint que ya serializa `OrderResponse`: `GET /orders` (`list_orders`, la
que sondea la terminal) resuelve el conjunto de ids ya facturados **en una sola consulta** para
todos los pedidos de la respuesta antes de serializar (evita N+1 sobre el listado completo);
`GET /orders/{id}` (`get_order`, vía `_load_order`) reutiliza el mismo helper para un solo id. Sin
nuevo endpoint, sin nueva petición del frontend.

**Compatibilidad**: campo de salida aditivo — ningún consumidor existente de `OrderResponse` deja
de recibir los campos que ya tenía; los que ignoran campos desconocidos (comportamiento estándar
de deserialización JSON) no se ven afectados.

### `DiningOrder.paid: boolean` (`pos-heladeria/.../interfaces/dining.interface.ts`)

Espejo en el frontend del campo anterior, mismo nombre. Usado por:
- `deriveTableStatus()` (insignia del listado de mesas).
- El texto de cabecera del pedido seleccionado (`pos-order-panel.component.ts`).
- La condición que oculta el botón "Anular" cuando el pedido ya está pagado.

## Restricción endurecida sobre un campo de request ya existente

### `CheckoutAndSendIn.discount` (`app/api/v1/orders/schemas.py:265`)

| Antes | Después |
|---|---|
| `Field(0, ge=0, max_digits=12, decimal_places=2)` — cualquier valor ≥ 0 | `Field(0, ge=0, le=0, max_digits=12, decimal_places=2)` — únicamente `0` es válido; cualquier otro valor responde `422` desde la validación del propio esquema |

Alcance **exclusivo** de este endpoint (`POST /orders/{order_id}/checkout-and-send`) — el campo
compartido `discount` de `sales/schemas.py` (usado por `create_sale`/`pay_order`, mostrador y
cierre de mesa unificado/dividido) no se modifica; ese campo es alcance de spec 011 (ver
research.md D4).

## Transiciones de comportamiento nuevas (lógica de servicio, no de esquema)

### T1 — Anulación bloqueada tras pago (D3, sustenta FR-005/FR-006/FR-007)

```
void_item(item_id, motivo, user)
        │
        ▼
  ¿existe Sale con customer_order_id == item.order_id?
  ¿o order.status == "pagada"?
        │
        ├─ SÍ → 409 "El pedido ya fue pagado y no puede anularse"
        │        (sin mutar el ítem, sin log de anulación, sin reversa de inventario)
        │
        └─ NO → comportamiento actual sin cambios
                 (valida `estado_cocina != "anulado"`, reversa de inventario si
                 `pendiente`, registra `OrderItemVoidLog`, aplica reemplazo si corresponde)
```

**Invariante que se mantiene**: `transition_kitchen` y `mark_order_ready` no cambian — la cocina
sigue pudiendo avanzar ítems de una orden ya pagada (flujo "pagar primero, cocina después" de
spec 026/028), solo `void_item` gana el nuevo guard.

### T2 — Insignia "Listo" exige pago (D2, sustenta FR-012/FR-013/FR-014)

```
deriveTableStatus(orders, tableStatus)
        │
        ├─ algún pedido QR "recibida"           → 'por_confirmar'   (sin cambios)
        ├─ algún pedido "bloqueada"              → 'pago_pendiente'  (sin cambios)
        ├─ algún ítem vivo con estado_cocina
        │  en {pendiente, en_preparacion}        → 'en_preparacion'  (sin cambios)
        ├─ hay ítems, todos "listo",
        │  Y todas las órdenes relevantes
        │  tienen paid == true                   → 'listo'           (NUEVO: exige paid)
        ├─ hay ítems, todos "listo",
        │  pero alguna orden relevante
        │  tiene paid == false                   → 'pago_pendiente'  (NUEVO: antes cambiaba
        │                                                              a 'listo' sin más)
        └─ (resto de ramas, sin cambios)          → 'ocupada' / 'reservada' / 'libre'
```

**Compatibilidad**: `TableDisplayStatus` no gana ningún valor nuevo — se reutiliza
`'pago_pendiente'`, ya existente y ya incluido en `NEEDS_STAFF`. Ningún consumidor de
`TableDisplayStatus` (filtros del listado, `STATUS_META`) requiere cambios de forma, solo la nueva
condición dentro de `deriveTableStatus`.

## Concepto de negocio nuevo (sin campo nuevo persistido): "pedido pagado"

Esta spec formaliza que "un pedido ya está pagado" significa, de forma única y consistente para
toda la Terminal de Mesas, **"existe una `Sale` emitida para ese pedido"** — no un valor concreto
de `CustomerOrder.status`, porque los dos caminos de pago vigentes (QR y mostrador vía
`checkout_and_send`) dejan la orden en `abierta` con la venta ya emitida. El campo `paid` de
`OrderResponse` es la única superficie nueva que expone este concepto al frontend; el backend lo
calcula con la misma consulta en los dos puntos que lo necesitan (`void_item`, serialización de
`OrderResponse`), sin duplicar la definición.
