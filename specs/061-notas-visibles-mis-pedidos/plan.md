# Implementation Plan: Notas del ítem visibles en "Mis pedidos" del Menú QR

**Branch**: `061-notas-visibles-mis-pedidos` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/061-notas-visibles-mis-pedidos/spec.md`

## Summary

La sección "Mis pedidos" del Menú QR (`PublicMenuComponent`, bloque
`@if (section() === 'pedidos')`) ya itera `order.items` y muestra cantidad, variante y opciones
(`optionLabels(item)`) por cada línea, pero nunca lee `item.notes` — aunque ese campo ya existe en
`DiningOrderItem` (`dining.interface.ts:112`) y ya viaja completo del backend al frontend (la
terminal de personal ya lo muestra, `pos-terminal.store.ts:789`). Este plan agrega un único bloque
condicional `@if (item.notes)` dentro del `@for` de ítems existente
(`public-menu.component.ts:241-255`), al mismo nivel visual que ya usa `optionLabels` (texto
pequeño, bajo el nombre del producto). No se agrega ningún método nuevo al componente — `item.notes`
se lee directo del binding, igual que ya se hace con `item.quantity`. Sin cambios de modelo de
datos, de contrato de backend, ni de ningún otro flujo (carrito, terminal de personal).

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componente standalone, signals, control flow
`@if`/`@for`, ya en uso por `PublicMenuComponent`).

**Primary Dependencies**: ninguna nueva — `item.notes` ya es un campo tipado de `DiningOrderItem`
(`dining.interface.ts:112`), ya poblado end-to-end.

**Storage**: N/A — sin cambios de base de datos, de modelo de datos ni de contrato de API; el dato
ya existe y ya llega al frontend.

**Testing**: Vitest vía `@angular/build:unit-test`. Ya existe
`public-menu.component.spec.ts` con el mismo patrón de test usado por las specs 019/041
(`createComponent`, aserciones sobre `fixture.nativeElement.textContent`) — este feature agrega un
caso nuevo a ese archivo, sin crear ningún fichero de test nuevo.

**Target Platform**: Web — SPA Angular del flujo de pedido del comensal por QR, sección "Mis
pedidos" (`section() === 'pedidos'`). Sin cambios en `../pos-backend` ni en este repositorio de
specs.

**Project Type**: cambio acotado al repositorio hermano `../pos-heladeria` únicamente — un solo
archivo de producción.

**Performance Goals**: sin objetivos nuevos — un binding de lectura adicional sobre datos ya
presentes en memoria, sin ninguna llamada de red ni cómputo extra.

**Constraints**: la nota se lee directamente de `item.notes` (ya tipado `string | null | undefined`
en `DiningOrderItem`) — no requiere ningún método de formateo, a diferencia de `optionLabels(item)`
que sí necesita resolver IDs de opción contra el catálogo (`this.lookup()`); esa asimetría es
intencional, no una inconsistencia a corregir (spec.md Assumptions).

**Scale/Scope**: 1 archivo de producción (`public-menu.component.ts`: 1 bloque `@if` nuevo dentro
de un `@for` ya existente, sin métodos nuevos), 1 caso de test nuevo en el archivo de spec ya
existente (`public-menu.component.spec.ts`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/061-notas-visibles-mis-pedidos/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (checklist de calidad 16/16).
- **Principio II (Comportamiento existente protegido)** ✅ — cambio puramente aditivo: agrega la
  visualización de un dato que ya existe y ya viaja al frontend, sin retirar ni alterar ningún
  comportamiento actual de "Mis pedidos" (cantidad, variante, opciones, estado de cocina, pago —
  spec.md FR-002, FR-005). No aplica registrar nada en `registro-de-anomalias.md`: no hay
  comportamiento previo que cambie, solo una omisión visual que se corrige.
- **Principio III (Characterization tests)** — no aplica: no existe hoy ningún test con prefijo
  `"CONGELA comportamiento actual:"` sobre el bloque "Mis pedidos" de
  `public-menu.component.spec.ts` (los tests existentes cubren Bug 1 y Bug 3 de la spec 041, no la
  sección "pedidos"); ninguno se ve afectado.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el comportamiento nuevo está definido en
  spec.md, FR-001 a FR-005.
- **Principio V (No refactors oportunistas)** ✅ — no se toca `optionLabels`, `dining-cart.service.ts`,
  `pos-terminal.store.ts` ni ningún otro archivo fuera de `public-menu.component.ts`; no se
  introduce ningún método/helper nuevo porque `item.notes` no necesita transformación (Technical
  Context, Constraints).
- **Principio VI (Evolución incremental)** ✅ — una sola historia de usuario, acotada a un único
  bloque de template en un único componente; sin mezclar con ninguna refactorización ni cambio de
  arquitectura.
- **Principio VII (Datos históricos)** ✅ — no aplica; no se toca ninguna `Sale`/factura ni dato
  histórico, solo la vista en vivo de un pedido activo.
- **Principio VIII (Evolución del modelo de datos)** ✅ — no aplica; `notes` ya existe en el modelo,
  sin cambios de esquema.
- **Principio IX (Dependencias nuevas)** ✅ — ninguna dependencia nueva.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: caso de test nuevo que cubre FR-001 a FR-003 (ítem con nota la muestra, ítem
  sin nota no agrega nada, dos ítems idénticos con notas distintas se distinguen).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (el comensal necesita ver su
  nota) viene directamente del dueño/desarrollador en spec.md; la decisión técnica de este plan
  (leer `item.notes` directo en el template, sin método nuevo) es la forma más simple de lograr ese
  resultado, no una funcionalidad adicional no solicitada.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reportada directamente por el dueño/desarrollador,
  spec.md Input) → Spec 061 → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Sin desviaciones que justificar. Sin Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: no se generó `data-model.md` ni `contracts/` — spec.md no
tiene sección "Key Entities" con datos nuevos (`notes` ya existe en `DiningOrderItem`) y ningún
endpoint ni contrato de API cambia de forma (el cambio es 100% de template en el cliente, sobre un
dato que el frontend ya recibe hoy). `quickstart.md` cubre la validación manual y automatizada
completa de la única historia de usuario. Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/061-notas-visibles-mis-pedidos/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan) — decisión D1
├── quickstart.md                  # Fase 1 (/speckit-plan) — validación de la única historia
├── checklists/requirements.md     # /speckit-specify — checklist de calidad de la spec
└── tasks.md                       # Fase 2 (/speckit-tasks)
```

Sin `data-model.md` ni `contracts/` — no aplican a esta spec (ver Constitution Check, re-chequeo
post-diseño).

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). Sin cambios en `../pos-backend`.

```text
pos-heladeria/
└── src/app/modules/tables/pages/
    ├── public-menu.component.ts       # ÚNICO archivo de producción que cambia — bloque
    │                                     `@if (item.notes)` nuevo dentro del `@for` de ítems de
    │                                     "Mis pedidos" (líneas 241-255), sin métodos nuevos
    └── public-menu.component.spec.ts  # caso de test nuevo (archivo ya existente, mismo patrón
                                          usado para Bug 1/Bug 3 de la spec 041)
```

**Structure Decision**: sin componentes, servicios ni interfaces nuevas — la única historia cae
dentro de un bloque de template ya existente en un único archivo de producción; `item.notes` se lee
directo del binding porque `DiningOrderItem.notes` (`dining.interface.ts:112`) ya lo expone tipado.
No se toca `dining-cart.service.ts`, `pos-terminal.store.ts` ni ningún endpoint de `pos-backend`.

## Complexity Tracking

*Sin violaciones que justificar — ver "Resultado" en Constitution Check.*
