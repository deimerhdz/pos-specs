# Implementation Plan: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

**Branch**: `048-pestanas-pago-pendiente-pedido` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/048-pestanas-pago-pendiente-pedido/spec.md`

## Summary

Sobre la Terminal de Mesas (`pos-heladeria`, módulo `tables`): agregar dos computeds nuevos en
`pos-terminal.store.ts` — `hasPendingAndActiveOrders` (¿la mesa seleccionada tiene a la vez algún
pago pendiente de confirmar y algún pedido pagado/activo?) y `effectiveCentralView` (qué debe
renderizar el panel central: el `centralPanelTab` elegido por el cajero cuando hay ambos, o
`centralState()` sin cambios en cualquier otro caso) — más un signal nuevo, `centralPanelTab`, que
guarda cuál de las dos pestañas está activa (por defecto `'validar-pago'`, se reinicia junto con el
resto del estado transitorio de la selección en `resetTransient()`). En
`table-sessions.component.ts`, el `@switch` de contenido pasa de leer `store.centralState()` a leer
`store.effectiveCentralView()` (mismo cuerpo, sin cambios) y la etiqueta del encabezado gana una
rama nueva que pinta dos botones de pestaña en vez del texto plano cuando
`hasPendingAndActiveOrders()` es verdadero. Ningún componente hijo (`payment-validation-block`,
`pos-order-panel`, `pos-checkout-panel`) cambia — se siguen montando exactamente igual que hoy, solo
que ahora también coexisten en el tiempo (uno visible, el otro a una pestaña de distancia) en vez de
excluirse. Sin cambios de backend, sin cambios de modelo de datos.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow `@if`/`@for`/`@switch`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x, sin dependencias nuevas — el fix agrega dos `computed` y un `signal` sobre estado ya existente

**Storage**: N/A — esta spec no agrega ni modifica entidades de backend ni de frontend (spec.md, Key Entities); no se toca ningún endpoint

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados `*.service.spec.ts`/`*.component.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio, pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; alternar pestañas es un cambio de signal local, sin llamada de red

**Constraints**: Cero cambios de comportamiento en los tres bloques reutilizados (`payment-validation-block`, `pos-order-panel`, `pos-checkout-panel`) ni en `centralState()` en sí (spec.md, Out of Scope); no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` (0 en `pos-heladeria/src/`, confirmado igual que en specs 045/046/047); no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Dos archivos de producción — `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (signal `centralPanelTab`, computeds `hasPendingAndActiveOrders`/`effectiveCentralView`, un ajuste de una línea en `resetTransient()`) y `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` (encabezado con rama de pestañas nueva, `@switch` de contenido apuntando a `effectiveCentralView()`). Tests nuevos en `pos-terminal.store.spec.ts` y `table-sessions.component.spec.ts` (o el archivo de test que ya exista para esa página, a confirmar en research.md). Ningún otro componente ni endpoint de backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/048-pestanas-pago-pendiente-pedido/spec.md` existe, sin `[NEEDS CLARIFICATION]` pendiente (la decisión de diseño ya se resolvió con el dueño del producto antes de escribir la spec), antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — el spec documenta explícitamente que esta es una corrección de un vacío de spec 036 (FR-005), consecuencia directa de spec 047; no reabre ninguna regla de precio, inventario o facturación. No requiere entrada nueva en `registro-de-anomalias.md` más allá de referenciar las specs 036/047 ya existentes.
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (gate trivial, mismo hallazgo que specs 045/046/047).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el único comportamiento nuevo (pestañas cuando coexisten pago pendiente y pedido activo) está definido en spec.md, FR-001 a FR-006.
- **Principio V (No refactors oportunistas)** ✅ — el plan solo agrega estado nuevo y ajusta el punto donde el template decide qué renderizar; no se reescribe `payment-validation-block.component.ts`, `pos-order-panel.component.ts` ni `centralState()` en sí.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (navegación/estado de UI en dos archivos), sin migración de datos ni cambio de arquitectura ni de backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta ya cerrada; el fix es puramente de navegación entre dos vistas que ya existen.
- **Principio VIII (Evolución del modelo de datos)** N/A — spec.md, Key Entities: "no se le agregan ni modifican entidades de datos".
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` + `/speckit-implement`: agregar cobertura para `hasPendingAndActiveOrders`/`effectiveCentralView`/el reinicio de `centralPanelTab` en `pos-terminal.store.spec.ts`, y para el renderizado de las pestañas/el `@switch` actualizado en el test de `table-sessions.component.ts` (research.md confirmará si ya existe ese archivo de test o si hay que crearlo).
- **Principio XI (Negocio vs. técnico)** ✅ — la causa raíz es un hallazgo técnico (research.md); la única decisión de negocio (pestañas vs. vista combinada) ya quedó registrada en spec.md, Clarifications, tomada por el dueño del producto antes de este plan.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reporte con captura) → Spec 048 (amend spec 036 FR-005, consecuencia de spec 047) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: research.md, data-model.md y contracts/ui-store-contract.md
confirman que el diseño no agrega dependencias nuevas (Principio IX), no modifica el modelo de datos
de backend (Principio VIII, N/A), no toca ningún characterization test (Principio III), y no altera
ningún dato histórico (Principio VII). Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/048-pestanas-pago-pendiente-pedido/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/           # Fase 1 (/speckit-plan) — contrato de UI/store, sin API nueva
│   └── ui-store-contract.md
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/modules/tables/
├── services/
│   └── pos-terminal.store.ts        # HOY: `centralState()` (~447) resuelve un único valor
│                                      # excluyente → SE AGREGAN: signal `centralPanelTab`
│                                      # (`'validar-pago' | 'pedido'`, por defecto `'validar-pago'`),
│                                      # computed `hasPendingAndActiveOrders` (usa
│                                      # `pendingOfSelectedTable()` + el `ordersOfTable(tableId)`
│                                      # privado ya existente), computed `effectiveCentralView`
│                                      # (delega en `centralPanelTab()` cuando hay ambos, si no
│                                      # devuelve `centralState()` tal cual); `resetTransient()`
│                                      # (~949) gana una línea para reiniciar `centralPanelTab` al
│                                      # cambiar de selección
└── pages/
    └── table-sessions.component.ts  # HOY: encabezado con `@switch (store.centralState())` de solo
                                       # texto (~104-110) y `@switch (store.centralState())` de
                                       # contenido (~119-149) → el encabezado gana una rama
                                       # `@if (store.hasPendingAndActiveOrders())` con dos botones de
                                       # pestaña; el `@switch` de contenido pasa a leer
                                       # `store.effectiveCentralView()` (cuerpo sin cambios)
```

**Structure Decision**: Se modifican in-place dos archivos ya existentes (`pos-terminal.store.ts`,
`table-sessions.component.ts`), sin crear ningún componente nuevo — las pestañas son botones simples
en el encabezado ya existente del panel central, reutilizando los tres bloques hijos
(`payment-validation-block`, `pos-order-panel`, `pos-checkout-panel`) exactamente como se montan hoy.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
