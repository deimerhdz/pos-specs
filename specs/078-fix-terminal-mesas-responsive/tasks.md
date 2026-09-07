---

description: "Task list for spec 078 — correcciones responsive y de presentación de la Terminal de Mesas"
---

# Tasks: Correcciones responsive y de presentación de la Terminal de Mesas

**Input**: Design documents from `/specs/078-fix-terminal-mesas-responsive/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-terminal-mesas-fixes.md](./contracts/ui-terminal-mesas-fixes.md), [quickstart.md](./quickstart.md)

**Tests**: incluidos — **no son opcionales aquí**. El plan (Technical Context; Constitution Check,
Principio X — "Verificación Obligatoria") enumera los ocho `*.spec.ts` de las áreas afectadas y
`quickstart.md` valida cada historia por separado en los tres anchos. Mismo criterio que
[spec 073](../073-fix-descuento-cobro-terminal/tasks.md) y [spec 076](../076-terminal-mesas-rediseno-responsive/tasks.md).

**Organización**: por historia de usuario de `spec.md` (US1–US6, en su orden de prioridad
P1×3 → P2×3). **Un solo repositorio**: `../pos-heladeria` (frontend). Cero backend, cero endpoints,
cero migraciones, cero peticiones nuevas al servidor (spec.md, Out of Scope; plan.md, Summary). El
único artefacto fuera de `pos-heladeria` es la entrada `A-71` en el registro de anomalías de
`pos-specs` (T002), bloqueante **solo** para la Historia 6.

Las seis piezas son 1:1 con las seis historias y verificables por separado (research.md, resumen de
decisiones D1–D8):

1. **US1** — `toOrderCardView()` suma `delivery_fee` al total de la tarjeta de Domicilio.
2. **US2** — CTA de crear pedido en las tres pestañas + etiqueta a todos los anchos + tipo
   preseleccionado por pestaña (ruta nueva sin `:tableId` + `?tipo=`).
3. **US3** — la columna de detalle pasa a una sola columna flex acotada (`min-h-0`/`min-w-0`),
   scroll solo interno, sin scroll horizontal de página.
4. **US4** — secciones fijas `shrink-0`, la lista de productos como única región `flex-1`.
5. **US5** — datos del domicilio en una fila compacta junto a la insignia de estado; dirección
   envuelve, no se trunca.
6. **US6** — umbral del menú de navegación global `md` (768) → `lg` (1024), en toda la app.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1 a US6 — solo en fases de historia de usuario
- Rutas relativas a la raíz de `pos-heladeria/` salvo que se indique otra cosa

---

## Phase 1: Setup

**Purpose**: línea base verde antes de tocar código y autorización de proceso (Principio II) para US6.

- [X] T001 [P] Registrar la línea base de tests: correr `npx ng test --watch=false` en
      `pos-heladeria` y anotar el conteo en verde/rojo, con atención a los ocho ficheros de las áreas
      afectadas — `src/app/modules/tables/services/pos-terminal.store.spec.ts`,
      `src/app/modules/tables/pages/table-sessions.component.spec.ts`,
      `src/app/modules/tables/pages/manual-order-page.component.spec.ts`,
      `src/app/modules/tables/components/pos-order-panel.component.spec.ts`,
      `src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`,
      `src/app/modules/tables/components/order-summary-card.component.spec.ts`,
      `src/app/modules/dashboard/layout/dashboard-layout.component.spec.ts`,
      `src/app/modules/dashboard/layout/sidebar.component.spec.ts` — para distinguir después cualquier
      regresión de esta spec de fallos preexistentes (Principio X)
- [X] T002 [P] Registrar la anomalía **A-71** en
      `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`, con una entrada nueva justo
      después de `A-70`, con el mismo formato que `A-65`–`A-70` y el contenido del borrador de
      [contracts/ui-terminal-mesas-fixes.md §"Decisión de negocio a registrar…"](./contracts/ui-terminal-mesas-fixes.md):
      (1) menú de navegación global colapsable en tablet en **toda la app**; (2) nota de que la
      tarjeta de Domicilio pasa a mostrar el total real (US1). **Bloquea la Fase 8 (US6) y la tarea
      T005 (US1)** — el commit de T005 cita `A-71` punto 2. US2 a US5 NO dependen de esta tarea. T002
      es barata y se hace en el Setup, así que no retrasa el MVP (contracts, cierre del documento)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Nota**: no hay ninguna pieza de infraestructura que bloquee a la vez a **todas** las historias.
Esta spec no crea entidades, campos, componentes, servicios ni rutas compartidas: cada historia es
un ajuste acotado de un componente/store/shell ya existente (plan.md, Structure Decision;
data-model.md — "ninguna entidad nueva"). Las únicas dependencias reales son punto‑a‑punto entre
historias que comparten archivo (US2↔US3 en `table-sessions.component.ts`; US3↔US4 en
`pos-checkout-panel.component.ts`; US4→US5 en `pos-order-panel.component.ts`) y la dependencia
US4→US3 (US4 es consecuencia directa de la reestructura de US3). Todas se documentan en
"Dependencies & Execution Order" al final. **No hay tareas en esta fase.**

---

## Phase 3: User Story 1 - Ver en la tarjeta de un pedido de Domicilio el mismo total que se va a cobrar (Priority: P1) 🎯 MVP

**Goal**: la tarjeta de un pedido de Domicilio (pestaña "Domicilios") muestra el total real a
cobrar — `subtotal de productos post-descuento + delivery_fee` —, el mismo importe que el panel de
cobro y que la venta emitida, compuesto en el navegador sin ninguna petición nueva.

**Independent Test**: crear un pedido de Domicilio con subtotal $25.000 y `delivery_fee` $6.000, sin
descuento; abrir "Domicilios" y verificar que la tarjeta muestra **$31.000**; seleccionarlo y
verificar que el total del panel de cobro es idéntico ($31.000)
([quickstart.md §Historia 1](./quickstart.md)). No depende de ninguna otra historia.

### Tests for User Story 1

- [X] T003 [P] [US1] Tests en
      `src/app/modules/tables/services/pos-terminal.store.spec.ts` para el `totalLabel` de las
      tarjetas de `ordersByType('domicilios')` / `ordersByType('para-llevar')`: pedido `DELIVERY`
      con subtotal $25.000 + `delivery_fee` $6.000 → `totalLabel` = `fmt(31000)` (FR-001, SC-001);
      pedido `DELIVERY` con `delivery_fee` `0` o `null` → `fmt(25000)`, sin error (FR-006); pedido
      `DELIVERY` con promoción aplicada (descuento ya en `discounted_unit_price` de las líneas) →
      `fmt(subtotal_post_descuento + delivery_fee)` (FR-002); pedido `TAKEAWAY` / de mesa →
      `totalLabel` **idéntico** al de hoy (FR-005); un único string de total, sin desglose (FR-004).
      Añadir el caso de reconciliación de [research.md D1](./research.md): `totalLabel` == `total` de
      un `CheckoutPreview` simulado con el mismo `subtotal`/`descuento`/`delivery_fee` (FR-003).
      Añadir además: (a) caso de **promoción pausada/eliminada después de confirmar el pedido** — la
      tarjeta mantiene el `discounted_unit_price` congelado y el test documenta que el panel
      (recompute en vivo) es la autoridad (FR-003, spec.md Edge Cases); (b) línea con descuento y
      `quantity > 1` — `totalLabel` no pierde céntimos por el redondeo de
      `discounted_unit_price * quantity` (usar `discounted_line_total` si está presente)
- [X] T004 [P] [US1] Test en
      `src/app/modules/tables/components/order-summary-card.component.spec.ts`: la tarjeta sigue
      renderizando un único `totalLabel` bajo la etiqueta "Total" con la misma forma que hoy — solo
      cambia el número que llega en el input, no la plantilla (FR-004, no regresión)

### Implementation for User Story 1

- [X] T005 [US1] En `src/app/modules/tables/services/pos-terminal.store.ts`, `toOrderCardView()`
      (línea 856): cambiar `totalLabel: this.fmt(this.orderSubtotal(o))` por
      `totalLabel: this.fmt(this.orderSubtotal(o) + (o.delivery_fee ?? 0))`. `orderSubtotal(o)` ya es
      el subtotal **post-descuento** (usa `discounted_unit_price` de cada línea); `delivery_fee` es
      `null` para mesa/`TAKEAWAY` → `+ 0`, sin cambio para esos tipos (FR-005). Añadir un comentario
      que deje trazable que esto **revierte** la fila `totalLabel` de
      [`spec 059 data-model.md`](../059-terminal-mesas-carga-y-pedidos/data-model.md) y cita `A-71`
      punto 2 (research.md D1; data-model.md §`OrderSummaryCardView`). Hace pasar T003. Depende de
      T003 y de T002 (la entrada `A-71` que cita debe existir)
- [X] T006 [P] [US1] En `src/app/modules/tables/components/order-summary-card.component.ts` (línea
      39, etiqueta "Total"): revisar si el texto "Total" necesita una nota ahora que el número **es**
      el total real. Decisión esperada: **sin cambio** — "Total" pasa a ser exacto, no ambiguo
      (plan.md, Scale/Scope). Dejar constancia en el mensaje de commit de que se revisó y por qué no
      se toca. Hace pasar T004
- [ ] T007 [US1] Ejecutar [quickstart.md §Historia 1](./quickstart.md) (pasos 1–8) en los tres
      anchos de referencia, incluida la pestaña de red de DevTools (paso 8: abrir "Domicilios" **no**
      dispara una petición `checkout-preview` por tarjeta)
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: la tarjeta de Domicilio muestra el total real a cobrar — verificable de forma
completamente independiente del resto de historias. Resuelve el único defecto de los siete que
afecta dinero visible (spec.md, US1 "Why this priority").

---

## Phase 4: User Story 2 - Poder crear un pedido nuevo desde cualquier pestaña, con el tipo correcto y un botón que se entienda (Priority: P1)

**Goal**: el CTA "Crear pedido nuevo" está visible y habilitado en las tres pestañas y en los tres
anchos, con etiqueta de texto visible siempre (nunca solo el ícono `+`); desde "Domicilios" /
"Para llevar" abre el armado manual con ese tipo ya preseleccionado (editable), vía una ruta sin
`:tableId` que no exige mesa libre.

**Independent Test**: abrir las tres pestañas y verificar que el CTA está visible en todas y, en
móvil, que muestra texto; pulsarlo desde "Domicilios" → armado manual con tipo "Domicilio"; desde
"Para llevar" → "Para llevar"; desde "Mesas" → sin preselección, como hoy
([quickstart.md §Historia 2](./quickstart.md)).

### Tests for User Story 2

- [X] T008 [P] [US2] Tests en `src/app/modules/tables/pages/table-sessions.component.spec.ts`: el
      CTA de crear pedido se renderiza con `store.orderTypeTab()` en `'mesas'`, `'domicilios'` y
      `'para-llevar'` (sin tarjeta seleccionada) (FR-007); la etiqueta de texto "Crear pedido nuevo"
      es visible en los tres anchos — ya no lleva `hidden sm:inline` (FR-013, FR-015); pulsarlo con
      la pestaña en `'domicilios'` navega a `/dashboard/mesas-sesiones/orden-manual` con
      `queryParams: { tipo: 'domicilio' }`; en `'para-llevar'` con `{ tipo: 'para-llevar' }`; en
      `'mesas'` navega a `['/dashboard/mesas-sesiones', tableId, 'orden-manual']` **sin** `tipo`, y
      sigue deshabilitado si `!store.newOrderTableId()` solo en ese caso (FR-008–FR-010, FR-012)
- [X] T009 [P] [US2] Tests en `src/app/modules/tables/pages/manual-order-page.component.spec.ts`:
      `ngOnInit` con `?tipo=domicilio` llama `setOrderTypeTab('domicilios')` una sola vez;
      `?tipo=para-llevar` → `setOrderTypeTab('para-llevar')`; sin `tipo`, `?tipo=mesas` o valor
      inválido → **no** se llama `setOrderTypeTab` (comportamiento idéntico a hoy); tras la
      preselección el `<select>`/segmented de tipo sigue editable y cambiarlo se acepta (FR-008–FR-011)

### Implementation for User Story 2

- [X] T010 [US2] En `src/app/modules/dashboard/routes.ts` (tras la ruta
      `mesas-sesiones/:tableId/orden-manual`, línea 164): agregar una ruta hermana
      `{ path: 'mesas-sesiones/orden-manual', loadComponent: () => import('../tables/pages/manual-order-page.component').then((m) => m.ManualOrderPageComponent) }`
      (sin `:tableId`), con los mismos `canActivate`/guards que la ruta con parámetro (research.md D2)
- [X] T011 [US2] En `src/app/modules/tables/pages/table-sessions.component.ts`: sacar el bloque del
      CTA (líneas 87–113) del guard `@if (store.orderTypeTab() === 'mesas')` para que se renderice en
      las tres pestañas sin tarjeta seleccionada, conservando las reglas de permiso/habilitación. El
      `(click)` pasa a: si `store.orderTypeTab()` es `'domicilios'` → `router.navigate(['/dashboard/mesas-sesiones/orden-manual'], { queryParams: { tipo: 'domicilio' } })`;
      si `'para-llevar'` → `{ tipo: 'para-llevar' }`; si `'mesas'` → navegación actual con
      `store.newOrderTableId()`. El `[disabled]="!store.newOrderTableId()"` y su `[title]` aplican
      **solo** en la pestaña "Mesas" (Domicilio/Para llevar no exigen mesa —
      `createManualOrderFromDraft()`, research.md D2). Hace pasar T008. Depende de T010
- [X] T012 [US2] En `src/app/modules/tables/pages/table-sessions.component.ts` (línea 106): quitar `hidden sm:inline` del `<span>` de la
      etiqueta para que "Crear pedido nuevo" sea visible también en móvil; conservar el badge `[F3]`
      con `hidden md:inline` (atajo solo relevante con teclado). Si el ancho de móvil no deja la
      etiqueta completa junto al ícono, usar la forma corta "Crear pedido" — nunca solo `+`
      (FR-013, FR-014, spec.md Assumptions; research.md D8). Hace pasar T008
- [X] T013 [US2] En `src/app/modules/tables/pages/table-sessions.component.ts`, el manejador de
      teclado (`keydown`, líneas ~353–363) y `openManualOrder()` (~330–332): al pulsar `F3` con la
      pestaña en `'domicilios'` / `'para-llevar'`,
      navegar a la ruta nueva sin `:tableId` con el `?tipo=` correspondiente (mismo criterio que
      T011); con la pestaña en `'mesas'`, comportamiento actual sin cambios (spec.md, Edge Case "el
      cajero pulsa el atajo…"). Depende de T010, T011
- [X] T014 [US2] En `src/app/modules/tables/pages/manual-order-page.component.ts`, `ngOnInit()`
      (líneas 765–769): leer `this.route.snapshot.queryParamMap.get('tipo')`; si es `'domicilio'`
      llamar `this.setOrderTypeTab('domicilios')`, si es `'para-llevar'` llamar
      `this.setOrderTypeTab('para-llevar')` — **una sola vez, antes** de
      `this.applyDefaultCustomerName()` (línea 769); ausente / `'mesas'` / inválido → no hacer nada.
      Usar el método local `setOrderTypeTab()` (línea 796), no `store.setOrderTypeTab()` directo,
      para que el nombre de cliente por defecto se ajuste (research.md D3). Actualizar en la misma
      tarea el doc-comment obsoleto de la línea 28 ("'Domicilio' se mantiene deshabilitada — spec
      055, FR-012") — corrección de documentación desactualizada, no refactor (research.md D3,
      Principio V). Hace pasar T009
- [ ] T015 [US2] Ejecutar [quickstart.md §Historia 2](./quickstart.md) (pasos 1–10) en los tres
      anchos de referencia
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: el cajero crea un pedido nuevo desde cualquier pestaña, con el tipo correcto
preseleccionado y un botón que se entiende en móvil — independiente del resto de historias.

---

## Phase 5: User Story 3 - Ver el panel de detalle del pedido completo, sin importar el ancho de la pantalla (Priority: P1)

**Goal**: la columna de detalle/cobro pasa a ser **una sola columna flex vertical acotada al alto
disponible** (`min-h-0`, `min-w-0`, sin scroll horizontal de página), con scroll vertical **solo**
interno. Se elimina el doble/triple contenedor de scroll anidado que hoy deja contenido recortado y
fuerza scroll horizontal en tablet/móvil.

**Independent Test**: seleccionar un pedido y abrir la Terminal en los tres anchos (escritorio
≥1024px, tablet 768–1023px, móvil <768px); en cada uno, todo el contenido del panel (encabezado,
lista, totales, método de pago, acciones) queda dentro del área visible y la página no necesita
scroll horizontal; en tablet el panel reemplaza la grilla y al cerrarlo se vuelve al mismo estado
([quickstart.md §Historia 3](./quickstart.md)).

### Tests for User Story 3

- [X] T016 [P] [US3] Tests en `src/app/modules/tables/pages/table-sessions.component.spec.ts`: el
      contenedor exterior de la zona de contenido (hoy línea 119, `flex-1 flex flex-col lg:flex-row
      … min-h-0 overflow-y-auto lg:overflow-hidden`) ya **no** lleva `overflow-y-auto` (queda
      `overflow-hidden`/`min-h-0`), y sus hijos flex llevan `min-w-0`; la tarjeta de detalle
      (`data-testid="detail-column"`, líneas 138–239) es una única columna
      `flex flex-col min-h-0 min-w-0 overflow-hidden` **sin** el `<div class="flex-1 … overflow-y-auto">`
      envolvente de la línea 148 ni el `<div>` redundante de la línea 149; el botón de volver
      (`data-testid="page-back-button"`) y la barra de pestañas/campana (línea 171) siguen `shrink-0`;
      el `@switch (store.effectiveCentralView())` es la región `flex-1 min-h-0` — tanto la rama
      `@default` (`<app-pos-order-panel>`, `flex-1 min-h-0`) como la rama `'validar-pago'`
      (`<app-payment-validation-block>` dentro de su `<div class="flex-1 min-h-0 overflow-y-auto">`,
      línea 222) quedan acotadas y con scroll solo interno; `<app-pos-checkout-panel>` va `shrink-0`;
      por debajo de `lg` la tarjeta de detalle reemplaza a la grilla (`[class]` de las líneas 126/141
      intacto) y `store.cancelSelection()` sigue devolviendo a la grilla; los controles de cobro
      (totales, selector de método de pago, botones de acción) quedan dentro del `overflow` vertical
      y accionables — ninguno fuera del área visible en los tres anchos (FR-016–FR-021, FR-021a)

### Implementation for User Story 3

- [X] T017 [US3] En `src/app/modules/tables/pages/table-sessions.component.ts` (línea 119): quitar
      `overflow-y-auto` del contenedor `flex-1 flex flex-col lg:flex-row p-3 gap-3 min-h-0
      overflow-y-auto lg:overflow-hidden` — dejarlo `… min-h-0 overflow-hidden` en todos los anchos;
      añadir `min-w-0` a la grilla y a la columna de detalle para que el contenido ancho envuelva en
      vez de imponer su ancho mínimo y forzar scroll horizontal de página (FR-018, research.md D4)
- [X] T018 [US3] En `src/app/modules/tables/pages/table-sessions.component.ts`, el subárbol de la columna de detalle (líneas 138–239):
      **eliminar** el `<div class="flex-1 flex flex-col min-h-0 overflow-y-auto">` de la línea 148 y
      el `<div class="flex flex-col bg-white flex-1 min-h-0 lg:flex-1 lg:min-h-0">` redundante de la
      línea 149 que hoy scrollean el panel central + `app-pos-checkout-panel` juntos. La tarjeta de
      detalle (`detail-column`) queda `flex flex-col min-h-0 min-w-0 overflow-hidden`; dentro, en
      orden: el botón de volver `lg:hidden shrink-0` (líneas 153–161) y la barra de
      pestañas/campana `shrink-0` (líneas 171–218) intactos; el `@switch (store.effectiveCentralView())`
      (líneas 220–234) pasa a ser la región `flex-1 min-h-0` — envuélvelo en un `<div class="flex-1
      flex flex-col min-h-0">` o aplícale las clases al primer hijo de cada rama; y
      `<app-pos-checkout-panel>` (línea 236) con `shrink-0`. Las **dos** ramas del `@switch` quedan
      acotadas: `@default` → `<app-pos-order-panel>` (ya `flex-1 flex flex-col min-h-0` por su
      `host`); `'validar-pago'` → conservar el `<div class="flex-1 overflow-y-auto p-4">` de la línea
      222 (añadirle `min-h-0`) alrededor de `<app-payment-validation-block>` — la superficie "Pagos
      por confirmar" pasa por este mismo subárbol (comentario de las líneas 34–41) y **no** debe
      quedar recortada (FR-017). Conservar el patrón maestro-detalle que ya colapsa en `lg` (líneas
      126/141) — explícito para tablet, con el retorno por `store.cancelSelection()` sin perder
      pestaña/filtro/scroll (FR-016, FR-017, FR-019; research.md D4). Hace pasar T016. Depende de T017
- [X] T019 [US3] En `src/app/modules/tables/components/pos-checkout-panel.component.ts` (host línea
      57, `w-full shrink-0 flex flex-col border-t … min-h-0 bg-white`; scroll interno línea 59,
      `flex-1 overflow-y-auto p-4`): garantizar que el panel apilado tiene un alto máximo propio y
      su scroll es **interno**, sin competir con el de la columna ahora acotada; añadir `min-w-0`
      donde el texto largo (cliente, método) pueda ensancharlo (FR-019, FR-020; research.md D4/D5).
      Depende de T018
- [ ] T020 [US3] Ejecutar [quickstart.md §Historia 3](./quickstart.md) (pasos 1–6) en los tres
      anchos: sin recorte lateral, sin scroll horizontal de página, texto largo envuelve, y en
      tablet el panel reemplaza la grilla y vuelve al mismo estado al cerrarlo. Repetir con una mesa
      que tenga un pago QR pendiente (pestaña "🔔 Pagos por confirmar", `app-payment-validation-block`):
      su contenido también queda dentro del área visible y con scroll solo interno en los tres anchos
      (FR-017, FR-019 — superficie que las specs suelen olvidar)
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: el panel de detalle/cobro se ve completo y usable en escritorio, tablet y móvil, sin
scroll horizontal de página. Desbloquea la tarea central de la pantalla en anchos que no son el
ideal (spec.md, US3 "Why this priority").

---

## Phase 6: User Story 4 - Ver varios productos del pedido a la vez en el panel de detalle (Priority: P2)

**Goal**: con la columna ya acotada (US3), las secciones fijas (encabezado, fila de domicilio,
totales, acciones) pasan a `shrink-0` y la lista de productos queda como la **única** región
`flex-1 min-h-0 overflow-y-auto` — recibe todo el alto libre.

**Independent Test**: seleccionar un pedido de ≥ 6 productos; en escritorio y tablet se ven ≥ 4
productos a la vez sin desplazarse; en móvil ≥ 2, con los botones de cobro siempre alcanzables; el
scroll ocurre dentro del área de la lista, con encabezado y totales/acciones siempre visibles
([quickstart.md §Historia 4](./quickstart.md)).

**⚠️ Depende de US3** — es la consecuencia directa de la columna acotada (plan.md, pieza 4).

### Tests for User Story 4

- [X] T021 [P] [US4] Tests en `src/app/modules/tables/components/pos-order-panel.component.spec.ts`:
      el encabezado (línea 40), el bloque de chips de tipo de pedido (línea 126) y la barra de
      acciones (línea 271) llevan `shrink-0`; la lista de productos (línea 152) es la **única**
      región con `flex-1 min-h-0 overflow-y-auto`; un pedido de 1–2 productos no fuerza alto
      artificial ni deja hueco vacío (la lista es `flex-1`, encoge a su contenido) (FR-022–FR-024)
- [X] T022 [P] [US4] Test en `src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`:
      encabezado / totales (línea 254) / botones llevan `shrink-0` y solo la zona media (línea 59)
      es `flex-1 min-h-0 overflow-y-auto`; en móvil los botones de cobro quedan alcanzables sin
      desplazar toda la pantalla (FR-025)

### Implementation for User Story 4

- [X] T023 [US4] En `src/app/modules/tables/components/pos-order-panel.component.ts`: marcar
      `shrink-0` en el encabezado (línea 40, ya lo tiene — verificar), en el bloque de chips de tipo
      de pedido (línea 126) y en la barra de acciones (línea 271); a la lista de productos (línea
      152, hoy `flex-1 overflow-y-auto p-4 space-y-3`) añadirle `min-h-0` para que sea la única
      región que absorbe el alto libre; confirmar que el contenedor intermedio de la línea 36
      (`flex-1 flex flex-col min-h-0`) y el `host` (línea 23, `flex-1 flex flex-col min-h-0`) se
      conservan (research.md D5; FR-022, FR-023, FR-024). Hace pasar T021. Depende de T018 (US3)
- [X] T024 [US4] En `src/app/modules/tables/components/pos-checkout-panel.component.ts`: encabezado,
      la zona de totales (línea 254, ya `shrink-0`) y los botones → `shrink-0`; la zona media (línea
      59) como única `flex-1 min-h-0 overflow-y-auto`, con scroll interno que no compite con el de la
      columna (research.md D5; FR-025). Hace pasar T022. Depende de T019 (US3)
- [ ] T025 [US4] Ejecutar [quickstart.md §Historia 4](./quickstart.md) (pasos 1–5) en los tres
      anchos con un pedido de ≥ 6 productos y uno de 1–2 (SC-005)
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: la lista de productos recibe el espacio disponible — ≥ 4 productos visibles en
escritorio/tablet, ≥ 2 en móvil, con los botones de cobro siempre visibles. US3 + US4 funcionan
juntas y por separado.

---

## Phase 7: User Story 5 - Ver la información del domicilio de forma compacta, junto al estado del pedido (Priority: P2)

**Goal**: el bloque de dirección / teléfono / valor del domicilio pasa de un bloque vertical extenso
a una **fila compacta** (`flex flex-wrap`) dentro del área de cabecera, contigua a la insignia de
estado del pedido; la dirección se muestra completa envolviendo en 2–3 líneas (sin `truncate`); el
valor del domicilio es el mismo `delivery_fee` que suma el total de US1.

**Independent Test**: seleccionar un pedido de Domicilio con dirección larga y verificar que
dirección + teléfono + valor del domicilio aparecen en una fila compacta junto al estado, que la
dirección envuelve completa sin truncarse, y que el valor coincide con el sumado en el total; un
pedido de mesa / "Para llevar" no muestra esa fila
([quickstart.md §Historia 5](./quickstart.md)).

**Depende de US4** — mismo archivo `pos-order-panel.component.ts`: US4 reparte el alto (secciones
fijas `shrink-0` vs. lista `flex-1`); US5 reubica el bloque de domicilio dentro de esa estructura y
lo deja `shrink-0`. Coordinar el merge.

### Tests for User Story 5

- [X] T026 [P] [US5] Tests en `src/app/modules/tables/components/pos-order-panel.component.spec.ts`:
      con `!store.selectedTable() && store.selectedOrder()?.order_type === 'DELIVERY'`, la dirección,
      el teléfono y el valor del domicilio se renderizan en **una** fila compacta (`flex flex-wrap`)
      dentro de la cabecera, contigua a la insignia de estado — no como bloque vertical con
      `space-y-*` (FR-026); la dirección va en un `<span>` con `break-words` y **sin** `truncate` ni
      `line-clamp` (FR-028); el valor mostrado en `🛵` es `store.selectedOrder()?.delivery_fee`
      formateado con `store.fmt(...)` — el mismo número del total de la tarjeta (FR-029); con
      `selectedTable()` o `order_type` distinto de `'DELIVERY'` la fila **no** se renderiza (FR-030);
      los tres datos siguen presentes y legibles (FR-027)

### Implementation for User Story 5

- [X] T027 [US5] En `src/app/modules/tables/components/pos-order-panel.component.ts`, el bloque de
      las líneas 106–114 (`@if (!store.selectedTable() && store.selectedOrder()?.order_type ===
      'DELIVERY')` con `<div class="text-[12px] flex gap-2 text-[#6b7280] space-y-0.5">` y los
      `<p>📍/📞/🛵</p>`): reubicarlo como fila compacta dentro del área de cabecera (líneas 37–114),
      visualmente contigua a la insignia de estado; layout
      `flex flex-wrap items-start gap-x-3 gap-y-1` (quitar el `flex gap-2` + `space-y-0.5`
      contradictorio); cada dato un `<span>` con su ícono; la dirección en
      `<span class="min-w-0 break-words">{{ … delivery_address }}</span>` (sin `truncate`); el valor
      en `🛵` = `store.fmt(store.selectedOrder()?.delivery_fee ?? 0)`. Mantener **sin cambios** la
      condición de render de la línea 106. Marcar la fila `shrink-0` (parte de US4). (research.md D6;
      FR-026–FR-030). Hace pasar T026. Depende de T023 (US4)
- [ ] T028 [US5] Ejecutar [quickstart.md §Historia 5](./quickstart.md) (pasos 1–7): fila compacta
      junto al estado, dirección larga envuelve en 2–3 líneas, valor del domicilio == el del total
      (US1), pedido no-Domicilio sin la fila, alto recuperado a la lista de productos (SC-006)
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: la información del domicilio ocupa menos alto y la lista de productos gana ese
espacio. US4 + US5 entregan juntas la mejora de densidad del panel.

---

## Phase 8: User Story 6 - Aprovechar el ancho de la tablet ocultando el menú de navegación como en móvil (Priority: P2)

**Goal**: el menú de navegación global adopta en tablet (768–1023px) el mismo comportamiento
colapsable que ya tiene en móvil — oculto por defecto, desplegable con el mismo control,
superponiéndose al contenido — en **toda la aplicación**. En escritorio (≥ 1024px) no cambia.

**Independent Test**: abrir varias pantallas de la app (Terminal de Mesas, Órdenes, Inventario,
Reportes…) en ~900px y verificar que el menú está oculto por defecto y se despliega con el
botón/hamburguesa de móvil; repetir en ~1280px y verificar que sigue fijo como hoy
([quickstart.md §Historia 6](./quickstart.md)).

**⚠️ BLOQUEADA por T002** — esta pieza cambia comportamiento **fuera de la Terminal de Mesas**;
`A-71` debe estar registrada antes de tocar código (Constitution Check, Principio II; research.md
D7). Los commits de esta fase citan `A-71`. Independiente de US1–US5 (archivos distintos).

### Tests for User Story 6

- [X] T029 [P] [US6] Tests en `src/app/modules/dashboard/layout/dashboard-layout.component.spec.ts`
      y `src/app/modules/dashboard/layout/sidebar.component.spec.ts`: con `window.innerWidth` en el
      rango tablet (p. ej. 900) `LayoutService.sidebarOpen()` arranca en `false`; el backdrop usa
      `lg:hidden` (no `md:hidden`); el margen del contenido usa `lg:ml-64` (no `md:ml-64`); el
      auto-cierre en `NavigationEnd` ocurre cuando `window.innerWidth < 1024`; con `innerWidth` ≥
      1024 el menú arranca visible y no auto-cierra (FR-031–FR-036); `sidebar.component.ts` no
      cambia — su `translate` depende solo de `sidebarOpen()`

### Implementation for User Story 6

- [X] T030 [US6] En `src/app/modules/dashboard/layout/layout.service.ts`: `DESKTOP_BREAKPOINT_PX`
      (línea 6) `768` → `1024`; actualizar el doc-comment (líneas 3–5 y 10–18) de `md` a `lg`; el
      valor inicial de `sidebarOpen` (líneas 19–21) queda `window.innerWidth >= 1024`. Citar `A-71`
      en el commit (research.md D7). Depende de T002
- [X] T031 [US6] En `src/app/modules/dashboard/layout/dashboard-layout.component.ts`: backdrop
      `md:hidden` → `lg:hidden` (línea 23); `[class.md:ml-64]` → `[class.lg:ml-64]` (línea 40);
      auto-cierre por navegación `window.innerWidth < 768` → `< 1024` (línea 92). Verificar que
      `sidebar.component.ts` **no** necesita cambios (su `translate-x` ya depende solo de
      `sidebarOpen()`). Citar `A-71` en el commit. Hace pasar T029. Depende de T002, T030
- [ ] T032 [US6] Ejecutar [quickstart.md §Historia 6](./quickstart.md) (pasos 1–8) en varias
      pantallas de la app a ~900px y ~1280px, incluidos el cruce del umbral 1024px y el menú
      desplegado con un pedido seleccionado en la Terminal (Edge Cases)
      → **Pendiente** — recorrido visual; requiere navegador + `pos-backend` con datos sembrados (no ejecutable en esta sesión de implementación, ver T036).

**Checkpoint**: a ancho de tablet, en toda la app, el menú de navegación se oculta por defecto y el
ancho liberado queda para el contenido; escritorio y móvil sin cambios. Las seis historias
funcionan de forma independiente.

---

## Phase 9: Polish & Cross-Cutting Concerns (No regresión — FR-037 a FR-040)

- [X] T033 [P] Correr `npx ng test --watch=false` completo en `pos-heladeria` y confirmar **0 fallos
      nuevos** frente a la línea base de T001, con atención a los ocho `*.spec.ts` de las áreas
      tocadas
      → **Hecho (2026-09-07).** Línea base T001 (rama `develop`, commit `238ed5e`): 737 tests,
      650 verde / 87 rojo — 69 de esos 87 eran el mismo `NG0201: No provider found for SwPush`
      (regresión preexistente de spec 077) en `table-sessions.component.spec.ts` (19),
      `manual-order-page.component.spec.ts` (48) y `dashboard-layout.component.spec.ts` (2).
      Después de spec 078: **782 tests, 764 verde / 18 rojo. 0 fallos nuevos.** Los 3 ficheros de
      arriba quedaron **en verde** (stub local de `SwPush` en sus TestBeds — decisión de
      implementación, cubre el hueco de spec 077). Los 18 rojos restantes son todos preexistentes y
      fuera de alcance: `app.spec.ts` (2) y `auth.service.spec.ts` (10) — mismo `SwPush` de spec 077,
      no tocados por acuerdo (stub solo local); `menu.service.spec.ts` (1), `tenant.service.spec.ts`
      (3), `transfer-details-step.component.spec.ts` (1) — ajenos; y
      `pos-checkout-panel.component.spec.ts` "T032 Imprimir Pre-cuenta" (1) — ya rojo en la línea
      base, sin relación con esta spec. +45 tests nuevos, todos verde.
- [ ] T034 [P] Verificación cruzada de no regresión
      ([quickstart.md §"Verificación cruzada de no regresión"](./quickstart.md)): la grilla
      responsive de 3 variantes, la barra superior operativa, los contadores de ocupación y
      "Atendido por" de spec 076 se ven y se comportan igual (FR-037); el vocabulario de estados y
      la paleta de colores no cambian (FR-038); ninguna regla de cobro / facturación / creación de
      pedidos cambia — cobrar un pedido de Domicilio emite la misma venta que antes (FR-039); ninguna
      acción (buscar F2, filtrar, seleccionar, crear pedido F3, cobrar, liberar mesa) deja de estar
      disponible en los tres anchos (FR-040, SC-008); la pestaña "🔔 Pagos por confirmar"
      (`app-payment-validation-block`) sigue funcionando y sin recorte en los tres anchos tras la
      reestructura de US3. Dejar constancia en el commit
      → **Parcial (automatizado, 2026-09-07).** No regresión de código confirmada: `pos-tables-panel`,
      `pos-terminal-header`, `STATUS_META` (vocabulario + paleta) y las reglas de cobro/facturación/
      creación de pedidos **no están en el diff** (el cambio de store se limita a `toOrderCardView()`
      y a que `orderSubtotal()` prefiera `discounted_line_total` cuando el backend lo persiste —
      display-only, más preciso, sin recálculo). Suite completa: 0 fallos nuevos (ver T033). El
      `@switch 'validar-pago'` queda acotado con `flex-1 min-h-0 overflow-y-auto` (test de US3).
      **Pendiente la parte visual** (grilla 4/3/2, contadores, "Atendido por", recorrido de acciones
      en los tres anchos) — requiere navegador (ver T036).
- [X] T035 En `src/app/modules/tables/pages/table-sessions.component.ts` y
      `src/app/modules/tables/components/pos-order-panel.component.ts`: `grep` de `overflow-y-auto` /
      `overflow-x` tras US3–US5 y confirmar que **no** queda ningún contenedor de scroll anidado
      redundante ni ningún `overflow-x-auto` que reintroduzca el scroll horizontal de página
      (research.md D4). Si algún `overflow-x-auto` legítimo permanece (p. ej. la `<nav>` de pestañas,
      línea 70), dejar constancia de por qué se conserva
- [ ] T036 Ejecutar el recorrido manual completo de [quickstart.md](./quickstart.md) (Historias 1–6
      + verificación cruzada) contra `pos-backend` + `pos-heladeria` reales en los tres anchos de
      referencia (~1280px, ~900px, ~390px), confirmando SC-001 a SC-008. **Pendiente** — requiere
      navegador y datos sembrados (quickstart.md, Prerrequisitos); no ejecutable sin entorno visual

---

## Dependencies & Execution Order

### Phase Dependencies

- **Fase 1 (Setup)**: sin dependencias. **T002 (A-71) bloquea la Fase 8 (US6) y T005 (US1 cita `A-71` punto 2).**
- **Fase 2 (Foundational)**: vacía — ver nota en esa fase.
- **Fase 3 (US1)**: depende de la línea base (T001) y de T002 (T005 cita `A-71`). Sin dependencia de otras historias.
- **Fase 4 (US2)**: sin dependencia de otras historias. Comparte
  `table-sessions.component.ts` con US3 (regiones distintas — CTA vs. columna de detalle) →
  coordinar el merge.
- **Fase 5 (US3)**: sin dependencia de otras historias. Comparte `table-sessions.component.ts` con
  US2 y `pos-checkout-panel.component.ts` con US4.
- **Fase 6 (US4)**: **depende de US3** (T018/T019 — la columna acotada). Comparte
  `pos-order-panel.component.ts` con US5 y `pos-checkout-panel.component.ts` con US3.
- **Fase 7 (US5)**: **depende de US4** (T023 — mismo archivo `pos-order-panel.component.ts`;
  US4 reparte el alto, US5 reubica el bloque de domicilio).
- **Fase 8 (US6)**: **depende de T002 (A-71 registrada)**. Independiente de US1–US5 (archivos
  `layout.service.ts`, `dashboard-layout.component.ts` — sin solape).
- **Fase 9 (Polish)**: depende de todas las historias en alcance que se vayan a entregar.

### User Story Dependencies

- **US1 (P1)** — MVP. Totalmente independiente (1 línea + tests).
- **US2 (P1)** — independiente. Ruta nueva + CTA fuera del guard + lectura de `?tipo=`.
- **US3 (P1)** — independiente. Reestructura de la columna de detalle.
- **US4 (P2)** — **depende de US3**. Reparto de alto: consecuencia directa de la columna acotada.
- **US5 (P2)** — **depende de US4** (mismo archivo). Fila compacta de domicilio.
- **US6 (P2)** — independiente en código, pero **requiere `A-71` (T002)** antes de empezar.

### Within Each User Story

- Los tests que preceden a la implementación (T003→T005, T008/T009→T010–T014, T016→T017/T018,
  T021/T022→T023/T024, T026→T027, T029→T030/T031) deben **fallar antes** de implementar.
- En US2: la ruta (T010) antes del CTA que navega a ella (T011).
- En US3: quitar el scroll exterior (T017) antes de reestructurar el subárbol (T018).
- En US4/US5: US3 y US4 terminadas antes de tocar el mismo archivo aguas abajo.
- La tarea de recorrido manual de `quickstart.md` cierra cada fase de historia.

---

## Parallel Opportunities

- **Fase 1**: T001 (`ng test` en `pos-heladeria`) y T002 (`A-71` en `pos-specs`) en paralelo —
  repos/archivos distintos.
- **US1, US2, US3 y US6** pueden avanzar en paralelo con desarrolladores distintos: US1 toca solo
  `pos-terminal.store.ts` + `order-summary-card.component.ts`; US6 solo el shell de layout; US2 y
  US3 comparten `table-sessions.component.ts` (regiones distintas — coordinar el merge).
- **US4 → US5** es secuencial (mismo archivo `pos-order-panel.component.ts`).
- Dentro de cada historia, los tests marcados `[P]` (T003/T004, T008/T009, T021/T022) corren en
  paralelo entre sí.

### Ejemplo — reparto por desarrollador

```
Dev A: US1 → US3 → US4 → US5   (la cadena del panel: store + columna de detalle + densidad)
Dev B: US2                      (CTA en las 3 pestañas + ruta sin :tableId + ?tipo=)
Dev C: US6                      (menú de navegación en tablet, tras A-71)
```

### Ejemplo — tests en paralelo, Historia 2

```bash
Task T008: "table-sessions.component.spec.ts — CTA visible en las 3 pestañas + navegación con ?tipo="
Task T009: "manual-order-page.component.spec.ts — ngOnInit lee ?tipo= y llama setOrderTypeTab una vez"
# Luego T010 (ruta) → T011/T012/T013 (CTA) y T014 (ngOnInit).
```

---

## Implementation Strategy

### MVP (solo US1)

1. Fase 1 (Setup) — línea base de tests; `A-71` (T002) se puede registrar ya (barato, desbloquea US6).
2. Fase 3 (US1) — `toOrderCardView()` suma `delivery_fee`; tests del `totalLabel`.
3. **DETENERSE Y VALIDAR**: `quickstart.md` Historia 1 — la tarjeta de Domicilio muestra el total
   real y coincide con el panel de cobro (SC-001).
4. Desplegar — resuelve el único defecto de los siete que afecta dinero visible.

### Entrega incremental

1. Setup → línea base.
2. + US1 → la tarjeta de Domicilio muestra el total real (MVP). Validar SC-001. Desplegar.
3. + US2 → crear pedido desde cualquier pestaña, con tipo preseleccionado y botón claro. Validar
   SC-002/SC-003. Desplegar.
4. + US3 → panel de detalle completo en los tres anchos, sin scroll horizontal. Validar SC-004.
   Desplegar.
5. + US4 → la lista de productos recibe el alto disponible (≥ 4 / ≥ 2). Validar SC-005. Desplegar.
6. + US5 → información del domicilio compacta junto al estado. Validar SC-006. Desplegar.
7. + US6 → menú de navegación colapsable en tablet en toda la app (requiere `A-71`). Validar
   SC-007. Desplegar.
8. Fase 9 (Polish) sobre lo entregado — no regresión (FR-037–FR-040, SC-008).

Cada incremento agrega valor sin romper el anterior. US1, US2 y US6 pueden adelantarse en cualquier
orden; US3 → US4 → US5 van encadenadas.

---

## Notes

- `[P]` = archivos distintos, sin dependencia de una tarea sin terminar.
- La etiqueta `[Story]` mapea cada tarea a su historia para trazabilidad (Principio XII).
- Un commit por tarea (o grupo lógico), citando el `FR`/contrato; los commits de la **Fase 8**
  citan además `A-71`.
- Verificar que los tests fallan antes de implementar donde el test precede a la implementación.
- Detenerse en cada checkpoint para validar la historia de forma aislada y en los tres anchos.
- Evitar: llamar `checkout-preview` una vez por tarjeta (Assumption de spec.md — el total se compone
  en memoria, research.md D1); tocar la grilla de 3 variantes, la barra superior, los contadores o
  "Atendido por" de spec 076 (FR-037); cambiar el vocabulario de estados o la paleta (FR-038);
  cambiar cualquier regla de cobro / facturación / creación de pedidos (FR-039); truncar la
  dirección del domicilio o esconderla tras una interacción (FR-028); introducir un sistema de
  breakpoints nuevo o un control de menú nuevo solo para tablet (FR-036, research.md D7); recalcular
  cualquier venta ya emitida — `delivery_fee` nulo se trata como cero (Principio VII, spec.md Edge
  Cases); convertir el panel de detalle en un patrón maestro-detalle nuevo — el colapso a una vista
  por vez ya existe, solo se acota (spec.md, Out of Scope; research.md D4).
