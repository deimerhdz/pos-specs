# Quickstart: Validación de la Vista de Pasos para Revisión y Pago del Menú QR

## Prerrequisitos

- `pos-backend` corriendo localmente con un tenant de prueba que tenga al menos un método de
  transferencia activo y configurado con número de cuenta/celular **e** imagen de código QR (spec
  032), y otro método de transferencia configurado **sin** imagen de código QR.
- `pos-heladeria` corriendo localmente, apuntando a ese backend.
- Un QR (o enlace `menu/t/:token`) de una mesa de ese tenant, con productos activos en el catálogo.

## Escenario 1 — Recarga a mitad de pago (User Story 1, SC-001/SC-002)

1. Abrir el menú por el enlace de la mesa, agregar un producto al carrito, presionar "Enviar
   pedido".
2. En la vista de revisión, elegir un método de transferencia.
3. Recargar la página (F5) **antes** de cargar el comprobante.
   **Esperado**: la vista vuelve a mostrar los datos de pago de ese mismo método, sin pedir elegirlo
   de nuevo (FR-005).
4. Elegir un comprobante válido (aparece su vista previa y el botón independiente "Enviar
   pedido", sin que se haya enviado nada todavía) y, **antes** de presionar "Enviar pedido",
   recargar la página de nuevo.
   **Esperado**: la vista no pide seleccionar el archivo otra vez — vuelve a mostrar la vista
   previa del comprobante ya subido, lista para presionar "Enviar pedido" (FR-006). Presionar el
   botón y verificar en el panel de staff que se creó exactamente **un** pedido, no dos.
5. Con el pedido ya creado (efectivo confirmado o comprobante ya enviado), recargar la página.
   **Esperado**: se muestra la confirmación del pedido ya existente, no la vista de revisión
   (FR-008).

## Escenario 2 — Vista propia con pasos (User Story 2)

1. Presionar "Enviar pedido" y verificar que la revisión se abre como pantalla propia (no como una
   superposición sobre la carta), con un indicador visible de en qué paso se está (FR-001/FR-002).
2. Desde el paso de datos de transferencia, volver al paso de selección de método y elegir uno
   distinto. Verificar que no queda ningún rastro del método anterior (FR-003).
3. Salir de la vista sin completar el pago y verificar que el carrito conserva exactamente los mismos
   productos, sin que se haya creado ningún pedido (FR-004).

## Escenario 3 — Datos de pago con imagen de QR (User Story 3, SC-003)

1. Elegir el método de transferencia configurado **con** imagen de código QR.
   **Esperado**: el número de cuenta/celular se ve como texto legible; el código QR se renderiza como
   una imagen visible (no como una URL en texto), verificable inspeccionando que el elemento es un
   `<img>` (FR-011/FR-012). Contrastar contra `contracts/cart-payment-methods.md`.
2. Elegir el método de transferencia configurado **sin** imagen de código QR.
   **Esperado**: se muestran los demás datos de pago disponibles, sin espacio roto ni ícono de imagen
   no encontrada (FR-013).
3. (Requiere coordinación con un Tenant Admin de prueba) Desactivar, desde otra pestaña o el panel de
   administración, el método de transferencia que se había elegido en la vista de pago, y luego
   recargar la página del comensal. **Esperado**: el sistema indica que ese método ya no está
   disponible y pide elegir uno activo, conservando el resumen del pedido (FR-010).

## Escenario 4 — Iconos (User Story 4, SC-005)

1. Recorrer los cuatro pasos de la vista (revisión, método, transferencia, confirmación).
   **Esperado**: ningún icono es un emoji; todos provienen del componente `app-icon` ya existente en
   la aplicación (verificable inspeccionando el DOM: ningún carácter emoji suelto, solo elementos
   generados por `app-icon`).

## Verificación de no-regresión

- Confirmar que el modal de reintento de pago tras un rechazo de comprobante (spec 024, Historia 5,
  sobre un pedido **ya creado**) sigue funcionando exactamente igual — no debe haberse tocado.
- Confirmar que la pantalla de cobro en caja (spec 032, FR-012a) sigue mostrando los métodos de pago
  solo por nombre, sin datos de integración — el campo `fields` agregado en esta funcionalidad es
  exclusivo de la respuesta al comensal.
