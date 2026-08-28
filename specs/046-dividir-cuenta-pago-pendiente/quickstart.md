# Quickstart de Validación: Eliminación de Dividir Cuenta y de Combinar Método de Pago

Guía para verificar manualmente, de punta a punta, que el cambio cumple los criterios de aceptación
del [spec.md](./spec.md) una vez implementado. No sustituye los tests automatizados (ver
[research.md §5](./research.md) y [contracts/ui-store-contract.md](./contracts/ui-store-contract.md)).

## Prerrequisitos

- Repositorio `../pos-heladeria` con las dependencias instaladas.
- Backend `../pos-backend` corriendo localmente (esta spec no requiere cambios de backend, pero la
  Terminal de Mesas necesita datos reales de mesas/catálogo/turno de caja para probarse).
- Al menos: una mesa con un pago QR en efectivo pendiente de confirmar, una mesa con transferencia
  con comprobante ya subido pendiente de aprobar, una mesa con un pedido de mostrador listo para
  cobrar (sin pago pendiente), y una mesa con consumo de más de un comensal (para verificar que
  "Dividir por comensal" ya no existe como modo de cobro).

## Arrancar el entorno

```bash
cd ../pos-heladeria
pnpm start   # o el comando de desarrollo configurado en package.json
```

Abrir la Terminal de Mesas (`table-sessions` / ruta del módulo `tables`) en el navegador.

## Escenarios a validar (mapeados a Acceptance Scenarios del spec)

1. **"Liberar Mesa" oculto con pago pendiente** (Historia 1): seleccionar la mesa con efectivo
   pendiente de revisión y confirmar que el panel "Pedido de mostrador" no muestra "🔓 Liberar
   Mesa". Repetir con la mesa de transferencia pendiente de aprobar — mismo resultado.
2. **"Liberar Mesa" visible sin pago pendiente**: seleccionar la mesa de mostrador lista para cobrar
   (sin pago pendiente) y confirmar que "🔓 Liberar Mesa" se ve con su comportamiento de siempre.
3. **Reaparición inmediata** (FR-002/SC-003): sobre la mesa con efectivo pendiente, confirmar el
   pago desde el bloque de validación (botón "Confirmar efectivo") y verificar que "🔓 Liberar Mesa"
   aparece de inmediato en el panel de mostrador, sin recargar la página ni volver a seleccionar la
   mesa. Repetir aprobando la transferencia pendiente.
4. **"Dividir la cuenta entre varias personas" ausente en todos lados** (Historia 2): en el panel
   "Pedido de mostrador" (cualquier mesa, con o sin pedido), confirmar que el botón "Dividir la
   cuenta entre varias personas" ya no existe en ningún lugar. Abrir la "Cuenta de la mesa" de una
   sesión con consumo de más de un comensal y confirmar que no hay ningún toggle "Dividir por
   comensal" — solo se puede cobrar el total completo.
5. **Desglose por comensal sin cambios**: en esa misma "Cuenta de la mesa" con más de un comensal,
   confirmar que el desglose de solo lectura (qué pidió cada uno, con su subtotal) se sigue viendo
   igual que antes — no se retiró, solo dejó de poder usarse para cobrar por separado.
6. **"Combinar método de pago" ausente en el cobro de mostrador** (Historia 3): abrir el pedido de
   mostrador listo para cobrar y confirmar que `payment-input` ya no ofrece ningún checkbox
   "Combinar con otro método" ni campos de segundo método — solo un selector de método y un importe.
7. **"Combinar método de pago" ausente en "Cuenta de la mesa"**: repetir el punto 6 sobre el cobro
   de "Cuenta de la mesa" (modo único que queda tras el punto 4).
8. **Pago insuficiente se rechaza, no se completa con otro método**: en el cobro de mostrador,
   elegir un método, escribir un monto menor al total, y confirmar que el botón de cobrar queda
   deshabilitado con el mensaje "Faltan $X para cubrir la cuenta" — sin ninguna forma de agregar un
   segundo método para completarlo.
9. **Cambio en efectivo sin regresión**: en el mismo cobro, elegir efectivo y escribir un monto
   mayor al total; confirmar que "Vuelto" se sigue calculando y mostrando igual que antes.
10. **Bloque de confirmación de pago pendiente sin cambios** (FR-003, ya cumplido hoy): sobre la
    mesa con efectivo pendiente, confirmar que el bloque de validación nunca ofreció combinar
    método — solo el monto recibido y "Confirmar efectivo"/"Rechazar pedido". Escribir un monto
    menor al total del pedido y confirmar que el backend rechaza la confirmación (mensaje de error,
    HTTP 422) en vez de aceptarla parcialmente.
11. **Sin regresión de comportamiento**: ejecutar un ciclo completo de validación de pago QR
    (confirmar efectivo, aprobar comprobante) y un cobro manual completo de mostrador (método único,
    monto exacto), y confirmar que ambos terminan exactamente igual que antes de esta spec —
    facturación, envío a cocina y liberación de mesa sin cambios de lógica, solo la condición nueva
    del punto 1.

## Verificación automatizada

```bash
cd ../pos-heladeria
pnpm test -- pos-checkout-panel        # ajustado: T035 + caso nuevo "Liberar Mesa" oculto en 'validar-pago'
pnpm test -- session-bill-panel        # ajustado: sin casos de modo split/combinar
pnpm test -- payment-draft.util        # ajustado: sin casos de combinar
pnpm test -- payment-attempt-review-panel  # sin cambios de intención; agrega caso de monto insuficiente
pnpm test -- payment-validation-block  # sin cambios
pnpm test -- pos-terminal.store        # sin cambios
```

`split-bill-panel.component.spec.ts` se elimina junto con el componente — no debe aparecer en la
salida del test runner tras la implementación.

(Ajustar el runner exacto —`pnpm test`, `ng test`, `npx vitest`— al script real configurado en
`package.json` del proyecto en el momento de implementar; hoy es `ng test`.)
