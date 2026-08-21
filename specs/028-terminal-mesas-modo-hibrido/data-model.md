# Data Model: Rediseño Híbrido de la Terminal de Mesas

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Consistente con la sección **Key Entities** del spec: esta funcionalidad **no agrega entidades ni
columnas nuevas**. Reutiliza el modelo de datos ya existente en `pos-backend`. Esta página
documenta los campos y transiciones de estado relevantes tal como existen hoy, y las dos
transiciones lógicas nuevas que introduce esta spec (D1/D2 de `research.md`), que se implementan
como nueva lógica de servicio, no como cambios de esquema.

## Entidades reutilizadas (sin cambios de esquema)

### `CustomerOrder` (`app/models/customer_order.py`)

| Campo | Relevancia para esta spec |
|---|---|
| `status` | `recibida → abierta → bloqueada → pagada` (+ `cancelada` desde cualquier estado no terminal). Ver transición nueva D1 más abajo. |
| `channel` | `qr \| counter \| waiter` — ya existe, CHECK constraint en BD, default `qr`. **Es el campo que decide el modo de la barra lateral (FR-005)**; esta spec no lo modifica, solo lo lee. |
| `table_session_id` | agrupa las órdenes de una misma mesa/sesión — usado para la regla de no-mezcla de orígenes (FR-013). |
| `version` | lock optimista ya usado por `block_order`; reutilizado por el nuevo endpoint de D3 para la misma protección contra doble ejecución. |

### `OrderPaymentAttempt` (`app/models/order_payment_attempt.py`)

| Campo | Relevancia |
|---|---|
| `status` | `pendiente \| confirmado \| rechazado` — sin cambios. Índice parcial único garantiza un solo `pendiente` por orden (regla ya vigente reutilizada en Edge Cases del spec). |
| `rejection_reason` | CHECK constraint que exige motivo al rechazar — ya cubre FR-002 ("Rechazar" exige motivo). |
| `receipt_file_url` | ya servido por el backend; D4 solo cambia cómo el frontend lo muestra (modal vs. pestaña nueva), no el campo. |

### `TableSession` (`app/models/table_session.py`)

| Campo | Relevancia |
|---|---|
| `status` | `active` → cerrado (vía `checkout.close_table_sessions`). El endpoint nuevo de D2 reutiliza la misma transición de cierre, sin nuevo valor de estado. |
| `billing_mode` | `unified \| split`, se fija solo al cerrar cobrando (spec 010/026). La liberación pura de FR-016 no cobra nada, así que no fija `billing_mode` (queda como quedó del último cobro por comensal, si aplica, o `null` si todos los comensales ya pagaron individualmente antes del cierre). |

### `DiningTable` (`app/models/dining_table.py`)

| Campo | Relevancia |
|---|---|
| `status` | `libre \| ocupada \| reservada \| pendiente_pago` — FR-014 mapea estas insignias: `pendiente_pago`→"Por confirmar" (amarillo), `ocupada` con todas las órdenes ya pagadas→"En preparación" (azul), `libre`→"Libre" (gris/verde). No se agregan valores nuevos. |

### `Sale` / `Invoice` (`app/models/sale.py`, `app/models/invoice.py`)

Sin cambios. `Sale.status`: `issued | paid | void`. `Invoice.status`: `issued | void`. La relación
`Sale.invoice` (`viewonly`) ya está pensada para reimprimir (D6/FR-012) — se reutiliza tal cual.

## Transiciones de estado nuevas (lógica de servicio, no de esquema)

### T1 — Orden manual: creación diferida a cocina (D1, sustenta FR-011)

```
create_order(hold_for_payment=true)
        │
        ▼
   status = "recibida"          (igual que hoy hace una orden QR recién creada;
   channel = "counter"|"waiter"  sin inventario descontado, sin visibilidad en cocina)
        │
        │  cajero agrega/edita ítems (misma orden, aún "recibida")
        ▼
  checkout-and-send (D3)         (nuevo endpoint atómico)
        │
        ├─ build_sale(...)       → Sale + Invoice emitidos
        └─ _confirm_order_impl   → status = "abierta", inventario descontado,
                                    ítems visibles para cocina (estado_cocina="pendiente")
```

**Invariante que se mantiene**: una orden `recibida` con `channel="counter"|"waiter"` **nunca**
aparece en la cola de "Validación de Pago Requerida" (esa cola se filtra también por
`channel == "qr"`, no solo por `status == "recibida"` — ver D1 en research.md), porque no tiene
comprobante que revisar y FR-009 exige que el cobro manual por transferencia/datáfono no pase por
revisión.

**Compatibilidad**: `hold_for_payment` es un campo opcional nuevo con default `false`; cualquier
llamada existente a `POST /orders` que no lo envíe conserva el comportamiento actual exacto
(creación directa en `abierta`). No hay migración de datos — es un parámetro de la petición, no
una columna.

### T2 — Liberación pura de una mesa ya pagada (D2, sustenta FR-016)

```
Todas las órdenes de la sesión ya están "pagada" (o "cancelada")
        │
        ▼
  release_paid_session(table_session_id)      (nuevo, endpoint POST .../release)
        │
        ├─ lock de fila (mismo mecanismo RN-MESA-01 que close_session)
        ├─ rechaza si _billable_orders(...) no está vacío   (⇐ condición inversa a close_session)
        ├─ rechaza si _assert_closable(...) falla            (pedidos "recibida" sin confirmar,
        │                                                       o ítems de cocina sin terminar —
        │                                                       misma regla, sin reabrirla)
        └─ checkout.close_table_sessions(...) + table.status = "libre"
```

No crea `Sale` ni `Payment` (ya existen de los cobros previos por comensal/orden); solo cierra
`TableSession` y libera `DiningTable`.

## Concepto de negocio nuevo (sin campo nuevo): "origen de la orden"

Tal como documenta el spec, el **origen** (QR vs. manual) ya es el campo `channel` existente. Esta
spec no introduce una columna nueva para representarlo — introduce **reglas de UI y de servicio**
que lo consultan:

- FR-005 (modo de la barra lateral): lectura pura de `channel` en el frontend.
- FR-013 (bloqueo de mezcla de orígenes): al crear una orden (QR vía `POST /cart/submit`, o manual
  vía `POST /orders`), el servicio correspondiente verifica que no exista ya, en la misma
  `table_session_id` activa, una orden con `channel` de la familia opuesta (`qr` vs.
  `counter`/`waiter`) en estado no terminal.
