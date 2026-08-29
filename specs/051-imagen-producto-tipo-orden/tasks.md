---

description: "Task list template for feature implementation"
---

# Tasks: Imagen de producto en el catálogo y organización del tipo de orden — creación de orden manual

**Input**: Design documents from `/specs/051-imagen-producto-tipo-orden/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva; ni la tarjeta del catálogo ni el detalle de producto tienen hoy
cobertura sobre `image_url` (research.md confirma 0 tests `"CONGELA comportamiento actual:"` y que
`ProductSelectComponent` no tiene archivo de test propio), así que esta feature agrega esa
cobertura en vez de asumir el comportamiento.

**Organization**: Tres historias de usuario (spec.md: US1 P1, US2 P2, US3 P3). Todas las rutas de
archivo son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en este
repositorio de specs). No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend
(plan.md, Storage: N/A).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md).

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **490/501** (54/59 archivos); los 11 que fallan son preexistentes y no relacionados (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, `pos-checkout-panel.component.spec.ts` y `session-bill-panel.component.spec.ts` por la regresión conocida de `MoneyInputComponent`, ya documentada en spec 046/047)
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget (784.46 kB vs 500 kB) y CommonJS (`qrcode`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend ni
de store (plan.md, Storage: N/A; research.md D1/D3) — las tres historias tocan el mismo componente
(`manual-order-page.component.ts`) pero bloques de template independientes entre sí.

**Checkpoint**: Fase 1 completa — las historias de usuario pueden comenzar.

---

## Phase 3: User Story 1 - Ver la imagen del producto en la tarjeta del catálogo (Priority: P1) 🎯 MVP

**Goal**: Cada tarjeta del catálogo de la orden manual muestra la imagen del producto cuando existe,
y un estado neutro consistente cuando no existe.

**Independent Test**: Abrir la creación de orden manual y verificar, sin tocar ninguna otra parte de
la pantalla, que las tarjetas de productos con imagen la muestran y las que no tienen imagen
muestran el ícono de respaldo, ambas con nombre y precio legibles.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que renderice la grilla de catálogo con un producto cuyo `image_url` tenga valor (mock de `store.catalogProductsFiltered()` o del servicio que la alimenta) y confirme que la tarjeta correspondiente contiene un `<img>` con ese `src` (`fixture.nativeElement.querySelector`) (research.md D1) — confirmado en rojo contra el código anterior (`expected null to be truthy`) antes de T005-T007
- [X] T004 [P] [US1] Agregar en el mismo archivo un caso que renderice la grilla de catálogo con un producto cuyo `image_url` sea `null` y confirme que la tarjeta correspondiente NO contiene ningún `<img>` y sí contiene el ícono de respaldo (`app-icon[name="image-off"]`), sin dejar espacio en blanco desalineado respecto a una tarjeta con imagen (research.md D1) — confirmado en rojo contra el código anterior antes de T005-T007

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, agregar `IconComponent` (`import { IconComponent } from '../../../shared/icon/icon.component';`) al arreglo `imports` del `@Component` (línea 29) (research.md D1)
- [X] T006 [US1] En el mismo archivo, dentro de la tarjeta de catálogo (líneas 126-136), agregar un contenedor de imagen a ancho completo antes del bloque de texto: `@if (p.image_url) { <img [src]="p.image_url" [alt]="p.name" class="w-full h-full object-cover" /> } @else { <span class="w-10 h-10 text-gray-300"><app-icon name="image-off" /></span> }`, dentro de un `div` `relative w-full aspect-square bg-indigo-50 flex items-center justify-center overflow-hidden` — mismo patrón exacto que `public-menu.component.ts:359-372` (research.md D1)
- [X] T007 [US1] En el mismo archivo, reestructurar la tarjeta (línea 129, `<button>`) para que el badge de descuento (líneas 131-133) quede posicionado `absolute top-2 right-2` sobre el contenedor de imagen de T006 (no sobre toda la tarjeta), y el bloque de nombre/precio (líneas 134-135) quede en un `div` con su propio `p-3` debajo del contenedor de imagen, quitando el `p-3` que hoy envuelve toda la tarjeta (línea 129) — mismo patrón de dos bloques que `public-menu.component.ts:357-374` (research.md D2) — T003/T004 en verde tras este cambio

**Checkpoint**: En este punto, la Historia de Usuario 1 debe funcionar y poder probarse de forma
completa e independiente.

---

## Phase 4: User Story 2 - Ver la imagen del producto en el detalle al seleccionarlo (Priority: P2)

**Goal**: Confirmar, con cobertura de test, que la vista de detalle/opciones abierta desde el
catálogo de la orden manual sigue mostrando la imagen del producto (comportamiento ya existente en
`ProductSelectComponent`, sin cambios de código).

**Independent Test**: Desde el catálogo de la orden manual, seleccionar un producto con imagen y
confirmar que el modal de detalle la muestra; seleccionar uno sin imagen y confirmar que el modal se
abre sin imagen rota — sin depender de los cambios de la Historia 1 ni de la Historia 3.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir estos tests primero; deben pasar sin ningún cambio de código de esta historia (comportamiento ya existente) — si fallan, es una regresión a investigar antes de continuar**

- [X] T008 [P] [US2] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que simule click sobre la tarjeta de un producto con `image_url` (dispara `store.openConfig(p)` → `store.configuringProduct()`), confirme que se renderiza `<app-product-select>` y que dentro de él aparece un `<img>` con ese `src` (research.md D3)
- [X] T009 [P] [US2] Agregar en el mismo archivo un caso equivalente con un producto sin `image_url` y confirme que `<app-product-select>` se renderiza sin ningún `<img>` dentro y sin error de consola/excepción (research.md D3)

### Implementation for User Story 2

- [X] T010 [US2] Ejecutar T008-T009 contra el código actual (sin cambios de `product-select.component.ts`) y confirmar que pasan tal cual — si alguno falla, documentar la causa exacta antes de continuar (no se espera ningún cambio de código para esta historia; `product-select.component.ts:48-51` ya implementa el comportamiento, research.md D3) — confirmado: T008/T009 pasaron en verde desde el primer intento, sin necesidad de tocar `product-select.component.ts`

**Checkpoint**: En este punto, las Historias de Usuario 1 y 2 deben funcionar ambas de forma
independiente.

---

## Phase 5: User Story 3 - Título organizado sobre el listado de mesas (Priority: P3)

**Goal**: El listado de mesas de la creación de orden manual tiene un título propio, distinguible
del encabezado que agrupa las pestañas de tipo de orden.

**Independent Test**: Abrir la creación de orden manual con "En Mesa" seleccionado y verificar que
aparece un título inmediatamente encima del listado de mesas, visualmente distinto del encabezado
"Tipo de Orden".

### Tests for User Story 3 ⚠️

> **NOTE: Escribir este test primero y confirmar que falla antes de implementar**

- [X] T011 [P] [US3] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que confirme que existe un encabezado con texto "Mesas" (o el texto elegido en T012) inmediatamente antes del contenedor del listado de mesas en el DOM, y que ese encabezado es un elemento distinto del `<h2>` "Tipo de Orden" ya existente (research.md D4) — confirmado en rojo contra el código anterior (`expected undefined to be truthy`) antes de T012

### Implementation for User Story 3

- [X] T012 [US3] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, agregar un `<h3 class="text-xs font-semibold uppercase tracking-wide text-gray-400">Mesas</h3>` inmediatamente antes del `<div class="flex gap-2 overflow-x-auto pb-1">` del listado de mesas (línea 74), dentro del mismo `div.space-y-3` (línea 34), sin mover la posición relativa de las pestañas de tipo de orden (líneas 46-69) (research.md D4, FR-004/FR-005) — T011 en verde tras este cambio

**Checkpoint**: Las tres historias de usuario deben ser funcionales de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T013 [P] Ejecutar manualmente los 3 escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo (Principio X) — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA visual completa (los 3 escenarios de quickstart.md). Se verificó en su lugar: `ng build` sin errores y `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, más la cobertura automatizada de T003-T004/T008-T009/T011 que ejercita exactamente el mismo comportamiento a nivel de componente (imagen con/sin `image_url` en tarjeta y detalle, título "Mesas"). **Queda pendiente que el usuario/QA ejecute el recorrido visual real** (incluyendo confirmar que el badge de descuento no queda tapado por la imagen) antes de dar la spec por verificada en producción (Principio X)
- [X] T014 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos de T003-T004, T008-T009 y T011; confirmar que los 8 casos existentes de `manual-order-page.component.spec.ts` siguen en verde sin cambios de intención (research.md, "Resumen de impacto en tests existentes") — **495/506 tests pasan** (54/59 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, regresión de `MoneyInputComponent`); cero regresiones nuevas; los 8 casos originales de `manual-order-page.component.spec.ts` siguen en verde sin cambios de intención

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **User Story 2 (Phase 4)**: Puede empezar tras la Fase 1; independiente de la Historia 1 (no depende de T005-T007), aunque comparte el mismo archivo — recomendable secuenciarla después de la Historia 1 para evitar conflictos de edición simultánea sobre `manual-order-page.component.ts`.
- **User Story 3 (Phase 5)**: Puede empezar tras la Fase 1; independiente de las Historias 1 y 2 en el DOM que modifica (bloque de tipo de orden/mesas vs. bloque de catálogo), aunque comparte el mismo archivo — mismo criterio de secuenciación que la Historia 2.
- **Polish (Phase 6)**: Depende de que las Fases 3-5 estén completas.

### Within Each User Story

- Los tests de cada historia se escriben y deben fallar (US1, US3) o pasar sin cambios (US2) antes
  de su implementación.
- T006 depende de T005 (usa `<app-icon>`, que T005 agrega a `imports`).
- T007 depende de T006 (reestructura el mismo bloque que T006 crea).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests US1) en paralelo entre sí — mismo archivo pero casos independientes.
- T008/T009 (tests US2) en paralelo entre sí.
- Dentro de un mismo archivo de producción (`manual-order-page.component.ts`), las tareas de
  implementación de las tres historias (T005-T007, T010, T012) tocan bloques de template distintos
  y no dependen entre sí en lógica, pero al ser el mismo archivo se recomienda ejecutarlas de forma
  secuencial (US1 → US2 → US3) para evitar conflictos de merge, no por dependencia funcional real.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los dos tests de la Historia 1 (mismo archivo, casos independientes):
Task: "Agregar test de tarjeta con imagen (image_url con valor)"
Task: "Agregar test de tarjeta sin imagen (image_url null, ícono de respaldo)"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T007).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (imagen en tarjeta del
   catálogo, con y sin `image_url`).
5. Desplegar/demostrar si está listo.

### Incremental Delivery

1. Completar Setup + Foundational → base lista.
2. Agregar Historia 1 (imagen en tarjeta) → probar de forma independiente → Desplegar/Demo (MVP).
3. Agregar Historia 2 (verificación de imagen en detalle) → probar de forma independiente →
   Desplegar/Demo.
4. Agregar Historia 3 (título sobre listado de mesas) → probar de forma independiente →
   Desplegar/Demo.
5. Completar Fase 6: Polish (T013-T014).

---

## Notes

- [P] = archivos distintos o casos de test independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend` (plan.md, Storage: N/A).
- Verificar que los tests de US1 y US3 fallan antes de implementar; los de US2 deben pasar sin
  cambios (comportamiento ya existente que se está cubriendo con test, no construyendo).
- Commit tras cada tarea o grupo lógico.
