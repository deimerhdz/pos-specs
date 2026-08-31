# Quickstart: validar el ocultamiento de Unidades de Medida y Reportes de Inventario

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite lo ya
detallado en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas. Reutiliza
íntegramente los fixtures de plan de spec 033 (`app/characterization_tests/plan_fixtures.py`) —
esta spec no crea fixtures propios.

**Prerequisitos**: mismo entorno que spec 033 — `pos-backend` con el venv activado, `pos-heladeria`
con `npm install` ya corrido. No hace falta PostgreSQL real para el backend (SQLite en memoria vía
fixtures existentes).

```bash
cd ../pos-backend
source env/bin/activate
```

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. Ningún `"CONGELA comportamiento actual:"` asume acceso sin
restricción a `unit-measures`, `/reports/inventory` ni `/reports/profitability` (research.md,
Decisión 5) — ninguno se toca.

## US1 — Unidades de medida desaparece de Ajustes sin el módulo Inventario

Backend — no requiere un fichero de test nuevo (research.md Decisión 5): el comportamiento de
`require_module_access("inventario")` ya está cubierto genéricamente por
`test_plan_module_access.py` (spec 033). Verificación del wiring, por inspección:

1. `app/api/v1/unit_measures/router.py` — confirmar que `APIRouter(...)` incluye
   `dependencies=[Depends(require_module_access("inventario"))]` (mismo patrón que
   `inventory/router.py:28-31`).
2. Con un tenant `inventario_access=false` (fixture `fx.make_plan(db, inventario_access=False)`),
   invocar manualmente `require_module_access("inventario")(tenant=tenant, db=db, _user=user)` →
   `HTTPException(403, "Tu plan actual no incluye el módulo de inventario.")` — ya lo cubre
   `test_plan_module_access.py::test_modulo_no_incluido_deniega_con_mensaje_claro` de forma
   genérica (mismo mecanismo, cualquier router que lo declare).

Frontend — specs nuevos:

3. `settings-page.component.spec.ts` (nuevo): con `PlanSummaryService.summary()` indicando
   `modules.inventario = false`, `visibleTabs()` no incluye la pestaña "Unidades de medida"
   (Acceptance Scenario 1, FR-001). Con `modules.inventario = true`, sí la incluye (Acceptance
   Scenario 3). Con `summary() === null` (todavía no cargó), sí la incluye — fail-open, mismo
   criterio que `sidebar.component.spec.ts` (research.md Decisión 3).
4. `dashboard/routes.ts` (ajuste a su spec de configuración de rutas, si existe, o assertion
   directa sobre el arreglo de rutas): el hijo `ajustes/unidades` incluye
   `canActivate: [planModuleGuard('inventario')]` (Acceptance Scenario 2, FR-002, redirige a
   `/dashboard` — Clarifications de `spec.md`).

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## US2 — Los reportes de insumos de inventario desaparecen sin el módulo Inventario

Backend — mismo criterio de verificación por inspección que US1, sobre
`reports/router.py::inventory` (RF-063): confirmar
`dependencies=[Depends(require_module_access("inventario"))]` en esa ruta específica, y que
`sales`/`top-products`/`products`/`categories`/`cashiers` no lo ganan (Acceptance Scenario 3,
FR-006, [contracts/inventario-module-gating.md](./contracts/inventario-module-gating.md)).

Frontend — ajustes a `reports.service.spec.ts` (existente) y `reports-page.component.spec.ts`
(existente):

1. Con `PlanSummaryService.summary().modules.inventario = false`: `http.expectNone()` confirma
   que **no** se dispara ninguna petición a `/reports/inventory` (Acceptance Scenario 1, FR-006).
2. Con lo anterior, `svc.isLoading()` eventualmente llega a `false` y `svc.error()` es `null` —
   confirma que excluir la query del agregado evita el spinner infinito descrito en research.md,
   Decisión 4 (regresión que esta spec existe para prevenir, no solo el caso feliz).
3. `reports-page.component.spec.ts`: con el mismo plan, la sección "Insumos con stock bajo" no
   aparece en el DOM renderizado (Acceptance Scenario 1). Con `modules.inventario = true`, sí
   aparece y su petición sí se dispara (Acceptance Scenario 2).

## US3 — El indicador de Margen refleja que depende de datos de Inventario

Mismo mecanismo que US2, misma señal `inventarioIncluido()`:

1. `reports.service.spec.ts`: con `modules.inventario = false`, `http.expectNone()` confirma que
   no se dispara ninguna petición a `/reports/profitability` (Acceptance Scenario 1, FR-007,
   [contracts/inventario-module-gating.md](./contracts/inventario-module-gating.md)).
2. `reports-page.component.spec.ts`: con el mismo plan, la tarjeta "Margen" (`app-stat-tile
   label="Margen"`) no aparece en el DOM — no solo su valor en blanco, la tarjeta entera
   (Acceptance Scenario 1, FR-007). Con `modules.inventario = true`, aparece y calcula con
   normalidad (Acceptance Scenario 2).
3. Backend, por inspección: `reports/router.py::profitability` (RF-065) gana la misma
   dependencia que `::inventory` — confirmar que no se introdujo ningún valor especial (ej.
   `margin: null`) para el caso sin módulo, consistente con la decisión de ocultar en el
   frontend en vez de degradar la respuesta (contracts, sección de Historia 3).

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## Reasignación del módulo (Edge Case común a las tres historias)

1. Con un tenant ya bloqueado (`inventario_access=false` o vencido), el Super Admin le reasigna
   el módulo (o renueva su plan) vía `/super-admin/tenants` (spec 033, ya implementado).
2. Sin que el Tenant Admin cierre sesión: recargar Ajustes → la pestaña "Unidades de medida"
   reaparece; recargar Reportes → la sección de insumos con stock bajo y la tarjeta de Margen
   reaparecen y calculan con normalidad (FR-008, SC-005). Como esta spec no cachea
   `PlanSummaryService.summary()` de forma distinta a como ya lo hace `SidebarComponent` (misma
   sesión de trabajo), el criterio de frescura es el mismo que ya aceptó esa pieza: se refleja en
   la siguiente carga del layout del dashboard, no en tiempo real dentro de la misma pestaña
   abierta — no es una regresión de esta spec, es el comportamiento ya existente que hereda.

## Verificación manual end-to-end (no automatizable sin navegador real)

1. Como Super Admin, asignar a un tenant de prueba un plan sin `inventario_access` (spec 033, ya
   implementado). Como ese Tenant Admin: confirmar que Ajustes no muestra "Unidades de medida" y
   que entrar por URL directa a `/dashboard/ajustes/unidades` redirige a `/dashboard` (SC-001/
   SC-002).
2. Mismo tenant: abrir Reportes → confirmar que no aparecen ni la sección "Insumos con stock
   bajo" ni la tarjeta "Margen", y que el resto de la pantalla (Total cobrado, Cobros, Ticket
   promedio, gráficas de ventas/productos/categorías/cajeros) carga con normalidad, sin spinner
   permanente ni banner de error (SC-003/SC-004).
3. Como Super Admin, reasignarle el módulo Inventario a ese tenant → como Tenant Admin, recargar
   el dashboard → confirmar que las tres superficies reaparecen y funcionan con normalidad, sin
   cerrar sesión (SC-005).
