# Research: Corrección — "Liberar Mesa" bloqueada por un pedido ya cancelado

Ambas incógnitas de esta spec ya se resolvieron leyendo el código real (frontend y backend) antes
de escribirla — no hay dependencias externas nuevas que investigar.

## D1 — Dónde vive el fix del bug de fondo (mesa bloqueada)

**Decision**: Agregar `CustomerOrder.status != "cancelada"` al `where(...)` del query que arma
`orders` dentro de `release_paid_session` (`pos-backend/app/api/v1/table_sessions/service.py:
366-369`), justo antes de pasarlo a `_assert_closable(db, orders)` (línea 370). No se toca
`_assert_closable` en sí (`service.py:217-241`), que sigue recibiendo exactamente el mismo tipo de
lista (pedidos ya filtrados por quien la llama) y sirviendo sin cambios a `close_session` (que ya
filtra correctamente vía `_billable_orders`, `service.py:141-157`).

**Rationale**: `_assert_closable` documenta su propio contrato en el docstring
("Una sesión no se cierra con pedidos sin confirmar ni con comida en curso") asumiendo que la lista
recibida ya es la relevante — el bug no está en su lógica de validación (que es correcta: un pedido
activo con cocina en curso SÍ debe bloquear), sino en que `release_paid_session` le pasa una lista
sin filtrar cuando su propio docstring aclara la intención real: seguir cubriendo pedidos `'pagada'`
con cocina en curso (que `_billable_orders` excluiría), no pedidos `'cancelada'` (que nunca tuvo
intención de incluir — no hay ninguna razón de negocio para que un pedido cancelado bloquee nada).

**Alternatives considered**:
- *Hacer que `cancel_order` (`orders/checkout.py:519-566`) también marque `estado_cocina =
  "anulado"` en los ítems no terminales al cancelar el pedido*: rechazado — `cancel_order` ya tiene
  una lógica deliberada y delicada de ajuste de inventario que distingue `pendiente` (nunca se
  preparó, revertir) de `en_preparacion`/`listo` (el insumo ya se consumió, no revertir); esa lógica
  lee `estado_cocina` para decidir qué revertir. Tocarlo agregaría una escritura de estado dentro de
  una función ya cuidadosamente comentada sobre por qué NO hace justamente eso ("ítem `anulado`: ya
  lo resolvió `void_item`" — una acción deliberadamente separada). El fix del query es más pequeño,
  no toca esa lógica, y sigue el mismo patrón ya usado por `_billable_orders`/`has_billable_orders`.
- *Agregar una acción nueva en la UI para "anular" ítems de un pedido ya cancelado*: rechazado — no
  hay ninguna razón de negocio para que un pedido terminal (cancelado) necesite una acción más;
  el problema es que se le sigue mirando la cocina a algo que ya no importa, no que falte una acción.

## D2 — Cómo deduplicar avisos repetidos

**Decision**: En `ToastService.push()` (`pos-heladeria/src/app/shared/feedback/toast.service.ts:
36-42`), antes de agregar una tarjeta nueva, revisar si ya existe una en `this.toasts()` con el
mismo `kind` y el mismo `text`; si existe, no agregar una copia (se deja que la ya visible siga su
propio temporizador).

**Rationale**: es el único punto de entrada de todas las notificaciones (`success`/`error`/`info`
son wrappers de `push`), así que el fix cubre cualquier error futuro que se repita, no solo
"Liberar Mesa" — coincide con lo que pidió el usuario ("evitar que errores repetidos apilen toasts
duplicados", no un parche puntual). No cambia la firma pública de `success`/`error`/`info`, así que
ningún llamador en la aplicación necesita tocarse.

**Alternatives considered**:
- *Deshabilitar el botón "Liberar Mesa" con un texto de carga tipo "Liberando…"*: no ataca la causa
  — el botón ya se deshabilita correctamente durante la petición
  (`[disabled]="store.submitting()"`, `pos-checkout-panel.component.ts:218-224`;
  `submitting.set(true)` antes del `await` en `releaseTable()`,
  `pos-terminal.store.ts:1681-1695`); el problema es la ausencia de dedupe entre clics
  *sucesivos* después de que `submitting()` ya volvió a `false`, no una carrera dentro de un mismo
  clic.
- *Agregar un `confirm.ask()` antes de liberar la mesa*: cambiaría el flujo de UX de un botón que
  hoy no lo pide (fuera del alcance pedido — la corrección es sobre avisos duplicados, no sobre
  agregar una confirmación nueva a una acción que no la tenía).

## Resumen de impacto en tests existentes (Principio X)

0 tests con prefijo `"CONGELA comportamiento actual:"` que dependan de que `release_paid_session`
incluya pedidos cancelados en su chequeo de cocina, ni que `ToastService.push()` apile duplicados
— ambos son huecos de cobertura, no comportamiento protegido. El test ya existente
`test_release_paid_session_409_con_cocina_en_curso_sobre_pedido_pagado`
(`test_table_sessions_service.py:647`) sigue pasando sin cambios: usa `status="pagada"`, que el
fix de D1 no excluye.
