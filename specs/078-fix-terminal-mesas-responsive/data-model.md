# Data Model: Correcciones responsive de la Terminal de Mesas

**Ninguna entidad nueva, ningún campo nuevo, ninguna columna nueva, ninguna migración.** Esta spec
es 100% frontend (`spec.md`, Out of Scope: "Cualquier cambio de backend"; Constitution Check,
Principio VIII — no aplica). Este documento solo enumera los datos ya existentes que las seis
correcciones **leen** de forma distinta, y confirma que no hay ninguna evolución de esquema que
planificar, migrar ni revertir.

## Entidades ya existentes (sin cambios de esquema)

### `DiningOrder` (backend: `CustomerOrder`; frontend: `dining.interface.ts:204-232`)

| Campo | Tipo | Uso en esta spec |
|---|---|---|
| `id`, `status`, `channel`, `order_type` | — | sin cambios; ya se leen para la tarjeta y el panel |
| `delivery_fee` | `number \| null` | **US1** — se **suma** al total de la tarjeta de Domicilio (`toOrderCardView()`), además de mostrarse en la fila compacta del detalle (**US5**). Ya lo expone spec 056; ya lo recibe la Terminal. `null` (pedido histórico anterior a spec 056) se trata como `0` — mismo criterio que el cobro. |
| `delivery_address` | `string \| null` | **US5** — se muestra completa, envolviendo; hoy ya se lee (`pos-order-panel.component.ts:108`) |
| `delivery_phone` | `string \| null` | **US5** — se muestra en la fila compacta; hoy ya se lee |
| `items[]` + `items[].discounted_unit_price` | `DiningOrderItem[]` | **US1** — base de `orderSubtotal(o)`, que ya devuelve el subtotal **post-descuento** (el backend resolvió `discounted_unit_price` al confirmar el pedido, spec 063/073). No se recalcula ningún descuento en el navegador. |
| `paid` | `boolean` (computado) | sin cambios; ya decide visibilidad de la tarjeta en las pestañas Domicilio/Para llevar |

**Regla de composición del total de la tarjeta (US1, FR-001/FR-002)**:
`totalLabel = fmt( orderSubtotal(order) + (order.delivery_fee ?? 0) )`, donde `orderSubtotal()`
= Σ líneas a su `discounted_unit_price`. Reproduce `CheckoutPreview.total`
(`max(0, subtotal − descuento + domicilio)`) sin llamar al backend, porque el descuento ya está en
las líneas y estos pedidos no llevan impuesto ni propina (ver [research.md](./research.md) D1).

### `CheckoutPreview` (frontend: `dining.interface.ts:283-293`)

Sin cambios. `GET /orders/{id}/checkout-preview` lo sigue devolviendo tal cual para el **panel de
cobro** del pedido seleccionado (US1 exige que la tarjeta coincida con este `total`, FR-003 — la
composición de D1 lo garantiza). Esta spec **no** llama este endpoint una vez por tarjeta.

## View-models de frontend afectados (solo en memoria)

### `OrderSummaryCardView` (`pos-terminal.store.ts:120-131`)

| Campo | Antes | Después |
|---|---|---|
| `totalLabel` | `fmt(orderSubtotal(o))` (solo productos) | `fmt(orderSubtotal(o) + (o.delivery_fee ?? 0))` para Domicilio; sin cambio para mesa / Para llevar (`delivery_fee` nulo → mismo valor de hoy, FR-005) |

Revierte de forma explícita la decisión de
[`spec 059 data-model.md`](../059-terminal-mesas-carga-y-pedidos/data-model.md) (fila `totalLabel`
= `orderSubtotal(order)`, "no incluye el domicilio, no lo pidió el spec"). La forma del
view-model no cambia — solo el número que llega en ese campo.

### Estado del shell de navegación (`LayoutService`, `layout.service.ts`)

| Elemento | Antes | Después |
|---|---|---|
| `DESKTOP_BREAKPOINT_PX` | `768` | `1024` |
| `sidebarOpen` inicial | `window.innerWidth >= 768` | `window.innerWidth >= 1024` |

No es una entidad de datos: es estado de UI de una sesión de navegador, sin persistencia. Cambia
solo el ancho a partir del cual el menú arranca visible.

## Compatibilidad, migración y rollback

- **Compatibilidad con datos existentes**: total — no se lee ni se escribe ninguna tabla. Un
  `delivery_fee` nulo (pedidos anteriores a spec 056) produce el mismo total de tarjeta que hoy
  (`+ 0`). Ninguna venta ni factura emitida se toca (Principio VII).
- **Migración**: no aplica (no hay esquema que migrar).
- **Rollback**: revertir los commits de frontend restaura el comportamiento previo por completo;
  no queda ningún dato en un estado intermedio porque no se escribió ninguno.
