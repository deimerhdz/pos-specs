---

description: "Task list for spec 082 — estandarización de íconos en el panel de administración"
---

# Tasks: Estandarización de Íconos en el Panel de Administración

**Input**: Design documents from `/specs/082-estandarizacion-iconos-admin/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md,
contracts/icon-component-contract.md, quickstart.md

**Tests**: incluidos — el proyecto exige verificar toda funcionalidad nueva (Principio X de la
constitución) y el plan (Constitution Check) deja la verificación pendiente explícitamente para
esta fase, no como opcional.

**Organización**: por historia de usuario de spec.md (US1-US3, priorizadas P1/P1/P2). Todo el
código vive en un solo repositorio: `../pos-heladeria` (Angular). `pos-backend` no se toca (plan.md,
Technical Context).

**Nota de re-planeación (durante `/speckit-implement`)**: al implementar T008/T021 se descubrió que
2 de sus 3 archivos no usaban el emoji `✕` catalogado sino un ícono SVG artesanal escrito a mano —
un tercer patrón de "ícono artesanal" no detectado en `/speckit-plan` (ni `app-icon` ni emoji). Una
búsqueda amplia encontró 14 archivos / 43 íconos SVG artesanales adicionales en el panel de
administración, incluido un tercer punto de temática de heladería (ícono con forma de cono/copa en
`manual-order-page.component.ts`, usado para "Modificar Toppings"). El usuario decidió expandir el
alcance de esta misma feature (ver research.md, sección "Hallazgo durante `/speckit-implement`", y
`data-model.md`, sección "Descubiertos durante la implementación"). Las tareas de abajo ya
incorporan esa expansión — no reflejan el tasks.md original de `/speckit-tasks`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1, US2 o US3 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 [P] Registrar la línea base de tests: correr `ng test` en `pos-heladeria` y anotar el
      resultado (número de tests en verde/rojo) para poder distinguir después cualquier regresión
      introducida por esta spec de fallos preexistentes (Principio III/X de la constitución).
      **Resultado**: 822 en verde / 19 en rojo (841 totales, 6 archivos con fallas). Los 19 rojos ya
      fallaban antes de tocar código: `app.spec.ts` (2, conflicto de `TestBed`), `auth.service.spec.ts`
      (9), `tenant.service.spec.ts` (3), `menu.service.spec.ts` (1) — ninguno relacionado con íconos;
      y 2 en `pos-order-panel.component.spec.ts`/`pos-checkout-panel.component.spec.ts` ya
      documentados como preexistentes en spec 076 ("Imprimir Pre-cuenta"/"Marcar pedido listo").
- [X] T002 Instalar la dependencia `material-icons` (npm, autoalojable) en
      `pos-heladeria/package.json` (research.md, Decisión D1). **Resultado**: `material-icons@1.13.14`
      agregado a `dependencies`.
- [X] T003 Copiar los archivos de fuente de la variante **Outlined** de `material-icons` (desde
      `node_modules/material-icons`) a `pos-heladeria/public/fonts/material-icons/` (research.md,
      Decisión D1; depende de T002). **Resultado**: `material-icons-outlined.woff2` (155 KB) y
      `.woff` (182 KB, fallback) copiados.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: el nuevo componente de ícono reutilizable y su fuente autoalojada son un
prerrequisito técnico para migrar cualquier ícono — ninguna historia de usuario (US1/US2/US3) puede
empezar sin esto, sin importar su prioridad relativa.

**⚠️ CRÍTICO**: ninguna tarea de historia de usuario empieza hasta completar esta fase.

- [X] T004 [P] Crear el catálogo semántico de íconos en
      `pos-heladeria/src/app/shared/icon-mi/icon-catalog.ts` (`Record<string, string>` de nombre
      semántico → ligadura de Material Icons), usando exactamente las entradas de `data-model.md`
      (los 35 nombres hoy soportados por `app-icon` + los nombres semánticos derivados de los emoji
      catalogados). **Ampliado durante la implementación** con las 9 entradas nuevas de
      `data-model.md`, sección "Descubiertos durante la implementación" (`menu`, `notifications`,
      `expand_more`, `lock`, `add`, `add_circle`, `schedule`, `qr_code_scanner`, `delete`).
- [X] T005 Agregar la declaración `@font-face` (apuntando a los archivos copiados en T003) y la
      clase utilitaria `.material-icons-outlined` en `pos-heladeria/src/styles.css` (research.md,
      Decisión D1; depende de T003).
- [X] T006 [P] Test unitario en
      `pos-heladeria/src/app/shared/icon-mi/icon-mi.component.spec.ts` (escrito primero, debe fallar
      hasta T007): verifica que el componente renderiza la ligadura correcta para un `name` conocido
      del catálogo (T004), que por defecto queda `aria-hidden="true"`, que con el input `ariaLabel`
      queda `role="img"` + `aria-label` igual al valor recibido, y que un `name` desconocido cae al
      ícono de reserva `help_outline` en vez de quedar vacío
      (contracts/icon-component-contract.md).
- [X] T007 Crear el componente Angular standalone `IconComponent` (selector `app-mi-icon`, distinto
      de `app-icon`) en `pos-heladeria/src/app/shared/icon-mi/icon-mi.component.ts`: input `name`
      requerido (resuelto contra `icon-catalog.ts` de T004), input opcional `ariaLabel`, tamaño y
      color heredados del elemento padre (mismo patrón de herencia que usa hoy `app-icon`, ahora vía
      `font-size`/`color` en vez de `currentColor` de SVG), fallback `help_outline` para `name`
      desconocido. Debe hacer pasar T006. Depende de T004, T005.

**Checkpoint**: fundación lista — las historias de usuario pueden empezar (en paralelo si hay
capacidad, US1 y US2 no comparten archivos entre sí).

---

## Phase 3: User Story 1 - Íconos consistentes en el resto del panel de administración (Priority: P1) 🎯 MVP

**Goal**: reemplazar, en cada pantalla del panel de administración salvo el panel principal
(dashboard) y la marca del panel lateral (cubiertos por US2 porque su edición no puede separarse de
la remoción de la temática de heladería en esos mismos dos archivos — ver spec.md, Historia 2), todo
ícono mostrado hoy vía `app-icon`, vía emoji, o vía SVG artesanal escrito a mano, por el nuevo
componente `app-mi-icon` (T007).

**Independent Test**: navegar caja, inventario, mesas/terminal de personal, usuarios y
configuración, y confirmar que ningún ícono se ve como emoji o como SVG artesanal — todos vienen del
nuevo componente (quickstart.md, Escenario 1, sin contar panel principal). No depende de US2 para
ser verificable en estas pantallas.

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar antes de la implementación.

- [ ] T008 [P] [US1] Test en `cash-dashboard.component.spec.ts` (emoji `✕`),
      `cash-movement-modal.component.spec.ts`, `cash-arqueo-modal.component.spec.ts`,
      `inventory-item-form.component.spec.ts`, `stock-adjust-modal.component.spec.ts` y
      `purchase-form.component.spec.ts` (estos 5 últimos: ícono `<svg>` artesanal de cerrar, no
      emoji — corrección respecto al tasks.md original): ninguno renderiza ya el glifo/SVG anterior;
      todos renderizan `<app-mi-icon name="close" ariaLabel="Cerrar">` (`purchase-form` además en su
      botón de eliminar fila).
- [ ] T009 [P] [US1] Test en `inventory-page.component.spec.ts`: sin `✕` (línea ~345) ni los dos
      íconos `<svg>` de "+" (botones "Nuevo insumo"/"Nueva compra"); con `<app-mi-icon
      name="close">` y `<app-mi-icon name="add">` respectivamente.
- [ ] T010 [P] [US1] Test en `users-page.component.spec.ts` y `user-role-modal.component.spec.ts`:
      sin `👥` (estado vacío, línea 84) ni `✕` (línea 29); con `<app-mi-icon name="group">` y
      `<app-mi-icon name="close" ariaLabel="Cerrar">` respectivamente.
- [ ] T011 [P] [US1] Test en `tenant-info.component.spec.ts`: sin `🎨🏪🧾🖨️✓👤`; con
      `<app-mi-icon>` de `palette`, `storefront` (x2, líneas 61 y 170), `receipt`, `print`, `check` y
      `person`.
- [ ] T012 [P] [US1] Test en `tables-page.component.spec.ts`: sin `🪑📷✏️🔴🟢` (líneas 55, 80 y
      demás); con `<app-mi-icon>` de `table_restaurant`, `photo_camera`, `edit`, y `circle` para los
      indicadores de estado, conservando la clase de color roja/verde ya existente (FR-008).
- [ ] T013 [P] [US1] Test en `table-sessions.component.spec.ts`: sin `✅` (línea 282) ni el ícono
      `<svg>` de círculo+cruz del botón "nueva mesa"; con `<app-mi-icon name="check_circle">` y
      `<app-mi-icon name="add_circle">` respectivamente.
- [ ] T014 [P] [US1] Test en `pos-order-panel.component.spec.ts`: sin `🍽️📍📞🛵✓`; con
      `<app-mi-icon>` de `restaurant`, `location_on`, `call`, `delivery_dining` y `check`.
- [ ] T015 [P] [US1] Test en `pos-checkout-panel.component.spec.ts`: sin `✏️🧾🔓`; con
      `<app-mi-icon>` de `edit`, `receipt` y `lock_open`.
- [ ] T016 [P] [US1] Test en `pos-tables-panel.component.spec.ts`: sin `🧾` ni el ícono `<svg>` de
      lupa del buscador de mesas; con `<app-mi-icon name="receipt">` y
      `<app-mi-icon name="search">` respectivamente.
- [ ] T017 [P] [US1] Test en `cart.component.spec.ts` (componente compartido con el menú público —
      research.md, Decisión D6): sin `🛒✕`; con `<app-mi-icon>` de `shopping_cart` y `close`.
- [ ] T018 [P] [US1] Test en `product-select.component.spec.ts` (compartido, D6): sin `🏷️`; con
      `<app-mi-icon name="sell">`.
- [ ] T019 [P] [US1] Test en `payment-attempt-review-panel.component.spec.ts` (compartido, D6): sin
      `💳`; con `<app-mi-icon name="credit_card">`.
- [ ] T020 [P] [US1] Test en `pos-catalog-drawer.component.spec.ts` (compartido, D6): sin `🏷️`; con
      `<app-mi-icon name="sell">`.
- [ ] T021 [P] [US1] Test en `cash-page.component.spec.ts`: sin el ícono `<svg>` de reloj (duración
      del turno); con `<app-mi-icon name="schedule">`.
- [ ] T022 [P] [US1] Test en `cash-report.component.spec.ts`: sin el ícono `<svg>` de candado
      ("Turno cerrado"); con `<app-mi-icon name="lock">`.
- [ ] T023 [P] [US1] Test en `bill-summary.component.spec.ts`: sin el ícono `<svg>` de moto de
      reparto; con `<app-mi-icon name="delivery_dining">`.
- [ ] T024 [P] [US1] Test en `header.component.spec.ts`: sin los 6 íconos `<svg>` artesanales
      (menú hamburguesa, campana, flecha desplegable, "Mi plan", "Cambiar contraseña", "Cerrar
      sesión"); con `<app-mi-icon>` de `menu`, `notifications`, `expand_more`, `layers`, `settings` y
      `logout` respectivamente.
- [ ] T025 [P] [US1] Test en `pos-terminal-header.component.spec.ts`: sin los 3 íconos `<svg>`
      artesanales (turno/cajero, "Abrir turno de caja", "Bloquear terminal"); con `<app-mi-icon>` de
      `group`, `credit_card` y `lock` respectivamente.
- [ ] T026 [P] [US1] Test en `manual-order-page.component.spec.ts`: sin los 20 íconos `<svg>`
      artesanales catalogados en `data-model.md` (volver, buscar, escanear, badges de tipo de orden,
      pestañas, editar nombre x2, cliente, notas/toppings/eliminar x2, botón de enviar); cada uno con
      su `<app-mi-icon>` equivalente de `data-model.md` (incluye `tune` para "Modificar Toppings",
      que reemplaza el ícono con forma de cono/copa de helado — ver nota de heladería en
      `data-model.md`).

### Implementation for User Story 1

- [ ] T027 [P] [US1] Reemplazar el ícono de cerrar por `<app-mi-icon name="close"
      ariaLabel="Cerrar">` en `cash-dashboard.component.ts:164` (emoji `✕`),
      `cash-movement-modal.component.ts`, `cash-arqueo-modal.component.ts`,
      `inventory-item-form.component.ts`, `stock-adjust-modal.component.ts` y
      `purchase-form.component.ts` (cabecera + botón de eliminar fila; estos 5 usaban un `<svg>`
      artesanal, no emoji) — todos en
      `pos-heladeria/src/app/modules/{cash-register,inventory}/components/` (hace pasar T008).
- [ ] T028 [P] [US1] Reemplazar `✕` (línea ~345) y los dos íconos `<svg>` de "+" en
      `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts` por
      `<app-mi-icon name="close">` y `<app-mi-icon name="add">` (hace pasar T009).
- [ ] T029 [P] [US1] Reemplazar `👥` en
      `pos-heladeria/src/app/modules/users/pages/users-page.component.ts:84` y `✕` en
      `user-role-modal.component.ts:29` (hace pasar T010).
- [ ] T030 [P] [US1] Reemplazar `🎨🏪🧾🖨️✓👤` en
      `pos-heladeria/src/app/modules/settings/.../tenant-info.component.ts` (líneas 61, 170 y demás)
      por sus `<app-mi-icon>` equivalentes (hace pasar T011).
- [ ] T031 [P] [US1] Reemplazar `🪑📷✏️🔴🟢` en
      `pos-heladeria/src/app/modules/tables/pages/tables-page.component.ts` (líneas 55, 80 y demás),
      preservando las clases de color existentes en los indicadores de estado (FR-008; hace pasar
      T012).
- [ ] T032 [P] [US1] Reemplazar `✅` en
      `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts:282` y el ícono
      `<svg>` de círculo+cruz del botón "nueva mesa" por `<app-mi-icon name="check_circle">` y
      `<app-mi-icon name="add_circle">` (hace pasar T013).
- [ ] T033 [P] [US1] Reemplazar `🍽️📍📞🛵✓` en
      `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts` (hace pasar
      T014).
- [ ] T034 [P] [US1] Reemplazar `✏️🧾🔓` en
      `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts` (hace pasar
      T015).
- [ ] T035 [P] [US1] Reemplazar `🧾` y el ícono `<svg>` de lupa en
      `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts` por
      `<app-mi-icon name="receipt">` y `<app-mi-icon name="search">` (hace pasar T016).
- [ ] T036 [P] [US1] Reemplazar `🛒✕` en
      `pos-heladeria/src/app/modules/tables/components/cart.component.ts` (componente compartido con
      el menú público, research.md D6; hace pasar T017).
- [ ] T037 [P] [US1] Reemplazar `🏷️` en
      `pos-heladeria/src/app/modules/tables/components/product-select.component.ts` (compartido, D6;
      hace pasar T018).
- [ ] T038 [P] [US1] Reemplazar `💳` en
      `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`
      (compartido, D6; hace pasar T019).
- [ ] T039 [P] [US1] Reemplazar `🏷️` en
      `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` (compartido,
      D6; hace pasar T020).
- [ ] T040 [P] [US1] Reemplazar el ícono `<svg>` de reloj en
      `pos-heladeria/src/app/modules/cash-register/pages/cash-page.component.ts` por
      `<app-mi-icon name="schedule">` (hace pasar T021).
- [ ] T041 [P] [US1] Reemplazar el ícono `<svg>` de candado en
      `pos-heladeria/src/app/modules/cash-register/components/cash-report.component.ts` por
      `<app-mi-icon name="lock">` (hace pasar T022).
- [ ] T042 [P] [US1] Reemplazar el ícono `<svg>` de moto de reparto en
      `pos-heladeria/src/app/modules/tables/components/bill-summary.component.ts` por
      `<app-mi-icon name="delivery_dining">` (hace pasar T023).
- [ ] T043 [P] [US1] Reemplazar los 6 íconos `<svg>` artesanales en
      `pos-heladeria/src/app/modules/dashboard/layout/header.component.ts` por sus `<app-mi-icon>`
      equivalentes (`menu`, `notifications`, `expand_more`, `layers`, `settings`, `logout`) (hace
      pasar T024).
- [ ] T044 [P] [US1] Reemplazar los 3 íconos `<svg>` artesanales en
      `pos-heladeria/src/app/modules/tables/components/pos-terminal-header.component.ts` por sus
      `<app-mi-icon>` equivalentes (`group`, `credit_card`, `lock`) (hace pasar T025).
- [ ] T045 [US1] Reemplazar los 20 íconos `<svg>` artesanales catalogados en `data-model.md` en
      `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` por sus
      `<app-mi-icon>` equivalentes, incluido el reemplazo del ícono con forma de cono/copa de helado
      ("Modificar Toppings") por `tune` (hace pasar T026; FR-006 aplicado también aquí — ver
      data-model.md, nota de heladería).
- [ ] T046 [US1] Grep en `pos-heladeria/src/app/modules/dashboard/**` (rutas del panel de
      administración, research.md D5 — incluye `manual-order-page.component.ts`) buscando usos
      restantes de `<app-icon` y migrar cada uno al nuevo componente según `data-model.md`;
      documentar el resultado del grep final (0 coincidencias esperadas dentro de esas rutas).
      Depende de T027-T045 (evita volver a tocar archivos ya migrados).

**Checkpoint**: US1 completa y verificable de forma independiente en las pantallas que cubre
(quickstart.md, Escenario 1, sin panel principal).

---

## Phase 4: User Story 2 - Ningún ícono con temática de heladería (Priority: P1)

**Goal**: eliminar el cono de helado (🍦) de la marca por defecto del panel lateral y de la
tarjeta/acceso "Productos" del panel principal, reemplazándolo por íconos neutros.

**Independent Test**: revisar el panel lateral (tenant sin logo propio) y el panel principal, y
confirmar que ningún ícono hace referencia a helados (quickstart.md, Escenario 2). No depende de
US1 — toca únicamente `sidebar.component.ts` y `admin-dashboard.component.ts`, archivos que ninguna
tarea de US1 toca.

### Tests for User Story 2 ⚠️

- [ ] T047 [P] [US2] Test en `sidebar.component.spec.ts`: la marca por defecto ya no renderiza `🍦`
      ni `🛡️` como texto literal; renderiza `<app-mi-icon name="storefront">` cuando
      `isSuperAdmin()` es falso y `<app-mi-icon name="admin_panel_settings">` cuando es verdadero.
- [ ] T048 [P] [US2] Test en `admin-dashboard.component.spec.ts`: ninguna tile ni entrada de
      `quickActions` renderiza `🍦👥📋💰🍽️🧾🪑` como texto literal; la tile/acceso "Productos" usa
      específicamente `<app-mi-icon name="shopping_bag">` (SC-003).

### Implementation for User Story 2

- [ ] T049 [P] [US2] Reemplazar la expresión `{{ isSuperAdmin() ? '🛡️' : '🍦' }}` en
      `pos-heladeria/src/app/modules/dashboard/layout/sidebar.component.ts:41` por
      `<app-mi-icon name="storefront">` / `<app-mi-icon name="admin_panel_settings">` según
      corresponda (hace pasar T047; FR-006).
- [ ] T050 [P] [US2] Reemplazar `🍦` (líneas 50 y 179) y el resto de emoji de tiles/`quickActions`
      (`👥📋💰🍽️🧾🪑`) en
      `pos-heladeria/src/app/modules/dashboard/pages/admin-dashboard.component.ts` por sus
      `<app-mi-icon>` equivalentes (`shopping_bag`, `group`, `receipt_long`, `payments`,
      `restaurant`, `receipt`, `table_restaurant`) (hace pasar T048; FR-006).

**Checkpoint**: US2 completa — cero íconos de heladería en el panel de administración, verificable
de forma independiente (quickstart.md, Escenario 2).

---

## Phase 5: User Story 3 - Componente de ícono reutilizable sin referencias del artesanal en el panel (Priority: P2)

**Goal**: confirmar que el panel de administración ya no tiene ninguna referencia activa al
componente SVG artesanal (`app-icon`) ni a ningún ícono `<svg>` artesanal suelto, que dicho
componente permanece intacto sirviendo solo al flujo público, y dejar constancia de que el nuevo
componente queda disponible para reutilizarse en cualquier otra parte de la aplicación (FR-009). El
componente en sí ya se construyó en la fase Foundational (T004-T007) porque US1 y US2 lo
necesitaban para poder ejecutarse — esta historia verifica el resultado y cierra el criterio de
aceptación 2 de la Historia 3 de spec.md.

**Independent Test**: revisar el código del panel de administración y confirmar (por búsqueda) que
`app-icon` ya no tiene referencias activas ahí, que no queda ningún `<svg>` artesanal suelto, y que
el archivo de `app-icon` sigue existiendo sin cambios para el flujo público (quickstart.md,
Escenario 3). Depende de que US1 y US2 ya hayan migrado sus archivos.

### Implementation for User Story 3

- [ ] T051 [US3] Grep en `pos-heladeria/src/app/modules/dashboard/**` (research.md D5) confirmando 0
      usos de `<app-icon` y 0 etiquetas `<svg` sueltas (fuera de `app-icon` mismo); documentar el
      resultado. Depende de T046, T049, T050.
- [ ] T052 [US3] Confirmar (grep/diff) que
      `pos-heladeria/src/app/shared/icon/icon.component.ts` permanece sin modificar respecto al
      inicio de esta feature, y que sus únicos consumidores restantes son
      `public-menu.component.ts` y los cuatro `checkout/*-step.component.ts` (research.md, Decisión
      D2); documentar el resultado como evidencia de que el flujo público no fue tocado (FR-010).
- [ ] T053 [P] [US3] Agregar un comentario doc breve en
      `pos-heladeria/src/app/shared/icon-mi/icon-mi.component.ts` con un ejemplo de uso
      (`<app-mi-icon name="..." ariaLabel="...">`) para que cualquier otra parte de la aplicación
      pueda reutilizarlo sin consultar `contracts/icon-component-contract.md` (FR-009).

**Checkpoint**: las tres historias completas — panel de administración migrado en su totalidad
(spec.md, SC-001 a SC-004).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T054 Ejecutar `ng test` completo en `pos-heladeria` y comparar contra la línea base de T001 —
      confirmar cero regresiones nuevas introducidas por esta feature.
- [ ] T055 [P] Ejecutar quickstart.md, Escenario 4 (auditoría de accesibilidad) sobre las siete
      pantallas del panel de administración y documentar el resultado (SC-006).
- [ ] T056 [P] Ejecutar quickstart.md, Escenario 5 (sin conexión a internet) y documentar el
      resultado (SC-007).
- [ ] T057 [P] Ejecutar quickstart.md, Escenario 6 (regresión del flujo público de menú QR) y
      documentar el resultado (FR-010).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup (T003 depende de T002) — bloquea todas las historias
  de usuario.
- **User Stories (Phase 3-5)**: todas dependen de Foundational. US1 y US2 no comparten archivos
  entre sí y pueden avanzar en paralelo. US3 depende de que US1 y US2 hayan terminado (T051 depende
  de T046/T049/T050).
- **Polish (Phase 6)**: depende de que US1, US2 y US3 estén completas.

### Dependencias entre historias

- **US1 (P1)**: puede empezar apenas termina Foundational. No depende de US2 ni de US3.
- **US2 (P1)**: puede empezar apenas termina Foundational, en paralelo con US1 — no toca ningún
  archivo que US1 toque.
- **US3 (P2)**: depende de que US1 y US2 hayan migrado sus archivos (es una verificación de
  cierre, no puede confirmar "cero referencias" antes de que existan cero referencias).

### Dentro de cada historia

- Tests se escriben y deben fallar antes de la implementación correspondiente.
- Cada archivo (o grupo trivial de archivos con el mismo patrón, como el sweep de íconos de cerrar)
  se migra en una sola tarea (test + implementación); no hay tareas que dependan de otro archivo
  dentro de la misma historia, salvo el sweep final de US1 (T046), que depende de que los archivos
  individuales ya estén migrados.

### Parallel Opportunities

- Todas las tareas de Setup marcadas [P] pueden correr en paralelo.
- T004 y T006 (Foundational) pueden correr en paralelo entre sí; T005 y T007 tienen dependencias
  explícitas señaladas arriba.
- Todos los tests de US1 (T008-T026) pueden correr en paralelo entre sí.
- Todas las implementaciones de US1 marcadas [P] (T027-T044) pueden correr en paralelo entre sí —
  cada una toca un archivo distinto (o grupo trivial en el caso de T027).
- US1 y US2 completas pueden avanzar en paralelo (equipos distintos), ya que no comparten archivos.

---

## Parallel Example: User Story 1

```bash
# Lanzar todos los tests de User Story 1 juntos:
Task: "Test en cash-dashboard/cash-movement-modal/cash-arqueo-modal/inventory-item-form/stock-adjust-modal/purchase-form"
Task: "Test en inventory-page.component.spec.ts"
Task: "Test en users-page.component.spec.ts y user-role-modal.component.spec.ts"
Task: "Test en tenant-info.component.spec.ts"
# ...(T012-T026 igual)

# Lanzar las implementaciones independientes de User Story 1 juntas:
Task: "Reemplazar el ícono de cerrar en el sweep de modales"
Task: "Reemplazar ✕/+ en inventory-page.component.ts"
Task: "Reemplazar 👥/✕ en el módulo de usuarios"
# ...(T030-T044 igual)
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (crítico — bloquea todas las historias).
3. Completar Fase 3: User Story 1.
4. **DETENER y VALIDAR**: probar User Story 1 de forma independiente (quickstart.md, Escenario 1,
   sin panel principal).
5. Desplegar/demostrar si está listo.

### Entrega incremental

1. Completar Setup + Foundational → fundación lista.
2. Agregar User Story 1 → probar de forma independiente → desplegar/demostrar (MVP).
3. Agregar User Story 2 (puede hacerse en paralelo con la 1) → probar de forma independiente →
   desplegar/demostrar.
4. Agregar User Story 3 (verificación de cierre, depende de 1 y 2) → probar de forma independiente
   → desplegar/demostrar.
5. Fase de Polish → validar accesibilidad, modo offline y no-regresión del flujo público antes de
   dar la feature por completa (Principio X de la constitución).

### Estrategia con equipo paralelo

Con dos desarrolladores:

1. El equipo completa Setup + Foundational en conjunto.
2. Una vez lista la fundación:
   - Desarrollador A: User Story 1 (20 archivos/grupos, la mayoría independientes entre sí).
   - Desarrollador B: User Story 2 (2 archivos).
3. Cualquiera de los dos toma User Story 3 apenas ambas historias P1 estén completas.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (spec.md, Principio XII de la
  constitución).
- El componente SVG artesanal (`app-icon`) NO se toca en ninguna tarea de esta lista — permanece
  intacto para el flujo público (research.md, Decisión D2).
- Verificar que los tests fallan antes de implementar.
- Hacer commit después de cada tarea o grupo lógico.
- Detenerse en cualquier checkpoint para validar una historia de forma independiente.
- Evitar: tareas vagas, conflictos de mismo archivo entre tareas paralelas, dependencias entre
  historias que rompan su independencia.
