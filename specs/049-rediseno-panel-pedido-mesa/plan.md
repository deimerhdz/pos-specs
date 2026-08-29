# Implementation Plan: Rediseño del panel de pedido de mesa — cliente, pedidos y cuenta

**Branch**: `049-rediseno-panel-pedido-mesa` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/049-rediseno-panel-pedido-mesa/spec.md`

## Summary

Sobre la Terminal de Mesas (`pos-heladeria`, módulo `tables`): en `pos-order-panel.component.ts` se
retira el control "+ Nuevo pedido" y su método asociado en el store (`newOrderOnTable()`, sin otros
llamadores), se reemplaza el input editable "Cliente" por texto de solo lectura integrado en una
nueva cabecera de una sola fila (mesa + chip de estado + cliente), y las pestañas por pedido pasan
de rotularse con el nombre del cliente a "Pedido N", precedidas por una pestaña nueva "Todos los
pedidos (N)" que ofrece una vista agregada de todos los pedidos de la mesa a la vez. Las filas
Subtotal/Descuento/Total se retiran del panel de pedido y se agregan, como resumen aditivo, al
panel de cuenta (`session-bill-panel.component.ts`), sumando columnas que `SessionBill.split` ya
trae. En `pos-terminal.store.ts` se extrae la construcción de líneas de `cartView()` a un método
reutilizable (`persistedItemsView`) para poder mostrar los ítems de **todos** los pedidos de la
mesa a la vez, se agrega un signal de vista (`showAllOrders`) y un computed (`ordersView`), y se
generalizan `marcarListo`/`voidPersistedCombo`/`avanzarItem` para operar sobre el pedido de
cualquier tarjeta visible, no solo el seleccionado. Sin cambios de backend, sin cambios de modelo
de datos, sin dependencias nuevas — ver `research.md` (decisiones D1-D7) y
`contracts/ui-store-contract.md` para el detalle exacto de cada miembro tocado.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow
`@if`/`@for`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x — sin dependencias nuevas; se reutilizan
`order-status.util.ts` (`hasPendingKitchenWork`, `kitchenStatusLabel`/`kitchenStatusClass`) y las
funciones ya existentes en `pos-terminal.store.ts` (`deriveTableStatus`, `STATUS_META`)

**Storage**: N/A — no se agregan ni modifican entidades de backend ni de frontend (data-model.md);
no se toca ningún endpoint de `pos-backend`

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados
`*.component.spec.ts`/`*.service.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio,
pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de
backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; todo el trabajo nuevo es recomputar signals
locales (sumas sobre datos ya cargados) y alternar un signal de vista, sin llamadas de red
adicionales

**Constraints**: Cero cambios en la lógica de cálculo de subtotal/descuento/total, en el mecanismo
de cobro/cierre de sesión, en cómo se crean pedidos por canal QR o por la vista dedicada de pedido
manual, y en los estados de cocina por ítem en sí mismos (spec.md, Assumptions); 0 tests con
prefijo `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (research.md, confirmado igual
que specs 045/046/047/048); no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Tres archivos de producción —
`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (retira `newOrderOnTable`;
agrega `showAllOrders`, `ordersView`, `selectedTableStatusMeta`, `persistedItemsView`,
`tableItemsView`; modifica `orderTabs` (etiqueta), `cartView`, `marcarListo`, `voidPersistedCombo`,
`avanzarItem`), `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts`
(cabecera, pestañas y contenido rediseñados) y
`pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts` (agrega
`billSummary` y dos filas). Un test existente
(`pos-order-panel.component.spec.ts`, caso "el descuento mostrado en el total es siempre $0 sin
promociones activas") debe moverse/adaptarse a `session-bill-panel.component.spec.ts`
(research.md, "Resumen de impacto en tests existentes"). Ningún otro componente ni endpoint de
backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/049-rediseno-panel-pedido-mesa/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (las 3 decisiones de alcance se resolvieron con el dueño del
  producto antes de escribir la spec), antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente el
  amend a spec 036 (retira "+ Nuevo pedido" y el resumen de totales de `pos-order-panel`, mueve el
  resumen a `session-bill-panel`) con autorización directa del dueño/desarrollador el 2026-08-28.
  No reabre ninguna regla de precio, inventario o facturación (research.md D1: solo se suma lo que
  el backend ya entrega); no aplica una nueva entrada en `registro-de-anomalias.md` (mismo criterio
  que specs 045/048, reordenamiento de UI).
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en
  `pos-heladeria/src/` (research.md, mismo hallazgo que specs 045/046/047/048).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (retirar
  "+ Nuevo pedido", migrar totales, pestañas "Todos los pedidos"/"Pedido N", cliente de solo
  lectura) está definido en spec.md, FR-001 a FR-015.
- **Principio V (No refactors oportunistas)** ✅ — la extracción de `persistedItemsView` (D4) y el
  retiro de `newOrderOnTable()` (D2) están directamente causados por FR-001/FR-009/FR-011, no son
  limpieza no relacionada; ningún otro archivo del módulo `tables` se toca sin necesidad.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (navegación/visualización de
  UI sobre tres archivos ya existentes del mismo módulo), sin migración de datos ni cambio de
  arquitectura ni de backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta ya
  cerrada; el resumen agregado del panel de cuenta suma columnas que el backend ya entrega para la
  cuenta vigente, no reconstruye nada histórico.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: sin entidades ni campos
  nuevos de backend; todo lo nuevo son vistas derivadas en memoria del cliente.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde los `describe` no afectados de
  `pos-order-panel.component.spec.ts`/`session-bill-panel.component.spec.ts`/
  `pos-terminal.store.spec.ts`; mover/adaptar el caso de descuento señalado en research.md; agregar
  cobertura nueva para `showAllOrders`/`ordersView`/`selectedTableStatusMeta` y la generalización de
  `marcarListo`/`voidPersistedCombo`/`avanzarItem` (research.md D6).
- **Principio XI (Negocio vs. técnico)** ✅ — las 3 decisiones de negocio (cliente siempre de solo
  lectura, sin reemplazo para "+ Nuevo pedido", estado por ítem conservado) ya quedaron registradas
  en spec.md, Clarifications, tomadas por el dueño del producto antes de este plan; las decisiones
  de este documento (D1-D7) son todas técnicas (cómo, no qué).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (dos capturas + pedido directo) → Spec 049 (amend
  spec 036) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` y `contracts/ui-store-contract.md`
confirman que el diseño no agrega dependencias nuevas (Principio IX), no modifica el modelo de
datos de backend (Principio VIII, N/A), no toca ningún characterization test (Principio III), y no
altera ningún dato histórico ni regla de cálculo de facturación (Principio VII) — únicamente
reubica y reagrupa datos que el backend ya entrega. Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/049-rediseno-panel-pedido-mesa/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D7
├── data-model.md         # Fase 1 (/speckit-plan) — vistas derivadas nuevas/modificadas
├── quickstart.md         # Fase 1 (/speckit-plan) — validación manual, 3 escenarios
├── contracts/            # Fase 1 (/speckit-plan) — contrato de UI/store, sin API nueva
│   └── ui-store-contract.md
└── tasks.md              # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/modules/
├── tables/
│   ├── services/
│   │   └── pos-terminal.store.ts            # Se retira `newOrderOnTable()` (~918); se agregan
│   │                                          # `showAllOrders` (signal), `ordersView`,
│   │                                          # `selectedTableStatusMeta`, `persistedItemsView`
│   │                                          # (privado, extraído de `cartView()` ~556-617) y
│   │                                          # `tableItemsView`; se modifican `orderTabs` (~500,
│   │                                          # etiqueta "Pedido N"), `cartView` (delega en
│   │                                          # `persistedItemsView`), `marcarListo` (~1245,
│   │                                          # `orderId` opcional), `voidPersistedCombo` (~1195,
│   │                                          # busca en `orders()`) y `avanzarItem` (~1267, busca
│   │                                          # en `tableItemsView`)
│   └── components/
│       ├── pos-order-panel.component.ts     # HOY: cabecera apilada + input "Cliente" + pestañas
│       │                                      # con "+ Nuevo pedido" + carrito + resumen de
│       │                                      # totales (~24-174) → SE REEMPLAZA por cabecera de
│       │                                      # una fila (mesa + chip de estado + cliente de solo
│       │                                      # lectura), pestañas "Todos los pedidos (N)"/
│       │                                      # "Pedido N" sin "+ Nuevo pedido", y contenido que
│       │                                      # alterna entre `store.ordersView()` (todas las
│       │                                      # tarjetas) y la tarjeta única del pedido
│       │                                      # seleccionado — sin las filas de totales
│       └── session-bill-panel.component.ts  # HOY: desglose por comensal + "Total" (~44-89) → SE
│                                              # AGREGA `billSummary` (computed) y dos filas
│                                              # "Subtotal"/"Descuento" antes de "Total"; sin
│                                              # cambios en el resto del componente (pago, cobro,
│                                              # modo readOnly)
└── orders/
    └── order-status.util.ts                 # Sin cambios — se reutiliza tal cual
                                               # `hasPendingKitchenWork` para la pastilla de estado
                                               # por tarjeta (research.md, D3)
```

**Structure Decision**: Se modifican in-place tres archivos ya existentes del módulo `tables`
(`pos-terminal.store.ts`, `pos-order-panel.component.ts`, `session-bill-panel.component.ts`), sin
crear ningún componente nuevo — las tarjetas de pedido y el resumen agregado son bloques de
template dentro de los componentes ya existentes, reutilizando en todo momento las utilidades ya
existentes de `order-status.util.ts` y del propio store.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
