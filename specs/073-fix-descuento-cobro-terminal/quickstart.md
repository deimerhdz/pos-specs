# Quickstart: Validación — descuento por promoción en el cobro de la Terminal

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Guía de validación ejecutable por historia de usuario. No repite el detalle de contratos
(ver [contracts/](./contracts/)) ni de esquema (ver [data-model.md](./data-model.md)).

## Prerrequisitos

- `pos-backend` corriendo con la migración de [data-model.md](./data-model.md) aplicada
  (`alembic upgrade head`), un tenant de prueba con turno de caja abierto.
- Un producto ("Cono de helado", $8.000) con una promoción vigente del 50% llevando 2 (spec 063).
- `pos-heladeria` corriendo contra ese backend (`npm start`), sesión de cajero/mesero.

## Suites automatizadas (ejecutar antes y después de implementar — red de seguridad)

```bash
# Backend — characterization tests (no deben regresar) + tests nuevos de esta spec
cd pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
# especialmente: test_orders_checkout.py, test_table_sessions_service.py,
# test_cart_service.py, test_promotions_service.py,
# test_orders_payment_gate.py (US7 — chequeo previo del efectivo)

# Frontend — Vitest
cd pos-heladeria
npm test
# especialmente: pos-terminal.store.spec.ts, pos-checkout-panel.component.spec.ts,
# payment-input.component.spec.ts, manual-order-page.component.spec.ts,
# payment-attempt-review-panel.component.spec.ts (US7),
# payment-validation-block.component.spec.ts (US7)
```

Ninguna suite existente debe cambiar de resultado salvo los tests que la implementación actualice
explícitamente por estar ligados al comportamiento que esta spec deroga (research.md D3, D8) —
citando la decisión que lo autoriza, como exige la Constitución (Principio III).

## Historia 1 — Cobrar en efectivo un pedido con promoción

1. Crear un pedido de mesa (Terminal, no QR) con 2 conos de helado.
2. Abrir el panel de cobro. **Verificar**: `GET /orders/{id}/checkout-preview` responde
   `{subtotal: 16000, discount: 8000, total: 8000}`; la pantalla muestra `Subtotal $16.000`,
   `Descuento −$8.000`, `Total $8.000` (FR-004, Acceptance Scenario 1).
3. Elegir efectivo, ingresar $8.000. **Verificar**: sin mensaje de importe faltante, vuelto $0,
   botón "Cobrar" habilitado (Scenario 2).
4. Ingresar $10.000. **Verificar**: vuelto $2.000, calculado sobre $8.000 (Scenario 3).
5. Ingresar $5.000. **Verificar**: "Faltan $3.000", "Cobrar" deshabilitado (Scenario 4).
6. Cobrar con $8.000. **Verificar**: `GET /sales/{id}` devuelve `total: 8000`, `change_given: 0`
   (Scenario 5); `SC-002` — el total mostrado coincide al peso con el de la venta emitida.

## Historia 2 — Cobrar por transferencia

1. Mismo pedido, elegir un método no efectivo. **Verificar**: importe precargado $8.000, no
   $16.000 (Scenario 1).
2. Cobrar. **Verificar**: venta emitida por $8.000, sin el 422 "pagos que no son en efectivo...
   superan el total" (Scenario 2).
3. Repetir con un pedido a domicilio, valor de envío $5.000 y la misma promoción. **Verificar**:
   importe precargado = `subtotal − descuento + domicilio` = $13.000 (Scenario 3).

## Historia 3 — Para llevar y domicilio

1. Orden para llevar con promoción vigente. **Verificar**: total mostrado incluye el descuento,
   se cobra por ese importe (Scenario 1).
2. Orden a domicilio, envío $5.000, sin promoción. **Verificar**: fila `Domicilio $5.000` visible,
   total la incluye (Scenario 2).
3. Orden a domicilio, envío $5.000, promoción que descuenta $8.000 sobre $16.000. **Verificar**:
   `Subtotal $16.000`, `Descuento −$8.000`, `Domicilio $5.000`, `Total $13.000` (Scenario 3).
4. **Verificar** (`SC-003`): ningún pedido a domicilio con valor de envío queda bloqueado al
   cobrar.

## Historia 4 — Vigencia congelada al tomar el pedido

Requiere una promoción con franja horaria configurable (p. ej. vigente hasta las 20:00).

1. Crear un pedido a las 19:59 (dentro de la franja). Cobrarlo a las 20:05. **Verificar**: el
   descuento se aplica igual, mismo total que se mostró al tomar el pedido (Scenario 1).
2. Crear un pedido a las 19:59 con una promoción que **empieza** a las 20:00. Cobrarlo a las
   20:05. **Verificar**: el descuento NO se aplica (Scenario 2).
3. Sobre el pedido del punto 1, agregar un tercer cono a las 20:05 antes de cobrar. **Verificar**:
   el descuento se recalcula sobre los 3 conos usando la vigencia de las 19:59; el tercero queda a
   precio pleno (Scenario 3, FR-010).
4. Consultar/reimprimir una venta emitida antes de este cambio. **Verificar**: importe y desglose
   sin cambios (Scenario 4, FR-011, `SC-007`).
5. Mesa con ronda 1 a las 19:59 (1 cono) y ronda 2 a las 20:05 (1 cono), promoción vigente solo
   hasta las 20:00. Cobrar la cuenta completa a las 20:10. **Verificar**: los dos conos se evalúan
   juntos contra las 19:59, el descuento se aplica (Scenario 5, FR-012a).
6. **Verificar** (`SC-009`): abrir `Ventas → [la venta del punto 1] → detalle`; la fila
   «Promociones evaluadas con la vigencia del…» muestra las 19:59, explicando por sí solo el
   descuento sin abrir el pedido. (`GET /sales/{id}` devuelve `promotion_evaluated_at` con ese
   instante; una venta anterior a esta spec lo devuelve `null` y no pinta la fila.)

## Historia 5 — Ver el descuento al armar la orden manual

1. Pantalla de armado, agregar 1 cono. **Verificar**: `Total $8.000`, sin fila de descuento
   (Scenario 1).
2. Agregar un segundo cono. **Verificar**: `Subtotal $16.000`, `Descuento −$8.000`,
   `Total $8.000` (Scenario 2).
3. Agregar un tercer cono. **Verificar**: `Subtotal $24.000`, `Descuento −$8.000`,
   `Total $16.000` (Scenario 3).
4. Simular caída del `POST /orders/draft-preview` (desconectar backend o mockear error).
   **Verificar**: se ve el subtotal sin descuento + aviso "el descuento se confirma al cobrar", y
   "Confirmar pedido" sigue habilitado (Scenario 4, FR-015).
5. Con un borrador en `Total $8.000` y una promoción vigente hasta las 20:00, pulsar "Confirmar
   pedido" a las 20:01 cuando el total real ya subió a $16.000. **Verificar**: se muestra el total
   actualizado y se pide una segunda confirmación antes de crear el pedido (Scenario 5, FR-015a).

## Historia 6 — Condición de la promoción en el catálogo

1. Promoción `2 x -50%` sobre un cono de $8.000. Abrir el catálogo de la Terminal. **Verificar**:
   la tarjeta muestra la condición legible con equivalente por unidad (`2 x -50% · ≈ $4.000 c/u`),
   no la insignia suelta `-50%` (Scenario 1).
2. Regla de paquete `3 x $20.000`. **Verificar**: la tarjeta muestra esa condición, no un precio
   unitario suelto (Scenario 2).
3. Producto sin promoción vigente. **Verificar**: tarjeta igual que hoy (Scenario 3).
4. Regla con `min_qty = 1`. **Verificar**: la tarjeta puede mostrar el precio unitario con
   descuento (Scenario 4).

## Historia 7 — Confirmar el pago de un pedido de comensal por QR con promoción

El panel **"Pagos por confirmar"** (mesa con un pedido `qr` en `recibida`; el cajero pulsa
"Confirmar efectivo" o "Aprobar comprobante"). Requiere un pedido creado por el carrito del
comensal (QR), no por la Terminal.

1. Comensal pide 2 conos por QR con la promoción del 50% llevando 2 y declara pago en efectivo.
   Abrir "Pagos por confirmar" en la Terminal. **Verificar**: `GET /orders/{id}/checkout-preview`
   responde `{subtotal: 16000, discount: 8000, total: 8000}`; el panel muestra `Subtotal $16.000`,
   `Descuento −$8.000`, `Total $8.000` (FR-022, Scenario 1).
2. Registrar $10.000 en efectivo y pulsar "Confirmar efectivo". **Verificar**: la venta se emite
   por $8.000, `attempt.change_amount = 2000`, el pedido pasa a cocina — **sin** el mensaje "El
   monto recibido (10000) es menor al total de la orden (16000.00)" (Scenario 2, FR-023, SC-002a).
3. Repetir con $8.000 exactos. **Verificar**: vuelto $0, se registra al primer intento (Scenario 3).
4. Repetir con $5.000. **Verificar**: el panel indica que faltan $3.000 (sobre $8.000, no sobre
   $16.000) y no permite confirmar (Scenario 4).
5. Comprobante de transferencia por un pedido QR con promoción (total real $8.000). **Verificar**:
   el panel muestra `Total $8.000` antes de pulsar "Aprobar"; la venta se emite por $8.000, sin el
   error de "pago no efectivo supera el total" (Scenario 5).
6. Pedido QR creado a las 19:59 dentro de una promoción vigente hasta las 20:00, pago confirmado a
   las 20:05. **Verificar**: el descuento se aplica igual, `Total $8.000` (Scenario 6, FR-009 +
   instante congelado del flujo QR).
7. Pedido QR cuya promoción el administrador pausó entre el pedido y el cobro (total real sube de
   $8.000 a $16.000). Abrir "Pagos por confirmar". **Verificar**: el panel muestra `Total $16.000`,
   marca que cambió respecto al $8.000 que mostraba la tarjeta y exige que el cajero lo confirme
   antes de emitir (Scenario 7, FR-024 + FR-009a).
8. **Verificar** (`SC-002a`): en ningún caso un pedido QR con promoción vigente queda bloqueado por
   "el monto recibido es menor al total de la orden"; el total del panel == total de la venta
   emitida, al peso.

## Verificación cruzada de alcance protegido

- `SessionBillPanelComponent` (pedido de mesero ya enviado a cocina) y el flujo del comensal por
  QR **en su menú y su carrito**: repetir el Escenario 5 de Historia 4 sobre la sesión de mesa
  completa (mesero) y confirmar que el carrito QR también congela su instante al confirmar el
  pedido — FR-018. La **revisión de pago del cajero** para pedidos QR ("Pagos por confirmar") NO
  está protegida: la corrige la Historia 7 (arriba).
- **`_order_total` eliminado**: confirmar por `grep` sobre `pos-backend/app/` que no queda ninguna
  referencia a `_order_total` y que `confirm_cash_payment_attempt` usa
  `compute_checkout_preview(...).total` para su chequeo previo.
- **Mesas fusionadas (FR-018a)**: fusionar dos mesas, cada una con un pedido creado a hora distinta
  (una dentro de una franja de promoción, la otra fuera). Ver la cuenta consolidada y luego cobrar
  cada pedido por separado. **Verificar**: el descuento de cada pedido en la cuenta consolidada
  coincide al peso con lo que se le cobra individualmente — cada pedido va contra su propio
  instante, no contra un instante único de grupo.
- Confirmar que ningún descuento manual aparece disponible en ningún panel (FR-019) y que sigue
  existiendo un único método de pago por cobro (FR-020).
