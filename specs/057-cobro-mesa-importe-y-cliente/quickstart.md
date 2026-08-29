# Quickstart: importe fijo para pagos no efectivo + nombre de cliente en el desglose

Prerrequisitos: `pos-heladeria` corriendo local (`ng serve`), `pos-backend` corriendo local, sesión
de staff válida, una mesa con al menos un pedido con productos.

## Escenario 1 — El importe no se puede editar con un método no efectivo (US1)

1. Iniciar sesión como staff → Terminal de Mesas → seleccionar una mesa con consumo → panel de
   cobro ("Cuenta de la mesa" o "Cobrar pedido"/"Pedido de mostrador", según el origen del pedido).
2. En "Método de pago", elegir un método distinto a efectivo (p. ej. "Nequi").
   - **Esperado**: el campo de importe muestra el total exacto de la cuenta, con apariencia
     inactiva (fondo gris), y no admite clic ni escritura.
3. Intentar hacer clic sobre el campo y escribir un valor distinto.
   - **Esperado**: el valor no cambia; sigue mostrando el total exacto.
4. Verificar que el botón "Cobrar y cerrar mesa" (o "Cobrar, Facturar y Enviar a Cocina") ya está
   habilitado sin ninguna otra interacción.
5. Cambiar el método de pago de vuelta a "Efectivo".
   - **Esperado**: el campo ("Con cuánto paga") vuelve a ser editable, precargado con el total,
     igual que el comportamiento actual.

## Escenario 2 — Mismo comportamiento en el flujo de "Cobrar pedido"/mostrador (US1, FR-003)

Repetir el Escenario 1 pero desde un pedido de origen mostrador (panel "Cobrar pedido"): el campo
de importe de `app-payment-input` en `pos-checkout-panel.component.ts` debe comportarse
exactamente igual.

## Escenario 3 — El desglose muestra el nombre del cliente en vez de "Sin asignar (mesero)" (US2)

1. Crear un pedido manual con "Cliente" = "Ana Torres" (o dejar "Consumidor final"), sin asignarlo
   a ningún comensal específico.
2. Ir a Terminal de Mesas → seleccionar esa mesa → abrir "Cuenta de la mesa".
   - **Esperado**: la línea del desglose que antes decía "Sin asignar (mesero)" ahora muestra "Ana
     Torres" (o "Consumidor final").
3. Repetir con una orden que no tenga ningún nombre de cliente guardado (si aplica en el flujo
   actual del sistema).
   - **Esperado**: esa línea sigue mostrando "Sin asignar (mesero)", sin cambios.

## Tests automatizados (referencia, no reemplazan lo anterior)

- Frontend: `ng test --include='src/app/modules/tables/components/payment-input.component.spec.ts'
  --include='src/app/modules/tables/components/session-bill-panel.component.spec.ts'
  --include='src/app/modules/tables/components/pos-checkout-panel.component.spec.ts'` — corregir
  primero el selector obsoleto `input[type="number"]` (research.md Decisión 5, 8 fallos
  preexistentes) antes de agregar los casos nuevos de FR-001/FR-005.
