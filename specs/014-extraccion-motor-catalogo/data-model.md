# Phase 1 Data Model: Extracción del motor de catálogo

Esta spec no introduce ni modifica ninguna tabla, modelo ORM ni esquema de base de datos —
FR-004 exige explícitamente que las once funciones adaptadoras "conserven su interacción con los
modelos [...] sin alterar qué consultan ni cómo". Las "entidades" relevantes aquí son de código,
no de datos: los Key Entities que ya define `spec.md` (Núcleo puro, Adaptadores de I/O, Contrato
de comportamiento, Batería comparativa), detallados a nivel de función/fichero para guiar la
extracción.

## Entidad: Núcleo puro del motor (`app/catalog_engine/core.py`)

Funciones y tipo que no reciben `db: Session`, no importan `sqlalchemy`, y calculan un resultado
únicamente a partir de sus parámetros de entrada.

| Símbolo | Firma (sin cambios) | Origen legado | Comportamiento a preservar |
|---|---|---|---|
| `compute_line_price` | `(variant: ProductVariant, options: list[Option]) -> Decimal` | `line_pricing.py:191-196` | Precio de variante + suma de `extra_price` de cada opción. Sin redondeo explícito (A-36 punto 1, `PENDIENTE` — se reproduce tal cual, FR-006). |
| `_exige_maximo` | `(gid: UUID, lo: int, consumen: set[UUID]) -> bool` | `line_pricing.py:94-105` | `True` solo si `lo > 0` y `gid` está en `consumen` (el grupo descuenta inventario y es obligatorio). Helper privado — sigue sin exportarse en `__init__.py` salvo lo que ya exportaba `line_pricing.py` hoy (no lo exportaba; solo se llama internamente). |
| `ConsumptionLine` | `@dataclass(frozen=True)` con campos `inventory_item_id: UUID`, `quantity: Decimal`, `source: str` | `consumption_plan.py:49-56` | Inmutable (`frozen=True`); `source` sigue siendo uno de `'receta' \| 'variante' \| 'opcion'` (no se valida con `Literal`/enum hoy — no se introduce validación nueva). |

**Invariante de fichero**: `grep -n sqlalchemy app/catalog_engine/core.py` debe devolver vacío
(SC-006). Los únicos imports de `core.py` son de la biblioteca estándar (`dataclasses`,
`decimal`, `uuid`) y de los modelos ORM que aparecen solo como *type hints* de parámetros ya
cargados (`ProductVariant`, `Option`) — verificar que ese import de tipos no arrastre
transitivamente `sqlalchemy` en tiempo de import (los modelos de `app/models/*.py` sí importan
`sqlalchemy`, así que si se necesita el tipo para anotación, evaluar `from __future__ import
annotations` + import bajo `TYPE_CHECKING`, ya que el código legado (`line_pricing.py:13`) ya usa
`from __future__ import annotations`).

## Entidad: Adaptadores de I/O — grupo `pricing` (`app/catalog_engine/pricing.py`)

Envuelven una consulta o validación contra `db: Session` y delegan el cálculo puro en el núcleo
cuando aplica. Provienen de `line_pricing.py`.

| Símbolo | Firma (sin cambios) | Origen legado | Anomalía(s) que reproduce tal cual |
|---|---|---|---|
| `load_valid_options` | `(db: Session, option_ids: list[UUID], *, variant: ProductVariant \| None = None) -> list[Option]` | `line_pricing.py:43-66` | — |
| `grupos_que_descuentan` | `(db: Session, links: Sequence[VariantOptionGroup]) -> set[UUID]` | `line_pricing.py:69-91` | A-32 (criterio distinto al de `group_discounts` de `consumption.py`) |
| `validate_option_selection` | `(db: Session, variant: ProductVariant, options: Sequence[Option]) -> None` | `line_pricing.py:108-188` | A-05 (`STRICT_OPTION_SELECTION=False` por defecto, `[PROTEGIDA]`), A-06 (tolerancia de opción de grupo ajeno) |
| `check_availability` | `(db: Session, required: dict[UUID, Decimal], *, extra_context: str = "") -> None` | `line_pricing.py:199-220` | A-47 parcial (best-effort, sin reserva ni bloqueo) |

Dependen de los modelos `Option`, `OptionGroup`, `ProductVariant`, `InventoryItem`,
`VariantOptionGroup` y de `app.core.config.settings` / `app.core.crud.get_or_404` — mismas
dependencias que hoy, sin alterar qué consultan (FR-004).

## Entidad: Adaptadores de I/O — grupo `consumption` (`app/catalog_engine/consumption.py`)

Provienen de `consumption_plan.py`.

| Símbolo | Firma (sin cambios) | Origen legado | Anomalía(s) que reproduce tal cual |
|---|---|---|---|
| `load_recipe` | `(db: Session, variant_id: UUID) -> list[RecipeItem]` | `consumption_plan.py:59-65` | — |
| `load_variant_groups` | `(db: Session, variant_id: UUID) -> list[VariantOptionGroup]` | `consumption_plan.py:68-76` | — |
| `group_discounts` | `(db: Session, link: VariantOptionGroup) -> bool` | `consumption_plan.py:79-95` | A-32 (criterio distinto al de `grupos_que_descuentan` de `pricing.py`) |
| `plan_line_consumption` | `(db: Session, variant_id: UUID, quantity: int, options: Sequence[Option]) -> list[ConsumptionLine]` | `consumption_plan.py:98-140` | A-02 `[PROTEGIDA]` (el tamaño manda sobre la opción, nunca se suman) |
| `required_consumption` | `(db: Session, variant_id: UUID, quantity: int, options: Sequence[Option]) -> dict[UUID, Decimal]` | `consumption_plan.py:143-152` | Hereda A-02 vía `plan_line_consumption` |
| `variant_label` | `(db: Session, variant_id: UUID) -> str` | `consumption_plan.py:155-162` | — |
| `ensure_lines_consume_inventory` | `(db: Session, entries: Sequence[tuple[UUID, int, Sequence[Option]]]) -> None` | `consumption_plan.py:165-227` | A-33 (bloqueo 409 si un grupo opcional es la única fuente de consumo y nadie elige nada) |

Dependen de los modelos `Option`, `Product`, `ProductVariant`, `RecipeItem`,
`VariantOptionGroup` — mismas dependencias que hoy.

**Import interno nuevo**: `plan_line_consumption` y `required_consumption` construyen
`ConsumptionLine`, que ahora vive en `core.py` — `consumption.py` importa `ConsumptionLine` desde
`app.catalog_engine.core` (o desde `app.catalog_engine` si se resuelve vía el `__init__.py`,
evitando cualquier ciclo porque `core.py` no importa nada de `consumption.py`/`pricing.py`).

## Entidad: Contrato de comportamiento

El conjunto que arbitra si la extracción es equivalente (Constitución, Principio II). No es una
entidad de datos nueva — son artefactos ya existentes en `pos-backend`, sin modificar:

- 41 characterization tests: `test_catalog_line_pricing.py` (25) + `test_catalog_consumption_plan.py` (16).
- 1 golden master: `golden_master_core.py` + `test_golden_master_pricing_consumption.py` +
  `golden_master/pricing_consumption.master.json` (8-12 casos encadenados).
- Entradas citadas del registro de anomalías: A-02, A-03, A-05, A-06, A-32, A-33, A-36 (parcial),
  A-47 (parcial) — como referencia normativa de qué comportamiento es intencional, no como
  artefacto ejecutable.

## Entidad: Batería comparativa (Historia 2, temporal)

Vive en `app/characterization_tests/catalog_engine_equivalence_gate.py`. No persiste estado más
allá de la corrida del proceso de test — es un generador + comparador, no un fixture de datos
nuevo.

| Campo del reporte por caso | Descripción |
|---|---|
| `caso_id` | Índice determinista (0..N-1) dentro de la corrida, dado la semilla fija |
| `entrada` | Combinación generada: variante, opciones elegidas, cantidad, y el estado de stock del fixture usado |
| `legado` | Salida (`ok`, valor) o (`error`, `{status_code, detail}`) de la función ejecutada contra `app/api/v1/catalog/*` |
| `nuevo` | Misma forma, ejecutada contra `app/catalog_engine/*` |
| `campo_divergente` | Solo si `legado != nuevo`: ruta del campo que difiere (mismo estilo que `_diff_paths` de `test_golden_master_pricing_consumption.py`) |

**Ciclo de vida**: se crea en la Historia 2, se ejecuta como gate antes de la Historia 3, y se
retira o archiva una vez la conmutación queda verificada (Clarifications, sesión 2026-08-17) — no
forma parte de la red de regresión permanente, que queda cubierta por los characterization tests y
el golden master (FR-013).

## Transiciones de estado

No aplica en sentido de máquina de estados de negocio — la única "transición" relevante es la del
propio código a través de las tres historias:

1. **Historia 1**: `app/catalog_engine/` existe y pasa los 41 tests + golden master apuntando a
   él; `line_pricing.py`/`consumption_plan.py` siguen siendo la implementación real que usan los
   siete consumidores (el paquete nuevo existe en paralelo, sin nada que dependa de él todavía).
2. **Historia 2**: gate de equivalencia comparativa en verde (cero diferencias campo a campo).
3. **Historia 3**: `line_pricing.py`/`consumption_plan.py` pasan a ser fachadas puras de
   `app/catalog_engine/`; los siete consumidores, sin cambiar una línea, empiezan a ejecutar de
   hecho el código nuevo por transitividad de import.
