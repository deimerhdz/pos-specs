# Research: Guardado Unificado de Producto (Crear y Actualizar)

Fase 0 de `/speckit-plan`. Todos los hallazgos están verificados leyendo el código real de
`../pos-backend` y `../pos-heladeria` (siblings de `pos-specs`) el 2026-08-27, no inferidos de las
specs previas.

## Hallazgos de código (línea base antes de esta spec)

- **`ProductService.saveProduct`** (`pos-heladeria/src/app/modules/products/services/product.service.ts:439-536`)
  orquesta el guardado completo con hasta `1 + 1 + 3·N + M` peticiones HTTP secuenciales para un
  producto con `N` presentaciones vivas y `M` desactivadas removidas: `POST`/`PATCH /products`,
  `GET .../variants` (una o dos veces, según crear o editar), por cada presentación un
  `POST`/`PATCH /variants` + `PUT .../recipe` + `PUT .../option-groups`, un `DELETE /variants/{id}`
  por cada presentación quitada, y un `PATCH .../variants/reorder` final. El propio código lo
  documenta: *"Persists a whole ProductDraft, orchestrating the flat backend endpoints (there is no
  nested create) ... Runs sequentially: a failing step aborts the rest, so on create the product
  may remain partially saved (still editable afterwards)."*
- **El `ProductDraft`/`VariantDraft` que arma el formulario ya es el árbol completo** que esta spec
  necesita transportar en una sola petición: `product.interface.ts:249-292` ya agrupa `name`,
  `price`, `recipe: RecipeLineDraft[]` y `optionGroups: VariantOptionGroupDraft[]` por presentación,
  antes de que `saveProduct` lo desarme en llamadas sueltas.
- **`ProductService` (backend) ya cruza hacia `catalog.service`** para `ensure_default_variant`
  (`app/api/v1/products/service.py:19,56`) — no es un módulo nuevo el que se toca, es el mismo que
  ya provee `create_product`.
- **El patrón de "reemplazo total idempotente"** ya existe dos veces (`set_recipe`,
  `set_variant_option_groups`, `catalog/router.py:247-352`): borra todas las filas del sub-recurso y
  reinserta las del body. El guardado consolidado reutiliza el mismo patrón, solo movido a estar
  dentro de la misma transacción que el resto del árbol.
- **El patrón de "dos pasadas" para `display_order`** (`reorder_variants`,
  `catalog/service.py:110-153`: primero valores negativos, luego los definitivos) existe
  específicamente para no violar `UNIQUE(product_id, display_order)` en un estado intermedio, sin
  depender de diferir la constraint (spec 042, decisión revisada durante su propia implementación
  porque los tests corren sobre SQLite). Se reutiliza tal cual para asignar el orden de todas las
  presentaciones del árbol consolidado.
- **La reactivación de una presentación desactivada es hoy una llamada de red instantánea y
  deliberadamente separada del guardado** (`product-form.component.ts:914-921`, comentario propio:
  *"A diferencia del resto del formulario, se aplica en el momento: reactivar es la operación que
  libera el nombre, y diferirla al «Guardar» dejaría al usuario sin salida cuando el guardado es
  precisamente lo que está fallando."*). Retirar `PATCH /variants/{id}` (FR-007) obliga a resolver
  este caso de otra forma — ver Decisión 4.
- **El botón "Guardar" ya deshabilita durante el guardado** vía `service.isSubmitting()`
  (`product-form.component.ts:431-432,1114`) — FR-008 (doble clic) ya está implementado hoy; esta
  spec no necesita tocar nada ahí, solo preservarlo.
- **Backend, testing**: no hay `pytest.ini`/`pyproject.toml` con configuración de test en la raíz de
  `pos-backend`; los characterization tests son `unittest.TestCase` ejecutados vía
  `python -m unittest` (confirmado en `test_catalog_service_sku.py`, `test_product_variant_reorder.py`,
  `test_products_service.py`) — no `pytest`, a diferencia de lo que asumiría un proyecto FastAPI
  típico.
- **Sin migración pendiente**: los tres modelos involucrados (`ProductVariant`, `RecipeItem`,
  `VariantOptionGroup`) ya tienen todas las columnas que el árbol consolidado necesita
  (`app/models/product_variant.py`, `recipe_item.py`, `variant_option_group.py`) — esta spec no
  agrega ninguna tabla, columna ni constraint nueva.

## Decisiones

### Decisión 1 — Reusar `POST /products` y `PATCH`/`PUT /products/{id}`, extendidos de forma aditiva

**Decisión**: en vez de crear rutas nuevas, `ProductCreate` gana un campo opcional
`variants: list[VariantSaveIn] = []` y `ProductUpdate` gana `variants: list[VariantSaveIn] | None =
None` (`app/api/v1/products/schemas.py`). Ningún endpoint nuevo — los "dos endpoints, uno por
acción" que pide FR-003 ya existían (`POST` crea, `PATCH`/`PUT` actualiza); esta spec solo amplía lo
que aceptan y devuelven.

**Rationale**: cumple FR-001/FR-002/FR-003 sin duplicar superficie de API. Al ser `variants` opcional
(ausente en `ProductUpdate` = "no tocar variantes", igual convención que
`OptionUpdate.inventory_item_id` ya usa vía `model_fields_set`, `catalog/router.py:535`), **ningún
llamador existente que no lo envíe cambia de comportamiento** — el characterization test
`TestUpdateProductA44` (`test_products_service.py`) y cualquier otro consumidor de `POST`/`PATCH
/products` sigue funcionando exactamente igual sin modificarse (Principio III: nada que "descongelar").

**Alternativas consideradas**:
- Rutas nuevas dedicadas (p. ej. `POST /products/full`) — descartadas: duplican superficie de API
  sin aportar nada que la extensión aditiva no logre, y complican el retiro de FR-007 (dos
  generaciones de endpoint de creación en vez de una).
- Un único endpoint que infiera crear/actualizar según la presencia de `id` en el body — descartada
  explícitamente por FR-003.

### Decisión 2 — Una sola transacción, mismo patrón de dos pasadas para `display_order`

**Decisión**: toda la escritura (producto, altas/bajas/ediciones de presentaciones, receta, grupos
de opciones, orden) ocurre en la misma `Session` de SQLAlchemy con un único `db.commit()` al final.
Cualquier excepción antes de ese commit dispara `db.rollback()` sin persistir nada (FR-004). El
`display_order` de todas las presentaciones del árbol final se asigna con el mismo patrón de dos
pasadas que ya usa `reorder_variants` (negativos primero, definitivos después) para no violar
`UNIQUE(product_id, display_order)` en un estado intermedio.

**Rationale**: reutiliza un patrón ya probado en producción (spec 042) en vez de inventar uno nuevo;
una sola `Session`/commit es la forma más directa de lograr "todo o nada" sobre PostgreSQL sin
introducir un patrón de saga o compensación que esta spec no necesita (los datos viven todos en el
mismo schema de tenant).

**Alternativas consideradas**:
- Transacciones por sub-recurso con lógica de compensación manual si un paso posterior falla —
  descartada por complejidad innecesaria; SQLAlchemy ya da atomicidad ACID gratis dentro de una
  misma `Session`.
- Diferir la constraint `UNIQUE(product_id, display_order)` (`DEFERRABLE INITIALLY DEFERRED`) en vez
  del patrón de dos pasadas — descartada por la misma razón que la descartó spec 042: los
  characterization tests corren sobre SQLite, que no soporta diferir `UNIQUE`.

### Decisión 3 — Reconciliación de presentaciones por "lista completa de activas deseadas"

**Decisión**: el árbol de `variants` que llega en el body representa el conjunto completo de
presentaciones que deben quedar activas, en el orden en que aparecen en la lista. El backend
reproduce exactamente el diffing que hoy hace `saveExistingProduct` en el frontend
(`product.service.ts:492-536`), movido al servicio:
- Cada entrada **sin `id`** se crea (mismas validaciones que `create_variant` hoy: nombre no
  duplicado, SKU único o autogenerado).
- Cada entrada **con `id`** actualiza esa presentación existente (mismas validaciones que
  `update_variant` hoy).
- Cualquier presentación **activa** del producto cuyo `id` no aparece en la lista recibida se
  desactiva (`active=False`, soft-delete — `RN-CAT-10`), igual que hoy hace el bucle final de
  `DELETE /variants/{id}` en `saveExistingProduct`.
- Receta y grupos de opciones de cada presentación se reemplazan por completo (mismo patrón que
  `set_recipe`/`set_variant_option_groups` hoy).
- `display_order` de cada presentación = su posición (1-based) dentro de la lista `variants` del
  body — sustituye a `PATCH .../variants/reorder` como llamada separada.

**Rationale**: preserva exactamente la semántica de negocio ya vigente (crear/actualizar/soft-delete
de presentaciones, reemplazo total de receta y grupos, orden por posición) sin inventar ningún
mecanismo nuevo — solo cambia **dónde** se ejecuta (una función de servicio en vez de un bucle en el
cliente HTTP).

**Alternativas consideradas**:
- Exigir un campo explícito `_action: "create"|"update"|"delete"` por entrada — descartada: la sola
  presencia/ausencia de `id` ya es inequívoca, y es el mismo criterio (`v.id ? patchVariant(...) :
  postVariant(...)`) que el código actual ya usa.
- No reconciliar bajas automáticamente (exigir un `DELETE` aparte por cada presentación quitada) —
  descartada: contradice FR-002 ("presentaciones activadas o desactivadas" forma parte de lo que el
  guardado consolidado debe persistir) y reintroduce una petición HTTP adicional por baja.

### Decisión 4 — Reactivar una presentación desactivada deja de ser una llamada de red aparte

**Decisión**: en el nuevo modelo, reactivar una presentación es una entrada más en `variants` con su
`id` real (y sin `active: false`) — igual que cualquier otra actualización. La UI de "Restaurar" dentro
del formulario deja de llamar `PATCH /variants/{id}` de inmediato: en su lugar, agrega esa
presentación al `draft().variants` en memoria (con su id real, su receta y sus grupos ya
cargados, tal como hace hoy después del `PATCH`) y espera al siguiente "Guardar" para persistirla
junto con el resto — sin ninguna petición de red por sí sola.

**Rationale**: consecuencia directa de retirar `PATCH /variants/{id}` (FR-007, decisión ya tomada por
el usuario en Clarifications). El escenario que motivaba la reactivación instantánea —desatascar al
usuario cuando el 409 de nombre duplicado ocurre **durante** el guardado— se resuelve igual de bien
en el modelo nuevo: el guardado consolidado que falla por ese 409 ya identifica la presentación
desactivada en conflicto (mismo `detail.variant_id`/`active` de hoy, ver Decisión 5); el usuario
resuelve moviendo esa reactivación al draft (client-side, sin red) y reintenta "Guardar" una vez
más — sigue siendo como máximo 2 peticiones de escritura totales, no una petición aparte por cada
intento. Documentado también como cambio de comportamiento en
[`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md).

**Alternativas consideradas**:
- Mantener viva solo esa ruta puntual (`PATCH /variants/{id}`) como excepción al retiro de FR-007 —
  descartada: el usuario, al responder la clarificación de retiro, eligió "se retiran por completo"
  sin condicionar esta excepción, y el flujo alternativo (mutar el draft + reintentar guardar) no es
  una regresión de fondo — solo deja de ser una acción de red instantánea.

### Decisión 5 — Forma del error identificable (FR-004)

**Decisión**: los errores de validación de esquema (Pydantic, `422`) ya identifican la posición
exacta del campo que falló vía `loc` (p. ej. `["body", "variants", 2, "price"]`) — FastAPI los
genera automáticamente, sin cambio necesario. Los errores de regla de negocio que hoy se lanzan a
mano (`409` nombre/SKU duplicado, `422` insumo repetido en receta, grupo de opciones inactivo o
repetido) se homogenizan agregando un campo `variant_index` (posición dentro del array `variants`
del body, o `null` si el error es del producto en sí o no aplica) al `detail` que ya devuelven hoy,
**sin quitar ningún campo existente** (`error`, `variant_id`, `active` para el 409 de nombre siguen
igual).

**Rationale**: mínimo cambio sobre un formato de error que el frontend ya sabe leer
(`toNameConflict` en `product.service.ts:45-52` ya extrae `detail.variant_id`/`active`); solo se le
agrega un campo nuevo, no se rediseña. Contracts/product-save-endpoints.md documenta la forma
completa.

**Alternativas consideradas**:
- Adoptar un formato de error genérico tipo RFC 7807 (`application/problem+json`) — descartada por
  ser un cambio de superficie mayor no requerido por ningún FR de esta spec, y por romper el
  parseo que ya hace el frontend hoy.

### Decisión 6 — Sin optimización de query adicional para SC-002

**Decisión**: no se introduce paginación, batching de `INSERT` ni ningún cambio de estrategia de
acceso a datos más allá de agrupar en una transacción lo que hoy ya son N sentencias SQL
individuales (una por presentación/receta/grupo, igual que hoy).

**Rationale**: la lentitud que reportó el usuario es de **latencia de red acumulada** (N round-trips
HTTP secuenciales, cada uno con su propio overhead de conexión/autenticación), no de cómputo por
fila en la base de datos — agrupar el transporte en una sola petición ya cumple SC-002 (menos de 2s
para un producto típico) sin necesitar optimizar las consultas SQL en sí, que ya eran rápidas
individualmente.
