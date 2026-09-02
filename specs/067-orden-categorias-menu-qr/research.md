# Research: Orden personalizado de categorías en el filtro del menú QR

No quedó ningún `NEEDS CLARIFICATION` de negocio en `spec.md` (sesión `/speckit-clarify`,
2026-09-01, resolvió el default de categorías preexistentes y el default de categorías nuevas).
Las incógnitas de este research son puramente técnicas: cómo modelar el orden, dónde imponerlo, y
cómo reproducir exactamente el orden actual al desplegar. Esta planeación sí tuvo acceso directo a
`../pos-backend` y `../pos-heladeria` (siblings de este repo `pos-specs`), así que los hechos de
código citados abajo se verificaron leyendo los archivos reales, no se asumieron.

**Hechos verificados contra el código real** (2026-09-01):

- `Category` (`pos-backend/app/models/category.py:11-22`, tabla `tenant.categories`): `id` (UUID,
  `UUIDPrimaryKeyMixin`), `created_at`/`updated_at` (`TimestampMixin`), `name` (`String(255)`,
  `NOT NULL`, único, indexado), `description` (`String(255)`, nulable), `active` (`Boolean`,
  default `True`); relación `products`. `__table_args__` hoy es solo `({"schema": "tenant"},)` —
  ningún constraint declarado. **No existe ninguna columna de orden hoy.**
- `GET /categories` (`categories/router.py:40`, admin, alimenta `categories-page.component.ts`):
  `select(Category).order_by(Category.name)` — alfabético ascendente. Sin filtro de orden
  adicional.
- `_build_menu` (`menu/router.py:96-98`, Menú QR): `select(Category).where(Category.active.is_(True)).order_by(Category.name)`
  — también alfabético ascendente. Es la única consulta que este feature necesita tocar (FR-005).
- `MenuCategoryResponse` (`menu/schemas.py:105-108`): solo `id`, `name`, `products` — no expone
  ningún campo de orden, igual que `MenuVariantResponse` no expone `display_order` (spec 042). El
  orden viaja implícito en la posición del arreglo, no en un campo.
- `public-menu.component.ts:208`: `@for (category of categories(); track category.id)` sobre las
  pestañas del filtro — itera el arreglo tal como lo devuelve el backend, sin ningún
  `.sort()`/`.order_by` del lado del cliente. Confirma que basta con ordenar la consulta del
  backend; el frontend del comensal no necesita ningún cambio.
- `category-form.component.ts:106-172`: formulario reactivo con dos campos (`name`, `description`),
  usado tanto para crear como para editar (`@Input() category`). `CategoryFormComponent.onSubmit()`
  arma un objeto `CategoryForm` con esos dos campos y llama `createCategory`/`updateCategory` según
  haya o no `category`.
- `category.interface.ts`: `Category` (respuesta), `CategoryForm` (campos editables del
  formulario), `CategoryCreatePayload`/`CategoryUpdatePayload` (bodies HTTP) — los cuatro tipos
  necesitan el campo nuevo.
- `categories-page.component.ts:82-90`: tabla HTML con columnas Nombre/Descripción/Estado/Acciones
  — User Story 3 agrega una columna más, sin tocar el `order_by` de `categoryService.categories()`
  (que sigue viniendo de `GET /categories`, sin cambios).
- Patrón existente para "siguiente valor libre" de un campo de orden:
  `catalog/service.py` (spec 042) calcula el siguiente `display_order` de una presentación con
  `select(func.max(ProductVariant.display_order)).where(ProductVariant.product_id == product_id)`,
  y usa `(current_max or 0) + 1` — mismo patrón aplicable aquí sin el `WHERE product_id` (el orden
  de categorías es global, no por padre).
- Convención de `CheckConstraint` para campos numéricos no negativos: presente en varios modelos
  (`product_variant.py` `price >= 0`, `option.py` `extra_price >= 0`, `option_group.py`
  `min_select >= 0`, `customer_order.py` `discount >= 0`, entre otros) — todos con el patrón
  `CheckConstraint("<campo> >= 0", name="ck_<tabla>_<campo>_<descriptor>")`.
- Precedente directo más cercano: `c8ff3a5551cb_product_variants_display_order.py` (spec 042,
  `product_variants.display_order`) — mismo tipo de columna (`Integer`), mismo patrón de migración
  (`@for_each_tenant_schema`, backfill con `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY id)`,
  `op.alter_column(..., nullable=False)` después del backfill). Difiere de esta spec en que: (a)
  esa columna es `NOT NULL` sin default de aplicación (siempre se asigna al crear una presentación,
  vía un endpoint de reordenamiento dedicado), mientras que aquí el default se calcula solo cuando
  el formulario lo deja vacío; (b) esa columna tiene `UNIQUE (product_id, display_order)
  DEFERRABLE`, mientras que aquí no hay restricción de unicidad (spec.md, Assumptions); (c) el
  orden ahí es ascendente (1..N, primero = menor valor); aquí es descendente (mayor valor = primero,
  decisión explícita del negocio en el input original de la spec, "de mayor a menor").
- Alembic: 47 archivos en `alembic/versions/`. Resolviendo la cadena `down_revision` completa
  (incluyendo revisiones con múltiples padres declarados como tupla), el único `head` real es
  `a96852d7be6a` (`a96852d7be6a_option_group_selection_mode_and_item_option_quantity.py`,
  correspondiente a spec 065, la feature fusionada más recientemente según `git log`). La CLI
  `alembic heads` no corre hoy en este entorno por un `NameError` de un módulo no relacionado
  (`94b7e35f5e5e_063d_promociones_reglas_destructivo.py`, llama `op.f(...)` a nivel de módulo);
  no se toca ese archivo porque está fuera del alcance de esta spec.
- Framework de tests backend: `unittest`, ejecutado con
  `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`. No existe hoy
  ningún characterization test de `categories`.
- Framework de tests frontend: Vitest (`^4.0.8`) vía `@angular/build:unit-test`.

## Decisión 1 — Columna simple `categories.display_order`, sin tabla ni restricción de unicidad separada

- **Decisión**: agregar `categories.display_order` (`Integer`, `NOT NULL` tras backfill), con
  `CheckConstraint("display_order >= 0", ...)` pero **sin** `UNIQUE`.
- **Rationale**: spec.md (Assumptions, resuelto sin necesidad de clarificación) permite
  explícitamente que dos categorías compartan el mismo valor de orden, con desempate alfabético
  (FR-006). Sin restricción de unicidad, no hace falta ninguna estrategia de "dos pasadas" ni
  `DEFERRABLE` — muy distinto del caso de `product_variants.display_order` (spec 042), donde la
  unicidad por producto exige atomicidad de reordenamiento completo. Aquí cada categoría se edita
  de forma independiente, un valor a la vez, exactamente como ya se editan `name`/`description`.
- **Alternatives considered**: (a) reproducir el mismo patrón de spec 042 con `UNIQUE
  (display_order)` global — descartada porque obligaría a mover el valor de otras categorías cada
  vez que dos coincidan, una fricción que el negocio no pidió (el input original solo pide "definir
  un orden en el formulario", no "una posición única sin huecos"); (b) tabla de ordenamiento
  separada (`category_order(category_id, position)`) — descartada por la misma razón que en spec
  042: el orden es un atributo 1:1 de cada categoría, no una relación N:N.

## Decisión 2 — Reutilizar `POST`/`PATCH /categories` existentes, sin endpoint de reordenamiento dedicado

- **Decisión**: agregar `display_order: int | None = Field(None, ge=0)` a `CategoryCreate` y
  `CategoryUpdate`, y `display_order: int` a `CategoryResponse`. `create_category` calcula
  `MAX(display_order) + 1` cuando `body.display_order is None`; `update_category` asigna
  `body.display_order` solo si viene en el body (igual que ya hace con `name`/`description`).
- **Rationale**: a diferencia de spec 042 (arrastrar-soltar sobre una lista completa de
  presentaciones, que exige reasignar 1..N atómicamente), aquí el administrador escribe un número
  en un campo del formulario de **una sola** categoría a la vez — ya es exactamente la forma en que
  `update_category` trata `name`/`description`/`active` hoy (`categories/router.py:107-127`). No
  hay ninguna operación de "mover" que toque más de una fila.
- **Alternatives considered**: (a) endpoint `PATCH /categories/reorder` que reciba la lista
  completa — descartado por sobre-ingeniería: no hay ningún requisito de arrastrar-soltar ni de
  secuencia sin huecos (spec.md permite duplicados); (b) campo obligatorio sin default automático —
  descartado porque FR-004 exige explícitamente que el campo no bloquee la creación cuando se deja
  vacío.

## Decisión 3 — Orden descendente aplicado únicamente en `_build_menu` (Menú QR), no en el listado de administración

- **Decisión**: `menu/router.py:97` cambia de `.order_by(Category.name)` a
  `.order_by(Category.display_order.desc(), Category.name)`. `categories/router.py:40` (listado de
  administración) **no cambia**.
- **Rationale**: FR-005 dice explícitamente "el filtro de categorías del menú QR"; spec.md,
  Assumptions, acota el alcance a ese único lugar. Cambiar también el listado de administración
  sería una decisión de producto no pedida (¿debería el admin ver sus categorías en el mismo orden
  del menú, o alfabético para encontrarlas rápido?) — no está en el spec, así que no se toca
  (Principio V, evitar alcance no solicitado). El tie-break por nombre (FR-006) se aplica como
  segundo criterio del `order_by`, igual en ambos casos si en el futuro se decide extenderlo.
- **Alternatives considered**: (a) mover el `order_by` a la relación `Category.products` o a nivel
  de modelo (como hizo spec 042 con `Product.variants`) — no aplica aquí porque no hay una relación
  ORM que "categorías dentro de X" recorra en varios lugares; los dos `order_by` (admin y menú) ya
  son consultas independientes hoy, así que solo se toca la que el spec pide.

## Decisión 4 — Backfill con `ROW_NUMBER() OVER (ORDER BY name DESC)`

- **Decisión**: la migración asigna
  `display_order = ROW_NUMBER() OVER (ORDER BY name DESC)` a las categorías existentes.
- **Rationale**: el Menú QR hoy ordena por `Category.name` **ascendente** (ver hechos verificados).
  Este feature ordenará por `display_order` **descendente**. Para que el backfill reproduzca
  exactamente la secuencia visible hoy (FR-009, clarification 2026-09-01, SC-003), el valor más
  alto debe caer en el nombre alfabéticamente más pequeño. `ROW_NUMBER() OVER (ORDER BY name DESC)`
  asigna 1 al nombre alfabéticamente más grande y N al más pequeño; al ordenar luego por
  `display_order DESC`, el nombre más pequeño (valor N) sale primero, exactamente el orden ASC por
  nombre que ya existe. Verificado con un ejemplo de 3 categorías (`Bebidas`, `Helados`, `Postres`):
  `ORDER BY name DESC` da rn 1/2/3 a Postres/Helados/Bebidas; `display_order DESC` después ordena
  Bebidas(3)/Helados(2)/Postres(1) — igual que `ORDER BY name ASC` hoy.
- **Alternatives considered**: (a) `ROW_NUMBER() OVER (ORDER BY name ASC)` sin invertir —
  descartada porque produciría el orden alfabético **invertido** al combinarse con
  `display_order DESC` (justo el error que esta decisión evita); (b) backfill por `created_at`/`id`
  (como spec 042 hizo para presentaciones) — descartada porque, a diferencia de
  `product_variants` (que no tenía ningún `order_by` propio y mostraba orden de inserción), las
  categorías **sí** tienen un orden visible hoy (alfabético), y es ese el que hay que preservar,
  no el de creación.

## Decisión 5 — `CheckConstraint` de no negatividad a nivel de base de datos, además de la validación de Pydantic

- **Decisión**: `CheckConstraint("display_order >= 0", name="ck_category_display_order_non_negative")`
  en `Category.__table_args__`, además de `Field(None, ge=0)` en los schemas Pydantic.
- **Rationale**: FR-003 exige rechazar valores negativos. El patrón ya establecido en el
  codebase para campos numéricos no negativos (`price`, `extra_price`, `min_select`, `discount`,
  etc., research.md "hechos verificados") siempre combina validación de API con `CheckConstraint`
  de base de datos — defensa en profundidad contra cualquier ruta que escriba directo a la fila sin
  pasar por el endpoint (scripts, fixtures de test, futuras migraciones).
- **Alternatives considered**: solo validación de Pydantic, sin constraint de BD — descartada por
  inconsistencia con el resto del modelo de datos del proyecto.
