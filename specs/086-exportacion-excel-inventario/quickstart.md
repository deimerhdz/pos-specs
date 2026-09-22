# Quickstart: Exportación de Inventario a Excel

**Feature**: `086-exportacion-excel-inventario` | **Fecha**: 2026-09-22

Guía de validación manual end-to-end contra los criterios de aceptación del spec. No
reemplaza los tests automatizados de `tasks.md`; sirve para confirmar el comportamiento
real en `pos-backend` + `pos-heladeria` corriendo juntos.

## Prerrequisitos

- Rama creada en ambos repos siguiendo el Principio XIV de la constitución:
  `feat/086-inventory-excel-export` (o el nombre que se defina en `plan.md`), partiendo de
  la rama en la que se esté trabajando en cada repo.
- `pos-backend` corriendo localmente contra una base de datos con al menos un tenant de
  prueba, con el módulo `inventario` habilitado en su plan.
- Un usuario de prueba con rol `ADMIN` en ese tenant (para los escenarios de éxito) y otro
  con rol distinto, p. ej. `CASHIER` o `MESERO` (para el escenario de rechazo por rol).
- `pos-heladeria` corriendo localmente (`ng serve`) apuntando a ese backend.
- Dependencia `openpyxl` instalada en el entorno del backend (`pip install -r
  requirements.txt` tras agregarla, ver `research.md` §1).

## Setup de datos de prueba

1. Con el usuario `ADMIN`, entrar a Inventario (`/dashboard/inventario`) y asegurarse de
   tener registrados:
   - Al menos un insumo **activo** con stock por encima de su mínimo (para verificar
     "Estado" = `OK`).
   - Al menos un insumo **activo** con stock igual o por debajo de su mínimo (para
     verificar "Estado" = `Bajo mínimo`).
   - Al menos un insumo **inactivo** (dado de baja) — idealmente con stock por debajo de su
     mínimo, para confirmar que igual se exporta como `OK` (research.md §4).
   - Al menos un insumo con nombre que incluya tilde, eñe y/o comillas, p. ej.
     `"Champiñón"` o `Arequipe (500ml)`.
   - Un insumo con costo con decimales no enteros, p. ej. `15.00` o `8.75`.
   - Anotar el número total de insumos registrados (activos + inactivos) como `N`.

## Escenario 1 — Descarga del reporte completo (User Story 1, P1)

1. En la pantalla de Inventario, con el usuario `ADMIN`, aplicar cualquier filtro o
   búsqueda (p. ej. filtrar por tipo, o buscar un nombre parcial) para dejar la tabla en
   pantalla mostrando un subconjunto de insumos.
2. Presionar el botón "Exportar Inventario".
3. **Verificar**: el navegador descarga automáticamente un archivo `.xlsx`, sin diálogos ni
   pasos de confirmación adicionales (FR-002, SC-001).
4. Abrir el archivo descargado en una hoja de cálculo.
5. **Verificar**: el archivo contiene exactamente `N + 1` filas (1 encabezado + `N` datos),
   **sin importar** el filtro/búsqueda que estaba activo en el paso 1 (FR-003, Acceptance
   Scenario 2, SC-002) — debe incluir tanto los insumos activos como los inactivos
   anotados en el setup.
6. Repetir con el inventario del tenant vacío (o un tenant de prueba sin insumos):
   **verificar** que igual se descarga un `.xlsx` válido con solo la fila de encabezados
   (FR-008, Acceptance Scenario 3).

## Escenario 2 — Consistencia de columnas y datos (User Story 2, P2)

1. Con el archivo descargado en el Escenario 1, revisar la fila 1.
2. **Verificar**: los encabezados son, en este orden exacto, `Nombre`, `Tipo`, `Unidad`,
   `Stock`, `Mínimo`, `Costo`, `Estado` (FR-004, Acceptance Scenario 1).
3. Ubicar la fila del insumo con costo `15.00` (o el valor usado en el setup).
   **Verificar**: la celda de "Costo" contiene el número `15` (o `15.00` según el formato
   de celda), no el texto `"$15.00"` ni ningún símbolo de moneda (Acceptance Scenario 2).
4. Seleccionar la columna completa "Costo" en la hoja de cálculo y revisar la barra de
   estado. **Verificar**: muestra una suma automática (confirma que son valores numéricos
   nativos, no texto — FR-006, Acceptance Scenario 4, SC-003). Repetir para "Stock" y
   "Mínimo".
5. Ubicar la fila del insumo con tilde/eñe/comillas del setup. **Verificar**: el nombre se
   ve exactamente igual que en pantalla, sin símbolos corruptos (`�`, `Ã±`, etc.) (FR-007,
   Acceptance Scenario 3, SC-004).
6. Comparar, insumo por insumo, "Tipo" (debe leer `Materia prima` o `Empacado`, nunca
   `raw_material`/`packaged`) y "Unidad" (debe leer la misma abreviatura que la tabla en
   pantalla) contra lo que muestra la tabla de Inventario en pantalla para ese mismo insumo
   (FR-005).
7. Ubicar la fila del insumo activo con stock bajo su mínimo del setup. **Verificar**:
   "Estado" = `Bajo mínimo`. Ubicar el insumo inactivo con stock bajo su mínimo.
   **Verificar**: "Estado" = `OK` (no `Bajo mínimo`), replicando la misma regla que hoy
   aplica la pantalla (research.md §4).

## Escenario 3 — Rechazo por rol no autorizado (FR-009)

1. Autenticarse con un usuario que **no** tenga rol `ADMIN` (p. ej. `CASHIER` o `MESERO`).
2. **Verificar en el frontend**: la pantalla de Inventario no es accesible en absoluto para
   este usuario (comportamiento ya existente del `roleGuard`, sin cambios) — por lo tanto
   el botón "Exportar Inventario" nunca es visible para él.
3. Llamar directamente al endpoint con el token de ese usuario (p. ej. con `curl` o el
   cliente HTTP de preferencia):
   ```bash
   curl -i -H "Authorization: Bearer <token-no-admin>" \
     https://<host-local-backend>/api/v1/inventory/items/export
   ```
4. **Verificar**: la respuesta es `403 Forbidden`, y no se recibe ningún contenido de
   archivo `.xlsx` (contracts/inventory-export-api.md).

## Escenario 4 — Manejo de error sin archivo corrupto (Edge Case)

1. Simular una falla del backend durante la generación (p. ej. detener temporalmente la
   base de datos, o forzar una excepción en el endpoint en un entorno de prueba local).
2. Presionar "Exportar Inventario" desde el frontend con un usuario `ADMIN`.
3. **Verificar**: el frontend muestra un mensaje de error visible (toast) informando que la
   exportación falló, y **no** se descarga ningún archivo (ni completo ni parcial/corrupto).

## Resultado esperado

Todos los escenarios anteriores en verde confirman SC-001 a SC-004 del spec y los criterios
de aceptación de ambas historias de usuario. Complementar con la suite automatizada descrita
en `tasks.md` (tests de backend con `python -m unittest`, tests de frontend con `ng test`)
antes de dar la funcionalidad por completa (Principio X).
