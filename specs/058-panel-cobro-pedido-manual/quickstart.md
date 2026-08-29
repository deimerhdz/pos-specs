# Quickstart de validación: Ajustes al panel de cobro de pedido manual

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Prerrequisitos: turno de caja abierto, al menos una mesa libre o un pedido de mostrador ya creado
con `hold_for_payment` (`status: 'recibida'`, aún no enviado a cocina).

## Escenario 1 — Historia 1: nombre de facturación solo-lectura + editar (FR-001/FR-002/FR-003)

1. Abrir el panel de cobro de un pedido de mostrador o de mesa que aún no se ha enviado a cocina
   ("Cobrar pedido"/"Pedido de mostrador").
2. Verificar que "Facturar a nombre de" se ve con apariencia atenuada (fondo gris) y no admite
   escribir directamente sobre él; junto al campo hay un botón de editar (✏️).
3. Pulsar el botón de editar: el campo pasa a apariencia normal y admite escribir. Cambiar el
   nombre a, por ejemplo, "María Pérez".
4. Hacer clic en cualquier otra parte del panel (perder foco del campo): el campo vuelve a
   apariencia atenuada, mostrando "María Pérez".
5. Confirmar el cobro (ver Escenario 2) y verificar en la petición de red (`checkout-and-send`)
   que `billing_customer_name` es `"María Pérez"` — sin cambio respecto al comportamiento actual de
   facturación.

**Resultado esperado**: el campo se comporta igual que "Cliente" en la vista de armado de pedido
manual — solo lectura por defecto, editable a demanda, valor preservado al confirmar.

## Escenario 2 — Historia 2: texto del botón "Cobrar" (FR-004/FR-005/FR-006)

1. En el mismo panel, verificar que el botón principal dice "Cobrar" (no "Cobrar, Facturar y
   Enviar a Cocina").
2. Elegir un método de pago y completar el importe si aplica.
3. Pulsar "Cobrar": mientras la operación está en curso, el botón debe mostrar "Cobrando…".
4. Verificar que el pedido queda cobrado, facturado y enviado a cocina — mismo resultado final que
   hoy, solo cambia el texto visible del botón.

**Resultado esperado**: ningún cambio de comportamiento de cobro, solo el rótulo.

## Escenario 3 — Historia 3: "Imprimir Factura"/"Liberar Mesa" ocultos mientras el cobro está pendiente (FR-007/FR-008)

1. Abrir el panel de cobro de un pedido que aún no se ha enviado a cocina.
2. Verificar que **no** aparecen los botones "Imprimir Factura" ni "Liberar Mesa" en este estado
   (solo el formulario de cobro y "Rechazar pedido").
3. Confirmar el cobro (Escenario 2). Tras el éxito, la selección se limpia (comportamiento actual
   sin cambios) — el panel vuelve a "+ Crear pedido nuevo" o a otro estado, no queda un pedido ya
   cobrado mostrando el formulario de cobro pendiente.
4. Abrir por separado "Cuenta de la mesa" de un pedido ya enviado a cocina (cobro por cierre de
   sesión), o el modo resumen de un pedido de canal QR ya pagado: verificar que "Imprimir Factura"
   y "Liberar Mesa" siguen apareciendo exactamente igual que antes de este cambio.

**Resultado esperado**: las dos acciones post-cobro solo se ven cuando el cobro de ese pedido ya se
efectuó (o en los modos donde ya aplicaban hoy sin cambios), nunca mientras el formulario de cobro
sigue pendiente de confirmar.
