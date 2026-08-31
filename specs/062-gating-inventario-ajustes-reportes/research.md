# Research: Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado

**Feature**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — esta spec reutiliza
integralmente el mecanismo de gating por plan de la spec 033 (`require_module_access`,
`PlanSummaryService`, `planModuleGuard`), sin agregar dependencias, entidades ni endpoints
nuevos. Esta fase documenta las decisiones técnicas necesarias para conectar ese mecanismo a
tres superficies que hoy no lo usan, y una alternativa descartada por cada una.

## Decisión 1 — Reutilizar `require_module_access("inventario")` tal cual, sin variante nueva

**Decisión**: Las tres superficies (Unidades de Medida, insumos con stock bajo, Margen) usan
exactamente la misma clave de módulo `"inventario"` que ya gobierna la pantalla de Inventario
(spec 033) — no se introduce una clave nueva (ej. `"unidades_medida"` o `"reportes_inventario"`).

**Justificación**: código verificado — `unit_measure` (`app/models/unit_measure.py`) solo lo
referencia `InventoryItem` (`app/models/inventory_item.py`); no existe ningún otro consumidor en
`app/models/*.py` ni en `pos-heladeria/src/app/modules/{products,option-groups}` (grep dirigido,
0 resultados fuera de inventario). El costo (`cogs`) de `profitability_report`
(`app/api/v1/reports/service.py:170-213`, `_variant_unit_cost_map`) sale exclusivamente de
`RecipeItem.quantity * InventoryItem.unit_cost` — ninguna otra fuente de costo existe hoy. Las
tres superficies son, sin excepción, datos o configuración de Inventario vistos desde otra
pantalla — no un módulo nuevo que además necesite su propio toggle en el catálogo de planes.

**Alternativas consideradas**: una clave de módulo nueva y más granular (ej. separar "reportes de
inventario" de "gestión de inventario") — descartada porque el spec (Assumptions) ya establece
que las tres superficies son inseparables de Inventario, y una clave nueva exigiría una migración
de `shared.plans` y una decisión de negocio sobre qué planes existentes la incluyen, ninguna de
las cuales pidió la spec.

## Decisión 2 — Router-level para Unidades de Medida, route-level para los dos endpoints de Reportes

**Decisión**: `unit_measures/router.py` gana la dependencia a nivel de `APIRouter(...)` (cubre
sus 5 endpoints de una sola vez); `reports/router.py` la gana solo en las rutas `GET /inventory`
y `GET /profitability` (decorador por ruta), sin tocar `sales`/`top-products`/`products`/
`categories`/`cashiers`.

**Justificación**: mismo patrón exacto ya usado por spec 033 para el mismo tipo de situación —
`inventory/router.py:28-31` aplica `dependencies=[Depends(require_module_access("inventario"))]`
a nivel de `APIRouter` porque **todos** sus endpoints son de inventario; las 4 rutas de
`/purchases*` dentro de ese mismo router agregan además `require_module_access("compras")` **por
ruta**, porque solo esas 4 (de un router más grande) necesitan esa segunda restricción
(`inventory/router.py:186,201,214,227`). `unit_measures/router.py` está en la misma situación que
`inventory/router.py` (100% de sus rutas son de inventario); `reports/router.py` está en la
situación de las rutas `/purchases*` (una minoría de rutas dentro de un router más grande que
sirve otros propósitos).

**Alternativas consideradas**: mover `/reports/inventory` y `/reports/profitability` a un router
de inventario separado — descartada por ser una reorganización de código no pedida por la spec
(Principio V: nueva funcionalidad antes que refactorización oportunista); el decorador por ruta
logra el mismo resultado sin mover nada.

## Decisión 3 — Dónde vive la señal de gating en el frontend: por superficie, no una sola global

**Decisión**: cada superficie lee `PlanSummaryService.summary()` con su propia señal local, en
el sitio donde esa señal se necesita para algo más que solo "mostrar u ocultar":
- `SettingsPageComponent` filtra su arreglo de pestañas con una señal local (mismo patrón que
  `SidebarComponent.visibleItems`, modificado en esta misma sesión de trabajo) — es puramente
  cosmético, la ruta ya está protegida por el guard (Decisión 2, más FR-002).
- `ReportsService` (no `ReportsPageComponent`) expone `inventarioIncluido()`, porque esa señal
  también decide el `enabled:` de dos `injectQuery` que **viven en el servicio** — si viviera en
  el componente, el servicio no podría leerla para gatear sus propias queries.

**Justificación**: `inventory-page.component.ts` (spec 033) ya estableció el precedente de
`comprasIncluido()` como señal local de componente para ocultar una pestaña — pero ahí no hay
ninguna query que gatear (el tab de Compras dispara sus llamadas solo al abrirse, no de forma
eager). Reportes es distinto: sus seis queries se disparan todas al entrar a la pantalla, así
que la señal debe vivir donde vive la definición de la query, no donde vive el template.

**Alternativas consideradas**: una señal global compartida en `PlanSummaryService` (ej.
`inventarioIncluido` como propiedad de ese servicio, reutilizable desde cualquier componente) —
descartada por ahora: sería una abstracción nueva para dos consumidores (research.md no encontró
un tercero), y el spec no pide unificar el criterio de gating de UI en un solo sitio. Si una
cuarta superficie necesitara este mismo cálculo, sería el momento de promoverlo — no antes
(Principio V).

## Decisión 4 — Gating fail-closed de las queries de Reportes, y su exclusión del agregado de estado

**Decisión**: `inventoryQuery` y `profitabilityQuery` ganan `enabled: inventarioIncluido()`,
donde `inventarioIncluido()` es `false` tanto si el resumen de plan indica que el módulo no está
incluido (o el tenant está vencido) **como si el resumen todavía no cargó** (`summary() === null`)
— lo opuesto al criterio fail-open ya usado en `SidebarComponent`/`SettingsPageComponent`. Además,
el arreglo `queries` que alimenta `isLoading`/`error` deja de ser una lista fija de 6 queries y
pasa a ser `computed()`: incluye `inventoryQuery`/`profitabilityQuery` solo cuando
`inventarioIncluido()` es `true`.

**Justificación — por qué fail-closed aquí y fail-open en el resto**: en Sidebar/Ajustes, un
`false` incorrecto durante la carga es cosmético y de corta duración — a lo sumo el ítem aparece
un instante antes de que el resumen cargue, y el guard/backend son la barrera real. En Reportes,
un `true` incorrecto durante la carga dispara una petición real a un endpoint que, para un tenant
sin Inventario, devuelve 403 de inmediato — y `reports.service.ts:234-237` calcula `error()` como
"la primera query en `isError()`", sin distinguir cuál: ese 403 se mostraría como
`"No se pudieron cargar los informes."` sobre **toda** la pantalla, no solo sobre Margen o
insumos. Fail-closed evita ese disparo.

**Justificación — por qué excluir del agregado, no solo deshabilitar**: una `injectQuery`
deshabilitada (`enabled: false`) queda en `isPending() === true` indefinidamente mientras nunca
se habilite — no es "cargada con éxito", es "nunca se intentó". Si `inventoryQuery`/
`profitabilityQuery` siguieran en el arreglo `queries` de `isLoading` para un tenant sin
Inventario, `isLoading` sería `true` para siempre (nunca se cumple `!q.isPending()` para esas
dos), y el spinner de Reportes no desaparecería jamás para ese tenant. Excluirlas del agregado
cuando `inventarioIncluido()` es `false` es indispensable, no una optimización.

**Alternativas consideradas**:
- Dejar las queries habilitadas y solo ocultar el resultado en el template — descartada: es
  exactamente el bug que esta decisión previene (403 filtrándose al banner de error global).
- Deshabilitar las queries pero dejarlas en el agregado de `isLoading`/`error` — descartada:
  produce el spinner infinito descrito arriba.
- Un `try/catch` silencioso en el `queryFn` de esas dos queries (devolver un valor por defecto en
  vez de dejar que la petición falle) — descartada: seguiría disparando una petición HTTP real
  contra un endpoint al que el tenant no debería ni siquiera intentar llamar (contradice FR-006/
  "no se puede ni siquiera consultar"), y enmascararía errores reales (ej. caídas del backend)
  bajo el mismo criterio que un 403 esperado.

## Decisión 5 — Verificación del wiring backend: por inspección, mismo criterio que spec 033

**Decisión**: no se agrega un test de integración HTTP (`TestClient`) que confirme que
`unit_measures/router.py` y las dos rutas de `reports/router.py` efectivamente devuelven 403 sin
el módulo — se verifica por inspección del código (la dependencia declarada en el router/ruta),
igual que ya se aceptó para `inventory/router.py`/`promotions/router.py` en spec 033.

**Justificación**: búsqueda dirigida (`grep -rl "TestClient" app/characterization_tests | xargs
grep -l "require_module_access"`) no encontró ningún test existente que ejercite el wiring real
de `require_module_access` sobre un router vía HTTP — el único test relacionado
(`test_plan_module_access.py`) invoca la dependencia directamente como función, "para no depender
de la resolución de `Depends()` de FastAPI" (comentario propio del archivo). Ese test ya cubre de
forma genérica el comportamiento de `require_module_access("inventario")` para cualquier router
que lo declare; agregar un test de integración nuevo solo para esta spec sería un estándar más
alto que el que el propio código de spec 033 estableció para el mismo mecanismo, sin que el spec
de esta feature lo pida.

**Alternativas consideradas**: agregar tests `TestClient` nuevos para las 7 rutas afectadas —
descartada por inconsistencia de criterio (elevaría el estándar de verificación de esta spec por
encima del de la spec de la que depende, sin justificación en `spec.md`); queda como mejora
independiente si se decide en el futuro (no en el alcance de esta spec, Principio V).

## Decisión 6 — Impacto en tenants reales al día de hoy

**Hallazgo**: la migración `5eeb92818839_seed_transitional_plan.py` (spec 033) sembró un único
plan transicional ("Ilimitado (transición)") con `inventario_access = true` y sin vencimiento, al
que se backfillearon todos los tenants existentes en el momento del despliegue de esa spec. Por
lo tanto, ningún tenant que siga en ese plan transicional (o en cualquier plan que el Super Admin
haya creado después con Inventario incluido) verá cambio de comportamiento alguno con esta spec —
el ocultamiento solo se activa para un tenant al que el Super Admin le asigne, de forma explícita
y posterior, un plan sin `inventario_access` o lo deje vencer.

**Relevancia**: confirma Principio II (Constitution Check) sin necesitar una entrada en
`registro-de-anomalias.md` — no hay ningún tenant hoy que dependa de ver Unidades de Medida,
insumos con stock bajo o Margen sin tener derecho a Inventario, porque esa combinación no existía
antes de que el Super Admin empezara a usar planes reales (spec 033 ya es quien la hizo posible).
