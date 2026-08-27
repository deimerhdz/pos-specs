---

description: "Task list for: Corrección de bugs y mejoras — Menú QR"
---

# Tasks: Corrección de bugs y mejoras — Menú QR

**Input**: Design documents from `/specs/041-correccion-bugs-menu-qr/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/README.md](./contracts/README.md),
[quickstart.md](./quickstart.md)

**Tests**: `spec.md` exige explícitamente pruebas nuevas ("Se agregan pruebas para logout +
refresh", "Se agregan pruebas para logout + browser back/forward", criterios de aceptación de las
4 historias) y `plan.md`/Constitución (Principio X, Verificación Obligatoria) ya comprometen crear
el primer `*.spec.ts` de cada archivo sin cobertura hoy — los tests de abajo **no son opcionales**
en esta spec.

**Alcance**: todo el trabajo de código vive en el repositorio sibling `../pos-heladeria`
(Angular). **`../pos-backend` no se toca** — confirmado en `plan.md`/`contracts/README.md`. Rutas
de fichero abajo son relativas a la raíz de `pos-heladeria` salvo que se indique lo contrario.

**Nota sobre paralelismo**: las 4 historias tocan mayormente archivos distintos y son
independientes entre sí, con una única excepción: **US1 y US3 comparten**
`src/app/modules/tables/pages/public-menu.component.ts` y su spec nuevo
`public-menu.component.spec.ts` (US1 crea el archivo; US3 lo extiende). Las tareas que tocan ese
par de archivos **no se marcan `[P]` entre sí** y se ordenan secuencialmente (US1 antes que US3)
para evitar conflictos de edición — la dependencia es solo de archivo, no funcional (research.md).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3, US4)

## Phase 1: Setup

**Purpose**: confirmar la línea base antes de tocar código.

- [X] T001 Confirmar la línea base en `pos-heladeria`: verificar que **no** existen hoy
      `src/app/modules/tables/pages/public-menu.component.spec.ts`,
      `src/app/modules/tables/components/table-qr.component.spec.ts`,
      `src/app/modules/tables/services/table-qr.util.spec.ts`,
      `src/app/modules/tables/pages/table-qr-sheet.component.spec.ts` ni
      `src/app/modules/tables/services/diner-token.store.spec.ts` (los 5 archivos que esta spec
      crea por primera vez), y que
      `src/app/modules/products/pages/product-form.component.spec.ts` **sí existe** (el que esta
      spec extiende, no crea) — `research.md`/`plan.md` §Testing.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: confirmar que la suite de tests del frontend corre en verde antes de agregar
cobertura nueva — gate compartido por las 4 historias, sin infraestructura nueva que construir
(ninguna historia depende de código compartido nuevo).

- [X] T002 Ejecutar la suite de tests actual de `pos-heladeria` (`ng test` / el comando que use el
      builder `@angular/build:unit-test`, `angular.json`) y confirmar que pasa en verde antes de
      cualquier cambio de esta spec — línea base limpia para las 4 historias.

**Checkpoint**: línea base confirmada — cualquier historia de usuario puede empezar desde aquí.

---

## Phase 3: User Story 1 - Cerrar sesión del Menú QR invalida el acceso de verdad (Priority: P1) 🎯 MVP

**Goal**: después de cerrar sesión, ni recargar, ni "Atrás"/"Adelante", ni reabrir la misma URL en
la misma pestaña deben permitir crear una sesión nueva sin volver a escanear el QR.

**Independent Test**: abrir el menú vía QR, crear sesión, cerrar sesión, y verificar que reload,
"Atrás", "Adelante" y reabrir la URL en la misma pestaña **no** ofrecen la pantalla de nombre;
verificar que una pestaña privada nueva con la misma URL sí funciona (quickstart.md, Bug 1).

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar contra el código actual antes de implementar.

- [X] T003 [P] [US1] Crear `src/app/modules/tables/services/diner-token.store.spec.ts`: casos para
      el comportamiento ya existente de `set()`/`clear()`/lectura por query param (línea base,
      sin cambios) **más** los casos nuevos de la marca de acceso cerrado: `markExited(token)`
      persiste en `sessionStorage`; `isExited(token)` devuelve `true` solo para el token marcado;
      `isExited(otroToken)` devuelve `false` (FR-001; `research.md` Decisión 1;
      `data-model.md` §Marca de acceso cerrado).
- [X] T004 [US1] Crear `src/app/modules/tables/pages/public-menu.component.spec.ts` con los casos
      de Acceptance Scenarios 3-7 de `spec.md` User Story 1: tras `exit()`, un `ngOnInit`
      posterior con el mismo `:token` muestra el nuevo estado `'exited'` (no `'name'`) — cubre
      recarga, "Atrás" y "Adelante" (los tres invocan `ngOnInit` de la misma forma en el test, ya
      que Angular no distingue el disparador); reabrir con un `:token` **distinto** (simulando una
      pestaña/contexto nuevo sin la marca) sí muestra `'name'` (FR-002 a FR-006). No agregar aún
      los casos de Bug 3 (ícono) — los agrega US3 más adelante sobre este mismo archivo.

### Implementation for User Story 1

- [X] T005 [US1] Extender `src/app/modules/tables/services/diner-token.store.ts`: agregar
      `markExited(token: string): void` (escribe en `sessionStorage`, clave
      `pos.diner.exited_token`, con el mismo patrón `try/catch` que `set()`/`clear()` para modo
      privado/storage bloqueado) e `isExited(token: string): boolean` (lee esa clave y compara
      contra `token`) — hace pasar T003 (`research.md` Decisión 1; `data-model.md`).
- [X] T006 [US1] (depende de T005) Modificar `exit()` en
      `src/app/modules/tables/pages/public-menu.component.ts:1013-1032`: llamar a
      `this.tokenStore.markExited(this.token)` en el mismo punto donde hoy se limpia el token de
      sesión, antes de fijar el estado final de la vista (FR-002; `research.md` Decisión 1).
- [X] T007 [US1] (depende de T005) Modificar el tipo `MenuView`
      (`public-menu.component.ts:32`) agregando el estado `'exited'`, y ajustar `exit()` para usar
      `view.set('exited')` en vez de `view.set('error')` — agregar el bloque de template
      correspondiente con el mensaje "Acceso finalizado, vuelve a escanear el código QR de tu
      mesa", sin ningún botón/enlace que lleve a `confirmName()` u `openSession()` (Acceptance
      Scenario 3; FR-002, FR-006).
- [X] T008 [US1] (depende de T005-T007) Modificar `ngOnInit`
      (`public-menu.component.ts:651-689`): antes de la comprobación existente de
      `!this.tokenStore.token()` (línea 676-679), comprobar
      `this.tokenStore.isExited(this.token)`; si es `true`, fijar `view.set('exited')` y retornar
      sin ofrecer el flujo de nombre — cubre recarga, "Atrás", "Adelante" y reapertura de URL en la
      misma pestaña (FR-002 a FR-005; hace pasar T004).
- [X] T009 [US1] (depende de T005) Revisar `expireSession()`
      (`public-menu.component.ts:1164-1174`): si el `:token` actual está marcado como cerrado
      (`isExited`), no debe fijar `view.set('name')` — debe respetar el mismo estado `'exited'`
      (`research.md` Decisión 2, cierra el segundo camino hacia el formulario de nombre).
- [X] T010 [US1] Ejecutar manualmente los pasos 2-8 de `quickstart.md` §Bug 1 (flujo feliz, cerrar
      sesión, recarga, "Atrás", "Adelante", reapertura de URL, y la verificación positiva en
      pestaña privada nueva) y confirmar cada resultado esperado.

**Checkpoint**: User Story 1 (P1, MVP) funciona y se puede verificar de forma independiente —
`spec.md` SC-001/SC-002.

---

## Phase 4: User Story 2 - El QR descargado de una mesa se identifica y se ajusta al uso físico (Priority: P1)

**Goal**: el PNG descargado del QR de una mesa muestra su identificador real y se puede descargar
en dos tamaños ("Mostrador"/"Sticker"), sin cambiar el destino codificado.

**Independent Test**: descargar el QR de dos mesas distintas en ambos formatos, confirmar que cada
PNG muestra el número de mesa correcto, y que los 4 PNG decodifican la misma URL que el flujo
actual (quickstart.md, Bug 2).

### Tests for User Story 2 ⚠️

- [X] T011 [P] [US2] Crear `src/app/modules/tables/services/table-qr.util.spec.ts`: casos para
      `menuUrlForToken`/`qrDataUrl`/`buildTableQr` ya existentes (línea base, sin cambios) **más**
      la nueva función de composición (p. ej. `labeledQrDataUrl(signedToken, tableLabel, preset)`):
      el preset `'mostrador'` produce un canvas de mayor tamaño que `'sticker'`; ambos incluyen el
      texto del identificador de mesa recibido por parámetro; ambos decodifican al mismo
      `menuUrlForToken(signedToken)` que la función ya existente (FR-008, FR-010 a FR-013;
      `research.md` Decisión 3).
- [X] T012 [P] [US2] Crear `src/app/modules/tables/components/table-qr.component.spec.ts`: el
      modal muestra dos opciones "Mostrador"/"Sticker" en vez de un único botón "Descargar PNG";
      cada opción invoca la función de composición con el `table.number`/`table.name` correctos
      (no un índice de lista); el nombre del archivo descargado sigue incluyendo el identificador
      de la mesa (FR-008, FR-009, FR-014).

### Implementation for User Story 2

- [X] T013 [US2] Agregar a `src/app/modules/tables/services/table-qr.util.ts` la función de
      composición (p. ej. `labeledQrDataUrl`) que dibuja sobre un `<canvas>` 2D nativo la imagen ya
      generada por `qrDataUrl` (`QRCode.toDataURL`, sin cambios) junto con el identificador de mesa
      como texto, dentro de una zona de seguridad, con dos presets de dimensión — referencia:
      Mostrador ~900×1100 px (módulo QR ~700 px), Sticker ~380×460 px (módulo QR ~300 px) —
      exportando el resultado con `canvas.toDataURL('image/png')`; sin modificar
      `menuUrlForToken`/`qrDataUrl`/`buildTableQr` existentes (FR-008, FR-010 a FR-013;
      `research.md` Decisión 3; hace pasar T011).
- [X] T014 [US2] (depende de T013) Modificar
      `src/app/modules/tables/components/table-qr.component.ts`: reemplazar el botón único
      "Descargar PNG" por dos opciones claramente diferenciadas "Mostrador"/"Sticker" que llamen a
      `labeledQrDataUrl` con el `table.number`/`table.name` de `@Input() table`, actualizando
      `downloadPng()` (o el método equivalente) y el nombre del archivo descargado por variante
      (FR-008, FR-009, FR-014; hace pasar T012).
- [X] T015 [US2] Revisar `src/app/modules/tables/pages/table-qr-sheet.component.ts` (generación
      masiva): confirmar que sigue mostrando/usando el identificador correcto (`t.number`/`t.name`)
      de cada mesa al construir sus tarjetas — ajustar solo si la refactorización de T013 cambia la
      firma de alguna función que este archivo consume, sin alterar su comportamiento actual
      (FR-015, sin regresión). Confirmado sin cambios: `buildTableQr(tables, id, width)` conserva su
      firma; `table-qr-sheet.component.ts` sigue leyendo `t.number`/`t.name` directamente, sin tocar.
- [X] T016 [US2] Ejecutar manualmente los pasos de `quickstart.md` §Bug 2: descargar Mostrador y
      Sticker de 2 mesas distintas, confirmar el identificador correcto en cada PNG, escanear los 4
      archivos con una cámara de celular real y confirmar que decodifican la misma URL que el flujo
      actual, y validar la generación masiva sin regresión.

**Checkpoint**: User Story 2 (P1) funciona de forma independiente de User Story 1 — `spec.md`
SC-003 a SC-005.

---

## Phase 5: User Story 3 - El catálogo del Menú QR usa un placeholder neutro para productos sin imagen (Priority: P3)

**Goal**: un producto sin imagen en el catálogo del Menú QR muestra un ícono genérico de "sin
imagen" en vez del emoji de helado.

**Independent Test**: ver el catálogo con un producto sin imagen (ícono genérico) y uno con imagen
(su imagen real, sin cambios) — quickstart.md, Bug 3.

**Nota de secuencia**: esta historia edita `public-menu.component.ts`/`.spec.ts`, creados por US1
(Phase 3) — implementar después de US1 para evitar conflicto de edición sobre el mismo archivo
(no es una dependencia funcional: el ícono no depende de la marca de acceso cerrado).

### Tests for User Story 3 ⚠️

- [X] T017 [US3] Extender `src/app/modules/tables/pages/public-menu.component.spec.ts` (creado en
      T004): agregar casos para un producto sin `image_url` (debe renderizar
      `<app-icon name="image-off">`, no el emoji `🍦`) y uno con `image_url` (sigue mostrando su
      imagen real, sin cambios) — FR-016 a FR-018.

### Implementation for User Story 3

- [X] T018 [P] [US3] Agregar el caso nuevo `"image-off"` al `@switch (name)` de
      `src/app/shared/icon/icon.component.ts`: SVG de trazo único, mismo `viewBox`/`stroke-width`
      que los casos vecinos, representando una imagen con una diagonal (símbolo estándar de
      "imagen no disponible"), sin librería externa nueva (FR-017, FR-019; `research.md`
      Decisión 4). Este archivo es independiente de `public-menu.component.ts`, por lo que puede
      hacerse en paralelo con T017.
- [X] T019 [US3] (depende de T018) Reemplazar el emoji `🍦` en
      `src/app/modules/tables/pages/public-menu.component.ts:346-350` por
      `<app-icon name="image-off" />` — hace pasar T017 (FR-016, FR-018, FR-020: solo este
      archivo del catálogo del comensal, sin tocar las pantallas de staff/admin que reutilizan el
      mismo emoji por su cuenta).
- [X] T020 [US3] Ejecutar manualmente los pasos de `quickstart.md` §Bug 3: confirmar el ícono
      genérico en el producto sin imagen, la imagen real intacta en el producto con imagen, y la
      consistencia visual con el resto de íconos del menú.

**Checkpoint**: User Story 3 (P3) funciona de forma independiente — `spec.md` SC-006/SC-007.

---

## Phase 6: User Story 4 - "Copiar insumos" solo aparece cuando el producto maneja inventario (Priority: P3)

**Goal**: el botón "Copiar insumos y sabores..." del formulario de productos solo se muestra
cuando el switch "maneja inventario" está activado, y reacciona de inmediato al cambiarlo.

**Independent Test**: en el formulario de un producto con varios tamaños, apagar/encender el
switch y confirmar que el botón desaparece/aparece de inmediato, en creación y en edición
(quickstart.md, Bug 4).

### Tests for User Story 4 ⚠️

- [X] T021 [P] [US4] Extender `src/app/modules/products/pages/product-form.component.spec.ts`
      (ya existe): agregar casos para el botón "Copiar insumos..." — oculto con
      `tracks_inventory=false` y `hasSizes && variants.length > 1`; visible con
      `tracks_inventory=true` en las mismas condiciones; reacciona de inmediato a
      `toggleTracksInventory()` en ambos sentidos, sin recargar; comportamiento igual en creación y
      en edición de un producto con insumos ya guardados (FR-021 a FR-024).

### Implementation for User Story 4

- [X] T022 [US4] (depende de T021) Modificar la condición del template en
      `src/app/modules/products/pages/product-form.component.ts:384-389`: cambiar
      `draft().hasSizes && draft().variants.length > 1` por
      `draft().hasSizes && draft().variants.length > 1 && draft().tracks_inventory` — sin tocar
      `copyConfigToOthers()` (`:858-881`) ni ningún otro método, porque la operación ya es
      puramente en memoria hasta `save()` (FR-021 a FR-023, FR-026; `research.md` Decisión 5; hace
      pasar T021).
- [X] T023 [US4] Ejecutar manualmente los pasos de `quickstart.md` §Bug 4: creación y edición de un
      producto con varios tamaños, activar/desactivar el switch sin recargar, confirmar que
      variantes/tamaños/insumos ya guardados no se ven afectados (FR-024, FR-025).

**Checkpoint**: User Story 4 (P3) funciona de forma independiente — `spec.md` SC-008.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final combinada de las 4 historias, sin cambios de alcance nuevos.

- [X] T024 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que las
      pruebas nuevas/extendidas de T003, T004, T011, T012, T017, T021 pasan junto con el resto de
      la suite, sin regresiones (Principio X, Verificación Obligatoria).
- [X] T025 Ejecutar `quickstart.md` de punta a punta (las 4 secciones, pasos manuales incluidos) en
      un entorno con `pos-backend` y `pos-heladeria` corriendo, confirmando cada "Esperado" listado.
- [X] T026 [P] Revisar que ningún archivo de `pos-backend` quedó modificado (`git status` sobre ese
      repositorio) — confirma el alcance declarado en `plan.md`/`contracts/README.md` (sin cambios
      de backend en ninguna de las 4 historias).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — gate compartido, sin bloquear ninguna historia
  más allá de confirmar la línea base.
- **User Stories (Phase 3-6)**: todas dependen de Foundational. US1, US2 y US4 son mutuamente
  independientes y pueden avanzar en paralelo. **US3 depende de US1 solo por archivo compartido**
  (`public-menu.component.ts`/`.spec.ts`) — implementarla después de US1, no en paralelo con ella.
- **Polish (Phase 7)**: depende de las historias que se decida entregar en este incremento.

### User Story Dependencies

- **US1 (P1, MVP)**: sin dependencia de otra historia.
- **US2 (P1)**: sin dependencia de otra historia — archivos completamente distintos de US1.
- **US3 (P3)**: **depende de US1 por archivo** (`public-menu.component.ts`/`.spec.ts`), no
  funcionalmente — puede implementarse después de US1 aunque US2/US4 no estén.
- **US4 (P3)**: sin dependencia de ninguna otra historia.

### Dentro de cada historia

- Tests antes que implementación (T003→T005-T009; T011-T012→T013-T015; T017→T018-T019;
  T021→T022).
- Verificación manual de `quickstart.md` al final de cada historia, antes del checkpoint.

### Parallel Opportunities

- T003 y T004 no son `[P]` entre sí (T004 depende del mismo archivo que después extiende T017,
  pero T003/T004 sí pueden correr en paralelo entre ellos — archivos distintos:
  `diner-token.store.spec.ts` vs `public-menu.component.spec.ts`).
- T011 y T012 en paralelo (archivos distintos: `table-qr.util.spec.ts` vs
  `table-qr.component.spec.ts`).
- T018 (ícono) puede correr en paralelo con cualquier tarea de US1 — archivo independiente
  (`icon.component.ts`); solo T019 (que sí toca `public-menu.component.ts`) debe esperar a que
  terminen las tareas de US1 sobre ese mismo archivo.
- T021 (US4) es independiente de todo lo demás — puede correr en paralelo con cualquier otra
  historia.
- Una vez completadas todas las historias deseadas, T024/T026 pueden correr en paralelo entre sí;
  T025 (quickstart manual) se hace al final, con el código ya estable.

---

## Parallel Example: Fase de Setup + primeras tareas de historias

```bash
# Tras T001/T002 (Setup + Foundational), en paralelo:
Task: "Crear diner-token.store.spec.ts (T003, US1)"
Task: "Crear table-qr.util.spec.ts (T011, US2)"
Task: "Crear table-qr.component.spec.ts (T012, US2)"
Task: "Extender product-form.component.spec.ts (T021, US4)"
Task: "Agregar caso 'image-off' a icon.component.ts (T018, US3 — no depende de US1)"
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational).
2. Completar Phase 3 (User Story 1) — la corrección de mayor riesgo (seguridad/integridad de
   sesión).
3. **Detener y validar**: ejecutar `quickstart.md` §Bug 1 de forma aislada.
4. Desplegar/demostrar si está listo — MVP mínimo viable de esta spec.

### Entrega incremental

1. Setup + Foundational → línea base lista.
2. Agregar US1 (P1, seguridad) → validar → MVP.
3. Agregar US2 (P1, identificación física de mesas) → validar → siguiente incremento, en paralelo
   con el punto anterior si hay capacidad de equipo (archivos distintos).
4. Agregar US3 (P3, placeholder) → validar → después de US1 por archivo compartido.
5. Agregar US4 (P3, botón de insumos) → validar → en cualquier momento, independiente del resto.
6. Phase 7 (Polish) al cerrar el alcance que se decida entregar.

### Estrategia de equipo en paralelo

Con más de una persona: una completa US1, otra US2 en simultáneo (archivos distintos); US4 puede
tomarla una tercera persona en cualquier momento; US3 se agenda después de que US1 termine, por el
archivo compartido — no por bloqueo funcional.

---

## Notes

- `[P]` = archivos distintos, sin dependencias entre sí.
- `[Story]` mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- Los tests de esta spec no son opcionales (`spec.md`, criterios de aceptación de logout;
  Principio X de la Constitución).
- Verificar que los tests fallan antes de implementar (T003/T004, T011/T012, T017, T021 antes que
  su implementación correspondiente).
- Ningún archivo de `pos-backend` se toca en ninguna tarea de esta spec (T026 lo verifica al
  cierre).
