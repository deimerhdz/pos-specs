---

description: "Task list template for feature implementation"
---

# Tasks: Mejora de Visibilidad del Buscador y Filtros en Inventario

**Input**: Design documents from `/specs/085-mejora-visibilidad-filtros-inventario/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md (N/A), quickstart.md

> **Regeneración**: esta versión de `tasks.md` reemplaza por completo la anterior
> (T001-T013, ya marcada como completa). Esa versión anterior implementó una iteración
> *previa* del diseño (tarjeta de filtros separada, acento `indigo` genérico) que la sesión
> de clarificación del 2026-09-22 reemplazó explícitamente (ver spec.md, Clarifications; y
> "Estado actual del código" en research.md). El código en `pos-heladeria` (rama
> `feat/085-improve-inventory-filters-visibility`) todavía tiene esa iteración anterior —
> todas las tareas de abajo parten de cero e implementan el diseño vigente (encabezado
> fusionado en la tarjeta de la tabla, acento exacto `#4a3aff` del mockup).

**Tests**: No se generan tareas de test. El spec y el plan (research.md, punto 6) determinan
explícitamente que este es un cambio exclusivamente visual sin comportamiento nuevo que
probar; la suite existente (`inventory-page.component.spec.ts`) no ejercita el bloque de
filtros ni la tarjeta de la tabla y debe permanecer en verde sin modificaciones.

**Organization**: Tareas agrupadas por historia de usuario para permitir implementación y
validación independiente de cada una.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos o verificaciones sin dependencia
  entre sí)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2)
- Se incluye la ruta exacta de archivo en cada descripción

## Path Conventions

Proyecto web con un único repo afectado: `pos-heladeria/` (frontend Angular). `pos-backend`
queda fuera de alcance por completo (ver plan.md, Structure Decision). Archivo único
modificado en todo este feature:

- `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`

Referencia visual exacta para todas las tareas de implementación: subárbol
`data-purpose="table-card-header"` de
[design/inventory-search-filters-redesign.html](./design/inventory-search-filters-redesign.html)
(líneas 263-304), con las dos exclusiones de FR-009 (sin badge "⌘K", sin contador de
insumos).

---

## Phase 1: Setup

**Purpose**: Confirmar el punto de partida (iteración anterior aún en el código) antes de
reemplazarlo por el diseño fusionado vigente

- [X] T001 Verificar checkout de la rama `feat/085-improve-inventory-filters-visibility` en
  `pos-heladeria` y que las dependencias estén instaladas (`npm install` si hace falta)
- [X] T002 Ejecutar `npm test` (`ng test`) en `pos-heladeria` como línea base de esta
  iteración: confirmar que `inventory-page.component.spec.ts` pasa en verde antes de
  modificar el bloque de filtros/tabla, para poder atribuir cualquier falla posterior
  exclusivamente a este cambio
- [X] T003 Ejecutar `npm start` (`ng serve`) en `pos-heladeria`, navegar a **Inventario →
  pestaña "Insumos"** y confirmar visualmente el estado actual (iteración anterior): la zona
  de filtros (`inventory-page.component.ts` líneas 111-131, `bg-indigo-50/40 rounded-xl
  border border-indigo-100 p-4 flex flex-wrap gap-3`) es hoy una tarjeta **separada**,
  hermana de la tabla (líneas 134-199) — este es el punto de partida que las tareas
  siguientes reemplazan (research.md, "Estado actual del código")

---

## Phase 2: Foundational

**N/A para este feature.** No hay infraestructura, modelo de datos, autenticación ni
prerrequisitos bloqueantes compartidos entre historias más allá del Setup: es un cambio de
presentación acotado a clases Tailwind, estructura del DOM y un ícono decorativo dentro de
un único bloque de template ya existente (`data-model.md` confirma que no hay entidades ni
servicios nuevos). Las historias de usuario pueden comenzar directamente después de la
Fase 1.

---

## Phase 3: User Story 1 - Encontrar la zona de búsqueda y filtros de un vistazo (Priority: P1) 🎯 MVP

**Goal**: Fusionar la zona de búsqueda/filtros como encabezado superior de la misma tarjeta
que contiene la tabla de resultados (separada de las filas por un borde), para que se
distinga del fondo de la página y de los demás elementos sueltos (encabezado, tarjeta "Bajo
mínimo") de un solo vistazo.

**Independent Test**: Mostrar la pantalla de Inventario (pestaña Insumos) a un usuario y
pedirle que señale dónde buscaría un insumo por nombre; exitoso si lo ubica de inmediato,
sin necesidad de ningún otro cambio de la spec (spec.md, Historia 1).

### Implementation for User Story 1

- [X] T004 [US1] Eliminar el `<div>` de filtros como hermano separado de la tabla (`bg-indigo-50/40
  rounded-xl border border-indigo-100 p-4 flex flex-wrap gap-3`, líneas 111-131 de
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`) e insertar su
  contenido (buscador + 2 selects) como primer hijo del `<div>` de la tabla (línea 134),
  envuelto en un nuevo contenedor `<div class="flex items-center justify-between gap-4 p-4
  border-b border-gray-200 bg-gray-50/50 flex-wrap">` (replica `data-purpose="table-card-header"`
  del mockup, con `flex-wrap` añadido para FR-006), sin modificar ningún `[ngModel]`,
  `(ngModelChange)` ni `<option>` de los tres controles (FR-001; research.md punto 1)
- [X] T005 [US1] Actualizar las clases del `<div>` contenedor de la tabla (línea 134 antes de
  T004) de `bg-white rounded-xl border border-gray-100 overflow-hidden` a `bg-white
  rounded-xl shadow-sm border border-slate-200/80 overflow-hidden` (mockup), en
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`, para que la
  tarjeta fusionada (encabezado + tabla) se distinga con más peso visual del fondo de la
  página y de sus vecinos (encabezado, tarjeta "Bajo mínimo") (FR-001; research.md punto 4)
  — depende de T004
- [X] T006 [US1] Validación manual siguiendo quickstart.md sección "Historia 1" (pasos 1-3):
  confirmar que la zona de búsqueda y filtros, ya fusionada como encabezado de la tarjeta de
  tabla, se distingue a simple vista del encabezado, de la tarjeta "Bajo mínimo" y de las
  filas de la tabla, incluyendo cuando "Bajo mínimo" está activo como filtro (Acceptance
  Scenarios 1-2; Edge case de "Bajo mínimo" activo) — depende de T004, T005
  > Validado en el navegador (tab logueado en `heladeria.localhost:4200/dashboard/inventario`):
  > la zona fusionada se distingue del encabezado y de la tarjeta "Bajo mínimo" tanto en
  > estado normal como con "Bajo mínimo" activo como filtro (sin competir visualmente).

**Checkpoint**: En este punto, la Historia de Usuario 1 debe ser completamente funcional y
verificable de forma independiente (la zona fusionada ya se distingue del resto de la
pantalla).

---

## Phase 4: User Story 2 - Distinguir el buscador de los filtros dentro de la misma zona (Priority: P2)

**Goal**: Dentro de la zona ya fusionada y visible (Historia 1), agrupar el buscador a la
izquierda y los filtros a la derecha, y reforzar cada control individualmente (ícono, acento
`#4a3aff`, flecha de select) para que se perciban como controles interactivos distintos entre
sí.

**Independent Test**: Pedirle a un usuario que, dentro de la zona ya ubicada, señale
específicamente el campo de búsqueda por nombre y cada uno de los filtros; exitoso si
identifica correctamente cada control sin dudar (spec.md, Historia 2).

### Implementation for User Story 2

- [X] T007 [US2] Envolver el buscador en `<div class="relative flex-1 max-w-md">` (agrupación
  a la izquierda del encabezado fusionado de T004) y actualizar las clases del `<input>` a
  `w-full pl-10 pr-3 py-2 bg-white text-slate-800 placeholder-slate-400 text-sm rounded-lg
  border border-gray-300 focus:outline-none focus:ring-2 focus:ring-[#4a3aff]/30
  focus:border-[#4a3aff] shadow-sm transition-all` (mockup línea 270, con `pr-3` en vez de
  `pr-12` porque no se agrega el badge "⌘K" — FR-009), conservando `app-mi-icon name="search"`
  ya posicionado de forma absoluta y sin modificar `[ngModel]="searchSignal()"`,
  `(ngModelChange)="onSearchInput($event)"`, `type="text"` ni el `placeholder="Buscar por
  nombre..."`, en `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`
  (FR-002; research.md punto 3) — depende de T004
- [X] T008 [US2] Agrupar los dos `<select>` a la derecha del encabezado fusionado dentro de
  `<div class="flex items-center gap-3 flex-wrap">`, en
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts` (FR-003;
  research.md punto 4) — depende de T004
- [X] T009 [US2] Envolver el `<select>` de tipo de insumo en `<div class="relative">`,
  actualizar sus clases a `appearance-none border border-gray-300 bg-white hover:bg-slate-50
  text-slate-700 font-medium pl-3 pr-8 py-2 rounded-lg text-sm cursor-pointer
  focus:outline-none focus:ring-2 focus:ring-[#4a3aff]/30 focus:border-[#4a3aff] shadow-sm
  transition-colors` (mockup línea 277), y agregar el ícono de flecha decorativo dentro de
  `<div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2.5
  text-slate-400">` con un SVG `w-4 h-4` (`path d="M19 9l-7 7-7-7"`, mockup líneas 282-286),
  sin modificar `[ngModel]="service.itemsType()"`, `(ngModelChange)="onTypeFilterChange($event)"`
  ni las `<option>`, en
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts` (FR-003;
  research.md punto 4) — depende de T008
  > **Desviación necesaria**: se usó `<app-mi-icon name="expand_more" [size]="16">` en vez del
  > SVG artesanal indicado. `inventory-page.component.spec.ts` (spec 082) tiene una
  > aserción `expect(el.querySelector('svg')).toBeNull()` sobre **todo** el componente (no
  > solo el botón "Nuevo insumo"); el `<svg>` de la flecha literal, aunque replica el mockup,
  > rompía ese test existente. `expand_more` ya está en `ICON_CATALOG` (línea 79) con la
  > semántica correcta (chevron abajo) — se preserva el comportamiento protegido (Principio
  > II) sin perder fidelidad visual al mockup.
- [X] T010 [US2] Repetir el mismo tratamiento de T009 (wrapper `relative`, clases del select,
  flecha SVG) para el `<select>` de estado, sin modificar `[ngModel]="service.itemsActive()"`,
  `(ngModelChange)="onActiveFilterChange($event)"` ni las `<option>`, en
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts` (FR-003;
  research.md punto 4) — depende de T008
  > Misma desviación que T009: `<app-mi-icon name="expand_more" [size]="16">` en vez de SVG.
- [X] T011 [US2] Validación manual siguiendo quickstart.md sección "Historia 2" (pasos 1-3):
  confirmar que el campo de búsqueda se percibe como lugar para escribir texto de búsqueda, y
  que cada selector ("Tipo de insumo", "Estado") se percibe como control de filtro
  independiente, separado del buscador y entre sí (Acceptance Scenarios 1-2) — depende de
  T007, T009, T010
  > Validado en el navegador: buscador con ícono de lupa y acento `#4a3aff` al enfocar,
  > cada select con flecha `expand_more` y hover propio — se perciben como tres controles
  > independientes.

**Checkpoint**: En este punto, las Historias 1 y 2 deben funcionar juntas: la zona fusionada
se ubica de un vistazo y, dentro de ella, cada control se distingue con claridad.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verificaciones transversales de accesibilidad, alcance (FR-009) y no-regresión
que aplican al cambio completo (Historias 1 y 2 combinadas)

- [X] T012 [P] Confirmar que la zona reforzada **no** incluye el badge "⌘K" dentro del campo
  de búsqueda ni un contador tipo "N Insumos" junto a los filtros —ambos presentes en el
  mockup pero excluidos explícitamente por FR-009 (Clarification Q2 de spec.md)—, per
  quickstart.md sección 4
  > Confirmado por inspección del template (T007) y captura del navegador: ningún badge ni
  > contador presente.
- [X] T013 [P] Validar el edge case de ventana angosta (~375px, modo responsive de devtools):
  confirmar que el buscador y los selects se acomodan en varias líneas (`flex-wrap`) dentro
  del encabezado fusionado y que la zona sigue siendo igual de reconocible en ese acomodo,
  per quickstart.md sección 4 y FR-006
  > Validado a 375×812: buscador y ambos selects se acomodan en filas separadas dentro de la
  > misma tarjeta con borde, sigue siendo reconocible como unidad.
- [X] T014 [P] Confirmar que la tarjeta "Bajo mínimo" (líneas 102-109) y las filas, columnas
  y datos de la tabla conservan exactamente su apariencia y jerarquía visual actuales
  (comparar contra una captura de la rama `main` si hay dudas), per quickstart.md sección 4 y
  FR-008
  > Confirmado por captura: "Bajo mínimo" (ámbar) y filas/columnas de la tabla sin cambios.
- [X] T015 [P] Verificar contraste WCAG 2.1 nivel AA del nuevo acento `#4a3aff` (mínimo 3:1
  para bordes y anillos de foco) y del texto de la zona (mínimo 4.5:1: placeholder, texto
  ingresado, texto de las opciones de los selects), usando el inspector de contraste de
  devtools (Chrome/Edge) o axe, per quickstart.md sección 5 y FR-004/SC-005 (research.md
  punto 5 estima ≈6.3:1 para `#4a3aff` sobre blanco — verificar contra la implementación
  real)
  > Calculado con la fórmula de luminancia relativa WCAG sobre los colores reales
  > renderizados: `#4a3aff` sobre blanco ≈ **6.29:1** (> 3:1 requerido para bordes/anillos de
  > foco — confirma la estimación de research.md). `text-slate-800` (texto ingresado) ≈
  > **14.63:1** y `text-slate-700` (texto de opciones) ≈ **10.35:1** sobre blanco (ambos > 4.5:1).
  > **Hallazgo**: `placeholder-slate-400` (`#94a3b8`) sobre blanco ≈ **2.56:1**, por debajo de
  > 4.5:1 — viene directo del mockup exacto (línea 270, Clarification Q3) y WCAG 1.4.3 no
  > exige el mismo mínimo a placeholders (texto que desaparece al escribir, no es la única
  > etiqueta del campo — el ícono de lupa + `aria`/contexto cumplen ese rol), por lo que no se
  > modificó; queda documentado para que el usuario decida si lo acepta o pide ajustarlo.
- [X] T016 [P] Verificar cero regresión funcional: escribir un término en el buscador,
  cambiar el selector de "Tipo de insumo", cambiar el selector de "Estado", combinar ambos
  filtros con "Bajo mínimo" activo, y confirmar que la paginación (`app-pagination-bar`)
  sigue funcionando exactamente igual que antes del cambio, per quickstart.md sección 6 y
  FR-005/FR-007/SC-004
  > Validado en el navegador: búsqueda por nombre ("banano") filtra correctamente, filtro de
  > tipo ("Empacado") filtra correctamente, "Bajo mínimo" activo/inactivo funciona, y
  > `app-pagination-bar` ("Por página") sigue presente y funcional — sin cambios de
  > comportamiento. Estado del filtro restaurado al finalizar la prueba.
- [X] T017 [P] Re-ejecutar `npm test` (`ng test`) en `pos-heladeria` y confirmar que la
  suite existente (incluida `inventory-page.component.spec.ts`) sigue en verde sin haber
  sido modificada, comparando contra la línea base de T002 (Principio X; research.md
  punto 6)
  > `inventory-page.component.spec.ts` en verde (2/2) antes y después del cambio. Suite
  > completa: mismo conjunto de fallas preexistentes y no relacionadas (auth.service,
  > tenant.service, app.spec, menu.service, pos-checkout/order-panel — módulo `tables` y
  > servicios con dependencias HTTP, fuera de alcance de esta spec) tanto en la línea base
  > (T002) como después del cambio; ningún archivo del módulo `inventory` aparece entre las
  > fallas. Nota: T009/T010 requirieron el ajuste de ícono documentado ahí — con ese ajuste,
  > `inventory-page.component.spec.ts` pasa sin modificaciones.
- [X] T018 Mostrar el cambio ya implementado al usuario que reportó el problema original y
  registrar su confirmación explícita de que ahora distingue con claridad y de un vistazo
  dónde está la zona de búsqueda y filtros (SC-003), per quickstart.md sección 7 — criterio
  de éxito principal del spec; requiere confirmación humana, no puede registrarse de forma
  automática
  > Confirmado por el usuario (2026-09-22): "todo quedo bien", tras revisar el resultado
  > implementado. Criterio de éxito principal del spec (SC-003) satisfecho.
- [X] T019 [P] Validar el edge case de baja visión/daltonismo per quickstart.md sección 4
  ("Sin depender solo del color"): con la emulación de daltonismo del panel "Rendering" de
  Chrome DevTools (o una captura en escala de grises), confirmar que la zona de búsqueda y
  filtros se sigue distinguiendo del resto de la pantalla apoyándose en borde, espaciado e
  ícono de lupa, sin depender únicamente del acento `#4a3aff` (FR-004; Edge case de baja
  visión de spec.md)
  > Validado aplicando `filter: grayscale(1)` a la página completa: la zona sigue
  > distinguiéndose del resto por el borde `border-b`, el fondo `bg-gray-50/50` y el ícono de
  > lupa, sin depender del tono `#4a3aff`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Phase 2)**: N/A — no bloquea nada adicional; las historias parten
  directamente de Setup
- **User Story 1 (Phase 3)**: depende de Setup (T001-T003)
- **User Story 2 (Phase 4)**: depende de T004 (US1), porque envuelve/agrupa elementos dentro
  del mismo encabezado fusionado que T004 crea; en la práctica conviene completar el
  checkpoint de US1 antes de empezar US2 para evitar conflictos de edición en el mismo bloque
  de template, aunque conceptualmente ambas historias son verificables por separado
- **Polish (Phase 5)**: depende de que Historias 1 y 2 estén implementadas (T004-T010)

### User Story Dependencies

- **User Story 1 (P1)**: independiente a nivel de requisito (FR-001); en la práctica se
  implementa primero por ser la base estructural (fusión del encabezado) sobre la que se
  apoya US2
- **User Story 2 (P2)**: independiente a nivel de requisito (FR-002/FR-003), pero su
  implementación (T007-T010) edita elementos dentro del mismo encabezado que T004 crea, por
  lo que debe aplicarse después de T004 para evitar conflictos de merge en el mismo archivo

### Within Each User Story

- Implementación antes que validación manual
- Dentro de US2: T007 (buscador) y T008 (agrupación de selects) pueden hacerse en cualquier
  orden entre sí, pero T009 y T010 (cada select individual) dependen de T008 (el wrapper que
  los agrupa)
- US1 completa (checkpoint) antes de comenzar la implementación de US2, por compartir el
  bloque de encabezado fusionado

### Parallel Opportunities

- T012, T013, T014, T015, T016, T017, T019 (Phase 5) no dependen entre sí y pueden
  ejecutarse en paralelo por distintas personas una vez completadas US1 y US2
- No hay tareas `[P]` dentro de Setup, US1 ni US2: todas editan el mismo bloque de template
  (`inventory-page.component.ts`) o dependen del resultado de la tarea anterior

---

## Parallel Example: Phase 5 (Polish)

```bash
# Una vez completas US1 (T004-T006) y US2 (T007-T011), lanzar en paralelo:
Task: "Confirmar que NO se agregó el badge '⌘K' ni el contador de insumos (FR-009)"
Task: "Validar edge case de ventana angosta (~375px) per quickstart.md sección 4"
Task: "Confirmar que 'Bajo mínimo' y la tabla no cambiaron de apariencia (FR-008)"
Task: "Verificar contraste WCAG 2.1 AA del acento #4a3aff y del texto (FR-004/SC-005)"
Task: "Verificar cero regresión funcional de búsqueda/filtros (FR-005/FR-007/SC-004)"
Task: "Re-ejecutar `npm test` en pos-heladeria y comparar contra la línea base"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Fase 1: Setup
2. Fase 2: Foundational — N/A, sin tareas
3. Completar Fase 3: User Story 1 (T004-T006)
4. **DETENER y VALIDAR**: confirmar que la zona fusionada ya se distingue de un vistazo
   (checkpoint de US1) — esto por sí solo resuelve la queja original de mayor impacto
5. Mostrar/demo si está listo

### Incremental Delivery

1. Setup → línea base lista
2. Agregar US1 (T004-T006) → validar de forma independiente → demo (MVP)
3. Agregar US2 (T007-T011) → validar de forma independiente → demo
4. Fase 5 (Polish): alcance FR-009, edge cases, contraste AA, cero regresión, suite en
   verde, confirmación del usuario (T012-T019) → cierre del spec

### Notes

- `[P]` = verificaciones sin dependencia entre sí, ejecutables en paralelo (solo en Phase 5)
- `[Story]` mapea cada tarea de implementación a su historia de usuario
- No se agregan tests automatizados nuevos: es un cambio exclusivamente visual sin
  comportamiento nuevo que probar (ver sección "Tests" arriba)
- Un único archivo modificado en todo el feature:
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`
- `pos-backend` no se toca en ninguna tarea
