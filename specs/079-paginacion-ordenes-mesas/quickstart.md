# Quickstart — Validación: Paginación y filtros en Órdenes y Mesas

Guía para verificar que la funcionalidad cumple el spec de punta a punta. Los detalles de contrato están en [`contracts/orders-list-api.md`](./contracts/orders-list-api.md) y [`contracts/tables-list-api.md`](./contracts/tables-list-api.md); el mapeo de filtros, en [`data-model.md`](./data-model.md).

## Prerrequisitos

- `pos-backend` y `pos-heladeria` en local, apuntando a un tenant de pruebas con:
  - ≥ 130 órdenes (mezcla de estados `recibida`/`abierta`/`bloqueada`/`cancelada`, algunas con `Sale` emitida, algunas de cada `order_type` y **al menos una con `order_type = NULL`**).
  - ≥ 64 mesas.
- Entrada **`A-72`** ya creada en `specs/000-reconocimiento/registro-de-anomalias.md` (prerrequisito de las Historias 1 y 2 — Principio II).

## Comandos de verificación automática

```bash
# Backend — desde pos-backend/
python -m unittest discover -s app/characterization_tests -p 'test_*.py'
#   Debe pasar TODO, incluido test_orders_service.py sin modificar,
#   más los nuevos test_orders_pagination.py y test_orders_status_type_filters.py

# Frontend — desde pos-heladeria/
npm test -- --watch=false
#   Debe pasar TODO, incluido table.service.spec.ts sin cambios,
#   más orders-page.component.spec.ts (reescrito) y tables-page.component.spec.ts (nuevo)
```

---

## Historia 1 — Navegar las órdenes por páginas (P1)

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 1 | Abrir `/dashboard/orders` sin filtros | Se ven 20 filas (tamaño por defecto); "Página 1 de 7"; total 130 | FR-001, FR-002 |
| 2 | Pulsar "Siguiente" | 20 filas siguientes en fecha de creación descendente; "Página 2 de 7" | FR-002, FR-003 |
| 3 | En página 3, cambiar "Por página" a 50 | Vuelve a página 1; 50 filas | FR-001, FR-004 |
| 4 | Ir a la última página (10 resultados) | 10 filas; "Siguiente" deshabilitado | FR-002 |
| 5 | Recargar sin filtros; comparar el conjunto página a página contra el listado actual | Mismo conjunto y mismo orden que hoy, solo segmentado | FR-003, SC-008 |
| 6 | DevTools → Network al abrir la pantalla | La respuesta trae ≤ `size` órdenes; nunca el conjunto completo | FR-006, SC-002 |
| 7 | Con ≥ 5.000 órdenes en el tenant, medir el tiempo hasta la primera página | < 2 s, sin degradarse con el total | SC-001 |
| 8 | Forzar `?page=999` (o navegar y borrar órdenes en otra pestaña) | La pantalla muestra la última página con resultados, sin error | FR-005 |

## Historia 2 — Filtrar por estado y por tipo, con el tipo visible (P2)

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 1 | Filtro de tipo → "Para llevar" | Solo órdenes para llevar; total y páginas recalculados para ese subconjunto | FR-009, FR-007 |
| 2 | Añadir filtro de estado → "Abierta" | Solo órdenes para llevar **y** abiertas | FR-011, FR-012 |
| 3 | Estando en página 4, cambiar cualquier filtro | Vuelve a página 1 del nuevo resultado | FR-004 |
| 4 | Observar cualquier fila | Muestra el tipo (En mesa / Para llevar / Domicilio) sin abrir el detalle | FR-017, SC-004 |
| 5 | Mirar los filtros de estado | **No** existe el botón "Bloqueadas"; sí existe "Bloqueada" como opción del filtro de estado | FR-014, FR-015 |
| 6 | Sin filtro de estado | Las órdenes `bloqueada` siguen apareciendo | FR-015, SC-007 |
| 7 | Filtro de estado → "Pagada" | Aparecen las órdenes con venta emitida, aunque su `status` interno sea `abierta`/`bloqueada` | FR-016 |
| 8 | Filtro de tipo → "En mesa" | La orden con `order_type = NULL` **no** aparece | FR-018 |
| 9 | Filtro de tipo → "Todos" | La orden con `order_type = NULL` aparece y su fila dice "Sin especificar" | FR-018 |
| 10 | Combinación sin resultados (p. ej. "Domicilio" + "Bloqueada" si no hay ninguna) | Estado vacío "No hay órdenes con estos filtros"; controles muestran 0 resultados / 0 páginas | Edge Cases |
| 11 | Aislar "órdenes para llevar canceladas" | ≤ 3 interacciones; el total del subconjunto se ve de inmediato | SC-003 |
| 12 | Filtro de estado → "Por confirmar" | Solo órdenes `recibida` sin venta | FR-008 |

## Historia 3 — Navegar las mesas por páginas (P3)

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 1 | Abrir `/dashboard/mesas` (64 mesas) | 20 mesas por número ascendente; "Página 1 de 4" | FR-019, FR-020 |
| 2 | En página 2, cambiar el estado operativo de una mesa | La operación se aplica; se sigue viendo una página válida del listado | FR-022 |
| 3 | Crear una mesa nueva | La lista se actualiza; la mesa aparece en la página que le toca por su número | FR-022 |
| 4 | DevTools → Network al abrir | La respuesta trae ≤ `size` mesas | FR-021 |
| 5 | Editar / activar / desactivar / ver QR de una mesa | Todas las operaciones funcionan; al terminar, vista coherente (misma página cuando es posible) | FR-022 |
| 6 | Recién abierta, comparar conjunto/orden contra hoy | Idénticos salvo la segmentación | FR-020, SC-008 |
| 7 | Con 500 mesas, medir la carga | < 2 s | SC-005 |

## No-regresión (obligatoria — FR-023–FR-026, SC-006)

| # | Pasos | Resultado esperado |
|---|---|---|
| 1 | Abrir la Terminal de Mesas (`/dashboard/mesas-sesiones`) | Muestra todas las órdenes de sesiones activas y todas las mesas, exactamente como hoy |
| 2 | Abrir el Dashboard | Muestra todas las órdenes y mesas activas, como hoy |
| 3 | Abrir "Pagos por confirmar" (pestaña de la Terminal) | Sin cambios (no consume el listado paginado) |
| 4 | Abrir la hoja de impresión de QR (`/dashboard/mesas/qr`) | Lista todas las mesas activas, como hoy |
| 5 | Abrir el detalle de una orden con mesa | La etiqueta de mesa se resuelve, como hoy |
| 6 | `curl "$API/api/v1/orders"` sin parámetros | Array JSON + header `ETag`; segunda llamada con `If-None-Match` → `304` |
| 7 | `curl "$API/api/v1/orders/tables"` sin parámetros | Array JSON, orden por `number`, forma idéntica a hoy |
| 8 | Revisar diff de tests | Ningún test `"CONGELA comportamiento actual:"` modificado; ninguna migración de Alembic nueva |
