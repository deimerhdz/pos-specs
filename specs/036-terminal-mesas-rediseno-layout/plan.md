# Implementation Plan: Rediseño de Layout de la Terminal de Mesas — Grilla de Mesas, Pagos por Confirmar y Menú Central

**Branch**: `036-terminal-mesas-rediseno-layout` | **Date**: 2026-08-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/036-terminal-mesas-rediseno-layout/spec.md`
(reemplaza el plan del 2026-08-25, escrito antes de que la sesión de clarificación del 2026-08-26
reemplazara la imagen de referencia original por dos prototipos y corrigiera el diseño objetivo)

## Summary

Reorganizar el layout de la pantalla "Terminal de Mesas" (`pos-heladeria`, módulo `tables`) para que
coincida con los dos prototipos de referencia, sin cambiar ninguna regla de negocio ya implementada
(specs 010, 011, 024, 028): (1) la grilla de mesas por estado de ocupación (`pos-tables-panel.component.ts`)
se conserva sin cambios de comportamiento, pero gana pestañas de tipo de orden "Mesas"/"Domicilios"/
"Para llevar" por encima de sus filtros ya existentes — las dos últimas quedan vacías porque no existe
hoy ninguna vía para crear órdenes de esos tipos; (2) se agrega una sección nueva "Pagos por confirmar"
debajo de la grilla que agrupa, en un listado aparte, todos los pagos pendientes de revisión ya
existentes (efectivo/transferencia), reutilizando el `computed` `pendingOrders` ya presente en
`pos-terminal.store.ts` sin duplicar la lógica de confirmación; (3) el panel central deja de mostrar el
catálogo como overlay de pantalla completa (`pos-catalog-drawer.component.ts`) y en su lugar muestra
por defecto la lista de ítems del pedido (`pos-order-panel.component.ts`, patrón ya existente), con un
botón "+ Agregar producto" que abre, embebida en el mismo panel, la grilla de catálogo con el buscador
por nombre nuevo (pedido explícitamente por el usuario) combinado con el filtro de categoría ya
existente; (4) se agrega un botón para colapsar/expandir el menú de navegación global del dashboard, lo
que requiere extender `sidebar.component.ts`/`dashboard-layout.component.ts` con un comportamiento de
colapso de escritorio que hoy no existe (hoy `sidebarOpen()` solo controla el slide-over móvil). El
panel derecho de cobro (`pos-checkout-panel.component.ts`) no cambia de comportamiento, solo de
disposición visual — esto incluye "Dividir la cuenta entre varias personas"
(`split-bill-panel.component.ts`) y "Facturar a nombre de" (`billingCustomerName`), que se descubrió
durante esta misma planeación que **ya están completamente implementados** (spec 010 / spec 011), no
son funcionalidad nueva a construir (ver spec.md, Clarifications, sesión durante `/speckit-plan`). Las
pestañas "Domicilios"/"Para llevar" se agregan vacías: construir la creación real de órdenes de esos
tipos requeriría un campo de backend y un flujo de "venta de mostrador" que no existen hoy en el
código, y se especifica en una spec futura independiente.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow `@if`/`@for`/`@switch`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x / `@angular/cdk` 21.2.x, `@tanstack/angular-query-experimental` v5 (data fetching), RxJS 7.8, Tailwind CSS v4 (utilidades inline en `template:`, sin `.scss` por componente)

**Storage**: N/A — esta spec no agrega ni modifica entidades de backend (spec.md, Key Entities); reutiliza las mismas fuentes de datos ya consumidas por `pos-terminal.store.ts` (API de mesas/órdenes/pagos y de catálogo)

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados `*.component.spec.ts` / `*.service.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio, pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; alternar entre lista de ítems y grilla de "+ Agregar producto", filtrar por nombre/categoría, y confirmar un pago desde "Pagos por confirmar" deben sentirse instantáneos (sin parpadeo ni recarga de red), igual que la interacción ya existente hoy

**Constraints**: Cero cambios de comportamiento en validación de pago QR, cobro manual, cálculo de cambio, envío a cocina, facturación, división de cuenta o cierre de mesa (specs 010, 024, 028, reforzado por FR-013/SC-005 de esta spec); no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` (no existen en el módulo `tables` del frontend, ver research.md); no se agregan dependencias nuevas (Principio IX) — la agregación de "Pagos por confirmar" y el filtro combinado de catálogo se implementan con signals/computed de Angular ya presentes en el store

**Scale/Scope**: Una sola pantalla (`table-sessions.component.ts` y sus componentes hijos bajo `src/app/modules/tables/`); toca directamente `pos-tables-panel.component.ts`, `pos-catalog-drawer.component.ts` (se retira su overlay), `pos-order-panel.component.ts` (punto de anclaje del panel central), un componente nuevo para "Pagos por confirmar", y `pos-terminal.store.ts`; en menor medida `sidebar.component.ts`/`dashboard-layout.component.ts`/`layout.service.ts` del módulo `dashboard`. `pos-checkout-panel.component.ts`, `split-bill-panel.component.ts` y `session-bill-panel.component.ts` solo reciben ajustes de disposición visual, no de lógica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/036-terminal-mesas-rediseno-layout/spec.md` existe y fue clarificado (dos veces: sesión inicial y sesión de actualización de prototipos 2026-08-26) antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — el spec documenta explícitamente que esto es una reorganización visual; ninguna regla de negocio de spec 010/024/028 cambia (FR-008/FR-009/FR-010/FR-013, SC-005). No se requiere entrada en `registro-de-anomalias.md` porque no hay decisión de negocio que cambie comportamiento, solo presentación. Las piezas nuevas de comportamiento (toggle de sidebar global, buscador por nombre, agregación de "Pagos por confirmar") son aditivas y de solo-lectura sobre datos ya existentes — no modifican ninguna regla existente.
- **Principio III (Characterization tests)** ✅ — no existen tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/app/modules/tables/` (confirmado por búsqueda); los únicos existen en el backend y esta spec no lo toca. Gate trivialmente satisfecho.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el único comportamiento nuevo (toggle de sidebar global, búsqueda por nombre en catálogo, agregación visual de pagos pendientes) está definido en el spec (FR-007, FR-004, FR-012).
- **Principio V (No refactors oportunistas)** ✅ — el plan solo toca los archivos necesarios para las 3 historias de usuario; no se incluye ninguna refactorización no relacionada (en particular, `split-bill-panel.component.ts`/`session-bill-panel.component.ts` NO se reescriben, solo se reubican visualmente si el layout lo exige).
- **Principio VI (Evolución incremental)** ✅ — el hallazgo de que "Domicilio"/"Para llevar" mezclarían nueva funcionalidad + migración de datos + layout llevó a acotar esta spec a un solo tipo de cambio (layout/presentación) y diferir el resto a una spec futura (spec.md, Clarifications).
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni datos históricos.
- **Principio VIII (Evolución del modelo de datos)** N/A — no hay cambios de modelo de datos en esta spec (spec.md, Key Entities: "no se le agregan atributos nuevos").
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia; toda la lógica nueva (agregación de pagos pendientes, filtro combinado de catálogo, colapso de sidebar) se implementa con signals/computed de Angular ya usados en el proyecto.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` + `/speckit-implement`: se exige mantener en verde `pos-checkout-panel.component.spec.ts`, `pos-order-panel.component.spec.ts`, `split-bill-panel.component.spec.ts`, `session-bill-panel.component.spec.ts` y `pos-terminal.store.spec.ts` ya existentes, y agregar specs nuevos para `pos-tables-panel.component.ts` (sin cobertura hoy) y el nuevo componente de "Pagos por confirmar".
- **Principio XI (Negocio vs. técnico)** ✅ — la decisión de negocio de acotar "Domicilios"/"Para llevar" a pestañas vacías, y el hallazgo técnico de que "Dividir la cuenta" ya existía (corrigiendo un supuesto erróneo de la clarificación anterior), quedaron ambos registrados explícitamente en el spec (Clarifications), no decididos unilateralmente en este plan.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (prototipos del usuario) → Spec 036 (dos sesiones de clarificación) → Decisiones (Clarifications) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: research.md, data-model.md y contracts/ui-store-contract.md
confirman que el diseño no agrega dependencias nuevas (Principio IX), no modifica el modelo de datos de
backend (Principio VIII, N/A), no toca ningún characterization test (Principio III), y mantiene el
comportamiento de cobro/validación/facturación/división de cuenta intacto (Principio II) — incluyendo
la corrección de alcance sobre "Dividir la cuenta" descubierta durante esta fase (Principio XI). Único
ajuste de alcance detectado durante el diseño: `sidebar.component.ts`/`dashboard-layout.component.ts`
necesitan honrar `sidebarOpen()` también en escritorio (hoy solo aplica en móvil) — es una extensión
acotada de un componente ya existente, no una nueva pieza de arquitectura. Gate sigue PASANDO sin
violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/036-terminal-mesas-rediseno-layout/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/           # Fase 1 (/speckit-plan) — contratos de UI/store, sin API nueva
│   └── ui-store-contract.md
└── tasks.md             # Fase 2 (/speckit-tasks) — el existente quedó desactualizado por
                          # el rediseño de prototipos del 2026-08-26; se regenera en el próximo
                          # /speckit-tasks
```

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de specs
(`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec (ver Constraints).

```text
pos-heladeria/src/app/modules/tables/
├── pages/
│   └── table-sessions.component.ts      # Layout raíz de la pantalla (fila flex: grilla+pagos | centro | derecha)
├── components/
│   ├── pos-tables-panel.component.ts    # HOY: grilla de mesas + filtro de ocupación → SE LE AGREGAN pestañas de tipo de orden (Mesas/Domicilios/Para llevar) por encima; el filtro de ocupación no cambia
│   ├── pending-payments-panel.component.ts  # NUEVO — sección "Pagos por confirmar"; agrega tarjetas por orden con pago pendiente de revisión, reutilizando los mismos métodos de confirmación del store
│   ├── pos-catalog-drawer.component.ts  # HOY: overlay de pantalla completa → SE EMBEBE dentro del panel central (se retira el wrapper `fixed inset-0 bg-black/40`); se agrega buscador por nombre
│   ├── pos-order-panel.component.ts     # HOY: lista de ítems del pedido + "+ Agregar producto" (estado `'pedido'`) — YA implementa el patrón objetivo del panel central; solo cambia cómo se monta el catálogo al pulsar "+ Agregar producto" (embebido, no overlay)
│   ├── pos-checkout-panel.component.ts  # Sin cambio de comportamiento; solo ajuste de disposición visual (FR-009); sigue renderizando split-bill-panel y session-bill-panel tal cual
│   ├── split-bill-panel.component.ts    # Sin cambio de comportamiento (spec 010, ya implementado) — solo reubicación visual si el layout lo exige (FR-010)
│   ├── session-bill-panel.component.ts  # Sin cambio de comportamiento — solo reubicación visual si el layout lo exige
│   ├── payment-validation-block.component.ts  # Sin cambio de comportamiento (bloque por mesa seleccionada, spec 028)
│   └── payment-attempt-review-panel.component.ts  # Sin cambio de comportamiento; se evalúa reutilizarlo en modo compacto dentro de pending-payments-panel (ver research.md §2)
└── services/
    └── pos-terminal.store.ts            # Se agregan: `orderTypeTab` (mesas/domicilios/para-llevar), `pendingPaymentsView` (une pendingOrders() + tables()), `catalogSearchText` y `catalogProductsFiltered` (combina con catalogCategoryId ya existente)

pos-heladeria/src/app/modules/dashboard/layout/
├── layout.service.ts        # Ya expone sidebarOpen()/toggle(); SIN cambios de API, se reutiliza tal cual
├── sidebar.component.ts     # SE EXTIENDE: hoy fuerza `md:relative md:translate-x-0` (visible siempre en escritorio); se hace ese comportamiento condicional a sidebarOpen() también en escritorio
└── dashboard-layout.component.ts  # Se ajusta el margen/ancho del contenido cuando el sidebar está colapsado en escritorio
```

**Structure Decision**: Se modifica in-place el módulo `tables` ya existente en `pos-heladeria` (no se
crea un módulo nuevo ni una segunda app), agregando un único componente nuevo
(`pending-payments-panel.component.ts`) para la sección "Pagos por confirmar" que no tiene equivalente
hoy. El único archivo compartido fuera de `tables/` que se toca es el trío `layout.service.ts`/
`sidebar.component.ts`/`dashboard-layout.component.ts` del módulo `dashboard`, porque el botón de
colapsar/expandir vive en la Terminal de Mesas pero controla el menú de navegación global (decisión
registrada en spec.md, Clarifications).

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
