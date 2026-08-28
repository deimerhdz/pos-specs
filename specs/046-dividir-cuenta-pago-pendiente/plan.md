# Implementation Plan: Eliminación de Dividir Cuenta y de Combinar Método de Pago en Toda la Aplicación

**Branch**: `046-dividir-cuenta-pago-pendiente` | **Date**: 2026-08-28 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/046-dividir-cuenta-pago-pendiente/spec.md`

## Summary

Sobre la Terminal de Mesas (`pos-heladeria`, módulo `tables`): (1) condicionar el botón "🔓 Liberar
Mesa" de `pos-checkout-panel.component.ts` a que la mesa seleccionada no tenga ningún pago pendiente
de confirmar, reutilizando el `computed` `centralState()` ya existente en `pos-terminal.store.ts`
(`'validar-pago'` cuando `pendingOfSelectedTable().length > 0`) — cero estado nuevo; (2) eliminar por
completo el botón "Dividir la cuenta entre varias personas" y su componente `split-bill-panel.component.ts`
(spec 010), incluyendo el modo de cobro "Dividir por comensal" de `session-bill-panel.component.ts`
(spec 026, FR-007); (3) eliminar por completo la opción "Combinar con otro método" de
`payment-input.component.ts`/`payment-draft.util.ts` (spec 011), usada tanto por el cobro de
mostrador (`pos-checkout-panel.component.ts`) como por el cobro de "Cuenta de la mesa"
(`session-bill-panel.component.ts`, spec 026 FR-008) — cada cobro queda restringido a un único
método por el total exacto o más.

**Hallazgo clave de esta fase de investigación** (documentado en research.md §3): el bloque de
confirmación de pago pendiente (`payment-attempt-review-panel.component.ts`) **ya cumple FR-003 sin
ningún cambio** — nunca tuvo una opción de combinar método de pago, y el backend
(`confirm_cash_payment_attempt`, `pos-backend/app/api/v1/orders/checkout.py:942-970`) ya rechaza con
`422` un monto en efectivo menor al total de la orden (comentario existente en el código: "FR-010a:
impide confirmar si `amount_received < total_orden`"). Este hallazgo reduce el alcance real de
implementación a los puntos (1)-(3) del párrafo anterior — ningún cambio de backend, ningún cambio en
`payment-attempt-review-panel.component.ts`.

## Technical Context

**Language/Version**: TypeScript 5.9.2, Angular 21.1 (componentes standalone, control flow `@if`/`@for`/`@switch`, signals)

**Primary Dependencies**: `@angular/core` 21.1.x / `@angular/cdk` 21.2.x, `@tanstack/angular-query-experimental` v5 (data fetching, sin cambios en esta spec), RxJS 7.8, Tailwind CSS v4 (utilidades inline en `template:`)

**Storage**: N/A — esta spec no agrega ni modifica entidades de backend (spec.md, Key Entities); no hay migración de base de datos ni endpoint nuevo/eliminado. El endpoint `PUT /table-sessions/{id}/assignments` y el esquema `SplitPaymentIn`/`payments: list[PaymentIn]` (backend) permanecen intactos — dejan de tener llamador desde el frontend, pero no se tocan (ver Constitution Check, Principio VII)

**Testing**: Vitest vía el builder `@angular/build:unit-test`; specs colocados `*.component.spec.ts` / `*.service.spec.ts`

**Target Platform**: Web — SPA Angular servida al navegador de la terminal POS (escritorio, pantalla ancha)

**Project Type**: Aplicación web existente (frontend Angular `pos-heladeria`); sin cambios de backend en esta spec

**Performance Goals**: Sin objetivos numéricos nuevos; ocultar/mostrar "Liberar Mesa" y retirar los dos flujos eliminados deben sentirse instantáneos (mismo patrón reactivo por signals ya usado hoy, sin llamadas de red adicionales)

**Constraints**: Cero cambios de comportamiento en validación de pago QR, cobro por comprobante, cálculo de cambio de un único método, envío a cocina o facturación (specs 024, 028, 044); no se toca ningún test con prefijo `"CONGELA comportamiento actual:"` (0 en `pos-heladeria/src/`, confirmado; existen en `pos-backend/app/characterization_tests/` sobre `checkout_and_send`/`table_sessions.close`/participantes, pero esta spec no toca backend — quedan intactos); no se agregan dependencias nuevas (Principio IX)

**Scale/Scope**: `pos-checkout-panel.component.ts` (condicionar "Liberar Mesa", retirar botón/render/handler de "Dividir la cuenta"), `session-bill-panel.component.ts` (retirar modo "Dividir por comensal"), `payment-input.component.ts` + `payment-draft.util.ts` (retirar "Combinar con otro método"), `split-bill-panel.component.ts` (eliminación completa del archivo y su spec); `pos-terminal.store.ts` no gana ningún signal/computed nuevo (se reutiliza `centralState()`)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/046-dividir-cuenta-pago-pendiente/spec.md` existe y fue clarificado en tres rondas sucesivas (2026-08-28) antes de este plan.
- **Principio II (Comportamiento existente protegido)** ✅ — el spec documenta explícitamente, con justificación del dueño del producto (Clarifications), la eliminación de dos comportamientos ya implementados ("Dividir la cuenta", spec 010; "Combinar método de pago", spec 011) y la nueva condición sobre "Liberar Mesa" (amend spec 036 FR-010, spec 026 FR-007/FR-008, spec 028 FR-016). No se requiere entrada nueva en `registro-de-anomalias.md`: es reordenamiento de navegación/retiro de funcionalidad sobre una pantalla ya existente, sin tocar reglas de precio, inventario o facturación (spec.md, Autorización de negocio).
- **Principio III (Characterization tests)** ✅ — 0 tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (gate trivial). Los existentes en `pos-backend/app/characterization_tests/` sobre `checkout_and_send`, `table_sessions.close` y participantes no se tocan porque esta spec no cambia backend (research.md §5) — se verifica que sigan en verde como evidencia de que no hubo cambio de contrato.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — el único comportamiento nuevo (condicionar "Liberar Mesa" a `centralState() !== 'validar-pago'`) está definido en spec.md, FR-001/FR-002.
- **Principio V (No refactors oportunistas)** ✅ — el plan solo toca los archivos con un punto de entrada a alguno de los dos flujos eliminados o al botón condicionado; no se reescribe `payment-attempt-review-panel.component.ts` (FR-003 ya cumplido, research.md §3) ni ningún archivo de backend.
- **Principio VI (Evolución incremental)** ✅ — un solo tipo de cambio (retiro de UI/lógica de frontend + una condición nueva sobre un botón existente), sin migración de datos ni cambio de arquitectura. La decisión de **no** tocar el backend (endpoints `assignments`/`participants`/`payments: list[PaymentIn]`) evita mezclar un cambio de contrato de API — de mayor riesgo y alcance — dentro de este mismo incremento (ver research.md §4 para el razonamiento completo).
- **Principio VII (Datos históricos)** ✅ N/A parcial — no se toca facturación ni se recalcula ninguna venta ya cerrada. Explícitamente relevante aquí: el modelo `session_participant` y el esquema `payments: list[PaymentIn]` (backend) **no se eliminan**, precisamente porque son compartidos con la venta ya facturada de sesiones históricas divididas o pagadas con métodos combinados — alterarlos retroactivamente violaría este principio. Ver research.md §4.
- **Principio VIII (Evolución del modelo de datos)** N/A — spec.md, Key Entities: "no se le agregan ni modifican entidades de datos". Ningún cambio de esquema de backend.
- **Principio IX (Dependencias nuevas)** ✅ — no se agrega ninguna dependencia; toda la lógica (condicionar un botón, retirar dos componentes/opciones) usa signals/computed y control flow ya presentes en el proyecto.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` + `/speckit-implement`: mantener en verde `pos-checkout-panel.component.spec.ts` (ajustando el test T035 y los que referencian "Dividir la cuenta"), `payment-draft.util.spec.ts`, `session-bill-panel.component.spec.ts`, `payment-validation-block.component.spec.ts`, `payment-attempt-review-panel.component.spec.ts` (sin cambios de intención, deben seguir en verde) y `pos-terminal.store.spec.ts`; eliminar `split-bill-panel.component.spec.ts` junto con el componente; agregar tests nuevos para la condición de "Liberar Mesa" (FR-001/FR-002/SC-001/SC-005) y para la ausencia de "Combinar con otro método" (FR-004/FR-007/SC-002).
- **Principio XI (Negocio vs. técnico)** ✅ — la decisión de negocio (eliminar ambas funcionalidades) está en spec.md, Clarifications; el hallazgo técnico de que FR-003 ya está cumplido sin cambios (backend ya rechaza pagos insuficientes, el bloque de confirmación nunca tuvo combinar) queda registrado en este plan y en research.md, no decidido unilateralmente sin trazabilidad.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (reporte + tres rondas de clarificación) → Spec 046 → Decisiones (Clarifications) → este Plan → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se redactan en español de Colombia.

**Resultado**: Gate PASA sin violaciones. No se requiere la tabla de Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: research.md, data-model.md y
contracts/ui-store-contract.md confirman que el diseño no agrega dependencias nuevas
(Principio IX), no modifica el modelo de datos de backend (Principio VIII, N/A), no toca ningún
characterization test (Principio III) y no altera ningún dato histórico ni el esquema de
`payments`/`participants` que sostiene ventas y facturas ya emitidas (Principio VII). El único
ajuste de alcance detectado durante el diseño es el hallazgo de que FR-003 ya está satisfecho
(ver Summary) — reduce trabajo, no lo amplía. Gate sigue PASANDO sin violaciones.

## Project Structure

### Documentation (this feature)

```text
specs/046-dividir-cuenta-pago-pendiente/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/            # Fase 1 (/speckit-plan) — contrato de UI/store, sin API nueva
│   └── ui-store-contract.md
└── tasks.md              # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repositorio de la aplicación)

El código vive en el repositorio hermano `../pos-heladeria` (Angular), no en este repositorio de
specs (`pos-specs`). No se requiere ningún cambio en `../pos-backend` para esta spec (ver
Technical Context → Storage, y research.md §4).

```text
pos-heladeria/src/app/modules/tables/
├── components/
│   ├── pos-checkout-panel.component.ts        # HOY: "Liberar Mesa" (líneas 227-233) sin condición de pago pendiente →
│   │                                           # SE CONDICIONA a `store.centralState() !== 'validar-pago'`. Se retiran:
│   │                                           # el botón "Dividir la cuenta…" (líneas 98-104 y 137-143), el bloque
│   │                                           # `@if (splitOpen() …) { <app-split-bill-panel …> }` (líneas 238-247), el
│   │                                           # signal `splitOpen` (línea 253), el import de `SplitBillPanelComponent`
│   │                                           # (línea 13), y los métodos ahora huérfanos `sessionOrders()` (321-324),
│   │                                           # `tableLabel()` (327-330) y `onSplitSaved()` (333-336)
│   ├── split-bill-panel.component.ts           # SE ELIMINA por completo (archivo + su .spec.ts) — sin punto de entrada
│   │                                           # tras retirar sus dos únicos invocadores en pos-checkout-panel
│   ├── session-bill-panel.component.ts         # Se retira el toggle "Cuenta única"/"Dividir por comensal" (líneas
│   │                                           # 116-140), el signal `mode` (línea 222) y la rama `mode() === 'split'`
│   │                                           # de: la plantilla (líneas 156-173), `canSplit()` (línea 246), `ready()`
│   │                                           # (253-261), `ngOnChanges()` (277-292, ya no resetea `splits`),
│   │                                           # `setSplitPayment()` (299-303) y `buildPayload()` (334-352, ya no
│   │                                           # construye `SplitPayment[]`) — el desglose de solo lectura por
│   │                                           # comensal (líneas 70-100, `bill.split`) NO se toca (spec 026 FR-006,
│   │                                           # fuera de alcance, ver spec.md Out of Scope)
│   ├── payment-input.component.ts              # Se retira el checkbox "Combinar con otro método" (líneas 58-66), el
│   │                                           # bloque `@if (draft().combined) { … }` de segundo método (69-89), y
│   │                                           # los métodos `toggleCombined()`, `setSecondMethod()`,
│   │                                           # `setSecondAmount()`, `remainder()` (146-164) — usado tanto por
│   │                                           # `pos-checkout-panel.component.ts` (mostrador) como por
│   │                                           # `session-bill-panel.component.ts` (Cuenta de la mesa), un único punto
│   │                                           # de cambio cubre ambos call sites (FR-004/FR-007)
│   └── payment-attempt-review-panel.component.ts  # SIN CAMBIOS — ya cumple FR-003 (research.md §3)
├── services/
│   ├── payment-draft.util.ts                   # Se retiran del `PaymentDraft` los campos `combined`/`secondMethodId`/
│   │                                           # `secondAmount` (líneas 12-19); `emptyPaymentDraft()`, `paymentLines()`
│   │                                           # (26-34), `nonCashAmount()` (42-50) y `paymentIssue()` (70-89) se
│   │                                           # simplifican a un único método — `missingAmount()`/la regla "no
│   │                                           # efectivo no puede superar el total" quedan intactas (siguen
│   │                                           # aplicando sobre un único método)
│   └── pos-terminal.store.ts                   # SIN CAMBIOS — se reutiliza `centralState()` (líneas 432-441) y
│                                                 # `pendingOfSelectedTable()` (líneas 411-414) tal cual; `releaseTable()`
│                                                 # (líneas 1481-1495) no cambia su lógica interna
```

**Structure Decision**: Se modifica in-place el módulo `tables` ya existente en `pos-heladeria`
(sin módulo nuevo). El único archivo que se elimina por completo es `split-bill-panel.component.ts`
(y su `.spec.ts`); el resto son eliminaciones acotadas dentro de archivos que se conservan. No se
crea ningún componente ni servicio nuevo — condicionar "Liberar Mesa" reutiliza estado ya existente
en el store, sin ampliar su superficie pública.

## Complexity Tracking

*Sin violaciones que justificar — el Constitution Check pasa limpio (ver arriba).*
