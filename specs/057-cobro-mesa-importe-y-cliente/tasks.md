---

description: "Task list for spec 057 — importe fijo no efectivo + nombre de cliente en el desglose"
---

# Tasks: Importe fijo para pagos no efectivo y nombre de cliente en el desglose de cobro

**Input**: Design documents from `/specs/057-cobro-mesa-importe-y-cliente/`
**Prerequisites**: plan.md, spec.md, research.md, quickstart.md (sin data-model.md ni contracts/ —
no aplican a esta spec, ver plan.md)

**Tests**: incluidos — mismo criterio que specs 054-056 (Principio III/X de la constitución).

**Organización**: por historia de usuario de spec.md (US1/US2, en orden de prioridad). Todo el
código vive en el repositorio hermano `../pos-heladeria` (no en `pos-specs`); sin cambios en
`../pos-backend`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1 o US2 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar el baseline exacto: ejecutar `ng test --watch=false
      --include='src/app/modules/tables/components/pos-checkout-panel.component.spec.ts'
      --include='src/app/modules/tables/components/session-bill-panel.component.spec.ts'` dentro
      de `pos-heladeria` y verificar que fallan exactamente los 8 casos documentados en
      research.md Decisión 5 (5 en `pos-checkout-panel.component.spec.ts`, 3 en
      `session-bill-panel.component.spec.ts`), sin ningún otro fallo inesperado en esos dos
      archivos

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: dejar los dos archivos de test que ejercitan `PaymentInputComponent` en un estado
donde puedan localizar el campo de importe real, **antes** de escribir ningún caso nuevo de
FR-001/FR-005 sobre ellos — sin esto, cualquier test nuevo sobre el importe fallaría por la misma
causa raíz que los 8 ya rotos (research.md Decisión 5), no por el comportamiento que se quiere
verificar.

**⚠️ CRÍTICO**: esta corrección es un prerrequisito real (Principio V), no una limpieza aparte —
no se puede escribir ningún test de FR-001 sin poder ubicar primero el campo de importe en el DOM.

- [X] T002 [P] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.
      spec.ts`: corregir el selector obsoleto `input[type="number"]` (líneas ~96, ~390, ~423) por
      el selector correcto del `<input>` real que renderiza `app-money-input`
      (`type="text" inputmode="decimal"`, `money-input.component.ts:30-31`) — sin cambiar ninguna
      aserción de fondo, solo cómo se ubica el elemento (research.md Decisión 5)
- [X] T003 [P] Mismo fix en `pos-heladeria/src/app/modules/tables/components/
      session-bill-panel.component.spec.ts` (líneas ~87, ~150) (research.md Decisión 5)
- [X] T004 Ejecutar de nuevo los dos archivos de T001 y confirmar que 7 de los 8 casos antes rotos
      ahora pasan (el 8º, `'T032: ofrece "Imprimir Pre-cuenta"...'`, resultó tener una causa no
      relacionada — texto de botón obsoleto, nada que ver con el importe — research.md Decisión 5;
      queda fuera de alcance de esta spec, sin tocar), sin ningún cambio de comportamiento de
      producción todavía (el fix es puramente de selector, más un ajuste de valor esperado ya
      formateado por `MoneyInputComponent`). Depende de T002, T003.

**Checkpoint**: los dos archivos de test quedan en verde y listos para recibir casos nuevos. Las
historias de usuario pueden empezar.

---

## Phase 3: User Story 1 - El importe de un pago no efectivo no se puede editar (Priority: P1) 🎯 MVP

**Goal**: en cuanto el cajero elige un método de pago distinto a efectivo, el campo de importe
queda fijo en el total exacto, en los dos flujos de cobro existentes (Cuenta de la mesa y Cobrar
pedido/mostrador); con efectivo, el campo sigue editable exactamente como hoy.

**Independent Test**: abrir cualquiera de los dos flujos de cobro, elegir un método no efectivo, y
verificar que el campo de importe no admite edición y muestra el total exacto; cambiar a efectivo
y verificar que vuelve a ser editable (quickstart.md Escenarios 1-2).

### Tests for User Story 1

- [X] T005 [P] [US1] Crear `pos-heladeria/src/app/modules/tables/components/
      payment-input.component.spec.ts` (no existía): con efectivo elegido, el importe sigue
      editable (no regresión); con un método no efectivo elegido, el importe queda deshabilitado y
      muestra el total exacto; al volver a efectivo, vuelve a ser editable (FR-001, FR-002, FR-004)
- [X] T006 [P] [US1] Agregar en `session-bill-panel.component.spec.ts`: al elegir un método no
      efectivo, el campo de importe queda deshabilitado con el total exacto y el botón "Cobrar y
      cerrar mesa" se habilita sin ninguna otra interacción (FR-001, SC-002); con efectivo, el
      comportamiento actual no cambia
- [X] T007 [P] [US1] Agregar en `pos-checkout-panel.component.spec.ts` el mismo caso, sobre el
      flujo "Cobrar pedido"/"Pedido de mostrador" (FR-003)

### Implementation for User Story 1

- [X] T008 [US1] En `pos-heladeria/src/app/modules/tables/components/payment-input.component.ts`:
      agregar `[disabled]="!isCash(draft().methodId)"` al `<app-money-input>` ya existente
      (líneas ~52-56) — usa el `setDisabledState` que `MoneyInputComponent` ya implementa por
      completo, sin ningún componente ni lógica nueva (research.md Decisión 1). Hace pasar T005,
      T006, T007. Depende de T004.
- [ ] T009 [US1] Ejecutar manualmente quickstart.md Escenarios 1 y 2 (UI, ambos flujos de cobro)

**Checkpoint**: el importe de un cobro no efectivo ya no se puede editar en ningún flujo,
verificable de forma independiente.

---

## Phase 4: User Story 2 - Nombre de cliente en vez de "Sin asignar (mesero)" (Priority: P2)

**Goal**: en el desglose por comensal de "Cuenta de la mesa", la línea de ítems sin comensal
asignado muestra el nombre de cliente de la orden cuando existe, en vez de la etiqueta genérica
"Sin asignar (mesero)".

**Independent Test**: abrir "Cuenta de la mesa" de una orden con nombre de cliente guardado y con
ítems sin comensal asignado, y verificar que esa línea muestra el nombre del cliente
(quickstart.md Escenario 3). No depende de que US1 esté implementada.

### Tests for User Story 2

- [X] T010 [P] [US2] Agregar en `session-bill-panel.component.spec.ts`: con `@Input customerName`
      no vacío y una línea de `bill.split` con `display_label: null`, el desglose muestra ese
      nombre en vez de "Sin asignar (mesero)" (FR-005); con `customerName` vacío, esa línea sigue
      mostrando "Sin asignar (mesero)" (FR-006, no regresión); una línea con `display_label` no
      nulo no cambia su etiqueta (FR-007)

### Implementation for User Story 2

- [X] T011 [US2] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.
      component.ts`: cambiar `lineLabel(label)` (líneas ~271-273) a `return label ||
      (this.customerName.trim() || 'Sin asignar (mesero)')` (research.md Decisión 4). Hace pasar
      T010. Depende de T004.
- [ ] T012 [US2] Ejecutar manualmente quickstart.md Escenario 3 (UI)

**Checkpoint**: el desglose de "Cuenta de la mesa" muestra el nombre del cliente cuando existe,
verificable de forma independiente de US1.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [X] T013 Ejecutar la suite completa de `pos-heladeria` (`ng test`) y confirmar que 7 de los 8
      casos de T001 ahora pasan y que no aparece ningún fallo nuevo — los 3 fallos preexistentes
      ajenos a esta spec (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`) y el
      caso "T032" (research.md Decisión 5, causa no relacionada) deben seguir siendo los únicos
      restantes, sin relación con esta spec (Principio X)
- [ ] T014 Ejecutar los 3 escenarios de quickstart.md de punta a punta como validación final

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup (T001, para saber el baseline exacto a corregir).
  Bloquea ambas historias.
- **US1 (Phase 3)** y **US2 (Phase 4)**: dependen solo de Foundational (T004) — tocan archivos
  distintos de producción (`payment-input.component.ts` vs. `session-bill-panel.component.ts`) y
  pueden avanzar en paralelo entre sí, aunque ambas agregan casos nuevos al mismo archivo de test
  `session-bill-panel.component.spec.ts` (coordinar el orden de merge de T006 y T010 para evitar
  conflicto de líneas, no de lógica).
- **Polish (Phase 5)**: depende de que ambas historias estén completas.

### Dentro de Foundational

T001 → T002, T003 (corrigen exactamente los casos que T001 confirmó rotos). T002, T003 → T004.

### Dentro de cada historia

- US1: T005-T007 (tests) antes de T008 (implementación, hace pasar los tres). T009 al final.
- US2: T010 (test) antes de T011 (implementación, hace pasar T010). T012 al final.

### Parallel Opportunities

- Foundational: T002 y T003 en paralelo (archivos distintos).
- US1: T005, T006, T007 en paralelo entre sí (T005 es un archivo nuevo; T006/T007 archivos
  existentes distintos).
- US1 y US2 pueden trabajarse en paralelo una vez terminado Foundational (archivos de producción
  distintos); coordinar solo el merge de sus respectivos casos nuevos en
  `session-bill-panel.component.spec.ts` si ambas historias avanzan a la vez.

---

## Parallel Example: Foundational

```bash
Task: "Corregir selector obsoleto en pos-checkout-panel.component.spec.ts (T002)"
Task: "Corregir selector obsoleto en session-bill-panel.component.spec.ts (T003)"
```

## Parallel Example: User Story 1

```bash
Task: "Crear payment-input.component.spec.ts (T005)"
Task: "Caso nuevo en session-bill-panel.component.spec.ts (T006)"
Task: "Caso nuevo en pos-checkout-panel.component.spec.ts (T007)"
```

---

## Implementation Strategy

### MVP First (Foundational + User Story 1)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational) — deja los tests existentes en verde antes
   de tocar nada más.
2. Completar Phase 3 (US1) — entrega el ajuste que protege dinero real (importe fijo no efectivo),
   el más urgente de los dos.
3. **Detener y validar**: quickstart.md Escenarios 1 y 2.
4. US2 se entrega después, de forma incremental, sin romper US1 (archivos de producción distintos).

### Incremental Delivery

1. Foundational → tests existentes en verde, sin cambio de comportamiento observable todavía.
2. + US1 → importe fijo para no efectivo, en ambos flujos → validar → demo (MVP).
3. + US2 → nombre de cliente en el desglose → validar.
4. + Polish.

---

## Notes

- Ningún task de este documento agrega dependencias nuevas ni toca `pos-backend`
  (spec.md, Assumptions).
- T002/T003 son el prerrequisito más importante de todo el plan: sin ellos, ningún test nuevo de
  US1 puede siquiera ejecutarse (research.md Decisión 5) — confirmar T004 en verde antes de seguir.
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
