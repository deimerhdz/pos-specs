# Research: Orden de Presentaciones de un Producto

No quedó ningún `NEEDS CLARIFICATION` de negocio en `spec.md`. Las incógnitas de este research son
puramente técnicas: cómo modelar el orden, dónde imponerlo, y cómo no dejar huecos ni romper
FR-005/FR-007/FR-008 al crear, eliminar o reactivar una presentación. Esta planeación no tenía
acceso directo a `pos-backend`/`pos-heladeria` en su propia sesión, así que los hechos concretos de
código (columnas exactas de `ProductVariant`, revisión `head` de Alembic, líneas del formulario,
ausencia de `ORDER BY` hoy) se verificaron pidiéndoselos a una sesión hermana con los repos reales
abiertos — citados abajo tal como los devolvió, no inventados.

**Hechos verificados contra el código real** (sesión hermana, 2026-08-27):

- `ProductVariant` (`pos-backend/app/models/product_variant.py:14-47`, tabla `tenant.product_variants`):
  `id` (UUID, `UUIDPrimaryKeyMixin`), `created_at`/`updated_at` (`TimestampMixin`), `product_id`
  (UUID FK → `products.id`, `CASCADE`, `NOT NULL`, indexado), `name` (`String(255)`, `NOT NULL`,
  `server_default="Single"`), `sku` (`String(100)`, nulable, único), `price` (`Numeric(12,2)`,
  `NOT NULL`, default `0`), `active` (`Boolean`, default `True`); relaciones `recipe_items`/
  `option_groups` (`cascade="all, delete-orphan"`); constraints `CheckConstraint(price >= 0)` y
  `UniqueConstraint(product_id, name)`. **No existe ninguna columna de orden hoy** — se agrega
  después de `active` (línea ~33).
- Revisión `head` actual de Alembic: `187e491e597a`
  (`187e491e597a_order_item_discount_snapshot.py`, `down_revision='205f518df786'`), confirmada como
  la única no referenciada como `down_revision` de ningún otro archivo entre los 36 existentes en
  `alembic/versions/`. La migración de esta spec encadena `down_revision='187e491e597a'`.
- `@angular/cdk` (`pos-heladeria/package.json`, `^21.2.14`): `grep -rn "@angular/cdk" src/` sin
  resultados — confirmado sin uso en todo `src/`, ya instalado, mismo major que Angular en uso.
- `product-form.component.ts` (1107 líneas, `template:` inline en la línea 62, sin `templateUrl`):
  línea 181, `@for (v of draft().variants; track v.localId)` — franja de pestañas de las
  presentaciones **activas** (clic para fijar `activeLocalId`); línea 202,
  `@for (dv of draft().deactivated; track dv.id)` — lista **aparte** de presentaciones
  desactivadas, para reactivarlas. `draft` es un signal escribible declarado en la línea 429
  (`readonly draft = signal<ProductDraft>(this.emptyDraft())`); `draft().variants` es un array plano
  dentro del valor del signal, mutado hoy vía `this.draft.update(d => ({...d, ...}))`.
- Ninguna de las dos consultas que hoy listan presentaciones tiene orden explícito: la única
  `ProductVariant` con `.order_by(...)` en `catalog/service.py:48-55` es `find_variant_by_name_ci`
  (ordena por `active.desc()` para el chequeo de duplicados, sin relación con orden de despliegue);
  `Product.variants` (`app/models/product.py:52`) no tiene `order_by` en la relación; `menu/router.py:110-114`
  itera `p.variants` directamente, sin `.order_by()`. Confirma que hoy el orden es
  puramente el que devuelva la base de datos (efectivamente inserción/PK), sin ninguna regla que
  esta spec deba respetar más allá de reproducirlo como punto de partida (FR-009).

## Decisión 1 — Nueva columna `display_order` en `product_variants`, no una tabla aparte

- **Decisión**: agregar `product_variants.display_order` (`Integer`, `NOT NULL`), en vez de crear
  una tabla de ordenamiento separada (`product_variant_order(product_id, variant_id, position)`).
- **Rationale**: el orden es un atributo 1:1 de cada presentación, no una relación N:N ni algo que
  cambie de dueño — vive naturalmente en la misma fila que `name`/`sku`/`price`/`active` (spec 002).
  Una tabla aparte obligaría a un `JOIN` adicional en el único endpoint que realmente importa para
  esta spec (el detalle del Menú QR, FR-004) sin ganar nada, porque `product_variants` ya se
  consulta siempre completa por producto.
- **Alternatives considered**: (a) tabla de orden separada — descartada por la razón anterior; (b)
  orden implícito por posición en un array JSON en `products` — descartado porque rompe la
  consulta por índice/único que ya usa el resto del modelo (SKU único por tenant, nombre único por
  producto) y no es consultable con `ORDER BY` nativo de SQL.

## Decisión 2 — Reasignación completa vía endpoint nuevo, invocado dentro de `saveExistingProduct`

- **Decisión**: agregar un endpoint nuevo `PATCH /products/{product_id}/variants/reorder` que
  recibe la lista completa de IDs de presentaciones activas del producto en el orden deseado
  (`{"variant_ids": ["<id1>", "<id2>", ...]}`), valida que todos pertenecen a ese producto y están
  activos, sin duplicados/faltantes, y en una sola transacción asigna `display_order = 1..N` según
  la posición de cada ID en la lista recibida. En el frontend, `product.service.ts` gana un método
  (`reorderVariants(productId, orderedIds)`) invocado como un paso `await` más, **dentro** de
  `saveExistingProduct` (`product.service.ts:478-513`), junto a los demás pasos secuenciales que ya
  ejecuta esa función.
- **Rationale**: verificado contra el código real (sesión hermana, 2026-08-27) que
  `saveExistingProduct` guarda hoy un producto con una secuencia de llamadas HTTP **totalmente
  secuencial y sin transacción compartida** — un `PATCH /products/{id}` (líneas 480-485), luego un
  `for (const v of draft.variants)` que hace `await patchVariant/postVariant` uno a la vez (líneas
  499-504) seguido de dos `PUT` más por variante (`saveVariantConfig`, líneas 516-519), y por último
  un loop de `DELETE /variants/{id}` (líneas 507-511) — sin `Promise.all` y sin rollback si una
  falla a mitad de camino (el manejo de `VariantNameConflictError` en la línea 436-438 asume
  explícitamente que puede quedar a medio guardar). Repartir `display_order` en N `PATCH`
  individuales de ese mismo loop heredaría ese mismo riesgo de quedar a medio reordenar si una
  petición falla — exactamente el escenario que la unicidad `DEFERRABLE` de Decisión 3 no puede
  proteger entre peticiones HTTP separadas (solo protege dentro de una misma transacción). Un único
  endpoint atómico evita el problema por completo: o se guarda el orden completo, o ninguno.
  Corresponde además al patrón ya usado en la misma función para "reemplazo completo"
  (`setRecipe`/`setVariantOptionGroups` ya son `PUT` de reemplazo total, no `PATCH` incremental).
- **Alternatives considered**: (a) `PATCH /variants/{id} {"display_order": N}` uno por presentación
  movida, dentro del mismo loop que ya recorre `draft.variants` — descartado por la razón anterior
  (sin atomicidad entre peticiones). (b) Llamar `/reorder` de inmediato al soltar la fila
  (`drop`), en vez de dentro de `saveExistingProduct` — descartado por consistencia de UX: el resto
  de cambios en memoria del mismo formulario (nombre, precio, `tracksInventory`) solo se persisten
  al presionar "Guardar" (spec 027); el arrastre sigue la misma convención en vez de introducir un
  único campo que se guarda "antes" que los demás. (c) Guardar solo un `display_order` relativo por
  presentación movida (valores flotantes/fraccionarios) para no reescribir toda la secuencia —
  descartado como sobre-ingeniería no pedida por el spec (Principio V); el volumen esperado por
  producto (SC-001 habla de hasta 10) no lo justifica.

## Decisión 3 — Restricción de unicidad `DEFERRABLE`, no una unicidad estricta inmediata

- **Decisión**: agregar `UNIQUE (product_id, display_order)` como `DEFERRABLE INITIALLY DEFERRED`
  sobre `product_variants`, en vez de una unicidad estricta (`NOT DEFERRABLE`, el default de
  Postgres).
- **Rationale**: el endpoint de Decisión 2 reescribe el `display_order` de **todas** las
  presentaciones del producto en la misma transacción. Si dos filas intercambian posición (p. ej. la
  1 pasa a ser la 3 y la 3 pasa a ser la 1), cualquier orden de `UPDATE` fila-por-fila pasa por un
  estado intermedio donde dos filas comparten temporalmente el mismo `display_order` — una
  restricción `UNIQUE` no diferible fallaría en ese punto intermedio, aunque el estado final sea
  válido. Diferir la verificación hasta el `COMMIT` (comportamiento estándar de Postgres para
  constraints `DEFERRABLE INITIALLY DEFERRED`) permite la reescritura completa sin necesitar una
  segunda pasada con valores temporales negativos u otro truco de dos fases.
- **Alternatives considered**: (a) sin restricción única en base de datos, solo invariante de
  aplicación — descartado: el proyecto ya usa `CHECK`/`UNIQUE` como respaldo redundante de sus
  invariantes de negocio en `product_variants` (`ck_product_variant_price_positive`,
  `uq__product_variants__product_id__name`, spec 002), y esta spec no tiene razón para romper ese
  patrón. (b) Actualizar en dos pasadas (primero mover todo a valores negativos temporales, luego a
  los definitivos) para evitar una constraint diferible — descartado, es más código para lograr
  exactamente lo que Postgres ya resuelve de forma nativa con `DEFERRABLE`.

## Decisión 3b — El arrastre solo aplica sobre `draft().variants` (activas); `deactivated` no se toca

- **Decisión**: el `CdkDropList` se agrega únicamente alrededor del `@for` de la línea 181
  (`draft().variants`, presentaciones activas). La lista de la línea 202 (`draft().deactivated`) no
  se envuelve en `CdkDropList` ni participa del arrastre.
- **Rationale**: son dos `@for` separados sobre dos arrays distintos del mismo `ProductDraft`, no
  una sola lista filtrada — confirmado contra el código real. Arrastrar solo tiene sentido dentro de
  `variants`, que es la franja que el administrador ve y usa para elegir cuál presentación editar;
  `deactivated` ya tiene su propio flujo (reactivar) sin relación con el orden visible.
- **Alternatives considered**: fusionar ambas listas en una sola vista con arrastre unificado —
  descartado, exigiría rediseñar un componente que hoy separa claramente "presentaciones en uso" de
  "presentaciones retiradas", un cambio de UI no pedido por el spec (Principio V).

## Decisión 4 — `display_order` no se toca al desactivar ni al reactivar una presentación

- **Decisión**: el soft-delete (`DELETE /variants/{id}`, spec 002 `RN-CAT-10`) y la reactivación
  (`PATCH /variants/{id} {"active": true}`, `RN-CAT-09`) **no modifican** `display_order` en
  absoluto. La fila desactivada conserva su posición en la secuencia; simplemente deja de listarse
  en el Menú QR (filtro `active=True` ya existente en esa consulta, sin cambios).
- **Rationale**: aunque `deactivated` es una lista aparte en la UI (Decisión 3b), la fila sigue
  siendo la misma columna `display_order` en base de datos — desactivar es solo `active=False`
  (`RN-CAT-10`), no un `DELETE`. La fila desactivada **nunca sale** de la secuencia consecutiva de
  valores ya asignados; solo cambia si, tras reactivarse, un administrador vuelve a arrastrarla. Como
  consecuencia directa, FR-007 ("eliminar no debe
  dejar huecos") y FR-008 ("reactivar conserva el orden que tenía") quedan satisfechos por
  construcción, sin ningún código nuevo de renumeración en el `DELETE`/reactivación — ya es
  imposible que se genere un hueco, porque la fila desactivada nunca deja de ocupar su posición.
- **Alternatives considered**: renumerar excluyendo desactivadas al eliminar, y reinsertar al
  reactivar (recalculando dónde "le toca" volver a entrar) — descartado como complejidad
  innecesaria frente a la opción de no tocar nada; además introduciría la pregunta sin respuesta
  clara de "¿en qué posición exacta reaparece?", que el Edge Case de `spec.md` resuelve de forma
  más simple ("recupera el orden que tenía antes de desactivarse").

## Decisión 5 — Migración: backfill por `ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY id)`

- **Decisión**: la migración nueva agrega la columna con `server_default` temporal, hace backfill
  con una única sentencia de ventana por esquema de tenant, y luego agrega la constraint diferible
  de Decisión 3. Seudocódigo por schema (seguir el patrón `@for_each_tenant_schema` ya usado en
  migraciones previas, p. ej. spec 027 `research.md` Decisión 4):
  ```sql
  ALTER TABLE product_variants ADD COLUMN display_order integer;

  UPDATE product_variants pv
  SET display_order = sub.rn
  FROM (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY id) AS rn
    FROM product_variants
  ) sub
  WHERE pv.id = sub.id;

  ALTER TABLE product_variants ALTER COLUMN display_order SET NOT NULL;
  ALTER TABLE product_variants
    ADD CONSTRAINT uq__product_variants__product_id__display_order
    UNIQUE (product_id, display_order) DEFERRABLE INITIALLY DEFERRED;
  ```
  `ORDER BY id` como criterio de backfill usa el mismo `id` (UUID generado en el momento de creación
  vía `UUIDPrimaryKeyMixin`) que hoy determina el orden implícito que ve el Menú QR — confirmado
  arriba que ninguna consulta actual ordena explícitamente por otra cosa (FR-009, SC-004). Se
  prefiere `id` sobre `created_at` porque es la misma columna que la base de datos ya usa hoy de
  facto para el orden natural (índice de la llave primaria), sin depender de la resolución del
  timestamp entre filas creadas en la misma transacción.
- **Rationale**: satisface FR-009 exactamente — el orden inicial que recibe cada presentación ya
  existente es idéntico al que produce hoy el orden implícito de creación, así que desplegar esta
  funcionalidad no reordena visualmente nada hasta que un administrador arrastre algo (SC-004).
- **Alternatives considered**: usar `created_at` en vez de `id` en el `ORDER BY` del backfill —
  descartado porque dos presentaciones creadas en la misma transacción (p. ej. `ensure_default_variant`,
  spec 002) pueden compartir el mismo `created_at`, mientras que el orden de inserción de filas sí
  es determinista; usar `created_at` con empates dejaría el orden de esas filas sin definir. La
  migración nueva encadena `down_revision='187e491e597a'` (revisión `head` confirmada arriba).

## Decisión 6 — Frontend: `@angular/cdk/drag-drop`, sin dependencia nueva

- **Decisión**: usar `CdkDropList`/`CdkDrag`/`moveItemInArray` de `@angular/cdk/drag-drop` para el
  arrastre en `product-form.component.ts`, en vez de una librería de terceros o una implementación
  manual con eventos de puntero.
- **Rationale**: `@angular/cdk` ya está declarado en `pos-heladeria/package.json` (`^21.2.14`), pero
  el registro de riesgos de este repo (`specs/000-reconocimiento/registro-riesgos.md`, R21) señala
  que hoy **no se usa en ningún archivo de `src/`** — es una dependencia instalada sin consumidor.
  Usar su módulo de drag-drop no agrega ninguna dependencia nueva (Principio IX no aplica) y, como
  efecto colateral, le da un primer uso real a un paquete que hoy solo pesa en la instalación sin
  aportar nada — sin que esta spec se proponga "corregir" R21 por sí misma (eso sigue siendo una
  decisión aparte del registro de riesgos, fuera de alcance aquí).
- **Alternatives considered**: (a) implementación manual con `dragstart`/`dragover`/`drop` nativos
  del navegador — descartado, reimplementa lo que el CDK ya resuelve (reordenamiento visual,
  soporte de teclado/accesibilidad, scroll automático de contenedor) sin ganar nada. (b) una
  librería externa de drag-and-drop — descartado, violaría Principio IX (dependencia nueva sin
  justificación) cuando el proyecto ya tiene una dependencia instalada que resuelve exactamente
  esto.
- **Pendiente de verificar al implementar**: el paquete está presente y sin uso (confirmado arriba),
  pero no se ejecutó un build real durante esta planeación — confirmar en la fase de implementación
  que `ng build`/`vitest` resuelven el punto de entrada secundario `@angular/cdk/drag-drop` sin
  advertencias, antes de apoyar la Historia 1 completa en él.

## Decisión 7 — Ordenar en la relación ORM (`Product.variants`), no repetir `.order_by()` en cada consulta

- **Decisión**: agregar `order_by="ProductVariant.display_order"` a la relación `Product.variants`
  (`app/models/product.py:52`), en vez de agregar `.order_by(ProductVariant.display_order)` por
  separado en cada punto que hoy recorre esa relación.
- **Rationale**: confirmado que tanto el detalle del Menú QR (`menu/router.py:110-114`, itera
  `p.variants` directamente) como, previsiblemente, el propio `GET /products/{id}` que alimenta el
  formulario cargan las presentaciones de un producto a través de esa misma relación ORM, sin
  `.order_by()` propio hoy. Ordenar en la definición de la relación es un solo cambio de una línea
  que corrige el orden en **todo** lugar que haga `product.variants` (presente o futuro), en vez de
  depender de que cada nuevo endpoint recuerde agregar el mismo `.order_by()` — más robusto frente a
  omisiones futuras y más fiel a "el orden es un atributo de la relación", no de cada consulta que
  la usa. Ninguna de las dos formas cambia el *shape* de ninguna respuesta — solo el orden en que se
  serializa la lista; `display_order` no necesita exponerse en el JSON del Menú QR (no aporta nada
  al comensal), solo determina el orden ya aplicado antes de serializar.
- **Alternatives considered**: agregar `.order_by(ProductVariant.display_order)` explícito en cada
  consulta que arma la respuesta de `GET /products/{id}` y de `menu/router.py` por separado —
  descartado frente a la opción de la relación: exige tocar dos (o más) sitios en vez de uno, y
  cualquier consulta nueva que en el futuro haga `product.variants` sin saberlo volvería a mostrar
  el orden implícito de antes de esta spec.
