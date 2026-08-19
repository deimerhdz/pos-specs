# Quickstart: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

Validación ejecutable por historia de usuario. Los comandos asumen que se ejecutan desde la raíz
de cada repo (`../pos-backend`, `../pos-heladeria`, siblings de `pos-specs`).

## Prerrequisitos

- `pos-backend`: entorno virtual activado (`source env/bin/activate` o equivalente), base de datos
  de pruebas disponible (los characterization tests usan SQLite en memoria vía `fixtures.py`, no
  requieren Postgres real).
- `pos-heladeria`: `npm install` ya ejecutado.
- Spec, plan, research y data-model de esta funcionalidad ya revisados (no hay migraciones que
  aplicar — data-model.md).

## Historia 1 — Confirmar el pago envía el pedido a cocina en un solo paso

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_orders_checkout -v
python -m unittest app.characterization_tests.test_orders_payment_gate -v
```

- **Esperado**: el CONGELA de `confirm_order` (`test_orders_checkout.py`) sigue en verde con la cita
  de autorización actualizada (spec 026, FR-001) en el mismo commit que lo modifica.
- **Esperado (nuevo caso, `test_orders_payment_gate.py`)**: confirmar un pago en efectivo (o
  aprobar un comprobante) sobre una orden con stock suficiente deja, en una sola llamada, el
  intento `confirmado` **y** la orden en `abierta` con el inventario descontado — sin ninguna
  segunda llamada a `confirm_order`.
- **Esperado (fallo de stock, US1 escenario 3 / FR-002)**: confirmar un pago sobre una orden cuyo
  stock ya no alcanza deja el intento **sin** confirmar (sigue `pendiente`) y la orden sigue
  `recibida` — ninguna de las dos cosas ocurre a medias.

```bash
cd ../pos-heladeria
npx vitest run src/app/modules/tables/components/pending-orders-panel.component.spec.ts
```

*(este archivo de spec no existe todavía — se crea como parte de la implementación de esta
historia; hoy `pending-orders-panel.component.ts` no tiene cobertura de test propia).*

- **Esperado**: no existe ningún botón "Confirmar" ni llamada a `confirmOrder()` disparada desde
  este panel; el botón "Rechazar" sigue funcionando igual que antes.

## Historia 2 — El cajero ve el cambio a entregar al confirmar un pago en efectivo

```bash
cd ../pos-heladeria
npx vitest run src/app/modules/tables/components/payment-attempt-review-panel.component.spec.ts
```

*(este archivo de spec no existe todavía — se crea como parte de la implementación de esta
historia).*

- **Esperado**: tras confirmar un pago en efectivo con monto mayor al total, la vista persistente
  del intento confirmado (no solo el toast) muestra el monto recibido y el cambio calculado; con
  monto exacto, muestra el cambio como "$0" en vez de omitir el dato.
- **Manual (no automatizable por tipo)**: reabrir un pedido cuyo pago en efectivo ya fue confirmado
  hace rato y verificar que el monto recibido y el cambio siguen visibles sin tener que recordarlos.

## Historia 3 — Dividir la cuenta y cobrar con varios métodos de pago

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_table_sessions_service -v
python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v
```

- **Esperado**: ambos módulos CONGELA siguen en verde sin ninguna modificación — esta historia no
  toca `compute_bill`/`close_session` (research.md, Decisión 7).

```bash
cd ../pos-heladeria
npx vitest run src/app/modules/tables/components/session-bill-panel.component.spec.ts
npx vitest run src/app/modules/tables/components/split-bill-panel.component.spec.ts
```

*(`split-bill-panel.component.spec.ts` no existe todavía — se crea al auditar esta historia contra
el comportamiento actual del componente; `session-bill-panel.component.spec.ts` ya existe y debe
seguir en verde).*

- **Esperado**: el placeholder "Selecciona una mesa con consumo" solo aparece cuando corresponde
  (regresión ya cubierta); al seleccionar una mesa con consumo aparece el detalle completo de la
  cuenta sin salir de la Terminal de Mesas (FR-006); dividir por comensal solo ofrece asignación
  por ítem/unidad, nunca porcentual (FR-007); cobrar combinando efectivo + otro método calcula el
  cambio únicamente sobre el excedente en efectivo (FR-008); al cerrar, la factura queda generada
  (FR-009).

## Historia 4 — Legibilidad en tablet y escritorio

No automatizable por tipo de criterio (tamaño de texto/controles, distinción de estado). Checklist
manual sobre el build servido (`npm start` en `pos-heladeria`), en un viewport de tablet y uno de
escritorio:

- [ ] El texto esencial (mesa, productos, total, estado) es legible sin zoom; verificar en
      DevTools que el tamaño computado no baja de 16px (FR-010).
- [ ] Los controles de acción (confirmar pago, aprobar/rechazar comprobante, cobrar) miden al menos
      44×44px en su área táctil (FR-011).
- [ ] Cada estado de pedido (pago pendiente, pago rechazado, pago confirmado y en cocina) se
      distingue por una etiqueta de texto o ícono además del color — cubrir la pantalla con un
      filtro de escala de grises (simulación rápida de daltonismo) y confirmar que los estados
      siguen siendo distinguibles (FR-003).
- [ ] Ninguna información visible hoy en la comanda (ítems, notas por producto, método de pago,
      estado del pedido) desapareció (FR-012).

## Regresión general

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: toda la suite de characterization tests sigue en verde, salvo las citas de línea ya
  actualizadas en `test_orders_checkout.py` (Historia 1) — ningún otro módulo (`test_orders_kitchen`,
  `test_orders_consolidation`, `test_cart_*`) cambia de resultado.
