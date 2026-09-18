# Contrato: Bloqueo de "Configurar" en promociones activas (bug 4)

Cubre spec.md FR-008 a FR-011. Cita anomalía **A-76** (research.md D7) — debe existir antes de
mergear.

## Backend

**Sin cambio de contrato de API** (mismo request/response de `PATCH /promotions/{id}`).
`PromotionResponse.status` (`app/api/v1/promotions/schemas.py`) ya viaja con los 4 valores reales
(`draft`/`active`/`paused`/`finished`); el listado del frontend ya recibe todo lo necesario para el
`[disabled]` de abajo.

**Cambio de comportamiento confirmado (research.md D9)**: `PATCH /promotions/{promotion_id}`
(`app/api/v1/promotions/router.py:81`) llama a `service.update()`
(`app/api/v1/promotions/service.py:762-778`), que edita `name`/`description`/`ends_at`/
`days_of_week`/`start_time`/`end_time` **sin ninguna guarda de `status` hoy**. Se agrega, al inicio
de esa función:

```python
def update(db: Session, promo: Promotion, data) -> Promotion:
    if promo.status == "active":
        raise HTTPException(status.HTTP_409_CONFLICT,
                             "Pausa la promoción antes de modificarla")
    ...
```

**No** se reutiliza la condición de `update_shape` (`status not in ("draft", "paused")`) — esa
condición también bloquearía `finished`, y FR-009 exige que "Configurar" siga editable en
`Borrador`, `Pausada` **y `Finalizada`**. Solo `active` se bloquea aquí.

**Tests de characterization a actualizar** (Principio III, cita **A-76**): esta guarda rompe, tal
como están escritos hoy, `test_ca1_editar_escalares_de_una_activa` y
`test_editar_vigencia_de_promocion_multi_regla_afecta_a_todas_con_una_accion`
(`app/characterization_tests/test_promotions_rules_admin.py`, ambos llaman `service.update()`
sobre una promoción `active` y esperan éxito) — deben reescribirse para esperar `HTTPException`
409, citando A-76 en el mismo commit. Ningún otro test del archivo depende de `service.update()`
en estado `active` (confirmado en research.md D9).

## Frontend (`promotions-page.component.ts`)

**Botón "Configurar"** (línea ~317-323):

```html
<button
  [disabled]="p.status === 'active'"
  [title]="p.status === 'active' ? 'Pausa la promoción para poder configurarla' : ''"
  (click)="openEdit(p); closeActionsMenu()">
  Configurar
</button>
```

Mismo criterio que `canDelete(p)` (línea ~1458-1460, `return p.status !== 'active'`) — estado real
(`status`), no el badge visual derivado (`displayOf`/`PromoDisplay`).

**`openEdit(p)`** (línea ~1528): agrega una guarda de defensa en profundidad —
`if (p.status === 'active') { return; }` al inicio — por si se invoca desde otro punto además del
botón (p. ej. navegación directa por ruta con el id de una promoción `active`).

**Al pausar** (`changeStatus(id, 'paused')`, servicio ya existente): tras la respuesta exitosa, la
fila actualizada en el signal de la lista refleja `status='paused'` de inmediato (patrón ya usado
por el resto de las acciones de la tabla) — el botón "Configurar" queda habilitado sin recargar la
página (spec.md FR-010).

## Casos de aceptación cubiertos

- `status='active'` → botón deshabilitado, `openEdit` no abre nada aunque se fuerce el clic.
- `status` en `draft`/`paused`/`finished` → botón habilitado, comportamiento idéntico a hoy.
- Pausar desde el listado → botón se habilita sin recargar.
- Badge visual "Vencida"/"Fuera de horario" con `status='active'` real → sigue deshabilitado (el
  criterio es `status`, no el badge).
