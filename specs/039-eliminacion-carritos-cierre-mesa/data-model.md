# Data Model: Eliminación de Carritos al Liberarse la Mesa

Todas las entidades de esta spec viven en el schema **`tenant`** (por-tenant, vía
`@for_each_tenant_schema`). Las decisiones de diseño detrás de cada elección están en
[research.md](./research.md); este documento se limita a columnas, restricciones y transiciones.
**Ninguna tabla cambia de esquema en esta spec** (Principio VIII no aplica — no hay migración).

## Cart (`carts`) — SIN CAMBIO DE ESQUEMA

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id` | UUID (PK) | No | `UUIDPrimaryKeyMixin`. |
| `participant_id` | UUID (FK → `session_participants.id`) | No | Indexado. **Clave de selección del `DELETE` masivo de esta spec** (research.md Decisión 2). |
| `status` | `String(12)` | No | `'abierto'` \| `'confirmado'` \| `'abandonado'`. **Sin cambio en el `CheckConstraint`.** |
| `created_at` | `DateTime` | No | `server_default now()`. |

**Lo único que cambia es *cuándo desaparece la fila*, no su forma**: hoy, una fila de `Cart` de un
participante cuya sesión se cierra sobrevive para siempre (con `status='abandonado'` si estaba
`'abierto'`, o con el `status` que ya tuviera si era `'confirmado'`). A partir de esta spec, en el
instante en que la `DiningTable` de esa sesión pasa a `libre` en la misma operación, la fila se
**elimina físicamente** — sin importar en qué `status` estuviera (FR-001, edge case de spec.md:
"la eliminación no distingue por `status`").

**Restricción explotada, no modificada**: el índice único parcial `idx_open_cart_per_participant`
(`participant_id` `WHERE status = 'abierto'`) sigue garantizando un solo carrito `'abierto'` por
comensal — esta spec no lo toca. Un comensal cuya mesa se liberó y cuyo `Cart` fue borrado no tiene
ninguna fila (ni `'abierto'` ni de otro `status`); si esa misma persona (mismo `participant_id`)
llegara a operar de nuevo — no debería ser posible tras cerrar su sesión, pero el índice ni siquiera
entraría en conflicto porque no queda ninguna fila previa que choque.

## CartItem (`cart_items`) / CartItemOption (`cart_item_options`) — SIN CAMBIO DE ESQUEMA

Sin columnas nuevas, sin restricciones nuevas. Relevante para el borrado masivo de esta spec:
`cart_items.cart_id` tiene `ondelete="CASCADE"` (`app/models/cart_item.py:19`) y
`cart_item_options.cart_item_id` también (`cart_item.py:58`) — ambos ya existían antes de esta spec
(la spec 038 ya dependía del mismo mecanismo para su propio borrado físico vía `db.delete(cart)`).
Es lo que hace que un `DELETE FROM carts WHERE participant_id IN (...)` (research.md Decisión 2)
borre también, a nivel de base de datos, sus `CartItem`/`CartItemOption` sin que el código de esta
spec necesite tocarlos explícitamente ni cargarlos en memoria.

## SessionParticipant (`session_participants`) — SIN CAMBIO DE ESQUEMA

Su `id` es la clave que conecta "sesión que se cerró" con "carritos que se borran": la función
nueva de esta spec (`delete_orphan_carts`) resuelve, para cada `TableSession` que
`close_table_sessions` devolvió como cerrada, los `SessionParticipant.id` de esa sesión
(`table_session_id`, sin filtrar por `status` del participante — un comensal ya `closed` antes de
esta operación también cuenta, edge case "carrito huérfano de un comensal ya cerrado" de spec.md),
y borra los `Cart` cuyo `participant_id` esté en ese conjunto.

## TableSession (`table_sessions`) / DiningTable (`dining_tables`) — SIN CAMBIO DE ESQUEMA

`TableSession.status = 'closed'` sigue siendo condición **necesaria pero no suficiente** para que
se borren los carritos de sus participantes: la condición **suficiente**, añadida por esta spec, es
que la `DiningTable` asociada pase a `status = 'libre'` **en esa misma operación** (FR-003). No hay
ninguna columna nueva que registre "se le borraron los carritos" — el borrado es un efecto de
aplicación sin rastro en el esquema, igual que ya lo es el borrado físico de spec 038 sobre
`submit_cart`.

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Cuando la mesa pasa a `libre` en la misma operación que cierra sus sesiones, se borran físicamente todos los `Cart` de los participantes de esas sesiones, sin importar su `status` | `checkout.delete_orphan_carts`, invocada en los 5 call-sites (research.md D1) | US1, US2 |
| El borrado ocurre en la misma transacción que libera la mesa — si esa operación falla y hace `rollback()`, ningún `Cart` queda borrado ni modificado | Los 5 call-sites llaman `delete_orphan_carts` antes de su único `db.commit()`; ningún `commit()` intermedio se introduce | US1, US2, US3 (rollback) |
| Si la mesa NO pasa a `libre` (queda algo por cobrar, o un `CustomerOrder` huérfano de la misma mesa física), no se borra ningún `Cart` | `_sweep_schema`: `delete_orphan_carts` solo se llama dentro de `if quedo_libre:`; la rama RN-SCHED-03 (`has_billable_orders`) ni siquiera llama `close_table_sessions` | US3 |
| El borrado alcanza únicamente los participantes de la(s) sesión(es) que efectivamente pasaron a `closed` en esa liberación — ninguna otra sesión, mesa o tenant se ve afectado | `delete_orphan_carts(db, sessions)` recibe exactamente el `list[TableSession]` que `close_table_sessions` devolvió para esa mesa; el schema-per-tenant ya aísla tenants | US4 |
| Las condiciones que hoy deciden si una mesa pasa o no a `libre` no cambian | Ningún `if`/`return` de los 5 caminos se modifica — solo se agrega una llamada nueva en la rama donde la mesa ya queda `libre` | Todas (FR-005) |

## Ciclo de vida de `Cart` al liberarse la mesa (antes / después de esta spec)

```text
Antes de esta spec:                              Después de esta spec (039):

  Comensal sale / mesa se cobra / se libera        Comensal sale / mesa se cobra / se libera
  (cualquiera de los 5 caminos)                    (cualquiera de los 5 caminos)
       │                                                │
       ▼                                                ▼
  close_participants:                              close_participants:  (SIN CAMBIO)
    Cart 'abierto' → 'abandonado'                     Cart 'abierto' → 'abandonado'
  Cart 'confirmado' (huérfano): sin tocar            Cart 'confirmado' (huérfano): sin tocar
       │                                                │
       ▼                                                ▼
  ¿La mesa queda 'libre'?                          ¿La mesa queda 'libre'?
       │                                                │
   ┌───┴───┐                                        ┌───┴───┐
   sí       no                                      sí       no
   │         │                                      │         │
   ▼         ▼                                      ▼         ▼
 (nada      (nada                              delete_orphan_carts:   (nada
  más)       más —                               TODOS los Cart de     más —
             carrito                              los participantes    carrito
             sigue                                de las sesiones      sigue
             'abandonado'                          cerradas se         'abandonado'
             para                                  ELIMINAN, sin       para
             siempre)                              importar su         siempre —
                                                    status              FR-003,
                                                                        sin cambio)
```

`submit_cart` (spec 038, borrado al confirmar con éxito) y este ciclo (borrado al liberar la mesa)
son las dos únicas transiciones que hacen desaparecer una fila de `Cart` en todo el sistema tras
esta spec — entre ambas cubren, respectivamente, "el pedido se confirmó" y "la sesión terminó sin
que ese carrito llegara a confirmarse" (spec.md §Key Entities).
