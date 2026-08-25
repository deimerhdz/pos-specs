# Quickstart: Validación de Estado "Pagada" Correcto y Formato de Moneda Reutilizable

## Antes de implementar (Principio II)

- [ ] Registrar la decisión de negocio en
  `specs/000-reconocimiento/registro-de-anomalias.md` (ver `research.md`, Decisión 4): qué
  cambia, por qué, quién y cuándo la autoriza, y qué funcionalidades quedan afectadas
  (`orders/tables_advanced.py`). Sin esta entrada, la implementación de Historia 1 no cumple
  el Principio II de la Constitución.

## Prerrequisitos

- `pos-backend` corriendo localmente, con al menos una mesa con una comanda creada con
  `hold_for_payment=True` (nace `'recibida'`, canal `waiter`/`counter`) y un turno de caja
  abierto.
- `pos-heladeria` corriendo localmente, apuntando a ese backend, con un usuario de staff con
  acceso a Terminal de Mesas, Órdenes, y al menos otro módulo con un campo de precio/monto
  (por ejemplo, Caja → abrir turno).

## Escenario 1 — El estado queda "pagada" de verdad tras cobrar (Historia 1, SC-001)

1. Desde Terminal de Mesas, armar una comanda con al menos un ítem y cobrarla ("Cobrar y
   enviar"), sin haber marcado ningún ítem como `listo` todavía.
2. Consultar el pedido recién cobrado (`GET /orders/{id}` o el detalle en el dashboard).
   **Esperado**: `status = "pagada"` (no `"abierta"`), y `paid = true` — ambos coinciden
   (FR-001, contracts/checkout-and-send-status.md).
3. Verificar en el listado de Órdenes ("Comandas de la operación") que sigue mostrando
   "Pagada" con el mismo aspecto que antes de este cambio (FR-004/SC-003) — no debe haber
   ninguna diferencia visual.

## Escenario 2 — La mesa sigue protegida mientras cocina no termine (Historia 1, SC-005)

1. Con el pedido del Escenario 1 ya `"pagada"` y sus ítems todavía `pendiente`/
   `en_preparacion` (ninguno `listo` ni `anulado`), intentar liberar la mesa
   (`PATCH /orders/tables/{id}/status` → `"libre"`), moverla a otra mesa
   (`POST /orders/{id}/move`), o fusionarla con otra (`POST /orders/merge`).
   **Esperado**: las tres operaciones siguen respondiendo `409` — la protección no
   desapareció al pasar a `'pagada'` (FR-003).
2. Marcar todos los ítems del pedido como `listo` (`PATCH /orders/items/{id}/kitchen`).
3. Repetir las tres operaciones del paso 1.
   **Esperado**: ahora sí se permiten — sin ítems pendientes, una orden `'pagada'` ya no
   bloquea la mesa (mismo criterio que antes de esta spec para una orden verdaderamente sin
   nada pendiente).

## Escenario 3 — El camino QR no se ve afectado (Historia 1, SC-002)

1. Desde el menú QR de una mesa, un comensal envía un pedido y el staff lo confirma hacia
   cocina (sin que exista todavía ninguna venta para ese pedido).
   **Esperado**: el pedido queda `status = "abierta"`, exactamente igual que antes de esta
   spec (FR-002) — no se marca `'pagada'` sin que exista una venta real.

## Escenario 4 — Input de moneda con formato en vivo (Historia 2, SC-004)

1. Abrir cualquiera de los ~12 campos migrados (ver `spec.md`, Assumptions) — por ejemplo,
   "Fondo inicial (base de efectivo)" al abrir un turno de caja.
2. Escribir `500000`.
   **Esperado**: el campo muestra `$ 500.000` (o el número con separador de punto) mientras
   se escribe, no solo después de guardar (FR-007).
3. Guardar el formulario y verificar en el backend/consulta posterior que el valor persistido
   es el número `500000` limpio, sin ningún carácter de formato (FR-008).
4. Repetir en un campo que admite decimales (por ejemplo, costo unitario de un insumo) y
   verificar que se pueden escribir y ver centavos sin que el separador de miles interfiera
   (FR-010).
5. En el campo de descuento de una promoción, cambiar entre modo "%" y modo "monto en pesos"
   y verificar que el formato de moneda solo se aplica en el segundo modo (FR-011).
6. En el campo de precio de paquete del selector de alcance de promociones, dejarlo vacío y
   guardar. **Esperado**: se sigue interpretando como "hereda el valor por defecto del
   paquete" — no se guarda como `0` (FR-009).

## Verificación de no-regresión

- Confirmar que `test_orders_tables_advanced.py` (characterization tests) sigue pasando sin
  modificar ninguno de sus tests `"CONGELA comportamiento actual:"` (verificado en
  `research.md`, Decisión 2 — el único test que ejercita esta guarda usa una orden sin ítems,
  así que su resultado no cambia).
- Confirmar que `pay_order` y `close_session` (los otros dos caminos que ya dejaban el pedido
  en `'pagada'`) siguen comportándose exactamente igual — no forman parte del alcance de esta
  spec.
- Confirmar que anular un ítem de un pedido ya pagado (`void_item`, spec 029 Decisión D3)
  sigue bloqueado por la misma señal de siempre (`paid`/`order_has_sale`) — esta spec no la
  toca.
