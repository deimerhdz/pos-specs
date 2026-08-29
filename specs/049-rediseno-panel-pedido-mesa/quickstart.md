# Quickstart: validación manual del rediseño del panel de pedido de mesa

## Prerrequisitos

- `pos-heladeria` corriendo localmente contra un backend con datos de prueba (`pos-backend`).
- Un turno de caja abierto (necesario para que `session-bill-panel` ofrezca cobrar).
- Al menos una mesa con **dos** pedidos activos del mismo comensal (una ronda ya con algún ítem
  "Listo" y otra con ítems "Pendiente"), y otra mesa con **un solo** pedido activo — para cubrir
  ambos casos de FR-009/FR-010.
- Al menos un pedido con descuento por promoción activo, para validar la fila "Descuento" tanto en
  su nueva ubicación (cuenta) como su ausencia en el panel de pedido.

## Arrancar la app

```bash
cd pos-heladeria
npm start
```

Abrir la Terminal de Mesas (`/dashboard/mesas-sesiones`) con un usuario cajero.

## Escenario 1 — Subtotal/Descuento/Total viven solo en la cuenta (User Story 1)

1. Seleccionar la mesa con descuento activo.
2. En el panel central (pedido), confirmar que **no** aparece ninguna fila "Subtotal", "Descuento"
   ni "Total" bajo el carrito.
3. En el panel derecho ("Cuenta de la mesa"), confirmar que aparecen "Subtotal", "Descuento" (si
   > 0) y "Total", además del desglose por comensal ya existente.
4. Confirmar que los tres importes coinciden con lo que mostraba antes el panel de pedido para esa
   misma mesa (mismo cálculo, solo cambia dónde se ve) — comparar contra `store.totals()` antes del
   cambio si hay una build anterior a mano, o contra la suma manual de las líneas del desglose.
5. Cobrar la mesa normalmente y confirmar que el botón "Cobrar y cerrar mesa" y el selector de
   método de pago siguen funcionando sin cambios (FR-005).

**Resultado esperado**: SC-001 — un cajero resuelve cuánto cobrar mirando un único panel.

## Escenario 2 — Cliente y pedidos en el nuevo formato (User Story 2)

1. Seleccionar la mesa con dos pedidos activos.
2. Confirmar que la cabecera muestra, en una sola fila, el número de mesa, un chip de estado (p.
   ej. "Ocupada") y el nombre del cliente como texto — sin ningún campo editable ni etiqueta
   "Cliente" separada arriba de un input (FR-006/FR-008).
3. Confirmar que aparecen las pestañas "Todos los pedidos (2)", "Pedido 1", "Pedido 2".
4. Con "Todos los pedidos (2)" activa (debe ser la pestaña inicial), confirmar que se ven las dos
   tarjetas a la vez, cada una con su propia hora y su propia pastilla de estado ("Pendiente" o
   "Listo").
5. Dentro de cada tarjeta, confirmar que cada ítem muestra cantidad, nombre, variante/nota (si
   tiene) y precio unitario + total de línea, junto con su pill de estado por ítem y el botón "✓
   Listo" cuando aplica (FR-013/FR-014).
6. Pulsar "✓ Listo" sobre un ítem de la tarjeta que **no** es la seleccionada por defecto —
   confirmar que el ítem correcto (el de esa tarjeta) cambia de estado, no un ítem del otro pedido
   (valida research.md D6 — la generalización de `avanzarItem`).
7. Cambiar a la pestaña "Pedido 1" — confirmar que se ve solo esa tarjeta, con "+ Agregar producto"
   disponible; cambiar a "Pedido 2" y repetir.
8. Volver a "Todos los pedidos (2)" — confirmar que "+ Agregar producto" ya no está visible en ese
   modo (research.md, D5).
9. Seleccionar la mesa con **un solo** pedido activo — confirmar que no aparece ningún selector de
   pestañas (FR-010).

**Resultado esperado**: SC-002 — alternar entre ver todos los pedidos o uno solo toma un clic, sin
perder de vista cliente/estado de mesa.

## Escenario 3 — Sin "+ Nuevo pedido" (User Story 3)

1. En cualquier mesa ocupada con uno o más pedidos, revisar la fila de pestañas.
2. Confirmar que no hay ningún control "+ Nuevo pedido" ni equivalente.

**Resultado esperado**: SC-003 — 0 apariciones del control retirado.

## Regresión rápida (no debería haber cambiado)

- Pedido de canal QR ya pagado: sigue sin ofrecer "+ Agregar producto" en su tarjeta.
- Ítem anulado: sigue sin aparecer en el carrito/tarjeta.
- "Guardar pedido" con líneas en borrador: sigue funcionando igual dentro de una pestaña "Pedido N".
- Anular un combo desde una tarjeta que no es la seleccionada: anula el combo correcto (D6).
