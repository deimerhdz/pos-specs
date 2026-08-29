# Implementation Plan: Panel derecho unificado — tipo de orden, mesas y pedido en la creación de orden manual

**Branch**: `052-panel-derecho-orden-manual` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/052-panel-derecho-orden-manual/spec.md`

## Summary

Sobre la pantalla de creación de orden manual (`pos-heladeria`, módulo `tables`,
`manual-order-page.component.ts`): las pestañas de "Tipo de Orden" y el listado de "Mesas", hoy en
la barra superior de ancho completo, se trasladan al panel derecho, apilados justo arriba del
bloque "Nueva orden" que ya existe ahí (carrito, totales, "Confirmar y Enviar") — la barra superior
queda reducida a "← Volver a la Terminal". El listado de mesas pasa de scroll horizontal a una
rejilla de 4 columnas (sin scroll), y el panel derecho se ensancha de 320px a 400px para alojar las
dos secciones nuevas sin recortes. Un único archivo de producción modificado, sin cambios de store,
sin cambios de backend, sin dependencias nuevas — ver `research.md` (decisiones D1-D4). Construye
sobre spec 051 (título "Mesas"), aún no commiteada.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow
`@if`/`@for`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x — sin dependencias nuevas; no se agrega ningún
import nuevo (research.md D1)

**Storage**: N/A — no se agrega ni modifica ninguna entidad de backend ni de frontend
(data-model.md); no se toca ningún endpoint de `pos-backend`

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados
`*.component.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio,
pantalla ancha); sin cambios responsive para pantallas angostas (spec.md, Edge Cases/Assumptions)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de
backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; es reordenamiento de template y clases CSS
sobre datos ya cargados, sin llamadas de red adicionales

**Constraints**: Cero cambios en el comportamiento de selección de mesa, disponibilidad de tipos de
orden, cálculo de carrito/totales o confirmación de pedido (spec.md, FR-003/FR-004/FR-006); 0 tests
con prefijo `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (research.md, confirmado
igual que specs 045-051); no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: Un único archivo de producción,
`pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` — se simplifica la
barra superior (líneas 34-96 actuales), se trasladan sus bloques de pestañas y mesas al panel
derecho (línea 155 actual), se cambia el listado de mesas de scroll horizontal a rejilla
(research.md D2), y se ensancha el panel derecho de 320px a 400px (research.md D3). Tests
ajustados en `manual-order-page.component.spec.ts` (ninguno de los 13 casos existentes depende de
la posición geométrica de los bloques que se mueven, research.md). Ningún otro componente, servicio
ni endpoint de backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/052-panel-derecho-orden-manual/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (las 2 decisiones de alcance se resolvieron con el dueño del
  producto vía `AskUserQuestion` antes de escribir la spec), antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta el estado actual
  exacto (post-051) con autorización directa del dueño/desarrollador el 2026-08-28. No reabre
  ninguna regla de precio, inventario, disponibilidad de tipos de orden ni facturación; no aplica
  una nueva entrada en `registro-de-anomalias.md` (mismo criterio que specs 045/048/049/051,
  reordenamiento de UI).
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en
  `pos-heladeria/src/` (research.md, mismo hallazgo que specs 045-051).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (ubicación y
  ancho de los controles) está definido en spec.md, FR-001 a FR-008.
- **Principio V (No refactors oportunistas)** ✅ — solo se tocan los bloques directamente causados
  por FR-001/FR-002/FR-005/FR-007; ningún otro bloque del componente (buscador, categorías, grilla
  de productos, carrito, totales) cambia su lógica interna, solo su contenedor padre o ancho.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (reordenamiento/ensanchamiento
  de UI sobre un archivo ya existente), sin migración de datos, sin cambio de arquitectura ni de
  backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta
  ya cerrada.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: sin entidades ni campos
  nuevos.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde los 13 casos existentes de
  `manual-order-page.component.spec.ts`; agregar cobertura nueva para la nueva ubicación de "Tipo
  de Orden"/"Mesas" dentro del panel derecho y para el ancho del panel (quickstart.md, Escenarios
  1-2).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (unificar controles, ensanchar
  el panel) viene directamente del dueño/desarrollador en spec.md, Input/Clarifications; las
  decisiones de este documento (D1-D4) son todas técnicas (qué archivo tocar, qué clases CSS usar).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (feedback directo sobre captura de pantalla) →
  Clarifications (`AskUserQuestion`) → Spec 052 → este Plan → Tasks/Implementación/Tests
  (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` confirma que el diseño no agrega
dependencias nuevas (Principio IX), no modifica el modelo de datos de backend (Principio VIII,
N/A), no toca ningún characterization test (Principio III, 0 en `pos-heladeria/src/`), y no altera
ningún dato histórico ni regla de cálculo de facturación (Principio VII) — únicamente reordena y
redimensiona bloques de template sobre datos que el store ya expone. Gate sigue PASANDO sin
violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/052-panel-derecho-orden-manual/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D4
├── data-model.md        # Fase 1 (/speckit-plan) — sin entidades nuevas
├── quickstart.md        # Fase 1 (/speckit-plan) — validación manual, 3 escenarios
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

No se genera `contracts/`: esta spec no expone ni consume ninguna API HTTP nueva ni modificada, no
agrega ni modifica ningún método/computed del store, y no hay ningún contrato de UI/store nuevo que
documentar más allá de reordenar bloques de template ya existentes — mismo criterio que specs
047/051.

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/modules/tables/pages/
├── manual-order-page.component.ts       # HOY (post-051): barra superior de ancho completo con
│                                          # "Tipo de Orden" + pestañas + "Mesas" + listado
│                                          # (~34-96); panel derecho de 320px con "Nueva orden" +
│                                          # carrito + totales (~155-207) → SE SIMPLIFICA la barra
│                                          # superior a solo "Volver" (research.md D1); SE
│                                          # TRASLADAN las pestañas y el listado de mesas al inicio
│                                          # del panel derecho, como bloques `shrink-0` nuevos antes
│                                          # de "Nueva orden" (research.md D1, D4); SE CAMBIA el
│                                          # listado de mesas de `overflow-x-auto` a
│                                          # `grid grid-cols-4` (research.md D2); SE ENSANCHA el
│                                          # panel derecho de `sm:w-[320px]` a `sm:w-[400px]`
│                                          # (research.md D3)
└── manual-order-page.component.spec.ts  # Se agregan casos nuevos para la ubicación de los
                                           # controles dentro del panel derecho; los 13 casos
                                           # existentes (8 originales + 5 de spec 051) no cambian de
                                           # intención (research.md, "Resumen de impacto en tests
                                           # existentes")
```

**Structure Decision**: Se modifica in-place un único archivo de producción ya existente
(`manual-order-page.component.ts`), sin crear ningún componente nuevo — es un reordenamiento de
bloques de template ya existentes dentro del mismo componente, no una extracción a subcomponentes.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
