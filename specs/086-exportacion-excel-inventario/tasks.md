---

description: "Task list template for feature implementation"
---

# Tasks: Exportación de Inventario a Excel

**Input**: Design documents from `/specs/086-exportacion-excel-inventario/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/inventory-export-api.md, quickstart.md

**Tests**: Se generan tareas de test. El plan (Testing, Constitution Check Principio X) exige
tests nuevos del endpoint (backend, `unittest`) y del componente/servicio de exportación
(frontend, `ng test`), más la validación manual `quickstart.md`; no hay tests
`"CONGELA comportamiento actual:"` que congelar porque el endpoint y el botón son
funcionalidad nueva (Principio III no aplica).

**Organization**: Tareas agrupadas por historia de usuario para permitir implementación y
validación independiente de cada una.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencia entre sí)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2)
- Se incluye la ruta exacta de archivo en cada descripción

## Path Conventions

Aplicación web con cambios coordinados en dos repos hermanos (plan.md, Structure Decision):

- `pos-backend/app/api/v1/inventory/` — endpoint, service, helper de generación del `.xlsx`
- `pos-backend/app/characterization_tests/` — tests del endpoint nuevo
- `pos-heladeria/src/app/modules/inventory/services/` — método de descarga del blob
- `pos-heladeria/src/app/modules/inventory/pages/` — botón "Exportar Inventario" y su handler

---

## Phase 1: Setup

**Purpose**: Preparar ambos repos antes de tocar código de inventario

- [X] T001 Crear la rama `feat/086-inventory-excel-export` en `pos-backend`, partiendo de la
  rama actual de ese repo (Principio XIV de la constitución)
- [X] T002 Crear la rama `feat/086-inventory-excel-export` en `pos-heladeria`, partiendo de
  la rama actual de ese repo (Principio XIV de la constitución)
- [X] T003 [P] Agregar `openpyxl` a `pos-backend/requirements.txt` e instalarla
  (`pip install -r requirements.txt`) — única dependencia nueva de esta feature,
  justificada en `research.md` §1
- [X] T004 [P] Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  en `pos-backend` como línea base, confirmando que la suite existente de inventario
  (`test_inventory_stock.py`, `test_core_inventory_reasons.py`, `test_inventory_timezone.py`)
  pasa en verde antes de modificar `router.py`/`service.py`
- [X] T005 [P] Ejecutar `ng test` en `pos-heladeria` como línea base, confirmando que
  `inventory-page.component.spec.ts` pasa en verde antes de modificar
  `inventory-page.component.ts` e `inventory.service.ts`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Construir en el backend el artefacto compartido que ambas historias necesitan
— la consulta sin paginar y el helper que arma el workbook completo en memoria — antes de
exponerlo por el endpoint (Historia 1) y de que sus columnas/formatos se validen a fondo
(Historia 2)

**⚠️ CRITICAL**: Ninguna historia puede completarse sin este helper, porque no hay archivo
que descargar ni columnas que verificar sin él

- [X] T006 [P] Crear `list_all_items_for_export()` en
  `pos-backend/app/api/v1/inventory/service.py`: `Select` de **todos** los `InventoryItem`
  del tenant (activos e inactivos), sin paginar, `order_by(InventoryItem.name)` ascendente —
  mismo orden por defecto que `list_items_query()` (línea 34), sin aplicar ningún filtro de
  búsqueda/tipo/activo (FR-003, Clarification 1 del spec; data-model.md, "Alcance de filas")
- [X] T007 [P] Crear `pos-backend/app/api/v1/inventory/export.py` (NUEVO) con dos funciones
  de traducción puras: `_type_label(type: str) -> str` (`"Materia prima"` si
  `raw_material`, `"Empacado"` si `packaged`, replicando `typeLabel()` de
  `inventory-page.component.ts:488-490`) y `_status_label(active: bool, current_stock, min_stock) -> str`
  (`"Bajo mínimo"` si `active and current_stock <= min_stock`, si no `"OK"` — replica
  exacta de `isLow()`, `inventory-page.component.ts:491-493`, incluida la condición
  `active`, research.md §4)
- [X] T008 En `pos-backend/app/api/v1/inventory/export.py`, implementar
  `build_inventory_excel(items, unit_measures_by_id) -> io.BytesIO` con `openpyxl`
  (depende de T006, T007): fila 1 con los encabezados exactos `Nombre`, `Tipo`, `Unidad`,
  `Stock`, `Mínimo`, `Costo`, `Estado` en ese orden (FR-004); a partir de la fila 2, una fila
  por cada `InventoryItem` recibido con `Nombre` (texto), `Tipo` (vía `_type_label`),
  `Unidad` (`UnitMeasure.abbreviation` resuelta por `unit_measure_id`), `Stock`/`Mínimo`/`Costo`
  como valores numéricos nativos (`Decimal`, sin redondear ni truncar — FR-006) y `Estado`
  (vía `_status_label`); retorna el workbook completo en un `io.BytesIO` ya cerrado para
  escritura, sin persistir nada en disco (research.md §2, data-model.md)

**Checkpoint**: el helper de generación del `.xlsx` existe y produce el archivo completo en
memoria — las historias de usuario pueden implementarse a partir de aquí

---

## Phase 3: User Story 1 - Descarga del reporte completo de inventario (Priority: P1) 🎯 MVP

**Goal**: un botón "Exportar Inventario" en la pantalla de Inventario descarga
automáticamente un `.xlsx` con el total absoluto de insumos (activos e inactivos), sin
importar filtros/búsqueda/orden en pantalla, incluido el caso de inventario vacío, y
rechazado para usuarios sin rol autorizado.

**Independent Test**: entrar a Inventario, presionar "Exportar Inventario" y verificar que
el navegador descarga un `.xlsx` que abre sin errores, con `N + 1` filas para `N` insumos
registrados.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T009 [P] [US1] Crear `pos-backend/app/characterization_tests/test_inventory_export.py`
  (NUEVO) con un `TestClient` sobre `GET /api/v1/inventory/items/export`: (a) usuario
  `ADMIN` con `N` insumos (mezcla de activos e inactivos) → `200`, `Content-Type`
  `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `Content-Disposition`
  con `filename="inventario_<YYYY-MM-DD>.xlsx"`, y el workbook leído con
  `openpyxl.load_workbook(io.BytesIO(response.content))` tiene exactamente `N + 1` filas
  (FR-002, FR-003, SC-002, contracts/inventory-export-api.md); (b) inventario vacío (`N=0`)
  → `200` con workbook de exactamente 1 fila, solo el encabezado (FR-008); (c) usuario
  autenticado sin rol `ADMIN` (p. ej. `CASHIER`) → `403 Forbidden`, sin cuerpo de archivo
  (FR-009, research.md §3); (d) fallo simulado a mitad de la generación (mockear
  `list_all_items_for_export()` o `build_inventory_excel()` para lanzar una excepción) →
  `500 Internal Server Error`, sin encabezado `Content-Disposition` ni bytes de archivo
  `.xlsx` en la respuesta (Edge Case del spec, research.md §2)
- [X] T010 [P] [US1] Crear
  `pos-heladeria/src/app/modules/inventory/services/inventory.service.spec.ts` (NUEVO) con
  un test para `exportItems()`: verifica que hace `GET` a `${baseUrl}/items/export` con
  `responseType: 'blob'` y `observe: 'response'`, usando `HttpClientTestingModule` /
  `provideHttpClientTesting()` (mismo patrón de testing ya usado por el resto del proyecto
  para `HttpClient`), y que retorna la respuesta simulada sin transformarla
  (contracts/inventory-export-api.md)
- [X] T011 [P] [US1] Ampliar
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.spec.ts` con un
  test que simula clic en el botón "Exportar Inventario": en éxito, confirma que se llamó
  `service.exportItems()` y que el componente dispara la descarga (spy sobre la creación del
  blob URL / `<a download>`); en error (el observable emite error), confirma que se llama
  `toast.error(...)` y que no se crea ningún enlace de descarga (research.md §8); además,
  confirma que el nombre del archivo descargado se lee del encabezado `Content-Disposition`
  de la respuesta simulada, y que si ese encabezado no está presente, el componente arma el
  nombre de respaldo `inventario_<fecha-local>.xlsx` (research.md §7)

### Implementation for User Story 1

- [X] T012 [P] [US1] En `pos-backend/app/api/v1/inventory/router.py`, agregar
  `GET /items/export` en la sección `# ============================ Insumos ============================`
  (junto a `low_stock`/`list_items`): `Depends(require_tenant_admin)` (hereda además
  `require_module_access("inventario")` ya aplicado a nivel de router), llama a
  `service.list_all_items_for_export()` (T006) y `export.build_inventory_excel(...)` (T008),
  calcula el nombre `inventario_<date.today().isoformat()>.xlsx` (research.md §7) y retorna
  un `fastapi.Response` con `media_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"`
  y encabezado `Content-Disposition: attachment; filename="..."` (research.md §2,
  contracts/inventory-export-api.md)
- [X] T013 [P] [US1] En
  `pos-heladeria/src/app/modules/inventory/services/inventory.service.ts`, agregar
  `exportItems(): Observable<HttpResponse<Blob>>` que llama
  `this.http.get(`${this.baseUrl}/items/export`, { responseType: 'blob', observe: 'response' })`
  (contracts/inventory-export-api.md, research.md §6) — reutiliza el mismo `baseUrl` e
  interceptor de autenticación ya usados por `fetchItemsPage()`
- [X] T014 [US1] En
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts`, agregar el
  botón "Exportar Inventario" en el header de la tarjeta de la tabla de insumos (líneas
  113-140, junto al buscador y los selects de filtro `Tipo`/`Estado`, alineado a la
  derecha — FR-001) y un handler `exportInventory()` que: invoca `service.exportItems()`;
  en éxito, lee el nombre de archivo del encabezado `Content-Disposition` de la respuesta (o
  arma `inventario_<fecha-local>.xlsx` como respaldo si no llega) y dispara la descarga
  creando un `URL.createObjectURL(blob)` + `<a download>` sintético, liberando el URL con
  `URL.revokeObjectURL()` tras el clic (research.md §6, §7); en error, llama
  `this.toast.error(...)` (ya inyectado en el componente, línea 389) sin dejar ningún
  archivo descargado (edge case del spec, research.md §8)

**Checkpoint**: Historia 1 funcional de forma independiente — el botón descarga el archivo
completo, respeta el total absoluto sin importar filtros, funciona con inventario vacío y
rechaza usuarios sin rol `ADMIN`

---

## Phase 4: User Story 2 - Consistencia de columnas y datos exportados (Priority: P2)

**Goal**: el archivo exportado por la Historia 1 tiene exactamente las columnas y el orden
de `Nombre, Tipo, Unidad, Stock, Mínimo, Costo, Estado`, valores numéricos nativos que
permiten autosuma, etiquetas de `Tipo`/`Unidad`/`Estado` idénticas a las de pantalla, y
nombres con tildes/eñes/comillas sin corrupción.

**Independent Test**: abrir el archivo exportado en una hoja de cálculo, comparar
encabezados y orden contra la tabla en pantalla, y seleccionar la columna "Costo" o "Stock"
para confirmar que la barra de estado muestra una suma automática.

### Tests for User Story 2 ⚠️

- [X] T015 [US2] Ampliar
  `pos-backend/app/characterization_tests/test_inventory_export.py` con tests que abren el
  workbook de la respuesta con `openpyxl`: (a) la fila 1 es exactamente
  `["Nombre", "Tipo", "Unidad", "Stock", "Mínimo", "Costo", "Estado"]` en ese orden
  (FR-004, Acceptance Scenario 1 de US2); (b) las celdas de `Stock`, `Mínimo` y `Costo`
  tienen tipo de dato numérico (`cell.data_type == 'n'`), no texto, y conservan los mismos
  decimales que en base de datos sin truncar (FR-006, data-model.md); (c) un insumo con
  costo `15.00` registrado → la celda de `Costo` tiene el valor numérico `15.00`, no el
  string `"$15.00"` (Acceptance Scenario 2 de US2)
- [X] T016 [US2] Ampliar `test_inventory_export.py` con tests de las etiquetas: un insumo
  `type="raw_material"` exporta `"Materia prima"` y uno `type="packaged"` exporta
  `"Empacado"` en la columna `Tipo` (nunca el código interno); la columna `Unidad` exporta
  `UnitMeasure.abbreviation` (no el nombre largo de la unidad); un insumo activo con
  `current_stock <= min_stock` exporta `Estado = "Bajo mínimo"`, y el mismo insumo marcado
  `active=False` con igual nivel de stock exporta `Estado = "OK"` (no `"Bajo mínimo"`),
  replicando exactamente `isLow()` del frontend incluida la condición `active` (research.md
  §4, §5, FR-005)
- [X] T017 [US2] Ampliar `test_inventory_export.py` con un test de codificación: un insumo
  con nombre `"Champiñón"` y otro con comillas/paréntesis (p. ej. `Arequipe (500ml)`) se
  preservan carácter a carácter al leer la celda con `openpyxl`, sin símbolos corruptos
  (FR-007, Acceptance Scenario 3 de US2, SC-004)

**Checkpoint**: ambas historias de usuario son funcionales de forma independiente — el
archivo se descarga con el total correcto (US1) y sus columnas/datos son fieles y operables
matemáticamente en una hoja de cálculo (US2)

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verificación integral de no-regresión y validación manual contra los criterios
de aceptación de ambas historias (Principio X: Verificación obligatoria)

- [X] T018 [P] Ejecutar la suite completa de `pos-backend`
  (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`) y
  confirmar que no hay regresiones fuera de `test_inventory_export.py` (endpoints/tests
  existentes de inventario y del resto del sistema en verde)
- [X] T019 [P] Ejecutar `ng test` completo en `pos-heladeria` y confirmar que no hay
  regresiones fuera de `inventory.service.spec.ts` e `inventory-page.component.spec.ts`
- [X] T020 Ejecutar `quickstart.md` Escenario 1 (descarga del reporte completo) end-to-end
  con `pos-backend` + `pos-heladeria` corriendo localmente: aplicar un filtro/búsqueda antes
  de exportar y confirmar que el archivo trae el total absoluto (`N + 1` filas) igual,
  incluido el caso de tenant sin insumos (solo encabezado)
  — Verificado contra los servidores locales reales del usuario (backend `:8000` +
  `ng serve :4200`, tenant `heladeria`) vía navegador (chrome-devtools): con la tabla
  filtrada en pantalla a "Café" (1 fila visible), `GET /inventory/items/export` salió
  **sin** el parámetro `search` y devolvió el total absoluto (44 filas = 43 insumos +
  encabezado, `content-length: 6490`), igual que sin filtro. El caso de tenant vacío
  (`N=0`) no se repitió en vivo (el tenant de desarrollo ya tiene insumos) — queda
  cubierto por `test_inventario_vacio_descarga_solo_el_encabezado` (T009b).
- [X] T021 Ejecutar `quickstart.md` Escenario 2 (consistencia de columnas y datos)
  end-to-end: abrir el archivo descargado en una hoja de cálculo real (Excel/Google
  Sheets/LibreOffice) y confirmar visualmente encabezados/orden, autosuma real al
  seleccionar las columnas `Costo`/`Stock`/`Mínimo`, y nombres con tilde/eñe/comillas
  legibles sin corrupción
  — El `.xlsx` real descargado del tenant `heladeria` se abrió con LibreOffice Calc
  headless (`--convert-to pdf`, carga exitosa como documento Calc real, no solo
  `openpyxl`) y se confirmó visualmente: encabezados `Nombre, Tipo, Unidad, Stock,
  Mínimo, Costo, Estado` en orden, columnas numéricas alineadas a la derecha (Stock/
  Mínimo/Costo con `cell.data_type == 'n'`), y nombres con tilde (`Café`, `Maní`,
  `Piña`, `Maracuyá`) legibles sin corrupción. La autosuma real seleccionando la
  columna en una UI interactiva no se probó (LibreOffice corrió en modo headless, sin
  interfaz) — la condición que la habilita (celdas numéricas nativas, no texto) sí
  quedó verificada, y ya la cubre también T015 a nivel unitario.
- [X] T022 Ejecutar `quickstart.md` Escenario 3 (rechazo por rol no autorizado, FR-009):
  confirmar que el botón no es visible para un usuario sin rol `ADMIN` y que una llamada
  directa al endpoint con su token responde `403 Forbidden` sin contenido de archivo
  — Con un JWT real de un usuario `CASHIER` existente en el tenant de desarrollo: en el
  navegador, `/dashboard/inventario` redirigió de inmediato a `/dashboard/caja` (guard de
  rol existente, sin cambios de esta spec) — el botón nunca llega a renderizarse. Una
  llamada directa al endpoint (`curl` con ese mismo token) respondió `403 Forbidden`,
  `{"detail":"Tenant admin access required"}`, sin bytes de archivo.
- [ ] T023 Ejecutar `quickstart.md` Escenario 4 (manejo de error sin archivo corrupto):
  simular una falla del backend a mitad de la generación y confirmar que el frontend
  muestra un toast de error sin descargar ningún archivo completo ni parcial
  — No se reprodujo en vivo: hacerlo habría requerido romper temporalmente el código del
  backend real que el usuario tiene corriendo (con recarga en caliente activa), con
  riesgo de interrumpir su sesión de desarrollo. Cubierto en su lugar por dos tests
  automatizados aislados que juntos prueban ambas mitades del caso: T009d (backend,
  mock de fallo a mitad de generación → 500 sin `Content-Disposition` ni bytes de
  archivo) y el caso "en error" de T011 (frontend, `exportItems()` emite error → se
  llama `toast.error(...)` y no se crea ningún enlace de descarga). Pendiente si el
  usuario quiere un repro en vivo.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Phase 2)**: depende de Setup (T003, dependencia `openpyxl` instalada) —
  **bloquea** ambas historias de usuario
- **User Story 1 (Phase 3)**: depende de Foundational (T006-T008) completo
- **User Story 2 (Phase 4)**: depende de Foundational (T006-T008); en la práctica valida y
  amplía tests sobre el mismo endpoint que expone la Historia 1 (T012), por lo que conviene
  completar la Historia 1 antes, aunque sus tests (T015-T017) podrían escribirse en paralelo
- **Polish (Phase 5)**: depende de que Historias 1 y 2 estén completas

### User Story Dependencies

- **User Story 1 (P1)**: puede iniciar apenas termine Foundational — sin dependencia de
  otras historias
- **User Story 2 (P2)**: depende conceptualmente de que el endpoint de la Historia 1 (T012)
  exista para tener un archivo real que abrir y validar (spec.md: "Depende de que la
  Historia 1 exista")

### Within Each User Story

- Tests antes de implementación, y deben fallar antes de implementar
- Foundational (query + helper) antes que el endpoint que lo expone
- Servicio frontend antes que el componente que lo consume
- Historia completa antes de pasar a la siguiente prioridad

### Parallel Opportunities

- T003, T004, T005 (Setup) en paralelo
- T006 y T007 (Foundational) en paralelo — archivos distintos (`service.py` vs `export.py`)
- T009, T010, T011 (tests de Historia 1) en paralelo — tres archivos distintos, dos repos
- T012 y T013 (implementación de Historia 1) en paralelo — repos distintos; T014 depende de
  T013 (usa `exportItems()`)
- T018 y T019 (Polish) en paralelo — repos distintos

---

## Parallel Example: User Story 1

```bash
# Lanzar los tests de la Historia 1 juntos (archivos y repos distintos):
Task: "Crear pos-backend/app/characterization_tests/test_inventory_export.py con tests del endpoint"
Task: "Crear pos-heladeria/.../services/inventory.service.spec.ts con test de exportItems()"
Task: "Ampliar pos-heladeria/.../pages/inventory-page.component.spec.ts con test del botón"

# Lanzar la implementación backend/frontend en paralelo:
Task: "Agregar GET /items/export en pos-backend/app/api/v1/inventory/router.py"
Task: "Agregar exportItems() en pos-heladeria/.../services/inventory.service.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 únicamente)

1. Completar Fase 1: Setup
2. Completar Fase 2: Foundational (CRÍTICO — bloquea ambas historias)
3. Completar Fase 3: Historia 1 — botón descarga el archivo completo, total absoluto,
   caso vacío, rechazo por rol
4. **PARAR y VALIDAR**: ejecutar `quickstart.md` Escenarios 1, 3 y 4 contra la Historia 1
5. Ya es un MVP demostrable: el reporte se descarga de forma confiable

### Incremental Delivery

1. Setup + Foundational → helper de generación del `.xlsx` listo
2. Historia 1 → probar de forma independiente → MVP demostrable (descarga funcional)
3. Historia 2 → probar de forma independiente → archivo auditable (columnas/datos fieles,
   numéricos operables)
4. Polish → suites completas sin regresión + los 4 escenarios de `quickstart.md` en verde

---

## Notes

- [P] = archivos distintos, sin dependencia entre sí
- [Story] mapea la tarea a su historia de usuario para trazabilidad
- Confirmar que los tests fallan antes de implementar (T009-T011 antes de T012-T014)
- Commits solo bajo pedido explícito del usuario, en unidades pequeñas y lógicas, en inglés,
  sin marcas de autoría de IA (Principio XV)
- No se crean módulos, rutas de navegación ni entidades de datos nuevas — todo el cambio
  vive dentro del módulo `inventory`/`inventario` ya existente en ambos repos
