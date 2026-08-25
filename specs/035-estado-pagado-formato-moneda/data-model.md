# Data Model: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

Sin entidades ni campos nuevos, sin migración (Principio VIII n/a). Esta spec cambia el punto
del ciclo de vida en que un campo ya existente toma un valor ya válido, y la condición de
lectura de tres consultas ya existentes.

## CustomerOrder.status (existente, ciclo de vida ampliado)

`status: str` — ya definido en `pos-backend/app/models/customer_order.py`, con
`CheckConstraint("status IN ('recibida', 'abierta', 'bloqueada', 'pagada', 'cancelada')")`.

**Ciclo de vida antes de esta spec** (documentado en el propio modelo):

```text
recibida → abierta → bloqueada → pagada
         ↘  cancelada (terminal, desde cualquier estado no terminal)
```

En la práctica, antes de esta spec, solo dos caminos alcanzaban `'pagada'`:
`checkout.pay_order` (camino legado de cobro) y `table_sessions.close_session` (cierre de
sesión de mesa). El camino `checkout.checkout_and_send` (Terminal de Mesas, "Cobrar y
enviar") creaba una `Sale` pero dejaba la orden en `'abierta'`.

**Ciclo de vida después de esta spec**: `checkout_and_send` se suma a los caminos que
alcanzan `'pagada'` directamente, en la misma transacción en que se crea la `Sale`. El camino
QR (`confirm_order`/`approve_payment_attempt`/`confirm_cash_payment_attempt`, vía
`_deduct_and_open`) sigue avanzando únicamente a `'abierta'` — no crea ninguna `Sale` en ese
momento (FR-002 de `spec.md`).

**Relación con el campo calculado `paid`** (spec 029, `OrderResponse.paid`): sin cambios —
sigue siendo `True` exactamente cuando existe una `Sale` para el pedido
(`orders/service.py:order_has_sale`), independientemente de `status`. Esta spec hace que,
para el camino `checkout_and_send`, `status = 'pagada'` y `paid = True` coincidan desde el
mismo instante — antes podían divergir (`status = 'abierta'`, `paid = True`).

## OrderItem.estado_cocina (existente, nuevo consumidor)

`estado_cocina: str` — ya definido en `pos-backend/app/models/order_item.py`, valores
`'pendiente' | 'en_preparacion' | 'listo' | 'anulado'`. Sin cambios en el propio campo; esta
spec agrega un nuevo consumidor: las tres funciones de gestión de mesa en
`orders/tables_advanced.py` (ver `research.md`, Decisión 2) lo consultan para decidir si una
orden `'pagada'` todavía bloquea operaciones sobre su mesa.

**Dos predicados nuevos** (ver `research.md`, Decisión 2, para por qué no es uno solo):

- `_table_occupied_by_order(order)` — reemplaza a `status not in ('pagada', 'cancelada')` en
  `_active_orders_on_table` (que alimenta `set_table_status` y los chequeos de mesa
  destino/origen de `move_order`):

  ```text
  status == 'cancelada' → False
  status == 'pagada'    → existe algún OrderItem de la orden con estado_cocina EN_CURSO
                           ('pendiente' | 'en_preparacion')
  cualquier otro status → True
  ```

- `_order_locked_for_move_or_merge(order)` — reemplaza al chequeo puntual sobre la orden que
  se mueve en `move_order`, y sobre las órdenes a fusionar en `merge_orders`:

  ```text
  status == 'cancelada' → True
  status == 'pagada'    → existe algún OrderItem de la orden con estado_cocina EN_CURSO
  cualquier otro status → False
  ```

Ambos coinciden en el caso `'pagada'` (la pregunta de fondo, `_has_pending_kitchen_work`, es
la misma), pero divergen en `'cancelada'` (nunca ocupa la mesa, pero siempre impide
mover/fusionar esa orden en concreto) y en los estados no-terminales (siempre ocupan la mesa,
pero nunca impiden mover/fusionar esa orden). Antes de esta spec, `'pagada'` se comportaba
como `'cancelada'` en ambos sentidos a la vez; esta spec solo cambia el caso `'pagada'` con
ítems pendientes.

## Componente de moneda (frontend, sin persistencia propia)

`MoneyInputComponent` no introduce ninguna entidad de datos — es un control de UI que envuelve
un `number | null` ya existente en cada formulario que lo use. Su contrato de entrada/salida
se documenta en `contracts/money-input-component.md`, no aquí (no es un modelo de datos).
