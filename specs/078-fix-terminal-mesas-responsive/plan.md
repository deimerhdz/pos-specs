# Implementation Plan: Correcciones responsive y de presentación de la Terminal de Mesas

**Branch**: `078-fix-terminal-mesas-responsive` | **Date**: 2026-09-07 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/078-fix-terminal-mesas-responsive/spec.md`

## Summary

Siete correcciones sobre la Terminal de Mesas ya rediseñada por spec 076, **todas de frontend**
(`pos-heladeria`): cero cambios de backend, cero endpoints, cero migraciones, cero peticiones
nuevas al servidor (spec.md, Out of Scope y Assumptions). Seis son defectos de presentación/
responsive y una — el total de la tarjeta de Domicilio — es un defecto que le muestra al cajero un
importe menor del que realmente se cobra. Dos puntos introducen además comportamiento nuevo acotado
y autorizado por el negocio (tipo de pedido preseleccionado al crear desde una pestaña; colapso del
menú de navegación en tablet, en toda la app).

Las correcciones se agrupan en seis piezas verificables por separado, una por historia de usuario:

1. **Total de la tarjeta de Domicilio (US1, FR-001–FR-006)** — `toOrderCardView()`
   (`pos-terminal.store.ts:846-858`) arma `totalLabel` con `orderSubtotal(o)` (solo productos, ya
   con el descuento por promoción que el backend dejó en `discounted_unit_price` de cada línea).
   Se le suma `o.delivery_fee`, de modo que la tarjeta muestre el mismo `total` que
   `GET /orders/{id}/checkout-preview` (spec 073: `max(0, subtotal − descuento + domicilio)`; sin
   impuesto ni propina para estos tipos de pedido, igual que el contrato de spec 073). Revierte de
   forma trazable la decisión de [spec 059 `data-model.md`](../059-terminal-mesas-carga-y-pedidos/data-model.md)
   ("la tarjeta muestra `orderSubtotal()`, no incluye el domicilio, no lo pidió el spec"). Es una
   **corrección de un defecto de presentación** — la regla "el total de un domicilio incluye el
   valor del domicilio" ya está decidida y autorizada por spec 056 para el cobro (research.md D1).

2. **Botón de crear pedido en todas las pestañas + texto en móvil + tipo preseleccionado (US2,
   FR-007–FR-015)** — hoy el CTA "+ Crear pedido nuevo" solo se pinta con
   `store.orderTypeTab() === 'mesas'` (`table-sessions.component.ts:87`) y en móvil solo muestra el
   ícono `+` (`hidden sm:inline` en la etiqueta, `table-sessions.component.ts:106`). Se saca el CTA
   del guard de pestaña, se le da etiqueta de texto visible en los tres anchos, y se navega a la
   vista de armado manual pasando el tipo de la pestaña de origen como preselección. Requiere una
   **ruta sin `:tableId`** para Domicilio/Para llevar (que no exigen mesa —
   `createManualOrderFromDraft()`, `pos-terminal.store.ts:1360-1408`) y un parámetro de tipo que
   `manual-order-page.component.ts` lee en `ngOnInit` → `store.setOrderTypeTab(...)`; el tipo sigue
   siendo editable dentro del formulario (research.md D2, D3).

3. **Panel de detalle adaptado al ancho (US3, FR-016–FR-021)** — la columna de detalle de
   `table-sessions.component.ts:138-239` anida un `overflow-y-auto` externo (línea 148) alrededor de
   `app-pos-order-panel` **y** `app-pos-checkout-panel`, y además cada panel intenta su propio
   `flex-1 overflow`. Ese doble contenedor de scroll es lo que deja contenido recortado y fuerza
   scroll horizontal de la página en tablet/móvil. Se rehace la columna como una **única columna
   flex acotada al alto disponible** (`min-h-0`, `min-w-0`, sin scroll horizontal de página), con
   scroll vertical **solo** interno. En tablet el patrón maestro-detalle ya colapsa en el
   breakpoint `lg` (línea 126/141); esta pieza lo deja explícito y conserva el retorno a la grilla
   en el mismo estado (research.md D4).

4. **Espacio para la lista de productos (US4, FR-022–FR-025)** — consecuencia directa de la pieza
   3: encabezado, fila de datos del domicilio, totales y acciones pasan a `shrink-0`, y la lista de
   productos (`pos-order-panel.component.ts:152`) queda como la **única** región
   `flex-1 min-h-0 overflow-y-auto` del panel. ≥ 4 productos visibles en escritorio y tablet, ≥ 2 en
   móvil (SC-005). Mismo trabajo aplica al panel de cobro apilado debajo (research.md D5).

5. **Información del domicilio en línea (US5, FR-026–FR-030)** — el bloque de dirección/teléfono/
   valor del domicilio (`pos-order-panel.component.ts:106-114`, hoy `flex gap-2` + `space-y-0.5`
   contradictorios, apilado bajo la cabecera) pasa a una fila compacta junto a la insignia de
   estado del pedido; la dirección se muestra completa envolviendo en 2–3 líneas (sin `truncate`);
   el valor del domicilio de esa fila es el **mismo** `delivery_fee` que suma el total de la US1
   (research.md D6).

6. **Menú de navegación global en tablet (US6, FR-031–FR-036)** — `LayoutService`
   (`layout.service.ts:6,19-21`) usa hoy el umbral `md` (768px) para decidir "slide-over con
   backdrop" vs "panel colapsable de escritorio". Se mueve ese umbral a `lg` (1024px):
   `DESKTOP_BREAKPOINT_PX` 768→1024 y el valor inicial de `sidebarOpen` a `>= 1024`; en
   `dashboard-layout.component.ts` el margen `md:ml-64`→`lg:ml-64` (línea 40), el backdrop
   `md:hidden`→`lg:hidden` (línea 23) y el auto-cierre por navegación `window.innerWidth < 768`
   →`< 1024` (línea 92); `sidebar.component.ts` no cambia (su `translate` ya depende solo de
   `sidebarOpen()`). Aplica en toda la aplicación (research.md D7). **Cambio de comportamiento
   fuera de la Terminal de Mesas** — requiere entrada en `registro-de-anomalias.md` antes de
   implementar (ver Constitution Check, Principio II).

7. **No regresión (FR-037–FR-040)** — la grilla responsive de 3 variantes, la barra superior
   operativa, los contadores de ocupación, "Atendido por", el vocabulario de estados y la paleta de
   colores de spec 076 no se tocan; ninguna acción (buscar, filtrar, seleccionar, crear, cobrar)
   deja de estar disponible en ningún ancho.

El detalle de cada decisión técnica está en [research.md](./research.md) (D1–D8).

## Technical Context

**Language/Version**: TypeScript ~5.9.2 (Angular 21 standalone components + `signal`/`computed`,
sin `NgModule`). Sin cambio de versión.

**Primary Dependencies**: `@angular/*` ^21.1.0 y Tailwind CSS ^4.1.12 (utilidades responsive
`sm`/`md`/`lg`/`xl` ya en uso en esta misma pantalla). **Ninguna dependencia nueva** en ningún
repositorio (Principio IX — nada que justificar).

**Storage**: N/A. Esta spec no toca el modelo de datos: sin entidades, sin campos, sin columnas,
sin migración (spec.md, Out of Scope; Constitution Check, Principio VIII — no aplica). Usa
`DiningOrder.delivery_fee` (`dining.interface.ts:220`), que spec 056 ya expone y la Terminal ya
recibe.

**Testing**: `ng test` (Vitest ^4.0.8 + utilidades de testing de Angular), archivos `*.spec.ts`
junto a cada componente/store. Áreas afectadas y su archivo de test ya existente:
`pos-terminal.store.spec.ts`, `table-sessions.component.spec.ts`, `pos-order-panel.component.spec.ts`,
`pos-checkout-panel.component.spec.ts`, `manual-order-page.component.spec.ts`,
`order-summary-card.component.spec.ts`, `dashboard-layout.component.spec.ts`,
`sidebar.component.spec.ts`. **Ningún archivo de esas áreas lleva hoy el prefijo
`"CONGELA comportamiento actual:"`** (verificado por `grep -rln "CONGELA comportamiento actual"
src/app/modules/tables src/app/modules/dashboard/layout` — sin resultados) → no hay conflicto con
el Principio III.

**Target Platform**: Aplicación web (SPA Angular) servida al navegador; responsive dentro de la
misma build para escritorio (≥ 1024px), tablet (768–1023px) y móvil (< 768px) — los mismos tres
anchos y los mismos breakpoints `md`/`lg` que ya usa spec 076, sin sistema de breakpoints nuevo.

**Project Type**: Aplicación web de 2 repositorios ya en producción (`pos-backend` +
`pos-heladeria`). A diferencia de spec 073 (que tocó ambos), **esta spec toca solo `pos-heladeria`**.

**Performance Goals**: Ninguno nuevo — SC-001 a SC-008 son de percepción/uso (todo el contenido
dentro del área visible, sin scroll horizontal, ≥ N productos a la vez, el menú accesible en una
interacción). Se reutiliza el mismo mecanismo reactivo (`signal`/`computed`) ya presente.

**Constraints**:
- No introducir peticiones nuevas al servidor (spec.md, Assumptions) — el total de la tarjeta se
  compone localmente de datos ya cargados; **no** se llama `checkout-preview` una vez por tarjeta.
- Preservar el vocabulario de estados y la paleta ya establecidos (FR-038, spec 076 FR-029/FR-030).
- No cambiar ninguna regla de cobro, facturación ni creación de pedidos (FR-039).
- Reutilizar los breakpoints `md` (768px) / `lg` (1024px) ya usados (Assumptions).
- Principio VII — ningún pedido/venta ya emitido se toca; `delivery_fee` nulo (histórico anterior a
  spec 056) se trata como cero, igual que ya hace el cobro.

**Scale/Scope**: 1 pantalla (Terminal de Mesas) + el shell de navegación global. Frontend, ~9
archivos existentes a modificar, **cero archivos nuevos de componente**:
- `services/pos-terminal.store.ts` — `toOrderCardView()` suma `delivery_fee` al total de la tarjeta
  (US1); el CTA de crear pedido no vive aquí, pero `newOrderTableId()`/`setOrderTypeTab()` se
  reutilizan tal cual (US2).
- `pages/table-sessions.component.ts` — CTA de crear pedido fuera del guard de pestaña, con
  etiqueta a todos los anchos y destino con tipo preseleccionado (US2); reestructura de la columna
  de detalle a una sola columna flex acotada (US3/US4).
- `components/pos-order-panel.component.ts` — fila compacta de datos del domicilio (US5); regiones
  fijas (`shrink-0`) vs. lista de productos (`flex-1`) (US4).
- `components/pos-checkout-panel.component.ts` — encaje del panel apilado en la nueva columna
  acotada, sin scroll propio que compita con el del panel (US3/US4).
- `pages/manual-order-page.component.ts` — lee el tipo preseleccionado de la ruta en `ngOnInit`
  (US2).
- `components/order-summary-card.component.ts` — sin cambios de forma esperados; solo recibe un
  `totalLabel` ya distinto (US1). Se revisa por si el texto "Total" necesita nota.
- `modules/dashboard/layout/layout.service.ts` — umbral 768→1024 (US6).
- `modules/dashboard/layout/dashboard-layout.component.ts` — clases `md:`→`lg:` y auto-cierre por
  navegación (US6).
- `modules/dashboard/routes.ts` — ruta de armado manual sin `:tableId` para Domicilio/Para llevar
  (US2).

El árbol de tareas concreto (qué línea cambia en cada archivo) es responsabilidad de `tasks.md`
(`/speckit-tasks`), no de este plan.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md) aprobada, solicitada por el dueño el 2026-09-07 con dos sesiones de Clarifications el mismo día (ver "Autorización de negocio" y "Clarifications"). |
| II. El comportamiento existente sigue protegido | ⚠️→✅ | US1, US3, US4, US5 son **correcciones de defectos de presentación** (spec.md lo declara explícito; restauran el comportamiento pretendido) — no abren anomalía. US2 (tipo preseleccionado) es **comportamiento nuevo aditivo**: hoy el armado manual abre sin tipo; no reemplaza ninguna regla de negocio, solo fija un valor inicial editable (FR-011) → Principio IV, sin anomalía. **US6 sí cambia comportamiento fuera de la Terminal de Mesas** (a ancho de tablet, todas las pantallas ocultan el menú de navegación) → requiere entrada **`A-71`** en `specs/000-reconocimiento/registro-de-anomalias.md` **antes** de implementar la pieza 6 (spec.md §"Impacto…" punto 3). Esa misma entrada deja además constancia del defecto de visualización del total de la tarjeta de Domicilio (US1, spec.md §"Impacto…" punto 1). Bloqueante solo para las tareas de US6. |
| III. Los characterization tests protegen el comportamiento heredado | ✅ | Verificado (grep) que ningún `*.spec.ts` de `src/app/modules/tables` ni de `src/app/modules/dashboard/layout` lleva el prefijo `"CONGELA comportamiento actual:"`. No hay ningún test protegido que esta spec deba tocar bajo autorización especial. |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ | US2 (tipo preseleccionado al crear desde una pestaña) y US6 (menú colapsable en tablet) son comportamiento nuevo autorizado por el spec; el criterio de éxito es la conformidad con `spec.md`, no la equivalencia con el pasado. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ con atención | La reestructura de la columna de detalle (US3/US4) toca `table-sessions.component.ts` y `pos-order-panel.component.ts`, que también sirven mesas y Domicilio/Para llevar (spec 059). Las tareas se limitan a la contención del scroll y el reparto de alto — **no** se reescribe la lógica de carrito, cobro ni selección. El `flex gap-2 space-y-0.5` contradictorio de la fila de domicilio (`pos-order-panel.component.ts:107`) se corrige porque US5 lo pide, no "ya que estamos". |
| VI. Evolución incremental | ✅ | Seis piezas 1:1 con las seis historias, priorizadas (P1×3, P2×3), verificables por separado (cada una con su sección en `quickstart.md`): el total de la tarjeta no depende del layout; el CTA en las tres pestañas no depende del menú de tablet; el menú de tablet es un cambio de shell aislado. Sin migración, sin cambio de arquitectura, sin mezclar clases de cambio. |
| VII. Compatibilidad con datos históricos | ✅ | Ninguna venta ni pedido emitido se recalcula: US1 corrige solo lo que **muestra** la tarjeta de un pedido **pendiente de cobro**; un `delivery_fee` nulo (pedido anterior a spec 056) se trata como cero — mismo criterio que ya aplica el cobro (spec.md, Edge Cases). |
| VIII. Evolución del modelo de datos | ✅ N/A | Cero entidades, campos, columnas y migraciones (ver Storage y [data-model.md](./data-model.md)). Solo se **lee** `DiningOrder.delivery_fee`, ya expuesto por spec 056. |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva. Tailwind y Angular ya están en el proyecto. |
| X. Verificación obligatoria | Pendiente de Fase 2 | Cada historia necesita sus `*.spec.ts` (nuevos/actualizados) más la validación manual responsive de [quickstart.md](./quickstart.md) en los tres anchos. No bloquea el gate de este plan. |
| XI. Decisiones de negocio frente a técnicas | ✅ | Las decisiones de negocio (la tarjeta muestra el total real; el tipo llega preseleccionado; el menú se colapsa en tablet en toda la app; la dirección nunca se trunca) ya están tomadas en `spec.md` por el dueño. Las decisiones técnicas (dónde se compone el total de la tarjeta, ruta sin `:tableId`, contención del scroll, umbral de breakpoint) están en `research.md`, sin mezclarse con las de negocio. |
| XII. Trazabilidad | ✅ | Necesidad (siete defectos observados por el dueño al usar spec 076) → spec 078 (FR-001…FR-040) → 2 sesiones de Clarifications (2026-09-07) → este plan → research.md (D1–D8) → data-model.md (sin cambios) → contracts/ui-terminal-mesas-fixes.md → quickstart.md → `A-71` (US6 + nota US1) → tasks.md → tests. **US1 revierte de forma explícita** la decisión registrada en `spec 059 data-model.md` (tarjeta = solo productos) — cadena cerrada. |
| XIII. Todo en español de Colombia | ✅ | Este plan y los artefactos de Fase 0/1 se redactan en español de Colombia. Los textos de UI nuevos/ajustados (etiqueta del botón de crear pedido en móvil — reutiliza "Crear pedido nuevo", spec 076 FR-027) no introducen ningún término nuevo (FR-014). |

### Resultado del gate

Sin violaciones que exijan `Complexity Tracking`. El único punto ⚠️ (Principio II, US6) se resuelve
con un ítem de proceso — registrar `A-71` antes de implementar la pieza 6 —, no con una excepción a
la Constitución.

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar `research.md` (D1–D8), `data-model.md`, `contracts/ui-terminal-mesas-fixes.md`
y `quickstart.md`. **Sin violaciones nuevas.**

- **VIII confirmado — cero esquema**: `data-model.md` cierra que no hay entidad, campo, columna ni
  migración; solo se **lee** `DiningOrder.delivery_fee` (ya expuesto por spec 056). No hay
  estrategia de migración/rollback que declarar más allá de "revertir los commits de frontend".
- **II con más precisión**: el diseño de Phase 1 **no amplió** lo que cambia de comportamiento
  existente. US1/US3/US4/US5 quedan como correcciones de presentación (research.md D1, D4, D5, D6
  atacan la causa raíz, no el síntoma). US2 es aditivo (D2/D3: ruta y query param nuevos, ninguna
  regla de creación tocada). US6 (D7) sigue siendo el único punto que cambia comportamiento fuera
  de la Terminal → `A-71` redactada en `contracts/ui-terminal-mesas-fixes.md`, bloqueante solo para
  las tareas de esa pieza.
- **V confirmado con evidencia**: research.md D3 registra y **acota** la única "tentación" — el
  doc-comment obsoleto de `ManualOrderPageComponent` ("Domicilio deshabilitada") se corrige porque
  US2 toca ese archivo y el comentario mentiría sobre el cambio, no como refactor libre. Ninguna
  otra reescritura de lógica de carrito/cobro/selección entra en alcance.
- **VI confirmado**: los 7 contratos de UI (`C1`–`C7`) mapean 1:1 a las seis historias + no
  regresión; ninguno mezcla piezas. `quickstart.md` valida cada historia por separado y en los tres
  anchos.
- **XII confirmado**: `data-model.md` y `research.md` D1 dejan explícito que US1 **revierte** la
  fila `totalLabel` de `spec 059 data-model.md` — la cadena Necesidad→Spec→Decisión→Implementación
  queda trazable en ambos sentidos.
- **III confirmado**: sin tests `"CONGELA comportamiento actual:"` en las áreas afectadas
  (verificado por grep antes de Phase 0).

Ningún gate queda en rojo.

## Project Structure

### Documentation (this feature)

```text
specs/078-fix-terminal-mesas-responsive/
├── plan.md                               # Este archivo (/speckit-plan)
├── research.md                            # Fase 0 — decisiones D1–D8
├── data-model.md                          # Fase 1 — sin cambios de esquema (documenta qué se lee)
├── contracts/
│   └── ui-terminal-mesas-fixes.md         # Fase 1 — contratos de UI por historia + A-71
├── quickstart.md                          # Fase 1 — validación responsive por historia (1–6)
├── checklists/                             # (de /speckit-clarify, si aplica)
└── tasks.md                               # Fase 2 (/speckit-tasks — NO lo crea este comando)
```

### Source Code (repositorio existente, fuera de `pos-specs`)

`pos-specs` no contiene código. La implementación vive en `pos-heladeria`. Esta feature **no crea
directorios ni archivos nuevos** — solo modifica los ya existentes:

```text
../pos-heladeria/
└── src/app/modules/
    ├── tables/
    │   ├── pages/
    │   │   ├── table-sessions.component.ts        # US2: CTA de crear pedido en las 3 pestañas,
    │   │   │                                       #      etiqueta a todos los anchos, tipo preseleccionado
    │   │   │                                       # US3/US4: columna de detalle → 1 sola columna flex
    │   │   │                                       #      acotada (min-h-0/min-w-0), scroll solo interno
    │   │   ├── table-sessions.component.spec.ts
    │   │   ├── manual-order-page.component.ts       # US2: lee el tipo preseleccionado de la ruta
    │   │   └── manual-order-page.component.spec.ts
    │   ├── components/
    │   │   ├── pos-order-panel.component.ts         # US5: fila compacta de datos del domicilio junto
    │   │   │                                        #      al estado; dirección envuelve, no trunca
    │   │   │                                        # US4: regiones fijas shrink-0 vs. lista flex-1
    │   │   ├── pos-order-panel.component.spec.ts
    │   │   ├── pos-checkout-panel.component.ts       # US3/US4: encaje del panel apilado sin scroll
    │   │   │                                         #      propio que compita con el de la columna
    │   │   ├── pos-checkout-panel.component.spec.ts
    │   │   ├── order-summary-card.component.ts        # US1: recibe un totalLabel ya = total real
    │   │   │                                          #      (revisar si "Total" necesita nota)
    │   │   └── order-summary-card.component.spec.ts
    │   └── services/
    │       ├── pos-terminal.store.ts                 # US1: toOrderCardView() suma delivery_fee al total
    │       │                                          #      de la tarjeta de Domicilio (revierte spec 059)
    │       └── pos-terminal.store.spec.ts
    └── dashboard/
        ├── layout/
        │   ├── layout.service.ts                     # US6: DESKTOP_BREAKPOINT_PX 768→1024; inicial >=1024
        │   ├── dashboard-layout.component.ts          # US6: md:ml-64→lg:ml-64, md:hidden→lg:hidden,
        │   │                                          #      auto-cierre por navegación <768→<1024
        │   ├── dashboard-layout.component.spec.ts
        │   └── sidebar.component.spec.ts               # US6: cobertura del rango tablet (sidebar.ts no cambia)
        └── routes.ts                                  # US2: ruta de armado manual sin :tableId
```

**Structure Decision**: se reutiliza íntegramente la estructura de módulos ya existente en
`pos-heladeria` (patrón "Option 2: Web application" del template, materializado como repositorio de
frontend separado del backend). Cero archivos nuevos: las seis piezas son extensiones/ajustes de
componentes, del store y del shell de layout ya existentes, más una entrada de ruta adicional. El
detalle línea por línea es responsabilidad de `tasks.md`.

## Complexity Tracking

> Sin violaciones del Constitution Check que justificar — tabla omitida. El único punto ⚠️
> (Principio II, US6) se resuelve con un ítem de proceso (`A-71` antes de implementar la pieza 6),
> no con una excepción a la Constitución.
