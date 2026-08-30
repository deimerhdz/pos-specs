# Quickstart: Validación de la Terminal de Mesas — carga diferida y tarjetas de pedido

Guía para verificar manualmente, de punta a punta, que las tres historias de usuario del spec
(`spec.md`) funcionan una vez implementadas. No sustituye los tests unitarios de
`pos-terminal.store.spec.ts`/`pos-tables-panel.component.spec.ts`/etc. (Principio X de la
Constitución) — es la verificación de comportamiento observable en el navegador.

## Prerrequisitos

- `pos-backend` corriendo localmente (o apuntando al backend de desarrollo ya configurado en
  `environment.ts`), con al menos un tenant con turno de caja abierto y algún método de pago
  configurado (spec 032).
- `pos-heladeria` con dependencias instaladas (`npm install`, ya asumido si el repo está en uso).
- Sesión de staff con acceso a "Terminal de mesas" (`/dashboard/mesas-sesiones`).

## Arrancar la app

```bash
cd pos-heladeria
npm start   # ng serve — http://localhost:4200 por defecto
```

Abrir las herramientas de desarrollador del navegador (pestaña Network) antes del siguiente paso.

## Escenario 1 — Carga diferida (Historia 1, FR-001 a FR-004)

1. Navegar a "Terminal de mesas" con la pestaña Network abierta y el filtro puesto en
   `Fetch/XHR`.
2. **Verificar** en la lista de peticiones disparadas al cargar: aparecen `tables`, `orders`
   (o equivalente `active_sessions_only=true`), la carga del menú/catálogo y de promociones
   activas — **no aparece** ninguna petición a `sales/payment-methods` (ni
   `?available=true`) ni a turno de caja (`cash-shifts/...`).
3. Seleccionar una mesa **libre** (sin pedido). **Verificar**: sigue sin aparecer ninguna petición
   de métodos de pago ni turno de caja; el panel derecho solo muestra "+ Crear pedido nuevo".
4. Seleccionar una mesa **con pedido activo** (o crear uno primero desde otra mesa). **Verificar**:
   en ese momento aparecen, por primera vez, las peticiones a métodos de pago (`load` +
   `?available=true`) y turno de caja — exactamente una vez cada una.
5. Seleccionar una segunda mesa con pedido. **Verificar**: no se repiten esas peticiones (se
   reutiliza lo ya cargado).

**Resultado esperado**: la grilla de mesas y el conteo/total de cada tarjeta se ven correctos
desde el primer render (sin "saltos" de precio), pero las peticiones de cobro solo aparecen tras
seleccionar un pedido real.

## Escenario 2 — Tarjetas de pedido Domicilio/Para llevar (Historia 2, FR-005 a FR-009)

1. Ir a "Terminal de mesas" → botón "Pedido de mostrador" (o F3) → pantalla de creación manual.
2. Elegir la pestaña "🛍️ Para Llevar", agregar al menos un producto, confirmar el pedido (dejar el
   nombre "Consumidor final" por defecto). Repetir una vez más con "🛵 Domicilio", diligenciando
   cliente/dirección/valor del domicilio.
3. Volver a "Terminal de mesas" (navegación automática tras crear el pedido).
4. Abrir la pestaña "Para llevar". **Verificar**: aparece una tarjeta con el mismo formato visual
   que las tarjetas de mesa (insignia de estado, cantidad de productos, total), mostrando
   "Consumidor final" como referencia de cliente.
5. Abrir la pestaña "Domicilios". **Verificar**: aparece la tarjeta del pedido de domicilio, con el
   nombre de cliente diligenciado — no mezclada con la de "Para llevar".
6. Abrir una pestaña de un tipo sin pedidos pendientes. **Verificar**: sigue mostrando el mensaje
   informativo de listado vacío ya existente (sin regresión).

## Escenario 3 — Seleccionar y cobrar un pedido sin mesa (Historia 3, FR-010 a FR-013)

1. Desde el estado del Escenario 2, seleccionar la tarjeta "Para llevar" creada.
2. **Verificar**: el panel central muestra el detalle del pedido (productos, cantidades, precio,
   estado) — mismo diseño que el panel de una mesa con pedido.
3. **Verificar**: el panel derecho ofrece el flujo de cobro completo (método de pago, cálculo de
   cambio) para ese pedido específico — cerrando la brecha donde antes no había ninguna forma de
   volver a encontrar el pedido tras crearlo.
4. Completar el cobro. **Verificar**: el pedido desaparece de la pestaña "Para llevar" (FR-008).
5. Seleccionar la tarjeta "Domicilio" creada. **Verificar** además que el panel de detalle muestra
   dirección, teléfono (si se diligenció) y valor del domicilio (FR-012).
6. Con una mesa previamente seleccionada, seleccionar en cambio una tarjeta de pedido sin mesa.
   **Verificar**: la mesa queda deseleccionada (FR-013) — no quedan dos selecciones resaltadas a la
   vez.

## Verificación automatizada (referencia)

```bash
cd pos-heladeria
npm test -- --run src/app/modules/tables
```

Debe incluir, como mínimo (tareas de la fase de implementación, no de este documento):
- `pos-terminal.store.spec.ts`: contrato de carga diferida (Contrato 3, `contracts/ui-contracts.md`)
  y de `hasActiveSelection`/`ordersByType` (Contrato 2).
- `order-summary-card.component.spec.ts` (nuevo): contrato de inputs/outputs (Contrato 1).
- `pos-tables-panel.component.spec.ts`: filtrado por pestaña y reutilización del componente de
  tarjeta.
- `pos-order-panel.component.ts` / `pos-checkout-panel.component.ts`: siguen pasando sus specs
  existentes sin regresión (ninguno de sus tests actuales queda en rojo por este cambio).
