# Contrato: API pública de `app/inventory_engine/` y fachada

Esta spec no expone ningún endpoint HTTP nuevo ni modifica `router.py`/`schemas.py` — el
"contrato" aquí es la superficie de importación Python que el resto de `pos-backend` (cuatro
ficheros consumidores + `app/characterization_tests/`) consume del motor de stock. Es un contrato
de **estabilidad de símbolos** (nombre, firma, ubicación de import), no de wire format.

## Contrato A — API pública de `app/inventory_engine/`

`app/inventory_engine/__init__.py` DEBE exponer, importables directamente como
`from app.inventory_engine import <símbolo>`, exactamente estos tres símbolos, con la misma firma
que tienen hoy en el código legado (FR-001):

```text
lock_items(db: Session, item_ids: Iterable[UUID]) -> dict[UUID, InventoryItem]

record_movement(
    db: Session,
    inventory_item_id: UUID,
    *,
    type: str,
    quantity: Decimal,
    reason: Optional[str] = None,
    reference_type: Optional[str] = None,
    reference_id: Optional[UUID] = None,
    user_id: Optional[UUID] = None,
    allow_negative: bool = False,
) -> InventoryMovement

apply_adjustment(
    db: Session,
    inventory_item_id: UUID,
    *,
    signed_delta: Decimal,
    reason: Optional[str] = None,
    user_id: Optional[UUID] = None,
) -> InventoryMovement
```

**Verificación** (Acceptance Scenarios 2-3 de la Historia 1):
- Las tres funciones reciben `db: Session` como primer parámetro posicional, sin excepción — no
  hay función pura en este contrato (a diferencia de `catalog_engine`, ver research.md Decisión 2).
- `.with_for_update()` aparece exactamente una vez por función, en el mismo punto lógico del flujo
  que en el fichero legado (verificable por inspección estática, SC-006).
- `record_movement` sigue rechazando `type="adjustment"` con `ValueError` (Acceptance Scenario 4).
- `apply_adjustment` sigue propagando `ValueError` sin capturar ante `signed_delta=0` (Acceptance
  Scenario 5, A-35 sub-hallazgo 3).

## Contrato B — Fachada post-conmutación (Historia 3)

Tras la Historia 3, `app/api/v1/inventory/stock.py` DEBE seguir exponiendo, desde exactamente la
misma ruta de import que hoy, los mismos tres símbolos:

```text
# app.api.v1.inventory.stock  (fachada de app.inventory_engine)
lock_items, record_movement, apply_adjustment
```

**Regla de forma**: cada símbolo se reexporta como import directo (`from app.inventory_engine
import lock_items as lock_items`, o equivalente vía el `__init__.py` del paquete) — nunca como una
función wrapper que reimplementa la firma y delega. FR-005 exige que la fachada "no contenga
lógica propia de cálculo, validación o consulta"; un wrapper cuenta como lógica propia (aunque sea
un solo `return`), un reexport de nombre no.

**Consumidores que dependen de este contrato exacto** (ninguno cambia salvo el ajuste mínimo de
import de `inventory/service.py`, FR-006/FR-008/SC-004):

| Fichero | Símbolos que importa | Ruta de import (sin cambios, salvo ajuste mínimo permitido) |
|---|---|---|
| `inventory/router.py:18` | `apply_adjustment` | `app.api.v1.inventory.stock` |
| `inventory/service.py:17` | `record_movement` | `app.api.v1.inventory.stock` — único fichero donde FR-008 permite ajustar el import (directo a `app.inventory_engine` o vía la fachada), sin ningún otro cambio |
| `sales/consumption.py:16` | `lock_items`, `record_movement` | `app.api.v1.inventory.stock` |
| `orders/consumption.py:18` | `lock_items`, `record_movement` | `app.api.v1.inventory.stock` |

## Contrato C — Consumidores de test (Historia 1)

Los mismos 16 characterization tests cambian **únicamente** su import (de
`app.api.v1.inventory.stock` a `app.inventory_engine`) para verificar la Historia 1 de forma
aislada, antes de que exista ninguna fachada — sin modificar ni una aserción (FR-007). Esta es la
única excepción documentada a "los cuatro consumidores no cambian": los tests no son uno de los
cuatro ficheros consumidores de producción, son el árbitro (Constitución, Principio II).

## Fuera de este contrato

- `router.py`, `schemas.py` de `app/api/v1/inventory/`: no se tocan (`router.py` solo consume
  `apply_adjustment`, sin cambiar su propia superficie).
- `inventory/service.py` más allá del import de `record_movement`: su propia extracción queda
  fuera de alcance (FR-008, Assumptions de `spec.md` — requiere spec de prerrequisito aparte por no
  tener characterization tests confirmados).
- Cualquier endpoint HTTP: esta spec no añade, quita ni modifica rutas — el motor siempre se
  consumió como librería interna, nunca como servicio propio.
- Golden master de inventario: no se construye en esta spec (research.md Decisión 4); no hay
  contrato de fichero JSON de golden master que mantener.
