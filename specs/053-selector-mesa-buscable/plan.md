# Implementation Plan: Selector de mesa buscable en la creación de orden manual

**Branch**: `053-selector-mesa-buscable` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/053-selector-mesa-buscable/spec.md`

## Summary

Sobre la pantalla de creación de orden manual (`pos-heladeria`, módulo `tables`,
`manual-order-page.component.ts`): la rejilla de botones de mesas (post-052, dentro del panel
derecho) se reemplaza por el select buscable ya existente en el proyecto
(`SearchableSelectComponent`, `shared/searchable-select/`), alimentado por un `computed` nuevo
(`mesaOptions`) que mapea `store.tablesView()` a `{ id, label: "Mesa N — Estado", disabled }`. El
componente compartido gana un campo opcional `disabled?: boolean` en `SearchableSelectOption` (y la
guarda correspondiente en `selectOption()`) para poder mostrar mesas ocupadas visibles pero no
seleccionables — retrocompatible con sus 4 consumidores actuales. Sin cambios de backend, sin
dependencias nuevas — ver `research.md` (decisiones D1-D4).

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow
`@if`/`@for`, signals, `ControlValueAccessor`)

**Primary Dependencies**: `@angular/core` 21.1.x, `@angular/forms` (`FormsModule`, ya dependencia
del proyecto, se agrega a `imports` de `ManualOrderPageComponent` para poder usar `[ngModel]` sobre
`app-searchable-select`) — sin dependencias nuevas

**Storage**: N/A — no se agrega ni modifica ninguna entidad de backend ni de frontend
(data-model.md); no se toca ningún endpoint de `pos-backend`

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados
`*.component.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio,
pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de
backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; `mesaOptions` es un `computed` que mapea una
lista ya cargada (`tablesView()`), sin llamadas de red adicionales

**Constraints**: Cero cambios en la disponibilidad de mesas, en la lógica de `selectTable()`, en el
tipo de orden, el catálogo, el carrito o la confirmación de pedido (spec.md, FR-006); 0 tests con
prefijo `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (research.md, confirmado igual
que specs 045-052); no se agregan dependencias nuevas (Principio IX); el campo `disabled` nuevo en
`SearchableSelectOption` debe ser retrocompatible con sus 4 consumidores actuales (research.md D2)

**Scale/Scope**: Dos archivos de producción —
`pos-heladeria/src/app/shared/searchable-select/searchable-select.component.ts` (campo `disabled?:
boolean` en `SearchableSelectOption`, guarda en `selectOption()`, clase condicional en el `<li>`) y
`pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` (agrega `FormsModule` +
`SearchableSelectComponent` a `imports`, agrega `computed mesaOptions`, reemplaza el bloque de
rejilla de mesas por `<app-searchable-select>`). Tests ajustados en
`searchable-select.component.spec.ts` (cobertura nueva para `disabled`) y
`manual-order-page.component.spec.ts` (el caso de cambio de mesa se reescribe para interactuar con
el select en vez de botones `M{n}`; el resto de los 17 casos no cambia, research.md). Ningún otro
componente, servicio ni endpoint de backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/053-selector-mesa-buscable/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente, antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta el estado actual
  exacto (post-052) con autorización directa del dueño/desarrollador el 2026-08-28/29. No reabre
  ninguna regla de disponibilidad de mesas ni de negocio; no aplica una nueva entrada en
  `registro-de-anomalias.md` (mismo criterio que specs 045/048/049/051/052, cambio de control de
  UI).
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en
  `pos-heladeria/src/` (research.md, mismo hallazgo que specs 045-052).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (select
  buscable, mesas ocupadas visibles-pero-no-seleccionables) está definido en spec.md, FR-001 a
  FR-006.
- **Principio V (No refactors oportunistas)** ✅ — la extensión de `SearchableSelectComponent`
  (`disabled`) está directamente causada por FR-004, es retrocompatible, y no se reescribe ninguna
  otra parte del componente compartido (filtrado, teclado, cierre) sin necesidad.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (sustituir un control de UI
  por otro ya existente en el proyecto), sin migración de datos, sin cambio de arquitectura ni de
  backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta
  ya cerrada.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: sin entidades ni campos
  de backend nuevos; `disabled` es un campo de UI opcional sobre un tipo de frontend ya existente.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia; se reutiliza
  `SearchableSelectComponent`, ya en el repositorio.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde los casos existentes de ambos archivos de test afectados;
  agregar cobertura nueva para `disabled` en `SearchableSelectComponent` y para la interacción
  completa del select de mesas en `manual-order-page.component.spec.ts` (quickstart.md, Escenarios
  1-2).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (select buscable, mostrar
  nombre + estado) viene directamente del dueño/desarrollador en spec.md, Input; las decisiones de
  este documento (D1-D4) son todas técnicas (qué componente reutilizar, cómo extenderlo).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedido directo del dueño/desarrollador) → Spec
  053 → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` confirma que el diseño no agrega
dependencias nuevas (Principio IX), no modifica el modelo de datos de backend (Principio VIII,
N/A), no toca ningún characterization test (Principio III, 0 en `pos-heladeria/src/`), y no altera
ningún dato histórico (Principio VII) — únicamente cambia el control de UI usado para seleccionar
una mesa, reutilizando datos que el store ya expone. Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/053-selector-mesa-buscable/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D4
├── data-model.md        # Fase 1 (/speckit-plan) — campo `disabled` en SearchableSelectOption
├── quickstart.md        # Fase 1 (/speckit-plan) — validación manual, 2 escenarios
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

No se genera `contracts/`: esta spec no expone ni consume ninguna API HTTP nueva ni modificada; el
único "contrato" que cambia es la interfaz de un componente de UI compartido
(`SearchableSelectOption`), documentado por completo en data-model.md — mismo criterio que specs
047/051/052.

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/
├── shared/searchable-select/
│   ├── searchable-select.component.ts       # HOY: `SearchableSelectOption { id, label }`, sin
│   │                                          # concepto de opción no seleccionable → SE AGREGA
│   │                                          # `disabled?: boolean`; `selectOption()` retorna sin
│   │                                          # efecto si `opt.disabled`; el `<li>` de la opción
│   │                                          # agrega estilo condicional para verse no
│   │                                          # seleccionable (research.md D2)
│   └── searchable-select.component.spec.ts  # Se agregan casos nuevos para `disabled` (no
│                                              # selecciona, sigue visible); los 8 casos existentes
│                                              # no cambian (research.md, impacto en tests)
└── modules/tables/pages/
    ├── manual-order-page.component.ts       # HOY (post-052): rejilla `grid grid-cols-4` de
    │                                          # botones de mesa dentro del panel derecho → SE
    │                                          # REEMPLAZA por `<app-searchable-select>` alimentado
    │                                          # por un `computed` nuevo `mesaOptions` (mapea
    │                                          # `store.tablesView()` a `SearchableSelectOption[]`,
    │                                          # research.md D2/D3); se agregan `FormsModule` y
    │                                          # `SearchableSelectComponent` a `imports`
    └── manual-order-page.component.spec.ts  # El caso "el selector de mesas permite cambiar..."
                                               # se reescribe para abrir el select y escribir en su
                                               # buscador en vez de buscar botones `M{n}`; se agrega
                                               # cobertura nueva para nombre+estado visibles y para
                                               # que una mesa ocupada no se pueda seleccionar; el
                                               # resto de los 17 casos no cambia (research.md)
```

**Structure Decision**: Se modifican in-place dos archivos de producción ya existentes
(`searchable-select.component.ts`, `manual-order-page.component.ts`), sin crear ningún componente
nuevo — se reutiliza y extiende (de forma retrocompatible) el select buscable ya compartido en el
proyecto, en vez de construir un picker de mesas específico.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
