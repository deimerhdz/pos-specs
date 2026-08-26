# Data Model: Vaciado del Carrito del Participante al Crear el Pedido (Menú QR)

Todas las entidades de esta spec viven en el schema **`tenant`** (por-tenant, vía
`@for_each_tenant_schema`) — a diferencia de `UserInvitation`/`Tenant`/`User` (spec 037, schema
`shared`). Las decisiones de diseño detrás de cada elección están en [research.md](./research.md);
este documento se limita a columnas, restricciones y transiciones.

## Cart (`carts`) — SIN CAMBIO DE ESQUEMA

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id` | UUID (PK) | No | `UUIDPrimaryKeyMixin`. |
| `participant_id` | UUID (FK → `session_participants.id`) | No | Indexado. |
| `status` | `String(12)` | No | `'abierto'` \| `'confirmado'` \| `'abandonado'`, `server_default='abierto'`. **Sin cambio en el `CheckConstraint`** (research.md Decisión 2) — `'confirmado'` sigue siendo un valor legítimo, producido por `consolidation.py::consolidate_table` (vía del mesero, fuera de alcance de esta spec), aunque `submit_cart` deje de producirlo. |
| `created_at` | `DateTime` | No | `server_default now()`. |

**Lo único que cambia es *comportamiento*, no esquema**: al confirmar un pedido desde el menú QR
(`submit_cart`), la fila de `Cart` del participante que lo originó se **elimina** (`db.delete(cart)`)
en vez de pasar a `status='confirmado'`. Ningún consumidor fuera de `app/api/v1/cart/` se ve
afectado — ver research.md Decisión 10.

**Restricciones** (sin cambios):
- Índice único parcial `idx_open_cart_per_participant` sobre `participant_id` `WHERE status =
  'abierto'` — sigue garantizando un solo carrito `'abierto'` por comensal. Con el borrado físico,
  después de confirmar un pedido el comensal pasa a tener **cero** filas de `Cart` (no una fila
  `'confirmado'` conviviendo con una `'abierto'` nueva) hasta que agrega un ítem nuevo o consulta
  `GET /cart`, momento en el que `_get_or_create_open_cart` le abre una — sin conflicto con este
  índice, igual que hoy.

## CartItem (`cart_items`) / CartItemOption (`cart_item_options`) — SIN CAMBIO DE ESQUEMA

Sin columnas nuevas. Relevante para el borrado físico: `cart_items.cart_id` tiene
`ondelete="CASCADE"` y `Cart.items` declara `cascade="all, delete-orphan"` — ambos ya existían antes
de esta spec (para el caso de limpieza de sesión) y son lo que hace que `db.delete(cart)` borre
también sus `CartItem`/`CartItemOption` sin código adicional. Mismo mecanismo en cascada para
`cart_item_options.cart_item_id`.

## OrderItem (`order_items`) — MODIFICADA (2 columnas nuevas)

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id`, `order_id`, `participant_id`, `product_variant_id`, `quantity`, `unit_price`, `combo_id`, `estado_cocina`, `void_de`, `notes` | — | — | **Sin cambios.** `unit_price` sigue siendo el precio unitario **sin** descuento, snapshot copiado del `CartItem` origen (FR-002 — "precio unitario aplicado"). |
| `discounted_unit_price` | `Numeric(12, 2)` | **Sí** | **NUEVA.** Precio unitario ya con el mejor descuento vigente aplicado en el instante de confirmar, o `NULL` si ninguna promoción aplicó a esa línea (o es una línea de combo — su ahorro es aparte). Mismo nombre/semántica que `CartItemResponse.discounted_unit_price` (research.md Decisión 7). |
| `discounted_line_total` | `Numeric(12, 2)` | **Sí** | **NUEVA.** `quantity × discounted_unit_price` (ya redondeado, `ROUND_HALF_UP` a centavos), o `NULL` en las mismas condiciones que la columna anterior. Mismo nombre/semántica que `CartItemResponse.discounted_line_total`. |

**Poblado**: ambas columnas se calculan en `submit_cart` con el mismo motor que ya usa
`serialize_cart` para pintar el carrito — `promotions.active_discount_promotions(db, now)` +
`promotions.best_line_discount(...)`, vía la función compartida que introduce research.md Decisión
4 — evaluado en el instante exacto de confirmar, no en el instante en que la línea se agregó al
carrito (mismo criterio que ya rige lo que el comensal ve en el carrito antes de confirmar).

**Compatibilidad con datos existentes (FR-015, Principio VII)**: las filas de `OrderItem` creadas
antes de esta spec quedan con ambas columnas en `NULL` — la migración no lleva `server_default` y
no hay ningún `UPDATE` de backfill. El endpoint de pedidos enviados devuelve `null` tal cual para
esas filas; el frontend lo interpreta como "sin descuento", sin recalcular contra el motor de
promociones vigente hoy (que podría dar un resultado distinto al que existía cuando se creó ese
pedido histórico).

## OrderItemOption (`order_item_options`) — SIN CAMBIO DE ESQUEMA

## CustomerOrder (`customer_orders`) — SIN CAMBIO DE ESQUEMA

Sin columnas nuevas. `idx_active_order_per_participant` (índice único parcial sobre
`participant_id` `WHERE status NOT IN ('pagada', 'cancelada')`) sigue siendo, sin cambios, la
garantía de última instancia contra una segunda orden activa del mismo comensal (FR-008) — esta
spec no la toca; solo mejora, en `submit_cart`, el mensaje que la aplicación devuelve *antes* de
llegar a depender de ese índice (research.md Decisión 3).

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| El carrito del participante se construye/lee del servidor, nunca del cuerpo de la petición de confirmación | `submit_cart` (sin cambio, FR-001) | US1 |
| Cada línea de pedido conserva producto, variante, opciones, cantidad, precio unitario, descuento (unitario y de línea) y total ya descontado, sin depender de que la fila de carrito siga existiendo | `OrderItem` + `OrderItemOption` (nuevas columnas + columnas existentes) | US1 |
| El carrito y sus líneas se eliminan físicamente al confirmar, en la misma transacción que el pedido | `submit_cart`, `db.delete(cart)` antes del único `commit()` (research.md D1) | US1 |
| Si la creación del pedido falla por cualquier motivo, el carrito no se borra ni se modifica | El único `try/except` de `submit_cart` envuelve ambas operaciones; `rollback()` deshace las dos | US1 |
| Si el snapshot de descuento no puede resolverse, se rechaza sin crear pedido ni borrar el carrito | Mismo `try/except` (research.md D5) | US1 |
| Solo se elimina el carrito del participante que confirmó — los demás carritos de la misma mesa u otra no se tocan | `_load_open_cart(db, participant.id)`, filtro por `participant_id` (sin cambio) | US4 |
| Segundo intento de confirmar sin carrito (ya vaciado) y con pedido no terminal previo → mensaje explícito de "ya fue enviado", sin crear nada | `submit_cart`, validaciones reordenadas (research.md D3) | US2 |
| Carrito vacío sin ningún pedido no terminal previo → aviso genérico de carrito vacío, distinto del anterior | `submit_cart` (research.md D3) | US2 |
| Garantía de última instancia: nunca más de un pedido no terminal por participante, incluso en confirmaciones casi simultáneas | `idx_active_order_per_participant` (spec 025, sin cambios) | US2 |
| Tras confirmar, el frontend vacía su copia local del carrito solo después de la respuesta exitosa del servidor | `payment-method-step.component.ts`/`transfer-details-step.component.ts` (sin cambio, ya lo hacían) | US1 |
| Al recargar, el frontend reconstruye el carrito consultando al servidor, nunca de una copia local | `DiningCartService.load()` (sin cambio) | US1 |
| Segunda ronda tras confirmar: el próximo `add_item`/`get_cart` abre un carrito `'abierto'` nuevo, vacío, sin arrastrar líneas de la ronda confirmada | `_get_or_create_open_cart` (sin cambio — ya no encuentra una fila `'abierto'` porque la anterior fue borrada, no marcada) | US3 |

## Ciclo de vida de `Cart` (antes / después de esta spec)

```text
Antes de esta spec:                          Después de esta spec (submit_cart, QR):

  ┌──────────┐   submit_cart          ┌──────────┐   submit_cart
  │ abierto  │ ───────────────►       │ abierto  │ ───────────────►  (fila eliminada,
  └────┬─────┘   cart.status =        └────┬─────┘   db.delete(cart) físicamente,
       │         'confirmado'              │         dentro de la   sin dejar
       │         (fila sobrevive)           │         misma          registro)
       │                                    │         transacción
       ▼                                    ▼
  ┌────────────┐                       (sin transición: la fila deja de existir;
  │ confirmado │  (nunca más se lee     el próximo add_item/get_cart crea una fila
  │ (huérfana) │   ni se borra)         'abierto' nueva, sin relación con la anterior)
  └────────────┘

  (sesión expira)                      (sesión expira: sin cambios)
  abierto → abandonado                 abierto → abandonado
```

`consolidate_table` (vía del mesero, `orders/consolidation.py`) sigue produciendo
`abierto → confirmado` exactamente como en la columna izquierda — esta spec solo cambia la
transición que ocurre dentro de `submit_cart` (menú QR).
