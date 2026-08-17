# Phase 1 Data Model: Extracción del motor de stock de inventario

Esta spec no introduce ni modifica ninguna tabla, modelo ORM ni esquema de base de datos — las
tres funciones movidas conservan exactamente su interacción con `InventoryItem`/`InventoryMovement`
(FR-001, FR-002). Las "entidades" relevantes aquí son de código, no de datos: los Key Entities que
ya define `spec.md` (Motor de stock, Contrato de comportamiento, Batería comparativa), detallados a
nivel de función para guiar la extracción.

## Entidad: Motor de stock (`app/inventory_engine/stock.py`)

Las tres funciones, todas impuras (`db: Session`, `.with_for_update()`) — no hay frontera
núcleo/adaptador en esta spec (ver research.md Decisión 2).

| Símbolo | Firma (sin cambios) | Origen legado | Comportamiento a preservar |
|---|---|---|---|
| `lock_items` | `(db: Session, item_ids: Iterable[UUID]) -> dict[UUID, InventoryItem]` | `stock.py:17-36` | Deduplica (`sorted(set(item_ids))`) y bloquea en una sola query, ordenado por `id` (orden canónico anti-deadlock, citado en A-17). Lista vacía devuelve `{}` sin consultar `db`. |
| `record_movement` | `(db: Session, inventory_item_id: UUID, *, type: str, quantity: Decimal, reason: Optional[str] = None, reference_type: Optional[str] = None, reference_id: Optional[UUID] = None, user_id: Optional[UUID] = None, allow_negative: bool = False) -> InventoryMovement` | `stock.py:39-89` | Rechaza `quantity <= 0` y `type` fuera de `("in", "out")` con `ValueError`; bloquea la fila (`.with_for_update()`), calcula `delta` con signo, rechaza con `InsufficientStockError` si `new_stock < 0` salvo `allow_negative=True` (A-35 sub-hallazgo 1, sin llamador visible, preservado), aplica `current_stock` e inserta `InventoryMovement`. |
| `apply_adjustment` | `(db: Session, inventory_item_id: UUID, *, signed_delta: Decimal, reason: Optional[str] = None, user_id: Optional[UUID] = None) -> InventoryMovement` | `stock.py:92-121` | Rechaza `signed_delta == 0` con `ValueError` sin capturar (A-35 sub-hallazgo 3, `ACCIDENTAL`, no se corrige aquí); bloquea la fila, suma `signed_delta`, rechaza con `InsufficientStockError` si quedaría negativo (sin `allow_negative`, nunca permite negativo); inserta `InventoryMovement` con `type="adjustment"`, `quantity=abs(signed_delta)`; `reason` sigue sin ser obligatorio (A-35 sub-hallazgo 2, preservado). |

**Dependencias del módulo** (sin cambios respecto al fichero legado, verificado —
`stock.py` no importa nada de `app/api/v1/`): `sqlalchemy.select`, `sqlalchemy.orm.Session`,
`app.core.exceptions.InsufficientStockError`, `app.models.inventory_item.InventoryItem`,
`app.models.inventory_movement.InventoryMovement`.

## Modelos ORM consultados (sin modificar — referencia)

Documentados aquí solo como referencia de los campos que las funciones leen/escriben; ningún campo,
constraint ni migración cambia en esta spec.

- **`InventoryItem`** (`app/models/inventory_item.py`, `tenant.inventory_items`): `current_stock`
  (`Numeric(12,3)`, mutado por las tres funciones), `name` (usado en los mensajes de error de
  `InsufficientStockError`); resto de campos (`unit_measure_id`, `type`, `min_stock`, `unit_cost`,
  `active`) no los toca el motor de stock.
- **`InventoryMovement`** (`app/models/inventory_movement.py`, `tenant.inventory_movements`):
  `inventory_item_id`, `type` (constraint `IN ('in','out','adjustment')`), `quantity` (constraint
  `> 0` — por eso `apply_adjustment` inserta `abs(signed_delta)`, nunca el valor con signo),
  `reason`, `reference_type`, `reference_id`, `user_id`, `moved_at` (`server_default=func.now()`).
- **`InsufficientStockError`** (`app.core.exceptions`): subclase de `HTTPException` (400) — no es
  parte de los modelos ORM, pero es la excepción de dominio que ambas funciones mutadoras lanzan;
  se importa y se relanza exactamente igual, sin envolver.

## Entidad: Contrato de comportamiento

El conjunto que arbitra si la extracción es equivalente (Constitución, Principio II). No es una
entidad de datos nueva — son artefactos ya existentes en `pos-backend`, sin modificar, más la
revisión manual nueva de esta fase:

- 16 characterization tests: `test_inventory_stock.py` (`RecordMovementTests`: 7,
  `ApplyAdjustmentTests`: 6, `LockItemsTests`: 3).
- Sin golden master nuevo (research.md Decisión 4) — sustituido por la revisión manual documentada
  de los tres sub-hallazgos de A-35 en alcance, ya registrada en research.md Decisión 4 (satisface
  FR-009 opción b y SC-002).
- Entradas citadas del registro de anomalías: A-17 (referencia positiva del patrón de bloqueo),
  A-35 sub-hallazgos 1-3 (en alcance) — como referencia normativa de qué comportamiento es
  intencional, no como artefacto ejecutable.

## Entidad: Batería comparativa (Historia 2, temporal)

Vive en `app/characterization_tests/inventory_engine_equivalence_gate.py`. No persiste estado más
allá de la corrida del proceso de test — es un generador + comparador, no un fixture de datos
nuevo. Mismo formato de reporte que `catalog_engine_equivalence_gate.py` de la spec 014.

| Campo del reporte por caso | Descripción |
|---|---|
| `caso_id` | Índice determinista (0..N-1) dentro de la corrida, dado la semilla fija |
| `entrada` | Combinación generada: función a ejercitar (`record_movement`/`apply_adjustment`), tipo de movimiento o `signed_delta`, cantidad, `allow_negative`, y el `current_stock` inicial del insumo de fixture usado (incluyendo casos en y cerca de cero) |
| `legado` | Salida (`ok`, `{current_stock resultante, campos del InventoryMovement}`) o (`error`, `{tipo de excepción, mensaje}`) de la función ejecutada contra `app.api.v1.inventory.stock` |
| `nuevo` | Misma forma, ejecutada contra `app.inventory_engine` |
| `campo_divergente` | Solo si `legado != nuevo`: ruta del campo que difiere, incluyendo tipo/mensaje de excepción si divergen (Assumptions de `spec.md`: una diferencia de mensaje de error cuenta como diferencia) |

**Ciclo de vida**: se crea en la Historia 2, se ejecuta como gate antes de la Historia 3, y se
retira o archiva una vez la conmutación queda verificada (Edge Cases de `spec.md`) — no forma parte
de la red de regresión permanente, que queda cubierta por los 16 characterization tests (FR-007).

## Transiciones de estado

No aplica en sentido de máquina de estados de negocio — la única "transición" relevante es la del
propio código a través de las tres historias:

1. **Historia 1**: `app/inventory_engine/` existe y pasa los 16 tests apuntando a él;
   `app/api/v1/inventory/stock.py` sigue siendo la implementación real que usan los cuatro
   consumidores (el paquete nuevo existe en paralelo, sin nada que dependa de él todavía).
2. **Historia 2**: gate de equivalencia comparativa en verde (cero diferencias campo a campo);
   FR-009 ya resuelto por la revisión manual de research.md Decisión 4.
3. **Historia 3**: `app/api/v1/inventory/stock.py` pasa a ser fachada pura de
   `app.inventory_engine`; los cuatro consumidores, sin cambiar una línea salvo el ajuste mínimo de
   import en `inventory/service.py` (FR-008), empiezan a ejecutar de hecho el código nuevo por
   transitividad de import.
