# Data Model: Orden personalizado de categorías en el filtro del menú QR

Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a la columna, la restricción, la migración y la asignación de valores.

## Category (`categories`, schema `tenant`) — MODIFICADA

Columna nueva:

| Columna | Tipo | Nulable | Default (ORM) | Default (BD) | Notas |
|---|---|---|---|---|---|
| `display_order` | `Integer` | No | sin default de aplicación — se asigna explícitamente en `create_category` (research.md Decisión 2) | ninguno tras el backfill (research.md Decisión 4) | Posición de la categoría dentro del filtro de categorías del Menú QR. A mayor valor, aparece primero (FR-005). |

Restricción nueva:

- `CheckConstraint("display_order >= 0", name="ck_category_display_order_non_negative")`
  (research.md Decisión 5). Sin `UNIQUE` — dos categorías pueden compartir el mismo valor
  (FR-006, desempate alfabético por nombre).

Columnas sin cambio (confirmadas contra `app/models/category.py:11-22`): `id`, `created_at`,
`updated_at`, `name` (único, `String(255)`), `description` (`String(255)`, nulable), `active`
(`Boolean`, default `True`); relación `products`.

## Menú QR (`_build_menu`, `app/api/v1/menu/router.py:96-98`) — consulta modificada

```python
categories = db.execute(
    select(Category)
    .where(Category.active.is_(True))
    .order_by(Category.display_order.desc(), Category.name)
).scalars().all()
```

Único punto de lectura que este feature ordena por `display_order` (research.md Decisión 3).
`MenuCategoryResponse` no gana ningún campo — el orden viaja implícito en la posición del arreglo,
igual que hoy (`id`, `name`, `products`, sin cambio de forma).

## Listado de administración (`GET /categories`, `categories/router.py:40`) — SIN CAMBIO DE ORDEN

Sigue `select(Category).order_by(Category.name)`. Expone `display_order` en `CategoryResponse`
(para que la tabla de administración pueda mostrarlo, User Story 3) pero no ordena por él
(research.md Decisión 3, fuera de alcance de spec.md).

## Asignación de `display_order` por operación

| Operación | Efecto sobre `display_order` | Regla |
|---|---|---|
| Crear categoría con valor explícito (`POST /categories`, `body.display_order` presente) | Se guarda tal cual, validado `>= 0` (Pydantic `ge=0` + `CheckConstraint`). | FR-001, FR-003 |
| Crear categoría sin valor (`POST /categories`, `body.display_order` es `None`) | Recibe `MAX(display_order) + 1` entre **todas** las categorías existentes (activas e inactivas) — aparece primera en el filtro del Menú QR. | FR-004, clarification 2026-09-01 |
| Editar con valor explícito (`PATCH /categories/{id}`, `body.display_order` presente) | Se reemplaza por el nuevo valor, validado igual que al crear. | FR-002, FR-003 |
| Editar sin tocar el campo (`PATCH /categories/{id}`, `body.display_order` ausente/`None`) | Ninguno — conserva el valor que ya tenía, igual que `name`/`description` cuando no vienen en el body. | FR-002 (edición parcial) |
| Desactivar/reactivar (`DELETE /categories/{id}`, `PATCH ... {"active": ...}`) | Ninguno — `display_order` no se modifica; solo dejan de contarse en el filtro del Menú QR mientras estén inactivas (FR-007). | FR-007 |

## Migración de datos existentes

Ver research.md, Decisión 4, para la justificación completa. Resumen de la regla de backfill:

```sql
UPDATE tenant.categories c
SET display_order = sub.rn
FROM (
    SELECT id, ROW_NUMBER() OVER (ORDER BY name DESC) AS rn
    FROM tenant.categories
) sub
WHERE c.id = sub.id
```

Al ordenar después por `display_order DESC`, esto reproduce exactamente el orden alfabético
ascendente por `name` que el Menú QR ya muestra hoy — desplegar el feature no reordena ninguna
categoría existente por sí solo (FR-009, clarification 2026-09-01, SC-003).

Pasos de la migración (`@for_each_tenant_schema`, mismo patrón que
`c8ff3a5551cb_product_variants_display_order.py`):

1. `op.add_column("categories", sa.Column("display_order", sa.Integer(), nullable=True), schema=schema)`
2. `op.execute(...)` con el `UPDATE` de backfill de arriba (adaptado al `schema` de cada tenant).
3. `op.alter_column("categories", "display_order", nullable=False, schema=schema)`
4. `op.create_check_constraint("ck_category_display_order_non_negative", "categories", "display_order >= 0", schema=schema)`

**Rollback** (`downgrade`): `op.drop_constraint(...)` seguido de
`op.drop_column("categories", "display_order", schema=schema)` — reversible sin afectar ninguna
otra columna ni fila de `categories`.

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| El formulario permite escribir un valor de orden entero, `>= 0`, al crear o editar | `category-form.component.ts`, nuevo `FormControl('display_order', ...)`; `CategoryCreate`/`CategoryUpdate` (`Field(None, ge=0)`) + `CheckConstraint` de BD | US1 |
| Dejar el campo vacío al crear asigna `MAX(display_order) + 1` | `create_category`, `categories/router.py` | US1, Edge Cases |
| El Menú QR muestra las categorías activas ordenadas por `display_order DESC, name ASC` | `_build_menu`, `menu/router.py:97` | US2 |
| Categorías inactivas siguen excluidas del filtro sin importar su orden | `.where(Category.active.is_(True))`, ya existente, sin cambio | US2, Edge Cases |
| El listado de administración muestra el valor de orden de cada categoría | `CategoryResponse.display_order`; nueva columna en `categories-page.component.ts` | US3 |
| Categorías creadas antes del feature no cambian de posición al desplegar | Migración, backfill `ROW_NUMBER() OVER (ORDER BY name DESC)` | Edge Cases, FR-009 |
