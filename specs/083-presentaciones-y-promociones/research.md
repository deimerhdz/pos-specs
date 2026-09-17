# Research: Catálogo de Presentaciones y Rediseño de Promociones

Decisiones técnicas para implementar `spec.md`. Verificado contra el código real de
`../pos-backend` (rama `develop`, head Alembic `ef0abdf40889`) y `../pos-heladeria` (rama
`develop`) el 2026-09-16. Cada decisión cita el archivo/patrón existente que sigue.

## D1 — `Presentation` como tabla nueva desde cero, no una reactivación

**Decisión**: crear `app/models/presentation.py` (`class Presentation`) y
`app/models/category_presentation.py` (`class CategoryPresentation`, tabla puente) como
entidades completamente nuevas.

**Por qué**: la entidad `Presentation` de spec 040 fue eliminada físicamente por la migración
`ba4b6bd573a6` (`063b`) — `op.drop_table("presentations", ...)`. No queda ninguna tabla, modelo
ni endpoint vivo que reactivar; solo el `downgrade()` de esa migración conserva, como
documentación, el shape original (`id`, `name` único, `active`, timestamps) — que coincide con lo
que esta spec necesita de cualquier forma, así que no aporta atajo real. El directorio
`app/api/v1/presentations/` existe vacío en disco (solo `__pycache__` huérfano de los `.py`
borrados en `063b`): se reutiliza como ubicación, no como código.

**Alternativas consideradas**: revertir la migración `063b` con un `alembic downgrade` puntual —
descartada porque `063b` es una migración compartida que también retiró `promotion_targets`,
`promotion_combo_items`, `promotion_presentation_rules` y `promotions.priority`; revertirla
resucitaría estructura que otras 5 migraciones posteriores (`063c`, `063d`, y las de specs
066/079/080) ya asumen ausente.

## D2 — Patrón de modelo: espejo de `OptionGroup`/`VariantOptionGroup`

**Decisión**: `Presentation` sigue el molde exacto de `app/models/option_group.py` (catálogo
plano por tenant: `name` único, `active`, sin relación jerárquica propia). La asociación
Categoría↔Presentación sigue el molde de `app/models/variant_option_group.py` (tabla puente
simple: dos FK + `UniqueConstraint`, sin columnas de negocio propias — a diferencia de
`VariantOptionGroup`, que sí lleva `min_select`/`max_select`/`quantity_per_option`, esta puente no
necesita ninguna columna adicional porque FR-004 solo pide "pertenece o no pertenece").

**Por qué**: es el par de entidades más reciente y más parecido en forma (catálogo reutilizable +
tabla puente hacia otra entidad del dominio) ya construido y probado en el propio repositorio;
seguirlo evita inventar un segundo patrón para el mismo problema (Principio V).

## D3 — Sin gating de plan (`moduleKey`) para "Presentaciones"

**Decisión**: la sección "Presentaciones" y sus endpoints no pasan por
`require_module_access`/`ensure_module_access` (`app/core/plan_limits.py`), igual que
Categorías y Productos.

**Por qué**: `plan_limits.py` solo gatea tres módulos pagos: `inventario`, `compras`,
`promociones` (`labels` en `ensure_module_access`, línea 136). Presentaciones es catálogo base
para estandarizar nombres de variante, con el mismo rol que Categorías/Productos — ningún
Acceptance Scenario de `spec.md` la trata como módulo condicionado al plan. Promociones, que sí
está gateado hoy, sigue estándolo sin cambios (esta spec no toca `plan_limits.py`).

**Alternativa considerada**: gatear "Presentaciones" igual que "Promociones" ya que ambas viven en
el mismo flujo de negocio — descartada porque el catálogo de presentaciones es prerrequisito de la
gestión normal de categorías/productos (US2), no una funcionalidad premium independiente.

## D4 — Sin endpoint de borrado físico para `Presentation` (FR-002 se cumple por diseño)

**Decisión**: `Presentation` expone `GET`/`GET {id}`/`POST`/`PATCH {id}` únicamente — sin `DELETE`
que borre filas. El toggle activa/inactiva (en ambos sentidos, US1 Escenario 5) se hace con
`PATCH {id}` cambiando `active`, exactamente como ya reactiva una `Category` hoy (`PATCH
/categories/{id}` con `active: true`, no hay endpoint de "reactivar" separado).

**Por qué**: ningún catálogo plano del dominio (`Category`, `OptionGroup`) expone hoy un borrado
físico condicionado — `DELETE /categories/{id}` (`categories/router.py:150-159`) siempre hace
soft-delete (`category.active = False`), sin importar si la categoría tiene productos. Replicar
ese mismo criterio para `Presentation` cumple FR-002 ("impedir la eliminación física... permitir
desactivarla") de la forma más simple posible: el borrado físico simplemente nunca se ofrece como
operación, en vez de ofrecerse y luego bloquearse condicionalmente. No hace falta lógica de
"¿está referenciada?" en ningún endpoint.

**Alternativa considerada**: exponer `DELETE /presentations/{id}` con lógica condicional (soft-delete
si está referenciada, hard-delete si no) — descartada por Principio V: ningún Acceptance Scenario
pide borrado físico bajo ninguna condición, y sería la única entidad del dominio con ese
comportamiento dual, inconsistente con `Category`/`OptionGroup`.

## D5 — Asociación Categoría↔Presentación: reemplazo total embebido en `Category`, sin sub-router

**Decisión**: `CategoryCreate`/`CategoryUpdate` ganan `presentation_ids: list[UUID] | None`
(ausente/`None` = no tocar la asociación en `Update`, lista vacía = desasociar todas);
`CategoryResponse` gana `presentations: list[PresentationSummary]` (id + name, sin `active` porque
solo se listan las ya asociadas, que pueden incluir una desactivada después — FR-002). Cada
`POST`/`PATCH` con `presentation_ids` reemplaza el conjunto completo, mismo criterio que
`_replace_option_groups`/`_replace_recipe` en `app/api/v1/catalog/service.py`.

**Por qué**: no existe hoy sub-recurso `/categories/{id}/algo` en el dominio; la asociación N a N
más parecida (`VariantOptionGroup`) tampoco tiene endpoint propio — vive embebida en el payload de
`VariantSaveIn.option_groups` dentro de `POST/PATCH /products`. Embeber `presentation_ids` en el
propio payload de categoría evita una segunda ida y vuelta al servidor (crear la categoría, luego
asociar presentaciones) y es coherente con "reemplazo total" ya usado en el resto del catálogo.

**Alternativa considerada**: `PUT /categories/{id}/presentations` dedicado — descartada porque
introduce un patrón nuevo (sub-recurso de reemplazo) que no tiene precedente en el repo para
relaciones de catálogo, sin ninguna ventaja funcional sobre embeber el campo en el propio payload
de categoría.

**Nota de implementación**: dado que `categories/router.py` no tiene `service.py` propio hoy (CRUD
plano en el router), la función de reemplazo (`_replace_category_presentations`, análoga a
`_replace_option_groups`) puede vivir en un `app/api/v1/categories/service.py` nuevo o inline en el
router — decisión de detalle para `/speckit-tasks`, no bloquea el diseño.

## D6 — Herencia al crear producto: la rama `else` de `create_product` gana un paso previo

**Decisión**: en `ProductService.create_product` (`app/api/v1/products/service.py:57-92`), cuando
`data.variants` viene vacío (branch que hoy llama `ensure_default_variant`), el servicio primero
resuelve las presentaciones activas asociadas a `product.category_id` (`CategoryPresentation` join
`Presentation` filtrando `active=true`, ordenadas por un criterio estable — nombre o
`display_order` si se agrega uno a la asociación):

- Si hay ≥ 1 presentación asociada: crear una `ProductVariant` por cada una (`name` = nombre de la
  presentación, `price=0`, `active=True`), reutilizando `_save_variant_entry`/
  `_assign_display_orders` (`catalog/service.py:219-312`) con una entrada `VariantSaveIn` sintética
  por presentación — mismo camino de código que ya usa la creación con variantes explícitas
  (FR-005/FR-005, "cada una lista para que se le asigne su precio").
- Si no hay ninguna: seguir llamando `ensure_default_variant(db, product)` sin cambios de firma
  (FR-006).

**Por qué**: reutiliza el guardado consolidado ya existente (spec 043) en vez de duplicar la
lógica de creación de variantes; el único código nuevo es la resolución de qué presentaciones
aplican y el armado de la lista sintética de `VariantSaveIn`.

**Alternativa considerada**: crear las `ProductVariant` directamente con `db.add()` sin pasar por
`_save_variant_entry` — descartada porque duplicaría la asignación de `display_order`, generación
de `sku` y validación de nombre único por producto que esa función ya centraliza.

## D7 — Renombrar el default "Single" → "Presentación única" (cambio de comportamiento a registrar)

**Decisión**: `ensure_default_variant` (`catalog/service.py:84-102`) cambia el literal `name="Single"`
por `name="Presentación única"`; el `server_default="Single"` de `ProductVariant.name`
(`product_variant.py:25`) se actualiza al mismo texto por consistencia (documentación de columna,
no ruta de escritura real — el ORM siempre pasa `name` explícito).

**Por qué es un cambio de comportamiento, no solo una decisión técnica**: cambia un valor
observable que ya existe en producción (el nombre de variante que ve el administrador, que
aparece en recibos, en el Menú QR y en el selector de reglas de promoción) para todo producto
nuevo creado sin variantes explícitas ni presentaciones de categoría asociadas. La sesión de
Clarifications de este spec (2026-09-16, pregunta 2) ya es la decisión de negocio que lo autoriza
("es un renombre en la interfaz... no cambia el comportamiento de creación ni el precio inicial"),
pero Principio II exige además una entrada en `registro-de-anomalias.md` con quién/cuándo antes de
implementar — el último número usado ahí es **A-73** (spec 080), así que esta spec registra
**A-74** citando esta clarificación como decisión.

**Alcance del cambio**: solo el literal usado por `ensure_default_variant` para productos NUEVOS.
Ningún `ProductVariant.name = 'Single'` ya persistido en productos existentes se renombra
retroactivamente (fuera de alcance de esta spec, `spec.md` §Assumptions).

## D8 — Siembra inicial (FR-019): migración aditiva con paso de datos por tenant

**Decisión**: una migración Alembic nueva, `down_revision = "ef0abdf40889"` (head actual),
decorada con `@for_each_tenant_schema` (`app/scripts/tenant.py`), que:
1. Crea `tenant.presentations` y `tenant.category_presentations`.
2. Por cada schema (incluido `tenant_default`), ejecuta
   `INSERT INTO {schema}.presentations (id, name, active, created_at, updated_at) SELECT
   gen_random_uuid(), name, true, now(), now() FROM (SELECT DISTINCT name FROM
   {schema}.product_variants) AS v` — comparación exacta de texto (FR-019), sin normalizar
   mayúsculas/espacios (mismo criterio literal que ya usa `UniqueConstraint(product_id, name)` en
   `product_variants`).
3. No inserta ninguna fila en `category_presentations` (FR-019 explícito: la siembra no crea
   asociaciones).

Sigue el molde de paso de datos ya usado en `94144eaa60b5_categories_display_order.py` y
`387ef3e638cd` (`063a`): función pura `_seed_sql(schema) -> str` o equivalente, testeable sin
Postgres real contra los characterization tests con SQLite (`schema_translate_map={"tenant":
None}`).

**Por qué una sola migración aditiva, sin destructiva**: a diferencia de spec 063 (que retiraba
estructura vieja), esta spec no borra nada existente — es 100% aditiva, así que no aplica el
patrón aditivo+destructivo de dos migraciones.

**Downgrade**: `DROP TABLE` de ambas tablas nuevas (el `downgrade` no intenta revertir
selectivamente solo las filas sembradas — mismo criterio que `063a`/`94144eaa60b5`: revertir
estructura, no intentar reconstruir el estado de datos previo a un paso de datos ya aplicado).

## D9 — FR-013 (etiqueta de presentación en el selector de reglas de promoción): resuelto 100% en frontend

**Decisión**: ningún archivo de `app/api/v1/promotions/` cambia. El cálculo de qué etiqueta
mostrar por variante (nombre de la presentación si coincide con una activa asociada a la
categoría del producto, o "Presentación única"/nombre propio en caso contrario) se hace en el
frontend, en el mismo lugar donde hoy se arma el aplanado `CatalogVariant` de
`promotions-page.component.ts` — que ya combina `CategoryService` + datos de producto/variante.
Con `CategoryResponse.presentations` (D5) disponible por categoría, el frontend arma un mapa
`category_id -> Set<nombre_presentación_activa>` y decide la etiqueta variante por variante, sin
excluir ninguna (FR-013).

**Por qué**: `PromotionRuleIn.variant_ids` (`promotions/schemas.py:104-116`) ya acepta cualquier
`ProductVariant.id` sin restricción de forma — el backend de promociones nunca necesitó saber qué
es una "presentación" para funcionar, y FR-009 exige explícitamente no alterar su motor de
cálculo. Resolver la etiqueta en el cliente evita tocar `PromotionResponse`,
`serialize_promotion` o cualquier endpoint de promociones, y aísla 100% del cambio de esta spec
al catálogo de presentaciones + categorías.

**Alternativa considerada**: que el backend enriquezca cada `PromotionVariantResponse` con una
`presentation_label` calculada — descartada porque obligaría a `promotions/service.py` a conocer
`Presentation`/`CategoryPresentation` (acoplamiento nuevo entre dos dominios que hoy son
independientes) solo para resolver un texto de UI que el frontend ya puede calcular con datos que
de todas formas necesita cargar para otras partes de la misma pantalla (el buscador de productos
por categoría, FR-012).

## D10 — Rediseño de las 3 pantallas de promociones: mismo componente, sin `ReactiveFormsModule`

**Decisión**: el rediseño visual de listado/creación/configuración vive en
`src/app/modules/promotions/pages/promotions-page.component.ts` (o se divide en 2-3 componentes de
página con rutas propias si la fidelidad a los 3 prototipos HTML distintos lo justifica —decisión
de detalle para tasks), manteniendo el patrón `FormsModule`/`ngModel` ya usado (900+ líneas,
template-driven) en vez de migrar a `ReactiveFormsModule`. `PromotionService`, sus interfaces y
`promotion-pricing.util.ts`/`promotion-condition.util.ts` no cambian de contrato — solo ganan lo
necesario para leer `presentations`/`category.presentations` (D9).

**Por qué**: FR-009 exige fidelidad visual a los prototipos, no un cambio de arquitectura de
formulario; spec 063 ya evaluó y descartó `ReactiveFormsModule`/`FormArray` para este mismo
componente (su `research.md`, D-R4) con el mismo argumento (`*ngFor` + `ngModel` indexado ya
resuelve listas repetibles sin el cambio de patrón) — no hay ningún requisito nuevo en esta spec
que reabra esa decisión.

**Alternativa considerada**: partir en 3 rutas (`/promotions`, `/promotions/new`,
`/promotions/:id/configure`) en vez de 3 pantallas internas del mismo componente — viable y más
cercana a "página" por prototipo; se deja como decisión de implementación porque no cambia ningún
FR ni contrato de datos, solo la organización de archivos del frontend.
