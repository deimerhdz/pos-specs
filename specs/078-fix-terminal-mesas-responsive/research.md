# Phase 0 Research: Correcciones responsive de la Terminal de Mesas

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan: el stack, el storage y el
testing ya están determinados por `pos-heladeria`, y las nueve ambigüedades de negocio del pedido
original se cerraron en las dos sesiones de Clarifications del 2026-09-07 (ver `spec.md`). Esta
investigación traduce las seis historias a decisiones técnicas y verifica que ninguna choca con
comportamiento protegido.

Contexto de código ya confirmado por lectura directa (2026-09-07):

- El total de la tarjeta de Domicilio hoy = `this.fmt(this.orderSubtotal(o))`
  (`pos-terminal.store.ts:856`); `orderSubtotal()` (`:2192-2206`) suma **solo productos**, ya con
  el descuento por promoción que el backend dejó en `discounted_unit_price` de cada línea
  (`itemUnitPrice()`, `:439-443`) — no suma `delivery_fee`, impuesto ni propina.
- `GET /orders/{id}/checkout-preview` (spec 073, `CheckoutPreview` en `dining.interface.ts:283-293`)
  devuelve `total = max(0, subtotal − descuento + domicilio)` — **sin** impuesto ni propina.
- El CTA "+ Crear pedido nuevo" solo se pinta con `store.orderTypeTab() === 'mesas'`
  (`table-sessions.component.ts:87`); su etiqueta es `hidden sm:inline` (`:106`), así que en móvil
  (< 640px) queda solo el ícono `+`. Navega a `['/dashboard/mesas-sesiones', tableId, 'orden-manual']`
  y se deshabilita si no hay mesa libre (`store.newOrderTableId()`, `:98`).
- La ruta de armado manual es `mesas-sesiones/:tableId/orden-manual` (`dashboard/routes.ts:164`);
  `manual-order-page.component.ts` ya tolera no recibir `tableId` (`ngOnInit`: `if (tableId) …`) y
  `createManualOrderFromDraft()` (`pos-terminal.store.ts:1360-1408`) **no exige mesa** para
  `DELIVERY`/`TAKEAWAY` (spec 055/056).
- La columna de detalle (`table-sessions.component.ts:138-239`) contiene: un botón de volver
  `lg:hidden` compartido (`:153`), una barra de pestañas/campana `shrink-0` (`:171`, con las dos
  pestañas de `hasPendingAndActiveOrders()`), un `@switch (store.effectiveCentralView())` con las
  ramas `@default` (`<app-pos-order-panel>`) y `'validar-pago'` (`<app-payment-validation-block>` en
  `<div class="flex-1 overflow-y-auto p-4">`, `:220-234`), y `<app-pos-checkout-panel>` (`:236`) —
  todo envuelto en un `<div class="flex-1 … overflow-y-auto">` (`:148`) + un `<div>` redundante
  (`:149`); y **además** cada panel declara su propio `flex-1 overflow-y-auto`
  (`pos-order-panel.component.ts:152`, `pos-checkout-panel.component.ts:59`). Doble/triple contenedor
  de scroll anidado.
- La fila de datos del domicilio (`pos-order-panel.component.ts:106-114`) va **debajo** de la
  cabecera (donde vive la insignia de estado), con clases `flex gap-2` y `space-y-0.5`
  contradictorias, y `📍 {{ delivery_address }}` sin envoltura ni control de ancho explícito.
- El sidebar global: `LayoutService` (`layout.service.ts`) usa `DESKTOP_BREAKPOINT_PX = 768` y
  `sidebarOpen` inicial = `window.innerWidth >= 768`; `dashboard-layout.component.ts` da al
  contenido `md:ml-64` cuando el sidebar está abierto (`:40`), pinta el backdrop con `md:hidden`
  (`:23`) y auto-cierra el sidebar en `NavigationEnd` solo si `window.innerWidth < 768` (`:92`).
  `sidebar.component.ts` es `fixed` a todos los anchos y su `translate` depende **solo** de
  `sidebarOpen()` — no tiene ninguna clase de breakpoint propia.

---

## D1. Total de la tarjeta de Domicilio se compone en el navegador, sin petición por tarjeta

- **Decision**: en `toOrderCardView()` (`pos-terminal.store.ts:846-858`), cambiar
  `totalLabel: this.fmt(this.orderSubtotal(o))` por
  `totalLabel: this.fmt(this.orderSubtotal(o) + (o.delivery_fee ?? 0))`. No se llama
  `GET /orders/{id}/checkout-preview` una vez por tarjeta.
- **Rationale**: la Assumption explícita de `spec.md` ("ninguna de las siete correcciones requiere
  peticiones nuevas al servidor") descarta N peticiones para N tarjetas de la pestaña "Domicilios".
  `orderSubtotal(o)` ya es el **subtotal post-descuento** (usa `discounted_unit_price` de cada
  línea, que el backend resolvió al confirmar el pedido — spec 063/073), así que
  `orderSubtotal(o) + delivery_fee` reproduce exactamente la fórmula de `CheckoutPreview.total`
  (`max(0, subtotal − descuento + domicilio)`): impuesto y propina son cero para estos pedidos, tal
  como el propio contrato de spec 073 los omite. FR-003 ("tarjeta y panel coinciden en todo
  momento") se cumple porque ambos parten del mismo `delivery_fee` y del mismo descuento ya
  persistido en las líneas.
- **Alternatives considered**:
  - *Pedir `checkout-preview` por cada tarjeta visible* — rechazada: viola la Assumption de
    `spec.md`, agrega tráfico proporcional al número de domicilios pendientes y un estado de
    carga por tarjeta que el spec no pide.
  - *Exponer un `total` calculado en `OrderResponse` desde el backend* — rechazada: es un cambio de
    backend, explícitamente fuera de alcance (`spec.md`, Out of Scope, "Cualquier cambio de
    backend").
  - *Sumar también impuesto/propina "por si acaso"* — rechazada: estos pedidos no los llevan
    (constraint de spec 073, sin descuento manual / método único); introducir campos que siempre
    valen cero es ruido y arriesga divergir de `checkout-preview`, que tampoco los suma.
- **Riesgo y mitigación**:
  1. *Impuesto/propina futuros* — si un pedido de Domicilio llevara impuesto o propina > 0, la
     tarjeta quedaría por debajo del panel. Hoy no los llevan y `checkout-preview` los fija en cero.
  2. *Descuento recomputado vs. congelado* — `orderSubtotal(o)` usa el `discounted_unit_price`
     congelado por línea al confirmar el pedido (spec 038); `checkout-preview` recalcula
     `auto_discount()` contra el estado vivo de la promoción (spec 073 FR-009a). Si la promoción se
     pausa/elimina/cambia entre la confirmación y la vista, los totales divergen hasta el cobro
     (spec.md, Edge Cases; FR-003).
  3. *Redondeo sub-céntimo* — en una línea con descuento y `quantity > 1`,
     `discounted_unit_price * quantity` puede diferir de `discounted_line_total` en < 1 centavo;
     conviene sumar `discounted_line_total` cuando esté presente.
  Todo se cubre en `quickstart.md` (Historia 1, reconciliación tarjeta ↔ panel) y en tests de
  `pos-terminal.store.spec.ts` (T003).

## D2. Ruta de armado manual sin `:tableId` para Domicilio/Para llevar

- **Decision**: agregar en `dashboard/routes.ts` una ruta hermana
  `mesas-sesiones/orden-manual` (sin `:tableId`), apuntando al mismo `ManualOrderPageComponent`. El
  tipo preseleccionado viaja como **query param** `?tipo=domicilio|para-llevar`. La ruta con
  `:tableId` se conserva sin cambios para "Mesas" y para el atajo F3.
- **Rationale**: Domicilio y Para llevar no tienen mesa (`createManualOrderFromDraft()` ya lo
  contempla), así que forzar un `:tableId` "de relleno" — la primera mesa libre — es lo que hoy
  hace que el CTA se **deshabilite cuando no hay mesas libres** (`newOrderTableId()` → `null`),
  un bloqueo sin sentido para un domicilio. Una ruta sin parámetro elimina ese acoplamiento.
  Un query param (en vez de otro segmento de path) mantiene el tipo como un dato **opcional y
  editable**, coherente con FR-011 (es un valor inicial, no un modo).
- **Alternatives considered**:
  - *Seguir pasando la primera mesa libre + `?tipo=`* — rechazada: mantiene el CTA deshabilitado
    sin mesas libres, contradiciendo FR-007 ("visible y habilitado" en Domicilios/Para llevar).
  - *Segmento de path `orden-manual/domicilio`* — rechazada: sugiere un sub-recurso/modo fijo; el
    query param comunica mejor "preselección".
  - *Estado de navegación (`router.navigate(..., { state })`)* — rechazada: se pierde al recargar
    la página o compartir el enlace, y complica el test.

## D3. Cómo se aplica el tipo preseleccionado

- **Decision**: `ManualOrderPageComponent.ngOnInit()` lee `route.snapshot.queryParamMap.get('tipo')`
  y, si es `domicilio` o `para-llevar`, llama `this.setOrderTypeTab(tipo)` (el método local que ya
  sincroniza el nombre de cliente por defecto, `:796-803`) **una sola vez**, antes de
  `applyDefaultCustomerName()`. Sin valor, o con `mesas`/valor inválido, no hace nada — comportamiento
  idéntico al actual. El `<select>`/segmented de tipo dentro del formulario sigue plenamente
  editable.
- **Rationale**: reutiliza el punto de entrada y el método que ya existen; no toca
  `createManualOrderFromDraft()` ni la validación. FR-008/FR-009/FR-010/FR-011 se satisfacen sin
  lógica nueva de negocio.
- **Alternatives considered**: *setear `store.orderTypeTab` directo desde el componente* —
  rechazada: `setOrderTypeTab()` local también ajusta "Cliente" por defecto (spec 055/056); saltárselo
  dejaría un "Consumidor final" colado en un domicilio (research de spec 056, D8).
- **Nota de traza**: el doc-comment de `ManualOrderPageComponent` dice "'Domicilio' se mantiene
  deshabilitada — spec 055, FR-012"; es un comentario **obsoleto** (spec 056 habilitó Domicilio y
  el `@if (store.orderTypeTab() === 'domicilios')` del template ya lo prueba). Se actualiza el
  comentario en la misma tarea de US2, como corrección de documentación desactualizada, no como
  refactor (Principio V).

## D4. Contención del scroll en la columna de detalle (US3)

- **Decision**: rehacer el subárbol de `table-sessions.component.ts:138-239` como **una sola
  columna flex vertical acotada**. El contenedor de la tarjeta de detalle pasa a
  `flex flex-col min-h-0 min-w-0 overflow-hidden`. Dentro, en orden: botón de volver
  (`lg:hidden shrink-0`, `:153-161`) y barra de pestañas/campana (`shrink-0`, `:171-218`) intactos;
  el `@switch (store.effectiveCentralView())` (`:220-234`) se marca/envuelve `flex-1 min-h-0` como
  única región que absorbe el alto — la rama `@default` (`<app-pos-order-panel>`, ya `flex-1
  min-h-0`) y la rama `'validar-pago'` (conservar su `<div class="flex-1 overflow-y-auto p-4">`,
  `:222`, añadirle `min-h-0`) quedan acotadas y con scroll solo interno para que "Pagos por
  confirmar" tampoco quede recortada (FR-021a); `<app-pos-checkout-panel>` (`:236`) va `shrink-0`.
  Se **eliminan** el `<div class="flex-1 … overflow-y-auto">` (`:148`) y el `<div>` redundante
  (`:149`) que hoy scrollean panel central + cobro juntos. Todo ancestro relevante lleva `min-w-0` para que el texto largo
  envuelva en vez de ensanchar (FR-020). En `lg` la grilla y el panel siguen lado a lado; por
  debajo de `lg` el panel reemplaza a la grilla (patrón maestro-detalle ya presente, `:126`/`:141`)
  — se deja explícito para tablet y se conserva el retorno con `store.cancelSelection()` sin perder
  pestaña/filtro/scroll (FR-016).
- **Rationale**: el defecto real (spec.md, Clarification "el panel aparece incompleto… no se adapta
  al ancho en ninguna de las tres vistas") es el **doble contenedor de scroll anidado**: el
  `flex-1` interno de cada panel nunca recibe un alto acotado porque su ancestro ya scrollea, así
  que crece con su contenido y desborda. Una única cadena `min-h-0 → flex-1 → overflow-y-auto`
  resuelve FR-016 a FR-019 sin JavaScript de layout. El `min-w-0` es la pieza que evita el scroll
  horizontal de página (FR-018): sin él, un hijo flex con contenido ancho impone su ancho mínimo de
  contenido al contenedor.
- **Alternatives considered**:
  - *Dejar el scroll externo y solo poner `max-width` a los textos* — rechazada: no arregla el
    reparto de alto (US4) ni el desborde vertical; ataca el síntoma, no la causa.
  - *Convertir el panel en un overlay/drawer de pantalla completa en tablet* — rechazada: es
    "introducir un patrón maestro-detalle nuevo", explícitamente fuera de alcance (spec.md, Out of
    Scope); el colapso a una vista por vez ya existe, solo hay que acotarlo bien.
  - *CSS `dvh`/`svh` fijos por sección* — rechazada: frágil ante la barra superior y la sub-barra de
    altura variable; el modelo flex `min-h-0` se adapta solo.

## D5. Reparto de alto para la lista de productos (US4)

- **Decision**: dentro de `pos-order-panel.component.ts`, marcar `shrink-0` en la cabecera
  (`:40`), la nueva fila compacta de domicilio (US5) y la barra de acciones; la lista de productos
  (`:152`) queda como la **única** región `flex-1 min-h-0 overflow-y-auto`. El mismo criterio en
  `pos-checkout-panel.component.ts`: encabezado/totales/botones `shrink-0`, y si el panel de cobro
  necesita scroll, es interno y no compite con el de la columna. Un pedido de 1–2 productos no
  fuerza alto artificial (la lista es `flex-1`, encoge a su contenido sin `min-height` fijo) —
  FR-024.
- **Rationale**: es la consecuencia directa de D4; una vez la columna está acotada, `flex-1` sobre
  la lista le entrega todo el alto libre. SC-005 (≥ 4 en escritorio/tablet, ≥ 2 en móvil) se cumple
  porque las secciones fijas son de altura conocida y modesta; se verifica en `quickstart.md` con
  un pedido de ≥ 6 productos en los tres anchos.
- **Alternatives considered**: *altura mínima en `px`/`rem` para la lista* — rechazada:
  reintroduce el riesgo de empujar los botones de cobro fuera de vista en móvil (FR-025); `flex-1`
  con `min-h-0` es adaptable y no necesita número mágico.

## D6. Fila compacta de datos del domicilio junto al estado (US5)

- **Decision**: mover el bloque `📍 dirección / 📞 teléfono / 🛵 domicilio`
  (`pos-order-panel.component.ts:106-114`) a una fila compacta **dentro del área de cabecera**,
  visualmente contigua a la insignia de estado del pedido. Layout: `flex flex-wrap items-start
  gap-x-3 gap-y-1`, cada dato como un `<span>` con su ícono; la dirección en un `<span
  class="min-w-0 break-words">` (sin `truncate`, sin `line-clamp`) que envuelve en 2–3 líneas si
  hace falta (FR-028, Clarification "siempre completa aunque envuelva"). El valor mostrado en
  `🛵` es `store.selectedOrder()?.delivery_fee` — el **mismo** número que suma el total (FR-029).
  La fila solo se renderiza para `!selectedTable() && selectedOrder()?.order_type === 'DELIVERY'`
  (FR-030) — condición ya existente, no cambia.
- **Rationale**: "junto al estado del pedido" (spec.md, punto 6 y Clarification) = dentro del
  bloque de cabecera que ya contiene la insignia, no un bloque vertical aparte debajo. `flex-wrap`
  + `gap` da la presentación compacta sin ocultar nada (FR-027). `break-words` sobre un contenedor
  `min-w-0` es lo que permite envolver sin ensanchar (encaja con D4/FR-020).
- **Alternatives considered**:
  - *Truncar la dirección con tooltip / "ver más"* — rechazada explícitamente por la Clarification
    ("no se trunca ni se esconde tras una interacción").
  - *Tabla de dos columnas etiqueta/valor* — rechazada: es justo el "bloque vertical extenso" que
    el spec quiere eliminar.

## D7. Umbral de breakpoint del menú de navegación en tablet (US6)

- **Decision**: mover el umbral de "slide-over con backdrop" de `md` (768px) a `lg` (1024px), en
  tres archivos:
  - `layout.service.ts`: `DESKTOP_BREAKPOINT_PX` `768`→`1024`; `sidebarOpen` inicial
    `window.innerWidth >= 1024`.
  - `dashboard-layout.component.ts`: `[class.md:ml-64]`→`[class.lg:ml-64]` (`:40`); backdrop
    `md:hidden`→`lg:hidden` (`:23`); auto-cierre en `NavigationEnd` `window.innerWidth < 768`
    →`< 1024` (`:92`).
  - `sidebar.component.ts`: **sin cambios** — su `translate-x` ya depende solo de `sidebarOpen()`,
    que ahora arranca `false` en el rango tablet.
  Aplica en toda la app porque `DashboardLayoutComponent` es el shell de todas las rutas
  autenticadas (FR-032).
- **Rationale**: el comportamiento colapsable en móvil ya está **completo** (slide-over, backdrop,
  toggle, auto-cierre por navegación); "darle a tablet el mismo comportamiento que móvil"
  (Clarification) es literalmente **subir el umbral** de `768` a `1024`. No se crea ninguna
  variante nueva ni un control nuevo (FR-036) — el mismo botón/hamburguesa del header sigue
  llamando `layoutService.toggle()`.
- **Alternatives considered**:
  - *Un tercer estado "tablet" con su propia lógica* — rechazada: FR-035/FR-036 piden el mismo
    contenido y el mismo control; un estado nuevo es complejidad sin valor.
  - *Media query CSS pura sin tocar `LayoutService`* — rechazada: el valor inicial de `sidebarOpen`
    y el auto-cierre por navegación son TypeScript que consulta `window.innerWidth`; hay que
    moverlos igual, y dejar el umbral en dos sitios (CSS `md:` + JS `768`) es la trampa de tener
    dos fuentes de verdad.
- **Verificación del Principio II**: este es el punto que cambia comportamiento **fuera** de la
  Terminal de Mesas → entra `A-71` en `registro-de-anomalias.md` antes de implementar (ver
  `contracts/ui-terminal-mesas-fixes.md`).

## D8. Etiqueta del botón de crear pedido a todos los anchos (US2)

- **Decision**: quitar `hidden sm:inline` de la etiqueta del CTA (`table-sessions.component.ts:106`)
  para que el texto "Crear pedido nuevo" sea visible también en móvil; el badge `[F3]`
  (`hidden md:inline`) se conserva tal cual (atajo solo relevante con teclado). En tablet, como la
  Historia 3 hace que no haya un panel de bienvenida permanente, el CTA se presenta como acción
  fija sobre la grilla/lista con la misma etiqueta (FR-015). Vocabulario: exactamente
  "Crear pedido nuevo" (spec 076 FR-027), o la forma corta "Crear pedido" si el ancho de móvil no
  deja espacio junto al ícono (Assumption de spec.md) — nunca solo `+`.
- **Rationale**: FR-013/FR-014 piden texto explícito con el vocabulario ya establecido; es un
  cambio de una clase de visibilidad. En escritorio la presentación no cambia (FR-015): el CTA
  sigue fijo al final del panel de bienvenida con su etiqueta completa (`pos-checkout-panel.component.ts:141-149`).
- **Alternatives considered**: *un texto nuevo más corto tipo "Nuevo"* — rechazada por FR-014 (sin
  término nuevo).

---

## Resumen de decisiones

| # | Decisión | Historia | Archivos | Abre anomalía |
|---|---|---|---|---|
| D1 | Total de tarjeta = `orderSubtotal + delivery_fee`, compuesto local | US1 | `pos-terminal.store.ts` | No (nota en A-71) |
| D2 | Ruta `mesas-sesiones/orden-manual` sin `:tableId` + `?tipo=` | US2 | `dashboard/routes.ts` | No |
| D3 | `ngOnInit` lee `?tipo=` → `setOrderTypeTab()` una vez | US2 | `manual-order-page.component.ts` | No |
| D4 | Columna de detalle → 1 columna flex acotada, scroll solo interno, `min-w-0` | US3 | `table-sessions.component.ts` | No |
| D5 | Secciones fijas `shrink-0`, lista de productos única región `flex-1` | US4 | `pos-order-panel.component.ts`, `pos-checkout-panel.component.ts` | No |
| D6 | Fila compacta de domicilio junto al estado; dirección `break-words` | US5 | `pos-order-panel.component.ts` | No |
| D7 | Umbral del sidebar `md`→`lg` en 3 archivos; `sidebar.component.ts` sin cambios | US6 | `layout.service.ts`, `dashboard-layout.component.ts` | **Sí — A-71** |
| D8 | Quitar `hidden sm:inline` de la etiqueta del CTA de crear pedido | US2 | `table-sessions.component.ts` | No |
