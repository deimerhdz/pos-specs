# Research: Tipo de precio e inventario condicional en grupos de opciones

Fase 0 de `/speckit-plan`. Cada decisión cita el código real inspeccionado en
`../pos-backend` y `../pos-heladeria` (ambos en disco, sibling de `pos-specs`) — sin
`NEEDS CLARIFICATION` pendiente: las cuatro preguntas de diseño ya se resolvieron en
`spec.md` (sección Clarifications) antes de este plan.

## Decisión 1 — Columna y valores de `OptionGroup.pricing_type`

**Decisión**: agregar `pricing_type: str` a `option_groups` (schema `tenant`), con
`CheckConstraint("pricing_type IN ('incluido', 'con_recargo')", name="ck_option_group_pricing_type")`.
Sin tipo `ENUM` de PostgreSQL — el proyecto no usa enums nativos para este tipo de campo
(precedente: `Product.preparation_type` es `String(20)` + `CheckConstraint`, no un
`postgresql.ENUM`, `app/models/product.py`).

`server_default='con_recargo'` a nivel de columna (protege cualquier `INSERT` que no
mencione la columna — scripts, tests, código futuro). En el schema Pydantic
`OptionGroupCreate`, en cambio, `pricing_type` es **obligatorio, sin default** — FR-001
exige que el administrador lo elija explícitamente al crear un grupo nuevo; no hay un
valor "razonable por defecto" entre dos casos de uso igual de válidos (sabor vs. topping),
a diferencia de `Product.tracks_inventory` (spec 027), donde sí existía un default de
negocio claro ("apagado por defecto").

**Rationale**: mismo patrón de asimetría de defaults ya usado en spec 027 (research.md de
esa spec, Decisión 3: default `True` a nivel de ORM/BD para no romper código existente,
default `False` solo en el schema `ProductCreate` porque ahí sí hay una regla de negocio
que lo exige). Aquí la asimetría es la misma mecánica, con la diferencia de que el schema
de creación no tiene default en absoluto — es un campo requerido.

**Alternativas consideradas**:
- Enum nativo de PostgreSQL (`CREATE TYPE`): rechazado — agregar un valor nuevo a un enum
  nativo requiere una migración distinta (`ALTER TYPE ... ADD VALUE`, no transaccional en
  versiones viejas de PostgreSQL) que el `CheckConstraint` con `String` no necesita; el
  proyecto ya evita enums nativos en columnas equivalentes.
- Booleano (`is_included: bool`): rechazado — un booleano fuerza una polaridad ("incluido"
  = `true`/`false`) que no se lee igual de claro en consultas SQL de diagnóstico
  (`WHERE pricing_type = 'incluido'` es más legible que `WHERE is_included`), y no deja
  espacio para un tercer valor futuro sin otra migración de tipo. Un `String` + `CHECK` es
  el patrón ya establecido (`preparation_type`) para "un campo con un conjunto cerrado y
  pequeño de valores válidos, con posibilidad de crecer".

## Decisión 2 — Dónde se hace cumplir "Incluido implica precio $0"

**Decisión**: validación a nivel de servicio (no solo de UI) en tres puntos del backend:

1. `add_option` (`app/api/v1/catalog/router.py:347-377`): tras cargar el `OptionGroup`
   (`get_or_404`), si `group.pricing_type == "incluido"` y `body.extra_price != 0`, `422`
   ("Los grupos «Incluido» no permiten precio distinto de $0.").
2. `update_option` (`router.py:380-419`): mismo chequeo cuando `body.extra_price is not
   None` — se resuelve el grupo de la opción existente (`option.option_group_id`) antes de
   aplicar el nuevo `extra_price`.
3. `update_option_group` (`router.py:296-326`): cuando `body.pricing_type == "incluido"` y
   el valor anterior era `"con_recargo"`, el sistema fuerza `extra_price = 0` en **todas**
   las opciones del grupo (`UPDATE options SET extra_price = 0 WHERE option_group_id = :id`)
   como efecto del propio `PATCH` — no rechaza el cambio, lo aplica (FR-004).

**Rationale**: el proyecto ya tiene el precedente exacto de "un campo fuerza a cero a otro
campo relacionado del mismo recurso" en `update_option` mismo (`router.py:405-412`,
`RN-CAT-38`: desvincular `inventory_item_id` resetea `item_quantity` a `0` sin importar qué
valor traiga el resto del payload). La validación de servicio (no solo de formulario) es
necesaria porque este proyecto expone la API públicamente vía OpenAPI y no depende
únicamente del frontend Angular para garantizar sus invariantes de negocio (mismo criterio
de doble respaldo ya usado en `RN-CAT-03`/`RN-CAT-04`, spec 002: schema + `CHECK`, aunque
aquí el chequeo cruza dos tablas y no puede expresarse como un `CheckConstraint` de una sola
fila — vive en el servicio, no en la base de datos).

La confirmación explícita antes de aceptar el cambio de "Con recargo" a "Incluido" con
precios ya configurados (FR-004, User Story 2 escenario 3) es una decisión de **UX**, no de
contrato de datos — vive enteramente en el frontend (`ConfirmService`, mismo patrón que
`toggleTracksInventory()` en `product-form.component.ts:854-868`, spec 027). El backend no
la exige ni la valida: si el `PATCH` llega con `pricing_type: "incluido"`, se aplica
directamente (igual que `product-tracks-inventory-field.md` de spec 027 documenta para su
propio switch: "el backend no la exige ni la valida, es una decisión de UX").

**Alternativas consideradas**:
- Rechazar el cambio con `409` si hay opciones con precio, exigiendo que el cliente las
  ponga en 0 una por una antes: rechazado — obliga al frontend a hacer N llamadas
  adicionales antes de un cambio que el usuario ya confirmó, y contradice el patrón ya
  aceptado en `RN-CAT-38` (el backend resetea, no exige que el cliente resetee antes).

## Decisión 3 — Unificar el criterio de "¿esta opción descuenta inventario?" (A-32)

**Decisión**: modificar `grupos_que_descuentan` (`app/catalog_engine/pricing.py:62-84`)
para que su segunda condición (evaluada por opción) exija exactamente lo mismo que
`group_discounts` (`app/catalog_engine/consumption.py:70-86`):

```python
# Antes (pricing.py:73-78)
por_opcion = set(db.execute(
    select(Option.option_group_id)
    .where(Option.option_group_id.in_(gids), Option.item_quantity > 0)
    .distinct()
).scalars().all())

# Después — mismo criterio que group_discounts (consumption.py:78-84)
por_opcion = set(db.execute(
    select(Option.option_group_id)
    .where(
        Option.option_group_id.in_(gids),
        Option.active.is_(True),
        Option.inventory_item_id.is_not(None),
        Option.item_quantity > 0,
    )
    .distinct()
).scalars().all())
```

`group_discounts` no cambia — ya es el criterio elegido como canónico.

**Rationale**: no es una elección arbitraria entre las dos — el criterio de tres
condiciones de `group_discounts` **ya es el tratado como correcto en otro lugar del propio
código**: la migración `e3f4a5b6c7d8_products_tracks_inventory.py` (líneas 45-61, spec 027)
hardcodea exactamente `active = true AND inventory_item_id IS NOT NULL AND item_quantity >
0` en su SQL de backfill para decidir si un producto existente "ya superaba" `RN-CAT-34` —
nunca usó el criterio de una sola condición de `grupos_que_descuentan`. Unificar hacia el
criterio ya usado en esa migración es la opción de menor sorpresa: hace que
`load_valid_options`/`validate_option_selection` (que llaman a `grupos_que_descuentan`) y
`ensure_lines_consume_inventory` (que llama a `group_discounts`) respondan lo mismo ante
una opción a medio configurar (cantidad puesta, insumo sin enlazar o desactivado) — el caso
exacto que hoy puede mostrarle al cajero el mensaje equivocado (`RN-CAT-39`).

**Autorización de negocio**: este es un cambio de comportamiento sobre una regla ya
congelada por characterization tests (`app/characterization_tests/test_catalog_line_pricing.py`,
`test_catalog_consumption_plan.py::EnsureLinesConsumeInventoryTests` — ambos citan `A-32`
explícitamente). Principio III de la Constitución exige, para tocarlos: (1) un spec que
defina el nuevo comportamiento — `spec.md` FR-009, con su sesión de Clarifications
(2026-09-01); (2) una decisión de negocio registrada; (3) actualización explícita de los
tests afectados; (4) evidencia de que no afecta negativamente otros comportamientos
protegidos. Este plan agrega la tarea (Fase de tasks, fuera de este documento) de sumar una
entrada a `specs/000-reconocimiento/registro-de-anomalias.md` marcando `A-32` como
**resuelta** por spec 064, citando esta decisión — sin esa entrada, el Principio II
considera el cambio no autorizado aunque el spec lo describa.

**Impacto verificado, sin regresión**: dado que `group_discounts` ya es más estricto (exige
dos condiciones más), unificar hacia él solo puede **relajar** una exigencia hoy vigente en
`validate_option_selection` (una opción a medio configurar deja de forzar "elige
exactamente el máximo" cuando antes sí lo forzaba) — nunca la endurece. Ninguna venta que
hoy se acepta puede empezar a rechazarse por este cambio; sí es posible que una venta que
hoy se rechaza (por una opción con cantidad puesta pero sin insumo real) empiece a
aceptarse, lo cual es exactamente el comportamiento que FR-010 (mensaje correcto) necesita
para no confundir "sin insumo enlazado" con "sin receta en absoluto".

**Alternativas consideradas**:
- Unificar hacia el criterio de una sola condición de `grupos_que_descuentan` (relajar
  `group_discounts` en vez de endurecer `grupos_que_descuentan`): rechazado — invertiría el
  criterio que la migración de spec 027 ya trató como canónico, y volvería **más
  permisivo** el punto que hoy protege la venta real (`ensure_lines_consume_inventory`),
  aumentando el riesgo de ventas que no descuentan nada en silencio (el problema exacto que
  motivó `RN-CAT-34`).
- Dejar A-32 sin resolver (fuera de alcance): rechazado por decisión explícita del usuario
  en la sesión de Clarifications de `spec.md` — la pregunta se hizo y se respondió "sí,
  unificar".

## Decisión 4 — Gating por plan: campo, no ruta completa

**Decisión**: a diferencia del patrón de spec 062 (`require_module_access("inventario")`
como dependencia de **router completo** o de **ruta completa**, ej.
`unit_measures/router.py:15-18`), esta spec necesita un gating **a nivel de campo dentro de
un endpoint que debe seguir funcionando sin el módulo**: `POST`/`PATCH /products` deben
seguir creando/editando productos (con o sin inventario) para cualquier tenant, y
`POST /option-groups/{id}/options` / `PATCH /options/{id}` deben seguir permitiendo crear
toppings con precio y sin insumo para un tenant sin el módulo Inventario. Bloquear la ruta
completa (como `unit_measures`) rompería el caso de uso central de esta misma spec para
tenants sin Inventario (Historia 6, Edge Case: "grupos… configurados solo por precio, sin
ningún insumo enlazado… siguen funcionando con normalidad").

Se extrae la lógica interna de `require_module_access` (`app/core/plan_limits.py:126-149`)
a una función pura reutilizable:

```python
def ensure_module_access(db: Session, tenant: Tenant, module_key: str) -> None:
    """Misma comprobación que `require_module_access`, invocable directamente
    (no solo como dependencia de FastAPI) para gating a nivel de campo, no
    de ruta completa. Mismo mensaje y mismo criterio de vencimiento."""
    ensure_plan_not_expired(tenant)
    access_column = f"{module_key}_access"
    labels = {"inventario": "inventario", "compras": "compras", "promociones": "promociones"}
    plan = db.get(Plan, tenant.plan_id)
    if not getattr(plan, access_column):
        raise HTTPException(
            status.HTTP_403_FORBIDDEN,
            f"Tu plan actual no incluye el módulo de {labels.get(module_key, module_key)}.",
        )

def require_module_access(module_key: str) -> Callable:
    """Dependencia FastAPI — sin cambio de firma ni de comportamiento para
    los routers que ya la usan (spec 033/062)."""
    def _dependency(
        tenant: Tenant = Depends(get_tenant),
        db: Session = Depends(get_db),
        _user: User = Depends(get_current_user),
    ) -> None:
        ensure_module_access(db, tenant, module_key)
    return _dependency
```

Uso, mismo estilo que `enforce_plan_limit(db, tenant, "productos")` (ya invocado
directamente dentro de `create_product`, `products/router.py:82`, no como `Depends`):

- `ProductService.create_product`/`update_product` (`app/api/v1/products/service.py:55-65`,
  `87-109`): si `data.tracks_inventory is True`, llamar
  `ensure_module_access(db, tenant, "inventario")` antes de persistir. `update_product`
  hoy **no** recibe `tenant` (`products/router.py:104-112`) — gana
  `tenant: Tenant = Depends(get_tenant)` igual que ya tiene `create_product` (línea 79).
- `add_option`/`update_option` (`catalog/router.py:347-377`, `380-419`): si el payload deja
  a la opción con `inventory_item_id is not None` (nuevo o ya existente) **o**
  `item_quantity > 0`, llamar `ensure_module_access(db, tenant, "inventario")`. Ambos
  endpoints ganan `tenant: Tenant = Depends(get_tenant)` (hoy no lo reciben).

**Rationale**: `enforce_plan_limit` ya demuestra que este proyecto sabe expresar gating de
plan como función invocada condicionalmente dentro de un handler, no solo como dependencia
de ruta — es el precedente correcto a seguir aquí, no el de spec 062 (que sí aplicaba a
pantallas/rutas completas, un caso distinto).

**Sin gating nuevo para `quantity_per_option`** (en el guardado unificado de producto,
`_save_variant_entry` → `_replace_option_groups`, `app/api/v1/catalog/service.py:215`): un
tenant sin el módulo Inventario que de todas formas envíe `quantity_per_option > 0` no
necesita bloquearse explícitamente aquí, porque **ya es inerte** — `_tracks_inventory`
(`consumption.py:163-171`) ignora `quantity_per_option` en cualquier producto con
`tracks_inventory=False`, y `tracks_inventory=True` ya está bloqueado por el punto anterior
para ese mismo tenant. Añadir un segundo chequeo redundante sobre un valor que el propio
switch ya neutraliza no aporta nada verificable (mismo criterio de "no bloquear lo que ya es
inofensivo" que spec 027 aplicó a los "insumos fantasma", FR-008).

**Alternativas consideradas**:
- Aplicar `require_module_access("inventario")` como dependencia de router completo sobre
  `products/router.py` y `catalog/router.py`: rechazado de raíz — bloquearía crear
  cualquier producto o cualquier topping con precio para un tenant sin Inventario, violando
  el propósito central de esta spec.
- Validar el gating únicamente en el frontend (ocultar los campos) sin respaldo en backend:
  rechazado — mismo criterio de doble respaldo de Decisión 2; un cliente HTTP que no pase
  por el formulario Angular podría activar inventario sin el módulo si el backend no lo
  impide, y FR-012 exige explícitamente "por cualquier vía, no solo la pantalla".

## Decisión 5 — Gating en frontend: switch de producto vs. editor de catálogo compartido

**Decisión**: dos mecanismos distintos, porque el switch de producto y el editor de
opciones no comparten contexto (hallazgo de exploración: `OptionGroup`/`Option` son
entidades de catálogo compartidas entre productos, no propiedad de uno solo —
`option-groups-page.component.ts` no recibe ni conoce ningún producto concreto).

1. **`product-form.component.ts`** — el switch "Maneja inventario" (línea 181,
   `tracks_inventory`) se deshabilita (no se oculta: un producto existente puede ya tener
   `tracks_inventory=true` desde antes de que el tenant perdiera el acceso al plan, y debe
   seguir viéndose) cuando `!inventarioIncluido()`. Se agrega una señal computada
   **fail-closed** (mismo estilo que `comprasIncluido` en
   `inventory-page.component.ts:385-388`, no el fail-open de `settings-page.component.ts` —
   aquí ocultar de más mientras el plan carga es más seguro que dejar togglear un switch que
   el backend rechazará con `403` un instante después):
   ```typescript
   readonly inventarioIncluido = computed(() => {
     const summary = this.planSummaryService.summary();
     return summary !== null && summary.modules.inventario && !summary.vencido;
   });
   ```
   Las secciones "Insumos fijos"/"Sabores a elegir" (líneas 275-413) pasan a condicionarse
   por `sectionsEnabled()` (`draft().tracks_inventory && inventarioIncluido()`), no
   directamente por `draft().tracks_inventory` — sin tocar el valor guardado en el draft
   (FR-013: los datos no se pierden, solo dejan de mostrarse/editarse).

2. **`option-groups-page.component.ts` / `option-form.component.ts`** (editor de catálogo
   compartido, sin contexto de producto): los campos "Insumo que consume" e "item_quantity"
   (`option-form.component.ts:53-73`) se ocultan cuando `!inventarioIncluido()` — gating
   **por plan únicamente**, nunca por el switch de un producto específico, porque el mismo
   `OptionGroup` puede estar asignado a variantes de varios productos con distinto
   `tracks_inventory` (edge case ya documentado en `spec.md`).

**Rationale — restructuración necesaria de "Sabores a elegir" (hallazgo de exploración
frontend)**: hoy, todo el bloque "Sabores a elegir" dentro de `product-form.component.ts`
(líneas 302-408) — incluida la selección de **qué** grupo de opciones ofrece la variante
(`g.option_group_id`) y su `min_select`/`max_select` propio — vive dentro del mismo
`@if (draft().tracks_inventory)` que "Insumos fijos" (línea 275). Esto significa que **hoy
un producto sin inventario no puede ofrecer ningún grupo de opciones en absoluto** — no
solo que sus opciones no descuenten stock, sino que el propio menú de sabores/toppings es
inalcanzable para ese producto. Esto contradice directamente `spec.md` FR-006 (que exige
dejar editable "el nombre y el precio de cada opción" incluso sin inventario) y es, de
hecho, el problema concreto que motivó el pedido original del usuario ("el topping/sabor
tampoco debería manejar inventario" — hoy ni siquiera puede *existir* en un producto así).

La solución: separar el bloque en dos niveles de condicional dentro de "Sabores a elegir"
(ya no todo detrás de un único `@if`):
- **Siempre visibles** (con o sin `tracks_inventory`): selector de grupo
  (`g.option_group_id`), `min_select`/`max_select` de esa variante — un producto sin
  inventario sigue pudiendo decir "el cliente elige 1 de estos 3 sabores/toppings".
- **Solo si `sectionsEnabled()`**: el input "descuenta [cantidad] por cada uno"
  (`quantity_per_option`, línea 331-334), el resumen "Descuenta de: …" y su detalle
  (líneas 347-401) — la parte exclusivamente relacionada con inventario.
- "Insumos fijos" (receta fija, líneas 276-300) sigue detrás de un único `@if` sin
  fraccionar — es 100% un concepto de inventario, sin ninguna parte de precio o menú que
  deba sobrevivir sin él.

**Alternativas consideradas**:
- Ocultar el bloque "Sabores a elegir" completo también en la versión final (dejar el
  comportamiento actual tal cual): rechazado — contradice `spec.md` FR-006 explícitamente y
  el propósito central del pedido del usuario.
- Gating del editor de catálogo compartido (`option-groups-page.component.ts`) por el
  switch de "el último producto que lo editó" o algún producto de referencia: rechazado —
  no existe tal noción en el modelo (`OptionGroup` no tiene `product_id`), e inventarlo
  agregaría una relación nueva y ambigua para un problema que el gating por plan (más
  simple y ya establecido, spec 033) resuelve sin ambigüedad.

## Decisión 6 — Migración `option_groups.pricing_type`

**Decisión**: nueva migración `alembic/versions/<hex>_option_groups_pricing_type.py`,
`down_revision = '94b7e35f5e5e'` (head actual, confirmado sin otra migración que lo
referencie como su propio `down_revision`), mismo patrón de
`b8c9d0e1f2a3_option_group_active.py` (columna booleana/`CHECK` agregada a
`option_groups`, con guarda `_has_table`) combinado con el backfill por `EXISTS` de
`e3f4a5b6c7d8_products_tracks_inventory.py` (spec 027):

```python
@for_each_tenant_schema
def upgrade(schema: str) -> None:
    if not _has_table(schema, "option_groups"):
        return
    op.add_column(
        "option_groups",
        sa.Column("pricing_type", sa.String(length=20), nullable=False, server_default="con_recargo"),
        schema=schema,
    )
    op.create_check_constraint(
        "ck_option_group_pricing_type",
        "option_groups",
        "pricing_type IN ('incluido', 'con_recargo')",
        schema=schema,
    )
    op.execute(f"""
        UPDATE {schema}.option_groups og
        SET pricing_type = CASE
            WHEN EXISTS (
                SELECT 1 FROM {schema}.options o
                WHERE o.option_group_id = og.id AND o.extra_price > 0
            ) THEN 'con_recargo'
            ELSE 'incluido'
        END
    """)

@for_each_tenant_schema
def downgrade(schema: str) -> None:
    if not _has_table(schema, "option_groups"):
        return
    op.drop_constraint("ck_option_group_pricing_type", "option_groups", schema=schema)
    op.drop_column("option_groups", "pricing_type", schema=schema)
```

**Rationale**: FR-015 exige exactamente esta regla ("con al menos una opción con precio
mayor a $0 → Con recargo; todas en $0 → Incluido"), sin distinguir activas/inactivas en su
texto — el backfill considera todas las opciones del grupo, sin filtrar por `active`, para
igualar la literalidad del criterio de negocio ya aprobado en `spec.md` (a diferencia de
Decisión 3, donde el criterio de "descuenta inventario" sí exige `active=True` porque ese
es un criterio distinto, de comportamiento en venta, no de clasificación histórica única).

**Alternativas consideradas**:
- Backfill considerando solo opciones activas: rechazado — el texto de FR-015 no lo pide, y
  usar solo activas podría reclasificar como "Incluido" un grupo cuya única opción con
  precio fue desactivada hace tiempo, perdiendo la evidencia de que ese grupo sí fue
  diseñado para cobrar recargo (decisión más conservadora: preservar la intención original).

## Decisión 7 — `registro-de-anomalias.md`: A-32 pasa a resuelta

**Decisión**: agregar una entrada nueva en
`specs/000-reconocimiento/registro-de-anomalias.md` (no se edita la entrada original de
A-32, se agrega una nota de resolución con fecha, citando spec 064, la decisión de
unificar hacia el criterio de `group_discounts`, y los tests actualizados) — tarea que
`/speckit-tasks` debe incluir explícitamente en `tasks.md`, no ejecutada en este plan.

**Rationale**: Principio II de la Constitución exige que todo cambio de comportamiento
sobre una regla previamente documentada quede registrado en ese archivo con quién/cuándo/
qué/por qué — el spec por sí solo (aunque ya contiene la decisión) no sustituye ese
registro, que es el "libro de autorizaciones vivo" citado explícitamente en el §Alcance de
la Constitución.

## Resumen de versiones y entorno (sin cambios respecto a spec 062)

**Backend**: Python 3.14.4 (venv `pos-backend/env`), FastAPI + SQLAlchemy + Alembic,
PostgreSQL 16 schema-per-tenant. Sin dependencias nuevas.

**Frontend**: Angular 21.1.x (standalone components + signals), TypeScript 5.9.2, Node
24.16.0. Sin dependencias nuevas — reutiliza `PlanSummaryService`/`ModuleAccess` ya
existentes (`src/app/modules/plan/`).

**Testing**: backend — `unittest` vía `python -m unittest discover -s
app/characterization_tests -p 'test_*.py'`; frontend — Vitest vía `ng test`.
