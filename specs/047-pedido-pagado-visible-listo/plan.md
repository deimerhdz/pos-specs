# Implementation Plan: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

**Branch**: `047-pedido-pagado-visible-listo` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/047-pedido-pagado-visible-listo/spec.md`

## Summary

Corregir, en `pos-heladeria` (módulo `tables`), el filtro de frontend que decide qué pedidos cuentan
como "consumo vivo" de una mesa: `activeOrders` y `tableOrders(tableId)`
(`pos-terminal.store.ts`) excluyen hoy un pedido `'pagada'` en cuanto
`hasPendingKitchenWork(o)` se vuelve falso — justo al terminar de cocinar ("Marcar pedido listo") —
aunque la sesión de la mesa siga abierta. El fix retira ese conjunto de la condición en ambas
funciones: un pedido `'pagada'` (no cancelado) cuenta como activo para su mesa sin importar el
estado de cocina; la única autoridad sobre cuándo deja de contar es el backend
(`GET /orders?active_sessions_only=true`, que ya excluye un pedido pagado solo tras liberar la mesa
de verdad). Se revisó, uno por uno, cada consumidor de esas dos funciones
(`centralState`, `tablesView`, `orderTabs`, `resyncSelectedOrder`, `selectTable`, `ordersToCharge`,
`billOrphan`) y en ninguno la exclusión de un pedido pagado-y-listo era el comportamiento deseado —
en todos era exactamente el bug reportado. Sin cambios de backend, sin cambios de modelo de datos.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow `@if`/`@for`/`@switch`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x, sin dependencias nuevas — el fix retira una condición de un `computed`/método ya existente

**Storage**: N/A — esta spec no agrega ni modifica entidades de backend ni de frontend (spec.md, Key Entities); el endpoint `GET /orders?active_sessions_only=true` (`pos-backend`) no se toca, ya es la fuente de verdad correcta

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados `*.service.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio, pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; el fix es una simplificación de una condición booleana ya evaluada en cada `computed` reactivo, sin trabajo adicional

**Constraints**: Cero cambios de comportamiento para pedidos `'cancelada'`, `'recibida'`+`qr` pendientes de confirmar, o cualquier otro estado no relacionado con "pagada + cocina terminada" (spec.md, Out of Scope); no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` (0 en `pos-heladeria/src/`, confirmado igual que en specs 045/046); no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Un único archivo de producción, `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (dos funciones: `activeOrders` ~línea 377, `tableOrders(tableId)` ~línea 401, más el import de `hasPendingKitchenWork` en la cabecera y, opcionalmente, el comentario de la rama `'listo'` de `deriveTableStatus`); tests nuevos en `pos-terminal.store.spec.ts`. Ningún otro componente, servicio ni endpoint de backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/047-pedido-pagado-visible-listo/spec.md` existe, sin `[NEEDS CLARIFICATION]` pendiente, antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — el spec documenta explícitamente que esta es una corrección de un vacío de spec 035 (A-52), con justificación técnica y capturas del dueño del producto; no reabre ninguna regla de precio, inventario o facturación. No requiere entrada nueva en `registro-de-anomalias.md` más allá de referenciar la A-52 ya existente.
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (gate trivial, mismo hallazgo que specs 045/046).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el único comportamiento que cambia (un pedido `'pagada'` sigue contando como activo sin importar la cocina) está definido en spec.md, FR-001 a FR-005.
- **Principio V (No refactors oportunistas)** ✅ — el plan solo toca `activeOrders`/`tableOrders` y su import; no se reescribe `deriveTableStatus()` ni `hasPendingKitchenWork()` en sí (research.md §2), solo se refresca un comentario que cita una premisa que el fix vuelve obsoleta.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (corregir una condición de filtrado en un archivo), sin migración de datos ni cambio de arquitectura ni de backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta ya cerrada; el fix es puramente sobre qué se muestra en pantalla mientras una sesión de mesa sigue activa.
- **Principio VIII (Evolución del modelo de datos)** N/A — spec.md, Key Entities: "no se le agregan ni modifican entidades de datos".
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` + `/speckit-implement`: agregar cobertura nueva (research.md §3) para el caso `status: 'pagada'` a través de `tableOrders()`/`activeOrders()`/`centralState()`/`marcarListo()` — hoy sin cobertura, que es justo el gap que dejó pasar el bug — y confirmar que los `describe` existentes de `deriveTableStatus` (con `status: 'abierta'`) siguen en verde sin cambios de intención.
- **Principio XI (Negocio vs. técnico)** ✅ — la causa raíz y el fix son un hallazgo técnico (research.md), no una decisión de negocio nueva: el comportamiento correcto ya estaba definido por decisiones previas (spec 029 Historia 3, spec 035 A-52) que esta spec simplemente deja de bloquear.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reporte con capturas) → Spec 047 (amend spec 035 A-52) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: research.md y data-model.md confirman que el diseño no
agrega dependencias nuevas (Principio IX), no modifica el modelo de datos de backend (Principio
VIII, N/A), no toca ningún characterization test (Principio III), y no altera ningún dato histórico
ni el contrato de `GET /orders?active_sessions_only=true` (Principio VII). Gate sigue PASANDO sin
violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/047-pedido-pagado-visible-listo/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

No se genera `contracts/`: esta spec no expone ni consume ninguna API HTTP nueva ni modificada, y
no hay ningún contrato de UI/store nuevo que documentar más allá de simplificar una condición
booleana ya interna — el "contrato" relevante (qué consumidores dependen de `activeOrders`/
`tableOrders`) ya queda completamente enumerado en research.md.

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec (ver Technical
Context → Storage, y research.md §1).

```text
pos-heladeria/src/app/modules/tables/services/
├── pos-terminal.store.ts       # HOY: `activeOrders` (~377) y `tableOrders(tableId)` (~401) excluyen
│                                # un pedido 'pagada' en cuanto `hasPendingKitchenWork(o)` es falso →
│                                # SE QUITA ese conjunto de ambas condiciones; se actualizan sus
│                                # docblocks; se retira `hasPendingKitchenWork` del import (línea 34,
│                                # sin más consumidores tras el fix — `KITCHEN_NOT_READY` sigue vivo
│                                # en `kitchenReady()`/`ensureReadyToCharge()`, sin cambios); se
│                                # refresca, opcionalmente, el comentario de la rama `'listo'` de
│                                # `deriveTableStatus()` (~168-179), que hoy cita la premisa vieja
│                                # como la razón por la que puede llegar una orden 'pagada' con
│                                # ítems 'listo' — con el fix, esa razón cambia
└── pos-terminal.store.spec.ts  # Se agregan dos bloques de test nuevos (research.md §3); los
                                 # `describe` existentes (incluido `deriveTableStatus` con
                                 # `status: 'abierta'`) no se tocan
```

**Structure Decision**: Se modifica in-place un único archivo de producción ya existente
(`pos-terminal.store.ts`), sin crear ningún componente, servicio o módulo nuevo. Es la corrección
más acotada posible dado el diagnóstico: dos condiciones booleanas casi idénticas, un import, y
opcionalmente un comentario.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
