# Contrato: API pública de `app/catalog_engine/` y fachadas

Esta spec no expone ningún endpoint HTTP nuevo ni modifica `router.py`/`schemas.py` — el
"contrato" aquí es la superficie de importación Python que el resto de `pos-backend` (siete
ficheros consumidores + `app/characterization_tests/`) consume del motor de catálogo. Es un
contrato de **estabilidad de símbolos** (nombre, firma, ubicación de import), no de wire format.

## Contrato A — API pública de `app/catalog_engine/`

`app/catalog_engine/__init__.py` DEBE exponer, importables directamente como
`from app.catalog_engine import <símbolo>`, exactamente estos catorce símbolos (trece funciones +
un dataclass), con la misma firma que tienen hoy en el código legado (FR-001, FR-002):

```text
# Núcleo puro (re-expuesto desde core.py)
ConsumptionLine            # dataclass(frozen=True): inventory_item_id, quantity, source
compute_line_price(variant: ProductVariant, options: list[Option]) -> Decimal
_exige_maximo(gid: UUID, lo: int, consumen: set[UUID]) -> bool   # privada; no se garantiza
                                                                   # estable para consumidores
                                                                   # externos al paquete, pero
                                                                   # debe existir y comportarse
                                                                   # igual (Acceptance Scenario 3)

# Adaptadores — grupo pricing (re-expuestos desde pricing.py)
load_valid_options(db: Session, option_ids: list[UUID], *, variant: ProductVariant | None = None) -> list[Option]
grupos_que_descuentan(db: Session, links: Sequence[VariantOptionGroup]) -> set[UUID]
validate_option_selection(db: Session, variant: ProductVariant, options: Sequence[Option]) -> None
check_availability(db: Session, required: dict[UUID, Decimal], *, extra_context: str = "") -> None

# Adaptadores — grupo consumption (re-expuestos desde consumption.py)
load_recipe(db: Session, variant_id: UUID) -> list[RecipeItem]
load_variant_groups(db: Session, variant_id: UUID) -> list[VariantOptionGroup]
group_discounts(db: Session, link: VariantOptionGroup) -> bool
plan_line_consumption(db: Session, variant_id: UUID, quantity: int, options: Sequence[Option]) -> list[ConsumptionLine]
required_consumption(db: Session, variant_id: UUID, quantity: int, options: Sequence[Option]) -> dict[UUID, Decimal]
variant_label(db: Session, variant_id: UUID) -> str
ensure_lines_consume_inventory(db: Session, entries: Sequence[tuple[UUID, int, Sequence[Option]]]) -> None
```

**Verificación** (Acceptance Scenarios 3 y 4 de la Historia 1):
- `compute_line_price` y `_exige_maximo`: sin `sqlalchemy` en su import chain, sin parámetro
  `db: Session`.
- Las once restantes: reciben `db: Session` en la misma posición (primer parámetro posicional)
  que en el código legado.

## Contrato B — Fachadas post-conmutación (Historia 3)

Tras la Historia 3, `app/api/v1/catalog/line_pricing.py` y `consumption_plan.py` DEBEN seguir
exponiendo, desde exactamente la misma ruta de import que hoy, los mismos símbolos:

```text
# app.api.v1.catalog.line_pricing  (fachada de app.catalog_engine)
compute_line_price, load_valid_options, check_availability,
validate_option_selection, grupos_que_descuentan, _exige_maximo
# + reexport que ya hace hoy (line_pricing.py:31-36), preservado (FR-009):
ConsumptionLine, load_variant_groups, plan_line_consumption, required_consumption

# app.api.v1.catalog.consumption_plan  (fachada de app.catalog_engine)
ConsumptionLine, load_recipe, load_variant_groups, plan_line_consumption,
required_consumption, group_discounts, variant_label, ensure_lines_consume_inventory
```

**Regla de forma**: cada símbolo se reexporta como import directo (`from app.catalog_engine.core
import compute_line_price as compute_line_price`, o equivalente vía el `__init__.py` del
paquete) — nunca como una función wrapper que reimplementa la firma y delega. FR-008 exige que la
fachada "no contenga lógica propia de cálculo, validación o consulta"; un wrapper cuenta como
lógica propia (aunque sea un solo `return`), un reexport de nombre no.

**Consumidores que dependen de este contrato exacto** (ninguno cambia, FR-010/SC-004):

| Fichero | Símbolos que importa | Ruta de import (sin cambios) |
|---|---|---|
| `sales/service.py:23` | `compute_line_price`, `load_valid_options` | `app.api.v1.catalog.line_pricing` |
| `sales/consumption.py:17-21` | `ensure_lines_consume_inventory`, `plan_line_consumption`, `required_consumption` | `app.api.v1.catalog.consumption_plan` |
| `orders/service.py:28` | `compute_line_price`, `load_valid_options` | `app.api.v1.catalog.line_pricing` |
| `orders/consolidation.py:28` | `compute_line_price`, `load_valid_options` | `app.api.v1.catalog.line_pricing` |
| `orders/kitchen.py:22` | `compute_line_price`, `load_valid_options` | `app.api.v1.catalog.line_pricing` |
| `orders/consumption.py:19-23` | `ensure_lines_consume_inventory`, `plan_line_consumption`, `required_consumption` | `app.api.v1.catalog.consumption_plan` |
| `cart/service.py:31-36` | `check_availability`, `compute_line_price`, `load_valid_options`, `required_consumption` (este último vía el reexport de `line_pricing.py:31-36`) | `app.api.v1.catalog.line_pricing` |

## Contrato C — Consumidores de test (Historia 1)

Los mismos 41 characterization tests y el golden master cambian **únicamente** su import (de
`app.api.v1.catalog.*` a `app.catalog_engine`) para verificar la Historia 1 de forma aislada,
antes de que exista ninguna fachada — sin modificar ni una aserción (FR-011, FR-012). Esta es la
única excepción documentada a "los siete consumidores no cambian": los tests no son uno de los
siete ficheros consumidores de producción, son el árbitro (Constitución, Principio II).

## Fuera de este contrato

- `router.py`, `schemas.py`, `service.py` (SKU) de `app/api/v1/catalog/`: no exponen símbolos del
  motor de precio/consumo, no se tocan.
- Cualquier endpoint HTTP: esta spec no añade, quita ni modifica rutas — el motor siempre se
  consumió como librería interna, nunca como servicio propio.
