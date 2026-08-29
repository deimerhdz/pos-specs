# Implementation Plan: Campo "Cliente" con valor por defecto "Consumidor final" en la creación de orden manual

**Branch**: `054-campo-cliente-orden-manual` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/054-campo-cliente-orden-manual/spec.md`

## Summary

Sobre la pantalla de creación de orden manual (`pos-heladeria`, módulo `tables`,
`manual-order-page.component.ts`): se agrega un campo "Cliente" (entre "Mesas" y "Nueva orden") de
solo lectura por defecto, con "Consumidor final" ya diligenciado, y un botón ✏️ que activa su
edición (mismo patrón `relative` + botón `absolute` que ya usa `password-input.component.ts`). El
campo está atado a `store.customerName` — un signal que ya existe y que
`createManualOrderFromDraft()` ya envía como `customer_name` al crear la orden; el backend ya
acepta y persiste ese campo sin ningún cambio. El valor por defecto se aplica únicamente dentro de
este componente (no en el método compartido `PosTerminalStore.selectTable()`), para no afectar la
Terminal de Mesas. Un único archivo de producción modificado, sin cambios de backend, sin
dependencias nuevas — ver `research.md` (decisiones D1-D3).

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow
`@if`/`@for`, signals, `ControlValueAccessor`)

**Primary Dependencies**: `@angular/core` 21.1.x, `@angular/forms` (`FormsModule`, ya en
`imports` del componente desde spec 053) — sin dependencias nuevas

**Storage**: N/A — `CustomerOrder.customer_name` ya existe en `pos-backend`
(`app/models/customer_order.py:49`), ya es opcional, y `POST /orders` ya lo acepta y persiste sin
cambios (data-model.md); no se toca ningún endpoint ni modelo de `pos-backend`

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados
`*.component.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio,
pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de
backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; el campo lee/escribe un signal ya existente
del store, sin llamadas de red adicionales

**Constraints**: Cero cambios al método compartido `PosTerminalStore.selectTable()` ni a ningún
otro comportamiento de la Terminal de Mesas (spec.md, "Decisión de negocio"; research.md D1); el
nombre de cliente nunca se guarda vacío (FR-005); cero cambios al catálogo, carrito, totales o
tipos de orden deshabilitados (FR-007); 0 tests con prefijo `"CONGELA comportamiento actual:"` en
`pos-heladeria/src/` (research.md, confirmado igual que specs 045-053); no se agregan dependencias
nuevas (Principio IX)

**Scale/Scope**: Un único archivo de producción,
`pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` — se agrega el campo
"Cliente" al template (entre el bloque "Mesas" y "Nueva orden"), un signal local `editandoCliente`,
los métodos `toggleEditarCliente()`/`onClienteBlur()`/`applyDefaultCustomerName()` (privado), y se
llama a `applyDefaultCustomerName()` desde `ngOnInit()`, `selectTable(id)` y `confirm()`
(research.md D1/D3). Tests nuevos en `manual-order-page.component.spec.ts` (research.md, impacto en
tests). Ningún otro componente, servicio ni endpoint de backend se modifica.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/054-campo-cliente-orden-manual/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente, antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente que
  esta spec cambia un comportamiento observable (el nombre de cliente guardado pasa de `null` a
  "Consumidor final" por defecto) con autorización directa del dueño/desarrollador el 2026-08-29, y
  explica por qué no reabre la decisión de `pos-terminal.store.ts:1042-1044` (sección "Decisión de
  negocio" de spec.md). No aplica una nueva entrada en `registro-de-anomalias.md` más allá de citar
  esta spec como origen y autorización.
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en
  `pos-heladeria/src/` (research.md, mismo hallazgo que specs 045-053).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (campo
  "Cliente", valor por defecto, edición, persistencia) está definido en spec.md, FR-001 a FR-007.
- **Principio V (No refactors oportunistas)** ✅ — no se extiende `IconComponent` (se reutiliza el
  emoji ✏️ ya usado en `tables-page.component.ts`), no se crea un componente reutilizable nuevo sin
  segundo consumidor, y no se toca `PosTerminalStore.selectTable()` compartido (research.md D1, D2).
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (agregar un campo de UI
  atado a un signal ya existente, en un único archivo), sin migración de datos ni cambio de
  arquitectura ni de backend.
- **Principio VII (Datos históricos)** N/A — no se toca facturación ni se recalcula ninguna venta
  ya cerrada; solo afecta órdenes nuevas creadas desde ahora.
- **Principio VIII (Evolución del modelo de datos)** N/A — data-model.md: `customer_name` ya existe
  en el modelo, sin cambios de esquema ni migración.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde los 21 casos existentes de
  `manual-order-page.component.spec.ts`; agregar cobertura nueva para el valor por defecto, la
  edición, el caso de campo vacío (blur y confirmar sin blur), y que `customer_name` viaje
  correctamente en la petición de creación (quickstart.md, Escenarios 1-3).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (campo "Cliente" con
  "Consumidor final" por defecto, editable, persistido) viene directamente del dueño/desarrollador
  en spec.md, Input; las decisiones de este documento (D1-D3) son todas técnicas (dónde aplicar el
  default, qué patrón de UI reutilizar).
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedido directo del dueño/desarrollador) → Spec
  054 → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: `data-model.md` confirma que el diseño no agrega
dependencias nuevas (Principio IX), no modifica el modelo de datos de backend (Principio VIII,
N/A, campo ya existente), no toca ningún characterization test (Principio III, 0 en
`pos-heladeria/src/`), y no altera ningún dato histórico (Principio VII, solo afecta órdenes
nuevas). Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/054-campo-cliente-orden-manual/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones D1-D3
├── data-model.md        # Fase 1 (/speckit-plan) — campo ya existente, sin cambios de esquema
├── quickstart.md        # Fase 1 (/speckit-plan) — validación manual, 3 escenarios
└── tasks.md             # Fase 2 (/speckit-tasks) — aún no generado
```

No se genera `contracts/`: el contrato de red (`customer_name` en `OrderCreate`/`POST /orders`) ya
existe y no cambia — esta spec solo hace que el frontend empiece a poblarlo desde una pantalla que
hoy no lo toca. Mismo criterio que specs 047/051/052/053.

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec.

```text
pos-heladeria/src/app/modules/tables/pages/
├── manual-order-page.component.ts       # HOY (post-053): sin ningún campo "Cliente" → SE AGREGA
│                                          # un bloque nuevo entre "Mesas" (~139-145) y "Nueva
│                                          # orden" (~148-153): `<h3>Cliente</h3>` + input
│                                          # readonly/editable con botón ✏️ (research.md D2);
│                                          # se agrega el signal `editandoCliente`, los métodos
│                                          # `toggleEditarCliente()`, `onClienteBlur()` y
│                                          # `applyDefaultCustomerName()` (privado); se llama a
│                                          # `applyDefaultCustomerName()` desde `ngOnInit()`
│                                          # (~230-234), `selectTable(id)` (~240-242) y `confirm()`
│                                          # (~252-257) (research.md D1, D3)
└── manual-order-page.component.spec.ts  # Se agregan casos nuevos: valor por defecto visible,
                                           # edición + blur, campo vacío vuelve al default (blur y
                                           # sin blur vía confirmar), `customer_name` viaja en la
                                           # petición de creación con el valor correcto; los 21
                                           # casos existentes no cambian (research.md, impacto en
                                           # tests)
```

**Structure Decision**: Se modifica in-place un único archivo de producción ya existente
(`manual-order-page.component.ts`), sin crear ningún componente ni servicio nuevo — se reutiliza el
signal `customerName` ya expuesto por `PosTerminalStore` y el patrón visual de
`password-input.component.ts`.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
