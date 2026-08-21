# Quickstart: Validación del Rediseño Híbrido de la Terminal de Mesas

**Spec**: [spec.md](./spec.md) | **Contratos**: [contracts/api-contracts.md](./contracts/api-contracts.md)

Guía de validación manual/automatizada end-to-end, una escena por historia de usuario. No
sustituye los tests unitarios/característicos de cada repo — es la prueba de que el conjunto se
comporta como el spec exige.

## Prerrequisitos

- `pos-backend` corriendo localmente (API + PostgreSQL 16 con al menos un tenant de prueba) con las
  migraciones al día y un turno de caja (`cash_shift`) abierto para el cajero de prueba.
- `pos-heladeria` corriendo localmente contra ese backend (`ng serve` o equivalente).
- Un usuario staff (cajero) autenticado en la Terminal de Mesas.
- Al menos una mesa libre y un método de pago configurado por tenant (Nequi/Daviplata/efectivo,
  spec 024).

## Escenario 1 — Validar un pago QR sin el botón que falla (User Story 1)

1. Desde el menú QR (o directamente vía `POST /cart/submit`), crear un pedido en una mesa y subir
   un comprobante de transferencia.
2. Abrir esa mesa en la Terminal de Mesas.
   - **Esperado**: un único bloque "Validación de Pago Requerida" (no dos pestañas).
3. Pulsar "Ver Comprobante".
   - **Esperado**: la imagen se muestra en una vista ampliada sin salir de la pantalla (D4).
4. Pulsar "Confirmar y Enviar a Cocina".
   - **Esperado**: el pedido queda pagado, visible en cocina, y se genera la factura — sin ningún
     error de "pago no confirmado".
5. Mirar la barra lateral derecha.
   - **Esperado**: no aparece "Cobrar y cerrar mesa" ni selector de método de pago (FR-006).

## Escenario 2 — Mesa QR con varios comensales, uno confirmado y otro pendiente (User Story 1, clarificación por comensal)

1. Dos comensales en la misma mesa suben cada uno su propio comprobante.
2. Confirmar el pago de uno solo.
   - **Esperado**: los ítems de ese comensal ya están visibles en cocina; la insignia de la mesa
     sigue en "Por confirmar" (amarillo) porque el otro comensal sigue pendiente.
3. Confirmar (o rechazar y resolver) el segundo.
   - **Esperado**: la insignia pasa a "En preparación" (azul) recién ahí.

## Escenario 3 — Crear y cobrar una orden manual (User Story 2)

1. Seleccionar una mesa libre; pulsar "+ Crear Orden Manual" (o F3).
   - **Esperado**: se abre la construcción de pedido; la barra lateral pasa a modo
     "Terminal POS / Cobro Inmediato" (no "Resumen de Cuenta").
2. Agregar productos desde el catálogo.
3. Seleccionar "Efectivo", ingresar un monto mayor al total.
   - **Esperado**: el cambio se calcula y se muestra antes de confirmar.
4. Pulsar "Cobrar, Facturar y Enviar a Cocina".
   - **Esperado**: en una sola acción — pago registrado, factura emitida, pedido visible en cocina
     (verificar contra `POST /orders/{id}/checkout-and-send`, contrato en
     `contracts/api-contracts.md`).
5. Repetir con "Transferencia/Datafono" en vez de efectivo.
   - **Esperado**: no se pide comprobante ni aparece ningún paso de revisión (FR-009).

## Escenario 4 — Reimpresión, pre-cuenta y liberación de mesa (User Story 3)

1. Sobre una mesa con pago QR aún pendiente, pulsar "Imprimir Pre-cuenta".
   - **Esperado**: imprime el desglose actual sin cambiar el estado de pago.
2. Sobre una mesa ya pagada y facturada, pulsar "Reimprimir Factura POS".
   - **Esperado**: reimprime el mismo documento; no aparece una segunda venta en
     `GET /invoices?order_id=`.
3. Con la mesa ya pagada y sin comensales activos, pulsar "Cerrar Mesa" / "Liberar Mesa".
   - **Esperado**: la mesa vuelve a "Libre" de inmediato (verificar
     `POST /table-sessions/{id}/release`, no `close_session`).
4. Repetir el paso 3 sobre una mesa con algo todavía por cobrar.
   - **Esperado**: `409` con el motivo ya definido en spec 010 (nada que liberar).

## Escenario 5 — Orígenes mixtos bloqueados (Edge Cases, clarificación de simetría)

1. Sobre una mesa con una orden manual activa, intentar que un comensal escanee el QR de esa mesa
   (o invocar `POST /cart/submit` directamente).
   - **Esperado**: `409`, el pedido QR no se crea.
2. Sobre una mesa con un pedido QR activo, intentar "+ Crear Orden Manual".
   - **Esperado**: la acción no está disponible o devuelve `409` — misma regla, dirección opuesta.

## Escenario 6 — Insignias del listado de mesas (User Story 4)

1. Dejar simultáneamente: una mesa con pago QR pendiente, una con pedido ya en cocina, y una libre.
2. Observar el listado lateral de mesas.
   - **Esperado**: "Por confirmar" (amarillo) / "En preparación" (azul) / "Libre" (gris o verde),
     cada una con texto además de color.

## Verificación automatizada (por repo)

- **Backend** (`pos-backend`): `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  — deben seguir pasando los characterization tests existentes de `table_sessions`/`orders`
  (ninguno de los endpoints reutilizados cambia de comportamiento); agregar tests nuevos para
  `checkout-and-send` y `release` junto a los existentes de `close_session`/`pay_order`.
- **Frontend** (`pos-heladeria`): `ng test` — actualizar/agregar specs de
  `session-bill-panel.component`, `pending-orders-panel.component`,
  `payment-attempt-review-panel.component` y `pos-terminal.store` para cubrir los dos modos de
  barra lateral (FR-005) y la insignia por mesa (FR-014).
