# Implementation Plan: Ajustes al panel de cobro de pedido manual

**Branch**: `058-panel-cobro-pedido-manual` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/058-panel-cobro-pedido-manual/spec.md`

## Summary

Tres ajustes de presentación sobre la rama "Cobrar pedido"/"Pedido de mostrador" de
`PosCheckoutPanelComponent` (`pos-heladeria`), la única pantalla afectada (spec.md, "Alcance
concreto sobre el sistema actual"): (1) el campo "Facturar a nombre de" adopta el mismo patrón de
solo-lectura + botón "editar" que ya usa el campo "Cliente" en `manual-order-page.component.ts`
(spec 054) — señal local `editandoFacturacion`, reiniciada junto con `paymentDraft` al cambiar de
pedido (research.md D1-D2); (2) el botón principal cambia su texto a "Cobrar" sin tocar
`checkout()`/`checkoutAndSend()` (research.md D3); (3) "Imprimir Factura" y "Liberar Mesa" se
ocultan mientras ese mismo pedido sigue pendiente de cobrar, mediante un `computed` nuevo
(`pendingCheckout`) que envuelve el footer compartido, reutilizando `sidebarMode()` y
`showSessionCharge()` ya existentes (research.md D4). Ningún cambio de modelo de datos, de
contrato de backend, ni de lógica de cobro.

Hallazgo de la investigación técnica (research.md D4): implementar el ajuste 3 cambia el resultado
de 2 tests ya existentes y obliga a mover un tercero — `pos-checkout-panel.component.spec.ts:162-180`
y `:220-244` pasan de esperar que el botón aparezca a esperar que siga oculto (la regla nueva de
spec 058 es más estricta que la de spec 046 que esos tests protegían), y `:246-269` debe
reubicarse en el `describe` de "cobro por sesión de mesa" porque el estado que ejercitaba
("Liberar Mesa" visible sobre un pedido `'recibida'`) deja de existir. Ninguno de los tres tiene el
prefijo `"CONGELA comportamiento actual:"` (confirmado con `grep`), así que el Principio III no
los protege de edición — pero sí aplica el Principio II: el cambio de comportamiento que
protegían queda autorizado por spec 058, Historia 3.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals, control
flow `@if`/`@for`).

**Primary Dependencies**: `@angular/core` (ya en uso) — sin dependencias nuevas.

**Storage**: N/A — sin cambios de base de datos ni de modelo de datos (spec.md, Assumptions).

**Testing**: Vitest vía `@angular/build:unit-test`, specs colocados `*.component.spec.ts`.

**Target Platform**: Web — SPA Angular de la terminal POS (`pos-heladeria`), navegador de
escritorio/pantalla ancha. Sin cambios de backend (`pos-backend`) ni de contrato de ningún
endpoint.

**Project Type**: Cambio acotado al repositorio hermano `../pos-heladeria` únicamente — no toca
`../pos-backend` ni este repositorio de specs (`pos-specs`).

**Performance Goals**: Sin objetivos numéricos nuevos.

**Constraints**: Cero cambio en la llamada de red de `checkout()`/`checkoutAndSend()` (FR-006);
"Imprimir Factura"/"Liberar Mesa" deben seguir apareciendo sin cambios en los demás modos del panel
(FR-008); el valor de facturación enviado al backend no cambia (FR-003).

**Scale/Scope**: 1 archivo de producción
(`pos-checkout-panel.component.ts`: 2 señales/métodos nuevos, 1 computed nuevo, ajustes de
template), 1 archivo de test existente a modificar/mover (`pos-checkout-panel.component.spec.ts`:
3 tests existentes ajustados o reubicados, ver research.md D4, más casos nuevos para
FR-001/FR-002).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/058-panel-cobro-pedido-manual/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (checklist de calidad en 16/16).
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta los tres cambios de
  comportamiento observable (solo-lectura + editar; texto "Cobrar"; ocultar acciones post-cobro
  mientras el cobro está pendiente) con autorización directa del dueño/desarrollador el
  2026-08-29. En particular, el endurecimiento de la regla de spec 046 sobre "Liberar Mesa"
  (research.md D4, tercer punto) queda explícito en spec.md, Historia 3 y FR-007/FR-008 — no es un
  efecto colateral silencioso.
- **Principio III (Characterization tests)** — no aplica ningún veto: ninguno de los tests
  afectados (`:162-180`, `:220-244`, `:246-269`) lleva el prefijo `"CONGELA comportamiento
  actual:"` (confirmado con `grep` sobre el archivo). Se modifican/mueven como parte normal de
  implementar un cambio de comportamiento ya autorizado (Principio II), no se "sobornan" para
  ocultar una regresión no revisada.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el comportamiento nuevo está definido en
  spec.md, FR-001 a FR-009.
- **Principio V (No refactors oportunistas)** ✅ — no se toca ningún selector, botón ni archivo
  fuera de `pos-checkout-panel.component.ts` y su spec; el patrón reutilizado
  (`editandoCliente`/`toggleEditarCliente`/`onClienteBlur`) se copia, no se extrae a un helper
  compartido — extraerlo sería una refactorización no pedida por ninguna de las tres historias.
- **Principio VI (Evolución incremental)** ✅ — un solo componente, tres ajustes de presentación
  acotados y ya divididos en historias independientes (spec.md); sin mezclar con ninguna
  refactorización, migración ni cambio de arquitectura.
- **Principio VII (Datos históricos)** ✅ — no aplica; ninguna `Sale`/factura ya emitida se toca,
  solo la presentación del formulario antes de cobrar.
- **Principio VIII (Evolución del modelo de datos)** ✅ — no aplica; sin cambios de esquema
  (spec.md, Assumptions).
- **Principio IX (Dependencias nuevas)** ✅ — ninguna.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: ajustar/mover los 3 tests identificados (research.md D4) y agregar
  cobertura nueva para FR-001/FR-002 (modo edición del nombre de facturación) y FR-004/FR-005
  (texto del botón).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (los tres ajustes de UI)
  viene del dueño/desarrollador en spec.md; las decisiones de este plan (research.md D1-D5) son
  todas técnicas — en particular, qué tests mover o reescribir es una decisión técnica para
  verificar la regla de negocio ya autorizada, no una funcionalidad nueva.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reportada por el dueño/desarrollador al probar
  el panel de cobro en vivo) → Spec 058 → este Plan (research.md D1-D5) → Tasks/Implementación/
  Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Sin desviaciones que requieran justificación adicional a la ya
documentada en research.md D4. Sin Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: no se generó `data-model.md` ni `contracts/` — spec.md
no tiene sección "Key Entities" (sin datos nuevos involucrados, research.md D5) y no hay ningún
endpoint ni contrato de API que cambie de forma. `quickstart.md` cubre la validación completa de
las tres historias de usuario. Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/058-panel-cobro-pedido-manual/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan) — decisiones D1-D5
├── quickstart.md                  # Fase 1 (/speckit-plan) — 3 escenarios de validación
├── checklists/requirements.md     # /speckit-specify — checklist de calidad de la spec
└── tasks.md                       # Fase 2 (/speckit-tasks) — aún no generado
```

Sin `data-model.md` ni `contracts/` — no aplican a esta spec (ver Constitution Check, re-chequeo
post-diseño).

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). Sin cambios en `../pos-backend`.

```text
pos-heladeria/
├── src/app/modules/tables/components/pos-checkout-panel.component.ts        # señales
│                                                                               # editandoFacturacion/toggleEditarFacturacion/onFacturacionBlur,
│                                                                               # computed pendingCheckout, texto "Cobrar" (research.md D1-D4)
└── src/app/modules/tables/components/pos-checkout-panel.component.spec.ts   # 3 tests
                                                                                # ajustados/reubicados + casos nuevos FR-001/FR-002/FR-004/FR-005
                                                                                # (research.md D4)
```

**Structure Decision**: sin componentes nuevos — los tres ajustes caen dentro de un único archivo
de producción ya existente (`pos-checkout-panel.component.ts`); no se toca
`manual-order-page.component.ts` (research.md D1: se replica su patrón, no se extrae a un helper
compartido) ni `pos-terminal.store.ts` (el dato `billingCustomerName` y la llamada
`checkoutAndSend()` no cambian).

## Complexity Tracking

*Sin violaciones que justificar — ver "Resultado" en Constitution Check.*
