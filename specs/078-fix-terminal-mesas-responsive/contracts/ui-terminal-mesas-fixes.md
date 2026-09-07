# Contratos de UI: Correcciones responsive de la Terminal de Mesas

Esta spec no cambia ningún contrato de API (sin endpoints, sin schemas nuevos —
[data-model.md](../data-model.md)). Los "contratos" aquí son de **interfaz de usuario**: el
comportamiento observable que cada historia fija y contra el que se verifica la implementación
(mismo estilo que [`spec 059 contracts/ui-contracts.md`](../../059-terminal-mesas-carga-y-pedidos/contracts/ui-contracts.md)).

Anchos de referencia (reutilizados de spec 076, sin sistema nuevo):

| Vista | Rango | Breakpoint Tailwind |
|---|---|---|
| Escritorio | ≥ 1024px | `lg` y superiores |
| Tablet | 768–1023px | `md` … antes de `lg` |
| Móvil | < 768px | antes de `md` |

---

## C1 — Total de la tarjeta de un pedido de Domicilio (US1)

**Superficie**: cada tarjeta de la pestaña "Domicilios" de la Terminal de Mesas
(`app-order-summary-card` alimentada por `store.ordersByType('domicilios')`).

| Entrada | Regla de salida |
|---|---|
| Pedido de Domicilio, `delivery_fee > 0` | `totalLabel` = `fmt(subtotal_post_descuento + delivery_fee)` — el mismo `total` que `GET /orders/{id}/checkout-preview` devuelve para ese pedido (FR-001, FR-002, FR-003) |
| Pedido de Domicilio, `delivery_fee` = `0` o `null` | `totalLabel` = `fmt(subtotal_post_descuento)` — sin sumar nada por domicilio, sin error (FR-006) |
| Pedido de mesa o "Para llevar" | `totalLabel` **exactamente igual que hoy** — `delivery_fee` nulo, no se suma nada (FR-005) |
| Cualquier pedido de Domicilio | un **único** total combinado; sin línea aparte de "productos"/"domicilio" en la tarjeta (FR-004) |

**Invariante**: para un mismo pedido de Domicilio, `totalLabel` de la tarjeta === `total` del panel
de detalle/cobro **mientras la promoción que aplicó al pedido conserve el mismo estado y reglas que
tenía al confirmarlo** (FR-003). Ambos parten del mismo `delivery_fee` y del descuento congelado en
las líneas al confirmar (`discounted_unit_price`, spec 038). Si esa promoción se pausa/elimina/
cambia después, el panel —que recalcula el descuento en vivo (spec 073 FR-009a)— es la autoridad y
la tarjeta puede quedar transitoriamente por debajo hasta el cobro (spec.md, Edge Cases).

**Sin petición nueva**: el total se compone de datos ya en memoria (`order.items`,
`order.delivery_fee`). No se llama `checkout-preview` por tarjeta.

---

## C2 — Botón de crear pedido en todas las pestañas + tipo preseleccionado (US2)

**Superficie**: CTA "Crear pedido nuevo" de la Terminal de Mesas y la ruta de armado manual.

| Contexto | Contrato |
|---|---|
| Pestaña "Mesas", "Domicilios" o "Para llevar", **sin** tarjeta seleccionada | El CTA está visible y habilitado, con las mismas reglas de permiso que hoy (FR-007). Ubicación: escritorio → fijo al final del panel de bienvenida; tablet/móvil → acción fija sobre la grilla/lista (FR-007, FR-015) |
| Cualquier ancho, incluido móvil | El CTA muestra **texto visible** ("Crear pedido nuevo", o "Crear pedido" si el ancho no da) — nunca solo el ícono `+` (FR-013, FR-014) |
| Activar el CTA (o su atajo) desde "Domicilios" | Abre el armado manual con tipo **Domicilio** preseleccionado (FR-008) |
| Activar el CTA desde "Para llevar" | Abre el armado manual con tipo **Para llevar** preseleccionado (FR-009) |
| Activar el CTA desde "Mesas" o desde el panel sin selección | Abre el armado manual **sin** tipo preseleccionado — idéntico a hoy (FR-010) |
| Dentro del formulario de armado | El tipo preseleccionado se puede cambiar — es valor inicial, no bloqueo (FR-011) |
| Tras crear el pedido | Misma navegación destino, mismo atajo, mismos permisos, mismo resultado que hoy (FR-012) |

**Ruta** (decisión técnica, [research.md](../research.md) D2/D3):

| Ruta | Cuándo | `?tipo=` |
|---|---|---|
| `mesas-sesiones/:tableId/orden-manual` | "Mesas" (y atajo F3 con mesa seleccionada) | — (sin preselección) |
| `mesas-sesiones/orden-manual` (nueva, sin `:tableId`) | "Domicilios" / "Para llevar" | `domicilio` \| `para-llevar` |

`ManualOrderPageComponent.ngOnInit()` lee `?tipo=` y llama `setOrderTypeTab(tipo)` una vez; valores
ausentes/inválidos/`mesas` → sin efecto.

---

## C3 — Panel de detalle adaptado al ancho (US3)

**Superficie**: la tarjeta de detalle/cobro de la Terminal de Mesas con una mesa o un pedido
seleccionado.

| Ancho | Contrato |
|---|---|
| Escritorio (≥ 1024px) | Panel **al lado** de la grilla; todo su contenido dentro del panel, nada recortado por el borde, la página sin scroll horizontal (FR-016 esc. 1) |
| Tablet (768–1023px) | Al seleccionar, el panel **reemplaza** la grilla a todo el ancho; todo el contenido visible, sin recorte lateral ni scroll horizontal de página; al cerrar (`cancelSelection()`), vuelta a la grilla en el mismo estado — pestaña, filtro, scroll (FR-016 esc. 2) |
| Móvil (< 768px) | El panel ocupa el ancho del dispositivo, todo el contenido dentro de la pantalla (FR-016 esc. 3) |
| Los tres | Controles de cobro (totales, método de pago, botones) visibles y accionables sin salir del área visible (FR-017, FR-021) |
| Los tres, contenido > alto disponible | Desplazamiento **vertical dentro del panel** — nunca horizontal, nunca contenido recortado inalcanzable (FR-019) |
| Los tres, texto largo (producto, dirección, cliente) | Se ajusta al ancho del panel (envuelve/acomoda) sin ensancharlo ni empujar contenido fuera de vista (FR-020) |

**Mecánica** ([research.md](../research.md) D4): la columna de detalle es una sola columna flex
vertical acotada (`min-h-0` en toda la cadena de ancestros, `min-w-0` para no forzar ancho
horizontal); se elimina el contenedor de scroll externo que hoy envuelve el panel central + el de
cobro juntos.

---

## C4 — Espacio para la lista de productos (US4)

**Superficie**: la sección que lista los productos del pedido en el panel de detalle.

| Entrada | Contrato |
|---|---|
| Pedido de ≥ 6 productos, escritorio o tablet | ≥ 4 productos visibles a la vez, sin desplazarse (SC-005); la lista ocupa el alto vertical restante del panel (FR-022) |
| Pedido de ≥ 6 productos, móvil | Los que quepan tras encabezado + totales + acciones — al menos 2 (SC-005); los botones de cobro siguen alcanzables sin desplazar toda la pantalla (FR-025) |
| Lista con más ítems de los que caben | Desplazamiento **dentro del área de la lista**; encabezado del pedido y zona de totales/acciones permanecen visibles (FR-023) |
| Pedido de 1–2 productos | La lista no fuerza alto artificial ni deja hueco vacío con aspecto de error (FR-024) |

**Mecánica** ([research.md](../research.md) D5): encabezado, fila de domicilio (C5), totales y
acciones → `shrink-0`; la lista de productos → única región `flex-1 min-h-0 overflow-y-auto`.

---

## C5 — Información del domicilio en línea (US5)

**Superficie**: el panel de detalle de un pedido de Domicilio (sin mesa, `order_type === 'DELIVERY'`).

| Entrada | Contrato |
|---|---|
| Pedido de Domicilio seleccionado | Dirección + teléfono + valor del domicilio en una **fila compacta junto a la insignia de estado del pedido**, no como bloque vertical separado y extenso (FR-026) |
| Esa fila | Los tres datos completos y legibles — no se elimina ninguno (FR-027) |
| Dirección larga | Se muestra **completa, envolviendo en 2–3 líneas** si hace falta; sin `truncate`, sin ocultarse tras una interacción (FR-028) |
| Valor del domicilio en la fila | Es el **mismo** `delivery_fee` sumado en el total del pedido (C1/FR-001) — nunca dos cifras distintas (FR-029) |
| Pedido de mesa o "Para llevar" | La fila **no** se muestra — panel igual que hoy para esos casos (FR-030) |

---

## C6 — Menú de navegación global en tablet (US6)

**Superficie**: el shell `DashboardLayoutComponent` — todas las pantallas autenticadas de la app.

| Ancho | Contrato |
|---|---|
| Tablet (768–1023px), cualquier pantalla | Menú **oculto por defecto**, sin reservar ancho fijo permanente (FR-031, FR-033); el ancho liberado queda para el contenido |
| Tablet, al activar el control de menú (mismo botón/hamburguesa que en móvil) | El menú se despliega **superponiéndose** al contenido, con las mismas opciones que en escritorio/móvil (FR-031, FR-035) |
| Tablet, menú desplegado, al elegir una opción o tocar fuera | El menú se vuelve a ocultar — mismo comportamiento que móvil (FR-031) |
| Escritorio (≥ 1024px) | El menú **no cambia** — sigue fijo como hoy (FR-034) |
| Los tres anchos | Mismo contenido, mismas opciones, mismos destinos — solo cambia cómo se muestra/oculta (FR-035); mismo control, sin uno nuevo para tablet (FR-036) |

**Mecánica** ([research.md](../research.md) D7): umbral `md` (768) → `lg` (1024) en
`layout.service.ts` (`DESKTOP_BREAKPOINT_PX`, valor inicial de `sidebarOpen`) y
`dashboard-layout.component.ts` (`md:ml-64`→`lg:ml-64`, `md:hidden`→`lg:hidden`, auto-cierre
`< 768`→`< 1024`). `sidebar.component.ts` no cambia.

---

## C7 — No regresión (US "No regresión")

| Contrato | FR |
|---|---|
| La grilla responsive de 3 variantes, la barra superior operativa, los contadores de ocupación y "Atendido por" (spec 076) no cambian | FR-037 |
| El vocabulario de estados y la paleta de colores (spec 076 FR-029/FR-030) no cambian | FR-038 |
| Ninguna regla de cobro, facturación o creación de pedidos cambia | FR-039 |
| Ninguna acción (buscar, filtrar, seleccionar, crear pedido, cobrar) deja de estar disponible en ninguno de los tres anchos | FR-040, SC-008 |

---

## Decisión de negocio a registrar antes de implementar (Principio II)

La pieza 6 (US6) cambia comportamiento **fuera de la Terminal de Mesas**. Antes de implementar sus
tareas, se agrega a `specs/000-reconocimiento/registro-de-anomalias.md` la entrada siguiente
(borrador — la tarea T00x de `tasks.md` la crea con el número real disponible, previsiblemente
`A-71`):

> ### A-71 — [DECISIÓN DE NEGOCIO — spec 078] El menú de navegación global se colapsa en tablet en toda la aplicación; la tarjeta de Domicilio pasa a mostrar el total real
>
> - **QUÉ CAMBIA**:
>   1. A ancho de tablet (768–1023px), el menú de navegación global de la aplicación pasa a estar
>      oculto por defecto y desplegable bajo demanda (mismo comportamiento que ya tiene en móvil),
>      **en todas las pantallas**, no solo en la Terminal de Mesas. Antes, en tablet el menú se
>      mantenía fijo ocupando ancho, igual que en escritorio.
>   2. La tarjeta de un pedido de Domicilio en la pestaña "Domicilios" de la Terminal de Mesas pasa
>      a mostrar como total `productos + valor del domicilio` (el mismo importe que se cobra), en
>      lugar de solo el subtotal de productos.
> - **POR QUÉ DEBE CAMBIAR**:
>   1. Liberar ancho de pantalla en tablet para el contenido de cada pantalla (particularmente la
>      Terminal de Mesas) y unificar el comportamiento con el de móvil.
>   2. La tarjeta mostraba un importe menor del que realmente se factura — riesgo de cobrar de
>      menos en pedidos a domicilio. La regla "el total de un domicilio incluye el valor del
>      domicilio" ya fue autorizada por spec 056 para el cobro; la tarjeta simplemente no la
>      reflejaba (defecto de visualización).
> - **QUIÉN**: el dueño/desarrollador del proyecto.
> - **CUÁNDO**: 2026-09-07 (solicitud y dos sesiones de Clarifications, spec 078).
> - **QUÉ SE VE AFECTADO**:
>   1. Todas las pantallas de la aplicación a ancho de tablet — cambia únicamente la presentación
>      del menú de navegación (mostrar/ocultar). Ninguna opción, orden ni destino del menú cambia.
>      Escritorio y móvil no cambian.
>   2. Solo la visualización de la tarjeta de pedidos de Domicilio **pendientes de cobro**. Ninguna
>      venta ya emitida se recalcula (Principio VII); un `delivery_fee` nulo (pedidos anteriores a
>      spec 056) se trata como cero, igual que ya hace el cobro.
> - **CÓDIGO**: `pos-heladeria` — `layout.service.ts`, `dashboard-layout.component.ts` (menú en
>   tablet); `pos-terminal.store.ts` (`toOrderCardView`, total de la tarjeta).

`A-71` es bloqueante **solo** para las tareas de US6; US1 a US5 no dependen de ella (US1 se puede
implementar antes y la nota del punto 2 se incorpora a la misma entrada).
