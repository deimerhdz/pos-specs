# Implementation Plan: Mejora de Visibilidad del Buscador y Filtros en Inventario

**Branch**: `085-mejora-visibilidad-filtros-inventario` | **Date**: 2026-09-22 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/085-mejora-visibilidad-filtros-inventario/spec.md`

## Summary

**Nota de esta revisión**: este plan fue regenerado tras una sesión de clarificación
(2026-09-22, ver spec.md) que reemplazó las decisiones visuales de la primera iteración de
este spec. `inventory-page.component.ts` (`pos-heladeria`) tiene **hoy** implementada esa
primera iteración (`tasks.md` T001-T013, rama `feat/085-improve-inventory-filters-visibility`):
un `<div>` de filtros **separado** y hermano de la tabla (líneas 111-131,
`bg-indigo-50/40 rounded-xl border border-indigo-100 p-4 flex flex-wrap gap-3`), apilado con
`space-y-6` sobre el `<div>` de la tabla (líneas 133-199, `bg-white rounded-xl border
border-gray-100 overflow-hidden`). Ese código ya no cumple el spec vigente y debe
reconciliarse con lo descrito abajo — ver `research.md`, sección "Estado actual del código",
y el Completion Report de este comando para el detalle de qué queda pendiente.

La zona de búsqueda y filtros debe fusionarse como encabezado superior de la **misma**
tarjeta que contiene la tabla de resultados, separada de las filas por un borde
(`border-b border-gray-200`), replicando como referencia visual exacta el mockup provisto
por el usuario (`design/inventory-search-filters-redesign.html`, subárbol
`data-purpose="table-card-header"`): colores (acento morado exacto `#4a3aff` en vez del
`indigo` genérico que usa el resto del producto), bordes, espaciados y la estructura de
agrupación buscador-izquierda/selects-derecha — con dos exclusiones explícitas: no se agrega
el badge "⌘K" del input ni el contador "N Insumos" que sí aparecen en el mockup (FR-009).

El enfoque técnico sigue siendo exclusivamente de presentación, ahora sobre un bloque más
amplio del mismo archivo: eliminar el `<div>` de filtros como hermano separado de la tabla e
insertarlo como primer hijo del `<div>` de la tabla, en un contenedor
`flex items-center justify-between gap-4 p-4 border-b border-gray-200 bg-gray-50/50
flex-wrap`; el buscador conserva el ícono de lupa (`app-mi-icon name="search"`, ya
existente en el catálogo compartido) y cada `<select>` incorpora un wrapper con flecha
personalizada (`appearance-none` + SVG decorativo), replicando la estructura del mockup, no
solo su color. El contenedor de la tarjeta pasa de `border-gray-100` a `shadow-sm
border-slate-200/80` (mockup). No se modifica ningún binding, selector, servicio, ruta, fila,
columna ni dato de la tabla — el comportamiento de búsqueda/filtrado y la apariencia del
resto de la tabla quedan 100% intactos (FR-005, FR-007, FR-008).

## Technical Context

**Language/Version**: TypeScript ~5.9.2 sobre Angular 21.1 (standalone components)

**Primary Dependencies**: Angular (`@angular/core`, `@angular/forms` — `FormsModule`/
`ngModel`, ya en uso), Tailwind CSS v4 (utilidades, sin hojas de estilo propias en este
componente; el acento `#4a3aff` se aplica vía clases de valor arbitrario `[#4a3aff]`
directamente en el template, sin extender `tailwind.config` — ver research.md punto 2),
`IconMiComponent` (`shared/icon-mi`) — ya importado en `inventory-page.component.ts` y con
el ícono `search` ya disponible en `icon-catalog.ts`. No se introduce ninguna dependencia
nueva (N/A Principio IX).

**Storage**: N/A — sin cambios de datos, API ni persistencia.

**Testing**: `ng test` (`@angular/build:unit-test`) en `pos-heladeria`. Suite existente
relevante: `inventory-page.component.spec.ts` (hoy solo cubre estandarización de íconos,
spec 082; no ejercita el bloque de filtros, así que no hay riesgo de romperla). No existen
tests `"CONGELA comportamiento actual:"` sobre este componente (verificado por grep) — no
aplica el Principio III.

**Target Platform**: Web (SPA Angular), mismos anchos de ventana que ya soporta hoy la
página de Inventario (incluye acomodo en varias líneas vía `flex flex-wrap`, ya presente).

**Project Type**: Aplicación web — cambio acotado al frontend (`pos-heladeria`); no se toca
`pos-backend`.

**Performance Goals**: N/A — cambio de clases CSS y un ícono estático; sin impacto de
rendimiento medible.

**Constraints**:
- Contraste AA de WCAG entre texto y fondo en la zona de filtros (FR-004, SC-005).
- Cero cambios en resultados, orden o comportamiento de búsqueda/filtrado (FR-005, FR-007).
- No alterar la apariencia de la tarjeta "Bajo mínimo" ni de las filas/columnas/datos de la
  tabla de resultados (FR-008); la zona no debe competir visualmente con la tarjeta de
  alerta cuando esta está activa (Edge case).
- Debe seguir siendo reconocible en el acomodo de varias líneas en ventanas angostas
  (FR-006).
- Alcance exclusivo a la pestaña "Insumos"; las pestañas "Compras" y "Movimientos" quedan
  fuera (no tienen esta barra).
- El mockup (`design/inventory-search-filters-redesign.html`) es referencia visual EXACTA
  —colores, bordes, espaciados, estructura—, pero **solo** para el subárbol
  `data-purpose="table-card-header"` y el contenedor de la tarjeta que lo envuelve; no se
  extiende al resto de la página del mockup (sidebar, header superior, fondo de página) ni
  a las filas/columnas/datos de la tabla (research.md punto 2).
- No agregar el badge "⌘K" del input ni el contador "N Insumos" que el mockup sí incluye
  (FR-009, Clarifications Q2 de spec.md).

**Scale/Scope**: 1 archivo (`inventory-page.component.ts`), ~50-60 líneas de template: se
elimina el `<div>` de filtros como hermano de la tabla y se inserta su contenido fusionado
como encabezado de la tarjeta de la tabla (con wrapper de flecha personalizada en cada
select), sin componentes, rutas ni servicios nuevos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principio | Evaluación |
|---|-----------|------------|
| I | Nacen de un spec | **PASS** — `specs/085-mejora-visibilidad-filtros-inventario/spec.md` aprobado, define problema, alcance, criterios de aceptación. |
| II | Comportamiento existente protegido | **PASS** — cambio exclusivamente visual (estructura del DOM y clases, no lógica); FR-005/FR-007/FR-008 exigen conservar el 100% del comportamiento y de la apariencia de filas/columnas/datos de la tabla. No hay decisión de negocio que registrar en `registro-de-anomalias.md` porque no cambia ningún comportamiento. |
| III | Characterization tests | **N/A** — no existen tests `"CONGELA comportamiento actual:"` sobre `inventory-page.component.ts`; el único spec existente (icono, spec 082) no toca el bloque de filtros ni la tarjeta de la tabla. |
| IV | Nuevos specs pueden introducir nuevo comportamiento | **N/A** — este spec deliberadamente no introduce comportamiento nuevo. |
| V | Nuevas funcionalidades antes que refactors oportunistas | **PASS** — el cambio se acota al bloque de filtros y al contenedor de la tarjeta de la tabla que lo recibe; no se toca el resto del componente (filas/columnas/datos de la tabla, pestañas Compras/Movimientos, modales) ni se aprovecha para refactorizar. |
| VI | Evolución incremental | **PASS** — unidad única y verificable (un bloque de template en un archivo); no mezcla con arquitectura, migraciones ni otros cambios de comportamiento. |
| VII | Compatibilidad con datos históricos | **N/A** — no hay facturas ni datos históricos involucrados. |
| VIII | Evolución del modelo de datos | **N/A** — sin cambios de modelo de datos. |
| IX | Dependencias nuevas | **PASS** — cero dependencias nuevas; reutiliza Tailwind y `IconMiComponent`/ícono `search` ya existentes y ya importados. |
| X | Verificación obligatoria | **PASS** — se verifica con: suite existente en verde (`ng test`), y la guía manual de `quickstart.md` contra SC-001 a SC-005 (visibilidad, contraste AA, cero regresión funcional, confirmación del usuario que reportó el problema). La verificación ya ejecutada en `tasks.md` (T001-T013) corresponde a la iteración anterior del diseño y debe repetirse íntegra contra este plan regenerado. |
| XI | Decisiones de negocio vs técnicas | **PASS** — el spec es la decisión de negocio (mejorar visibilidad); esta es la decisión técnica de cómo lograrlo sin tocar comportamiento. |
| XII | Trazabilidad | **PASS** — cadena Necesidad (queja documentada en spec, sección Contexto) → Spec 085 → Implementación (este plan / tasks.md) → Tests (suite existente + quickstart) → Verificación, íntegra. |
| XIII | Español de Colombia | **PASS** — todos los artefactos de esta spec en español de Colombia. |

**Resultado**: sin violaciones. No aplica Complexity Tracking.

**Nota post-diseño**: `tasks.md` (Fase 2) ya fue regenerado vía `/speckit-tasks` contra el
diseño fusionado descrito en este plan y en `research.md` (encabezado fusionado en la
tarjeta de la tabla, acento exacto `#4a3aff`, T001-T018). La re-implementación en
`pos-heladeria` sigue pendiente de ejecutarse.

## Project Structure

### Documentation (this feature)

```text
specs/085-mejora-visibilidad-filtros-inventario/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan) — sin entidades, se documenta el N/A
├── quickstart.md        # Fase 1 (/speckit-plan) — guía de validación manual
├── design/
│   └── inventory-search-filters-redesign.html  # Mockup de referencia visual exacta (Q3)
├── checklists/
│   └── requirements.md  # Checklist de calidad del spec — todos los ítems en verde
└── tasks.md             # Fase 2 (/speckit-tasks) — ya regenerado contra el diseño
                          #   fusionado vigente (T001-T018)
```

No se genera `contracts/`: esta spec no expone ni consume ninguna interfaz externa (API,
CLI, esquema); es un ajuste de presentación dentro de un componente Angular ya existente,
sin cambios de contrato de datos ni de servicio.

### Source Code (repository root)

```text
pos-heladeria/                                    # único repo afectado (frontend)
└── src/app/modules/inventory/
    ├── pages/
    │   ├── inventory-page.component.ts           # ÚNICO archivo modificado:
    │   │                                          #   hoy: div "Filters" separado (líneas
    │   │                                          #   111-131) + div de tabla (133-199);
    │   │                                          #   objetivo: fusionar el primero como
    │   │                                          #   encabezado del segundo (research.md §1)
    │   └── inventory-page.component.spec.ts       # suite existente (spec 082) — sin cambios,
    │                                              #   debe seguir en verde
    ├── components/                                # inventory-item-form, stock-adjust-modal,
    │                                              #   purchase-form — sin cambios
    ├── interfaces/                                 # sin cambios
    └── services/                                    # inventory.service.ts, unit-lookup.ts —
                                                     #   sin cambios (cero lógica tocada)

pos-backend/                                       # NO se toca (sin cambios de API/datos)
```

**Structure Decision**: cambio de un único archivo del frontend Angular
(`pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`), acotado al
template inline del componente (clases Tailwind + un ícono decorativo reutilizado). No se
crean componentes, servicios, rutas ni archivos nuevos; `pos-backend` queda fuera de
alcance por completo, consistente con las Assumptions del spec.

## Complexity Tracking

*No aplica — el Constitution Check no registró violaciones.*
