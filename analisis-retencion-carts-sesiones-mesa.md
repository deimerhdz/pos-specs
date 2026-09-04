# Análisis: flujo de venta y retención de `carts`, `table_sessions`, `session_participants`

**Alcance de este documento**: análisis únicamente, sin cambios de código. Consolida decisiones ya
tomadas y especificadas en las specs del repo, y responde explícitamente a dos preguntas de
negocio:

1. ¿Vale la pena conservar registros de `carts`, `table_sessions` y `session_participants` cuando
   el **usuario/mesero cierra una sesión de mesa sin pedidos pendientes por cobrar**?
2. ¿Vale la pena conservarlos cuando **el cajero libera una mesa sin pedidos pendientes por
   cobrar**?

## 1. Resumen ejecutivo

| Tabla | ¿Se conserva hoy? | Decisión |
|---|---|---|
| `carts` | **No** — se elimina físicamente | Correcto conservar la eliminación: son filas transitorias sin valor una vez que la sesión terminó sin confirmarse en pedido. |
| `table_sessions` | **Sí, siempre** — nunca se borra, solo cambia `status` a `closed` | Correcto conservar: es el registro de auditoría de ocupación de mesa. |
| `session_participants` | **Sí, siempre** — nunca se borra | Correcto conservar: es el registro de quién estuvo en la sesión y la clave que usa el propio sistema para localizar carritos huérfanos al liberar la mesa. |

Esto **ya no es una decisión pendiente**: está especificado e implementado, principalmente en
`specs/039-eliminacion-carritos-cierre-mesa/` (para `carts`) y en
`specs/010-sesion-mesa-reparto-cierre-barrido/` (para el ciclo de vida de `table_sessions` y
`session_participants`). Este documento explica el porqué y señala el único vacío real: no existe
hoy una política de purga/archivado a largo plazo para `table_sessions`/`session_participants`.

## 2. Flujo completo de la venta

```
Comensal escanea QR
      │
      ▼
Se une a TableSession activa o se crea una nueva      [specs/007, specs/016]
      │
      ▼
Arma su Cart (status='abierto') con CartItem/CartItemOption   [specs/007, specs/015]
      │
      ▼
submit_cart → crea CustomerOrder (status='recibida')    [specs/007]
   Cart se BORRA físicamente si la confirmación tiene éxito   [spec 038, vigente]
   (todavía NO se descuenta inventario en este punto — best-effort, A-47)
      │
      ▼
Cocina confirma el pedido: confirm_order                [specs/008]
   → único punto real de descuento de inventario
   → CustomerOrder: 'recibida' → 'abierta'
      │
      ▼
Staff/cajero orquesta el cobro:                          [specs/017]
   block_order → pay_order (build_sale) → confirm_order → cancel_order (si aplica)
      │
      ▼
build_sale construye la venta (constructor único, compartido
por los 4 caminos de cobro) → invoices/service.py emite la factura   [specs/011]
   (regla protegida A-14a: un solo punto de emisión de factura,
    tras pérdida de facturas reales en producción)
      │
      ▼
Cajero cobra y cierra la sesión (close_session) o libera la mesa    [specs/010]
   → TableSession: 'active' → 'closed'
   → DiningTable: ¿pasa a 'libre'? (ver regla de la sección 3)
```

Nota sobre la evolución más reciente del pago (`specs/024-pagos-ordenes-mesa/` y
`specs/025-revision-pago-antes-envio/`): el flujo del comensal moderno **invierte el orden**
respecto al diagrama de arriba — el comensal resuelve primero el método de pago y el pedido solo
se registra una vez que ese paso está resuelto, con idempotencia para evitar doble creación ante un
doble envío. No cambia lo que le pasa a `carts`/`table_sessions`/`session_participants` al cerrar
la sesión; solo cambia cuándo se crea la `CustomerOrder` dentro del flujo del comensal.

## 3. Regla que decide si la mesa queda libre

Documentada en `specs/010-sesion-mesa-reparto-cierre-barrido/spec.md` (User Story 8, FR-028 a
FR-032) y es la misma para **ambos** escenarios que pregunta el usuario — cierre por el
usuario/mesero y liberación por el cajero:

> Una sesión de mesa vuelve a `libre` automáticamente solo cuando se cumplen **ambas** condiciones
> a la vez: ningún comensal sigue `open`, y no queda ningún pedido en un estado distinto de
> `pagada`/`cancelada`. Si falta cualquiera de las dos, la mesa sigue `ocupada`.

Un pedido `cancelada` **no** cuenta como "pendiente por cobrar" (corregido también para el camino
de liberación de sesión ya pagada en `specs/050-correccion-liberar-mesa-pedido-cancelado/`, que
arreglaba un bug donde pedidos cancelados bloqueaban indebidamente la liberación).

Esta condición doble es la que dispara — o no — la eliminación de `carts` descrita abajo. Aplica
igual sin importar si quien la satisface es el propio comensal cerrando su sesión o el cajero
usando "Liberar Mesa".

## 4. Qué le pasa a cada tabla en los dos escenarios pedidos

### Escenario A — el usuario/mesero cierra la sesión de mesa sin pendientes por cobrar

Camino típico: `try_release_if_empty` (último comensal activo sale, su token expira, o cancela su
único pedido vivo) — caracterizado en `specs/010/spec.md` líneas 27-29, disparador 1 de 5 en
`specs/039/spec.md`.

- **`carts`**: se **eliminan físicamente** todos los `Cart` de los participantes de la sesión
  cerrada, sin importar su `status` (`'abierto'`, `'abandonado'` o `'confirmado'`), en la misma
  transacción que libera la mesa (`specs/039`, FR-001/FR-002). Si la mesa **no** llegara a quedar
  libre en esa misma operación (p. ej. queda otro comensal activo), ningún carrito se toca.
- **`table_sessions`**: la fila **no se borra** — transiciona `status: 'active' → 'closed'`. Queda
  como registro permanente de que esa sesión existió, cuándo se abrió y cerró, y bajo qué
  `billing_mode`.
- **`session_participants`**: **no se borran** — quedan con su `status` de cierre (`closed`). Son,
  de hecho, la clave que el propio sistema usa (`participant_id`) para saber qué carritos borrar en
  esa misma operación.

### Escenario B — el cajero libera una mesa sin pendientes por cobrar

Caminos típicos: `close_session` (cobra y cierra manualmente), `release_paid_session` (libera una
sesión ya completamente pagada), o `release_table` / endpoint "Liberar Mesa"
(`POST /orders/tables/{table_id}/release`, regla dura de cero órdenes no terminales) —
disparadores 2, 3 y 4 de 5 en `specs/039/spec.md`.

- **`carts`**: mismo resultado que el Escenario A — se eliminan físicamente todos los `Cart` de los
  participantes de la(s) sesión(es) que se cerraron en esa operación, en la misma transacción
  (`specs/039`, User Story 2). Si algo queda pendiente por cobrar, la mesa no libera y ningún
  carrito se borra.
- **`table_sessions`**: no se borra — `status: 'active' → 'closed'`, con `closed_by_user_*` lleno
  (identifica al cajero que cerró, a diferencia del cierre por barrido automático, donde
  `closed_by=None`, RN-SCHED-05).
- **`session_participants`**: no se borran, igual que en el Escenario A.

**Conclusión operativa**: en ambos escenarios el comportamiento de las tres tablas es idéntico —
solo cambia *quién* dispara la liberación (comensal saliendo vs. cajero cobrando/liberando) y si
`closed_by_user_*` queda identificado. La tabla `carts` es la única que pierde filas; las otras dos
siempre sobreviven.

## 5. Ventajas de conservar vs. eliminar cada tabla

### `carts`

| | Eliminar (comportamiento actual, spec 039) | Conservar (comportamiento previo a spec 039) |
|---|---|---|
| A favor | No quedan filas huérfanas sin ningún consumidor — verificado explícitamente revisando los usos de `Cart` fuera del módulo `cart` (`orders/consolidation.py`, `orders/checkout.py`), que solo consultan carritos `'abierto'` de sesiones **todavía activas** (spec 039, Assumptions). Sin crecimiento indefinido de una tabla que antes de esta spec acumulaba filas muertas para siempre. | Se conservaría el detalle exacto de qué había armado un comensal (ítems, opciones) incluso si nunca llegó a confirmar pedido. |
| En contra | Se pierde ese detalle de "qué armó y no confirmó" si algún día se quisiera auditar intención de compra no concretada. | Antes de spec 039 esas filas quedaban huérfanas "para siempre" sin que nada las consultara — costo de almacenamiento y ruido sin ningún beneficio identificado hoy. |

No hay, en ninguna spec revisada, un requisito de negocio que necesite reconstruir un carrito ya
abandonado. La spec 039 lo verificó explícitamente antes de decidir el borrado.

### `table_sessions` / `session_participants`

| | Conservar (comportamiento actual) | Eliminar/purgar |
|---|---|---|
| A favor | Auditoría de ocupación de mesa (quién, cuándo, cuánto tiempo), trazabilidad de quién cerró/cobró cada sesión (`closed_by_user_*`), base para reportes de rotación de mesas y para investigar reclamos de un cliente sobre una cuenta ya cerrada. Es además la fuente de verdad que otras specs (039, 050) usan para decidir qué borrar en `carts` — perderla rompería esa lógica. | Ninguna ventaja identificada en las specs revisadas: no hay volumen reportado como problema, ni una spec que proponga purgarlas. |
| En contra | Crecimiento indefinido sin política de archivado documentada — a diferencia de `carts` (regla explícita de borrado) o de la auditoría de órdenes (`specs/074`, retención acotada vía Sentry: 7-30 días), estas dos tablas no tienen ningún límite de retención especificado. | Se perdería la auditoría de ocupación y la trazabilidad de cierre — alto costo para un beneficio de espacio no cuantificado como problema real hoy. |

## 6. Decisión recomendada

1. **`carts`**: mantener el comportamiento actual (eliminación física al momento en que la mesa
   queda libre, `specs/039`). No hay ninguna necesidad de negocio identificada que justifique
   conservar carritos huérfanos, y la spec que introdujo el borrado ya verificó que ningún otro
   módulo depende de esas filas después de que la mesa se libera.

2. **`table_sessions` y `session_participants`**: mantenerlas como registros permanentes (nunca
   borrar), en ambos escenarios (cierre por usuario o liberación por cajero) — su valor de
   auditoría y trazabilidad supera claramente el costo de almacenamiento, y son además la base que
   el propio sistema usa para decidir qué carritos limpiar.

3. **Vacío a considerar, no urgente**: a diferencia de `carts` (regla de borrado explícita) y de la
   auditoría de órdenes (`specs/074`, retención acotada a 7-30 días vía Sentry), no existe hoy
   ninguna spec que defina una política de retención/archivado a largo plazo para
   `table_sessions`/`session_participants`. Mientras el volumen no sea un problema operativo real,
   no hay necesidad de actuar; si en el futuro el crecimiento de estas tablas se vuelve un problema
   (tamaño de base de datos, performance de consultas históricas), el precedente de `specs/074`
   (mover el detalle histórico a un sistema externo con retención acotada, en vez de purgar sin
   más) es el patrón que este mismo proyecto ya usó para resolver un problema análogo.

## 7. Referencias

- `specs/007-menu-carrito-qr/spec.md` — flujo de carrito del comensal vía QR.
- `specs/015-caracterizacion-cart/spec.md`, `data-model.md` — characterization de `cart/service.py`.
- `specs/008-confirmacion-cobro-legado-y-cancelacion-de-pedidos/spec.md` — confirmación en cocina y descuento de inventario.
- `specs/017-caracterizacion-orders/spec.md`, `data-model.md` — orquestación de checkout/staff.
- `specs/011-venta-mostrador-constructor-factura/spec.md` — `build_sale` y emisión de factura.
- `specs/024-pagos-ordenes-mesa/spec.md` — métodos de pago configurables por tenant.
- `specs/025-revision-pago-antes-envio/spec.md` — inversión del orden pago→pedido, idempotencia.
- `specs/010-sesion-mesa-reparto-cierre-barrido/spec.md` — ciclo de vida completo de `TableSession`, reparto de cuenta, liberación automática y barrido.
- `specs/016-caracterizacion-table-sessions/spec.md`, `data-model.md` — characterization de `table_sessions`.
- `specs/039-eliminacion-carritos-cierre-mesa/spec.md`, `data-model.md` — decisión y especificación del borrado físico de `carts` al liberar la mesa.
- `specs/050-correccion-liberar-mesa-pedido-cancelado/spec.md`, `data-model.md` — corrección sobre pedidos cancelados y liberación de mesa.
- `specs/074-auditoria-ordenes/spec.md` — auditoría del ciclo de vida de la orden individual y su política de retención vía Sentry (referencia de contraste, fuera de alcance de estas tres tablas).
