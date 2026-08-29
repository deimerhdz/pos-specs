---

description: "Task list for spec 058 — nombre de facturación editable, texto 'Cobrar' y acciones post-cobro ocultas mientras el cobro está pendiente"
---

# Tasks: Ajustes al panel de cobro de pedido manual

**Input**: Design documents from `/specs/058-panel-cobro-pedido-manual/`
**Prerequisites**: plan.md, spec.md, research.md, quickstart.md (sin data-model.md ni contracts/ —
no aplican a esta spec, ver plan.md)

**Tests**: incluidos — mismo criterio que specs 054-057 (Principio III/X de la constitución).

**Organización**: por historia de usuario de spec.md (US1/US2/US3, en el orden en que aparecen en
spec.md). Todo el código vive en el repositorio hermano `../pos-heladeria` (no en `pos-specs`); sin
cambios en `../pos-backend`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes). Las tres
  historias modifican el mismo único archivo de producción
  (`pos-checkout-panel.component.ts`) y el mismo único archivo de test
  (`pos-checkout-panel.component.spec.ts`, research.md D5) — por eso casi ninguna tarea de
  implementación o de test lleva `[P]` entre historias distintas: marcarlas en paralelo
  introduciría conflictos de merge sobre las mismas líneas, aunque las historias sean
  funcionalmente independientes.
- **[Story]**: US1, US2 o US3 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar el baseline exacto: ejecutar `ng test --watch=false
      --include='src/app/modules/tables/components/pos-checkout-panel.component.spec.ts'` dentro
      de `pos-heladeria` y verificar 18 casos en verde y exactamente 1 fallo preexistente y ajeno a
      esta spec (`'T032: ofrece "Imprimir Pre-cuenta" cuando hay cuenta de sesión'`, ya
      documentado como causa no relacionada en spec 057 research.md Decisión 5 — texto de botón
      obsoleto, nada que ver con "Imprimir Factura"/"Liberar Mesa"), sin ningún otro fallo
      inesperado

---

## Phase 2: Foundational (Blocking Prerequisites)

**No aplica ninguna tarea foundational a esta spec**: las tres historias son independientes entre
sí en cuanto a comportamiento (spec.md) y no comparten ningún prerrequisito bloqueante distinto
del baseline confirmado en Setup (Phase 1) — a diferencia de spec 057, aquí no hay ningún test
roto por una causa ajena que corregir antes de empezar (T001 ya lo confirma). Las historias pueden
empezar directamente tras Phase 1.

---

## Phase 3: User Story 1 - El nombre de facturación se muestra como texto con un botón para editarlo (Priority: P1) 🎯 MVP

**Goal**: el campo "Facturar a nombre de" nace en modo solo lectura (apariencia atenuada) y un
botón "editar" habilita escribir sobre él; al perder foco, vuelve a solo lectura mostrando el valor
recién escrito. El valor enviado a facturación no cambia (FR-001, FR-002, FR-003).

**Independent Test**: abrir el panel de cobro de un pedido aún no enviado a cocina y verificar que
"Facturar a nombre de" se ve atenuado y no editable hasta pulsar el botón de editar; al perder foco
tras escribir, vuelve a verse atenuado con el nuevo valor (quickstart.md Escenario 1).

### Tests for User Story 1

- [X] T002 [US1] En `pos-heladeria/src/app/modules/tables/components/
      pos-checkout-panel.component.spec.ts`: ampliar el test existente `'el nombre de facturación
      va por defecto "Consumidor Final" (T024)'` (líneas ~113-117) para además afirmar que el
      input nace con el atributo `readOnly` (`billingInput!.readOnly` es `true`) — sin quitar la
      aserción de valor por defecto ya existente
- [X] T003 [US1] En el mismo archivo: agregar un caso nuevo que (a) ubique el botón de editar junto
      al campo (p. ej. por `title="Editar nombre de facturación"`), (b) simule un clic sobre él y
      confirme que el input deja de ser `readOnly`, (c) escriba un nombre nuevo y dispare `blur`, y
      (d) confirme que el input vuelve a `readOnly` mostrando el nombre recién escrito (FR-001,
      FR-002)
- [X] T004 [US1] En el mismo archivo: agregar un caso que confirme que, tras cobrar con éxito
      (`checkout-and-send` respondido con éxito) o al cambiar de pedido seleccionado
      (`store.selectedOrderId.set(...)` a otro id), el campo del pedido siguiente nace de nuevo en
      modo solo lectura, aunque el anterior se haya dejado en modo edición (research.md Decisión 2)

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/components/
      pos-checkout-panel.component.ts`: agregar la señal `editandoFacturacion = signal(false)` y
      los métodos `toggleEditarFacturacion()` (la pone en `true`) y `onFacturacionBlur()` (la pone
      en `false`), reflejo exacto de `editandoCliente`/`toggleEditarCliente`/`onClienteBlur` de
      `manual-order-page.component.ts:334-374` (research.md Decisión 1)
- [X] T006 [US1] En el mismo archivo: envolver el `<input>` de "Facturar a nombre de" (líneas
      ~142-153) en un `<div class="relative">`, agregar
      `[readOnly]="!editandoFacturacion()"`, `[class.bg-gray-50]="!editandoFacturacion()"`,
      `[class.text-gray-500]="!editandoFacturacion()"` y `(blur)="onFacturacionBlur()"` al input, y
      agregar el botón `✏️` (`(click)="toggleEditarFacturacion()"`,
      `title="Editar nombre de facturación"`, `absolute right-2 inset-y-0`) — mismo marcado que
      `manual-order-page.component.ts:168-187` (research.md Decisión 1). Hace pasar T002, T003.
- [X] T007 [US1] En el mismo archivo: agregar `this.editandoFacturacion.set(false);` dentro del
      `effect()` existente que ya reinicia `paymentDraft` al cambiar `store.selectedOrderId()`
      (líneas ~282-286) (research.md Decisión 2). Hace pasar T004. Depende de T005.
- [ ] T008 [US1] Ejecutar manualmente quickstart.md Escenario 1 (UI)

**Checkpoint**: el nombre de facturación se comporta como el campo "Cliente" de armado de pedido
manual, verificable de forma independiente.

---

## Phase 4: User Story 2 - El botón de cobro dice simplemente "Cobrar" (Priority: P2)

**Goal**: el botón principal del panel de cobro pendiente dice "Cobrar" (y "Cobrando…" durante el
envío), sin ningún cambio en la lógica de `checkout()`/`checkoutAndSend()` (FR-004, FR-005, FR-006).

**Independent Test**: abrir el panel de cobro de un pedido pendiente y verificar que el botón
principal dice "Cobrar"; confirmar el cobro y verificar que sigue cobrando, facturando y enviando a
cocina igual que antes (quickstart.md Escenario 2). No depende de US1 ni de US3.

### Tests for User Story 2

- [X] T009 [US2] En `pos-checkout-panel.component.spec.ts`: actualizar el test existente `'cobra,
      factura y envía a cocina con "checkout-and-send" (T025)'` (líneas ~135-151) para ubicar el
      botón por el texto exacto `'Cobrar'` (`.trim() === 'Cobrar'`, para no confundirlo con
      "Cobrando…" ni con "Rechazar pedido") en vez de `'Cobrar, Facturar y Enviar a Cocina'` — el
      resto del test (petición a `checkout-and-send`, `billing_customer_name`, etc.) no cambia

### Implementation for User Story 2

- [X] T010 [US2] En `pos-checkout-panel.component.ts`: cambiar el literal de la línea ~173 de
      `'Cobrar, Facturar y Enviar a Cocina'` a `'Cobrar'`, dejando intacto el operador ternario con
      `store.checkoutSubmitting() ? 'Cobrando…' : '...'` (research.md Decisión 3). Hace pasar T009.
- [ ] T011 [US2] Ejecutar manualmente quickstart.md Escenario 2 (UI)

**Checkpoint**: el botón dice "Cobrar" y el cobro sigue funcionando exactamente igual, verificable
de forma independiente de US1 y US3.

---

## Phase 5: User Story 3 - "Imprimir Factura" y "Liberar Mesa" no se muestran mientras el cobro sigue pendiente (Priority: P1)

**Goal**: mientras el panel muestra un pedido aún no enviado a cocina (cobro pendiente), no
aparecen "Imprimir Factura" ni "Liberar Mesa"; en los demás modos del panel (cuenta de mesa, resumen
QR) siguen apareciendo sin cambios (FR-007, FR-008, FR-009).

**Independent Test**: abrir el panel de cobro de un pedido pendiente y verificar que ninguno de los
dos botones aparece; abrir por separado "Cuenta de la mesa" o el modo resumen QR y verificar que
ambos siguen apareciendo igual que hoy (quickstart.md Escenario 3). No depende de US1 ni de US2.

### Tests for User Story 3

- [X] T012 [US3] En `pos-checkout-panel.component.spec.ts`, describe `'PosCheckoutPanelComponent —
      modo terminal-pos'`: invertir la aserción final del test `'T033/spec 029: ofrece "Imprimir
      Factura" para el pedido seleccionado, con cuenta de sesión'` (líneas ~162-180) — con
      `sessionBill` fijado sobre el pedido `'recibida'` (cobro pendiente), el botón debe seguir
      `toBeUndefined()` en vez de `toBeDefined()` (research.md Decisión 4, primer punto);
      actualizar el nombre del test para reflejar que ahora prueba lo contrario (p. ej. `'FR-007:
      no ofrece "Imprimir Factura" mientras el cobro sigue pendiente, aunque ya haya cuenta de
      sesión'`)
- [X] T013 [US3] En el mismo describe: invertir la aserción final del test `'spec 046, FR-002/
      SC-003: "Liberar Mesa" reaparece de inmediato al confirmarse el pago pendiente'` (líneas
      ~220-244) — el pedido propio (`'o1'`) sigue `'recibida'` (cobro pendiente) durante todo el
      test, así que "Liberar Mesa" debe seguir `toBeUndefined()` incluso después de que el pago QR
      pendiente se confirme (research.md Decisión 4, tercer punto: la regla nueva de spec 058 es
      más estricta que la de spec 046). Actualizar el nombre del test para documentar que la regla
      de spec 046 queda subsumida por FR-007
- [X] T014 [US3] En el mismo archivo: mover el test `'T035: "Liberar Mesa" pide la liberación y
      muestra el motivo del 409 si falla'` (líneas ~246-269) del describe `'modo terminal-pos'` al
      describe `'PosCheckoutPanelComponent — pedido ya en cocina, cobro por sesión de mesa'`
      (líneas ~329-492), reusando su `beforeEach` existente (`abiertaOrder()` + `bill` ya
      configurado) en vez de fijar `sessionBill` sobre un pedido `'recibida'` — en ese describe
      "Liberar Mesa" sigue apareciendo sin cambios (FR-008) y el caso de red/409 sigue siendo válido
      (research.md Decisión 4, cuarto punto)
- [X] T015 [P] [US3] En el describe `'pedido ya en cocina, cobro por sesión de mesa'`: agregar un
      caso nuevo que confirme que "Imprimir Factura" y "Liberar Mesa" siguen apareciendo sin cambios
      en ese modo (FR-008) — cobertura explícita que hoy no existe de forma directa (research.md
      Decisión 4, primer punto)

### Implementation for User Story 3

- [X] T016 [US3] En `pos-checkout-panel.component.ts`: agregar el `computed` `pendingCheckout =
      computed(() => this.sidebarMode() === 'terminal-pos' && !!this.store.selectedOrder() &&
      !this.showSessionCharge())` (research.md Decisión 4)
- [X] T017 [US3] En el mismo archivo: envolver el footer completo (líneas ~190-227, ambos botones
      "Imprimir Factura" y "Liberar Mesa") con `@if (!pendingCheckout())` anidado dentro del `@if
      (store.sessionBill(); as bill)` ya existente (research.md Decisión 4). Hace pasar T012, T013,
      T014, T015. Depende de T016.
- [ ] T018 [US3] Ejecutar manualmente quickstart.md Escenario 3 (UI)

**Checkpoint**: las dos acciones post-cobro solo aparecen cuando corresponden, verificable de forma
independiente de US1 y US2.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T019 Ejecutar la suite completa de `pos-checkout-panel.component.spec.ts` (`ng test
      --watch=false --include='...pos-checkout-panel.component.spec.ts'`) y confirmar que todos los
      casos pasan salvo el único fallo preexistente y ajeno a esta spec ya documentado en T001
      (`'T032: ofrece "Imprimir Pre-cuenta"...'`), sin ningún fallo nuevo (Principio X)
- [X] T020 Ejecutar la suite completa de `pos-heladeria` (`ng test`) y confirmar que no aparece
      ningún fallo nuevo fuera de los ya preexistentes y ajenos a esta spec
- [ ] T021 Ejecutar los 3 escenarios de quickstart.md de punta a punta como validación final

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: no aplica ninguna tarea (ver nota arriba) — no bloquea a las
  historias más allá de esperar T001.
- **US1 (Phase 3)**, **US2 (Phase 4)** y **US3 (Phase 5)**: dependen solo de Setup (T001). Son
  funcionalmente independientes entre sí (spec.md), pero las tres tocan
  `pos-checkout-panel.component.ts` y `pos-checkout-panel.component.spec.ts` — coordinar el orden
  de merge si se trabajan a la vez para evitar conflictos de línea (no de lógica).
- **Polish (Phase 6)**: depende de que las tres historias estén completas.

### Dentro de cada historia

- US1: T002-T004 (tests) antes de T005-T007 (implementación, hacen pasar los tres). T008 al final.
- US2: T009 (test) antes de T010 (implementación, hace pasar T009). T011 al final.
- US3: T012-T015 (tests, incluyendo mover/invertir los tres existentes) antes de T016-T017
  (implementación, hacen pasar los cuatro). T018 al final.

### Parallel Opportunities

- Al compartir ambos archivos entre las tres historias, no se recomienda paralelismo real de
  implementación entre historias (ver nota de formato arriba) — la única tarea marcada `[P]` es
  T015, por ser una adición nueva y acotada a un describe que ninguna otra tarea de esta spec toca.
- Dentro de US3, T012, T013 y T014 modifican tests ya existentes en el mismo archivo y conviene
  hacerlos en el orden dado (o al menos revisarlos juntos) para no pisarse las líneas.
- Sugerencia práctica: completar las tres historias de forma secuencial (US1 → US2 → US3, o el
  orden de prioridad que se prefiera) sobre el mismo archivo, en vez de forzar trabajo simultáneo.

---

## Parallel Example: User Story 3

```bash
Task: "Agregar cobertura de 'Imprimir Factura'/'Liberar Mesa' en el describe de cobro por sesión de mesa (T015)"
```

*(Único caso realmente paralelizable de esta spec — ver nota de formato arriba.)*

---

## Implementation Strategy

### MVP First (Setup + User Story 1)

1. Completar Phase 1 (Setup) — confirma el baseline (18/19 en verde, 1 fallo preexistente ajeno).
2. Completar Phase 3 (US1) — entrega el ajuste de mayor prioridad compartida (P1) más directamente
   ligado a la experiencia de edición ya conocida por el cajero.
3. **Detener y validar**: quickstart.md Escenario 1.
4. US2 y US3 se entregan después, de forma incremental, sin romper US1 (mismo archivo, regiones de
   template distintas).

### Incremental Delivery

1. Setup → baseline confirmado, sin cambio de comportamiento observable todavía.
2. + US1 → nombre de facturación editable con toggle → validar (MVP parcial).
3. + US2 → botón "Cobrar" → validar.
4. + US3 → acciones post-cobro ocultas mientras el cobro está pendiente → validar (incluye mover/
   invertir 3 tests existentes, research.md Decisión 4).
5. + Polish.

---

## Notes

- Ningún task de este documento agrega dependencias nuevas ni toca `pos-backend` (spec.md,
  Assumptions).
- T012, T013 y T014 son el punto más delicado de todo el plan: invierten o mueven tests que hoy
  pasan y protegían un comportamiento que spec 058 autoriza explícitamente a endurecer
  (research.md Decisión 4) — no se trata de "arreglar" tests rotos, sino de actualizar
  deliberadamente lo que protegen, con la autorización de negocio ya citada en spec.md.
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
