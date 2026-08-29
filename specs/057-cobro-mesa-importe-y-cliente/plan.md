# Implementation Plan: Importe fijo para pagos no efectivo y nombre de cliente en el desglose de cobro

**Branch**: `057-cobro-mesa-importe-y-cliente` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/057-cobro-mesa-importe-y-cliente/spec.md`

## Summary

Bloquea el campo de importe de `PaymentInputComponent` (`pos-heladeria`) cuando el método de pago
elegido no es efectivo, conectando el `[disabled]` reactivo que `MoneyInputComponent` ya soporta
por completo (`ControlValueAccessor.setDisabledState`) — sin componente ni lógica de cálculo
nuevos, porque el importe ya se precarga con el total exacto al elegir cualquier método
(`setMethod()`). Como `PaymentInputComponent` es el único punto de captura de importe en todo el
proyecto (usado por `session-bill-panel.component.ts` y directamente por
`pos-checkout-panel.component.ts`), un solo cambio cubre ambos flujos de cobro (FR-003).

Además, cambia `lineLabel()` en `session-bill-panel.component.ts` para que la línea "sin
comensal asignado" del desglose de "Cuenta de la mesa" muestre el `@Input customerName` que el
componente ya recibe (el mismo nombre de facturación que ya usa para el `customer_name` que se
envía al backend al cobrar) antes de caer a "Sin asignar (mesero)".

Hallazgo de la investigación técnica (research.md Decisión 5): 8 tests ya fallan hoy, antes de
cualquier cambio de esta spec, en los dos archivos de test que ejercitan este componente
(`pos-checkout-panel.component.spec.ts`, `session-bill-panel.component.spec.ts`) — un selector
`input[type="number"]` obsoleto de antes de que `MoneyInputComponent` (que renderiza
`type="text"`) reemplazara el campo de importe. Corregirlo es un prerrequisito real para poder
escribir cualquier test de FR-001, no una limpieza oportunista — se corrige como parte de esta
spec.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals, control
flow `@if`/`@for`, Angular Forms con `ControlValueAccessor`).

**Primary Dependencies**: `@angular/core`, `@angular/forms` (ya en uso) — sin dependencias nuevas.

**Storage**: N/A — sin cambios de base de datos ni de modelo de datos (spec.md, Assumptions).

**Testing**: Vitest vía `@angular/build:unit-test`, specs colocados `*.component.spec.ts`.

**Target Platform**: Web — SPA Angular de la terminal POS (`pos-heladeria`), navegador de
escritorio/pantalla ancha. Sin cambios de backend (`pos-backend`) ni de contrato de ningún
endpoint.

**Project Type**: Cambio acotado al repositorio hermano `../pos-heladeria` únicamente — no toca
`../pos-backend` ni este repositorio de specs (`pos-specs`).

**Performance Goals**: Sin objetivos numéricos nuevos.

**Constraints**: Cero cambio de comportamiento para pagos en efectivo (FR-002); ambos flujos de
cobro (mesa y mostrador) deben quedar cubiertos por el mismo cambio, sin divergir (FR-003); no se
agregan dependencias nuevas (Principio IX de la Constitución).

**Scale/Scope**: 2 archivos de producción (`payment-input.component.ts`,
`session-bill-panel.component.ts`), 2 archivos de test existentes a corregir/extender
(`pos-checkout-panel.component.spec.ts`, `session-bill-panel.component.spec.ts`), 1 archivo de
test nuevo (`payment-input.component.spec.ts`, no existía).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/057-cobro-mesa-importe-y-cliente/spec.md` existe,
  sin `[NEEDS CLARIFICATION]` pendiente (ambos ajustes tenían un valor por defecto razonable, ya
  fundamentado en código existente — ver spec.md checklist).
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente los
  dos cambios de comportamiento observable (importe fijo para no efectivo; nombre de cliente en
  vez de "Sin asignar (mesero)") con autorización directa del dueño/desarrollador el 2026-08-29,
  tras probar el flujo en vivo.
- **Principio III (Characterization tests)** — vigilar: los 8 tests que hoy fallan por el
  selector obsoleto (research.md Decisión 5) **no son characterization tests en rojo por esta
  spec** — ya estaban rotos antes de cualquier cambio (confirmado reproduciendo el mismo fallo
  contra el código sin tocar). Corregir su selector no es "editar un test protegido hasta que esté
  de acuerdo con el código nuevo" (lo que el Principio III prohíbe): es arreglar una aserción que
  ya no podía ejecutarse por una causa ajena al comportamiento que protegía.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el comportamiento nuevo (importe bloqueado,
  nombre de cliente en el desglose) está definido en spec.md, FR-001 a FR-008.
- **Principio V (No refactors oportunistas)** ✅ — la única corrección fuera del pedido explícito
  del usuario (el selector obsoleto, research.md Decisión 5) es un prerrequisito real para poder
  verificar FR-001 con un test automatizado, no una refactorización no relacionada; no se toca
  ningún otro selector ni archivo que no esté directamente involucrado en esta spec.
- **Principio VI (Evolución incremental)** ✅ — un solo frente de cambio (un componente compartido
  de pago + un método de una sola línea en el desglose), sin mezclar con ninguna otra
  funcionalidad, migración ni refactorización.
- **Principio VII (Datos históricos)** ✅ — no aplica; no se toca ninguna `Sale`/factura ni dato
  persistido, solo la edición/presentación en el frontend antes de cobrar.
- **Principio VIII (Evolución del modelo de datos)** ✅ — no aplica; sin cambios de esquema ni de
  columnas (spec.md, Assumptions).
- **Principio IX (Dependencias nuevas)** ✅ — ninguna.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: corregir primero los 8 tests rotos (research.md Decisión 5), luego
  agregar cobertura nueva para FR-001/FR-002/FR-003/FR-004 (importe fijo/editable según método, en
  ambos flujos) y FR-005/FR-006/FR-007 (nombre de cliente en el desglose).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (bloquear el importe no
  efectivo, mostrar el nombre de cliente) viene del dueño/desarrollador en spec.md; las decisiones
  de este plan (research.md D1-D5) son todas técnicas — en particular, la corrección del selector
  obsoleto es una decisión técnica para poder verificar la regla de negocio, no una funcionalidad
  nueva.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reportada por el dueño/desarrollador tras probar
  spec 056 en vivo) → Spec 057 → este Plan (research.md D1-D5) → Tasks/Implementación/Tests
  (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Sin desviaciones respecto a una lectura literal de spec.md que requieran
justificación más allá de la corrección de tests ya documentada (research.md Decisión 5, amparada
por Principio V). Sin Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: no se generó `data-model.md` ni `contracts/` — spec.md no
tiene sección "Key Entities" (sin datos nuevos involucrados) y no hay ningún endpoint ni contrato
de API que cambie de forma (research.md D1-D4 confirman que todo el cambio es de presentación/
edición sobre datos que el frontend ya recibe). `quickstart.md` cubre la validación completa de
ambas historias de usuario en los dos flujos de cobro. Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/057-cobro-mesa-importe-y-cliente/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan) — decisiones D1-D5
├── quickstart.md                  # Fase 1 (/speckit-plan) — 3 escenarios de validación
└── tasks.md                       # Fase 2 (/speckit-tasks) — aún no generado
```

Sin `data-model.md` ni `contracts/` — no aplican a esta spec (ver Constitution Check, re-chequeo
post-diseño).

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). Sin cambios en `../pos-backend`.

```text
pos-heladeria/
├── src/app/modules/tables/components/payment-input.component.ts        # [disabled] reactivo en
│                                                                          # <app-money-input> según isCash() (research.md D1)
├── src/app/modules/tables/components/payment-input.component.spec.ts   # NUEVO — casos de
│                                                                          # FR-001/FR-002/FR-004
├── src/app/modules/tables/components/session-bill-panel.component.ts   # lineLabel(): usa
│                                                                          # customerName antes de "Sin asignar (mesero)" (research.md D4)
├── src/app/modules/tables/components/session-bill-panel.component.spec.ts   # corrige selector
│                                                                              # obsoleto (research.md D5) + casos nuevos FR-005/FR-006/FR-007
└── src/app/modules/tables/components/pos-checkout-panel.component.spec.ts   # corrige selector
                                                                               # obsoleto (research.md D5) + casos nuevos FR-001/FR-003
```

**Structure Decision**: sin componentes nuevos — todo el cambio de producción cae dentro de dos
archivos ya existentes. `pos-checkout-panel.component.ts` no necesita ningún cambio de producción
propio (solo su test) porque hereda el comportamiento nuevo automáticamente al usar
`PaymentInputComponent` ya corregido (research.md Decisión 3).

## Complexity Tracking

*Sin violaciones que justificar — ver "Resultado" en Constitution Check.*
