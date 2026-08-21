# Research: Rediseño Híbrido de la Terminal de Mesas

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-08-20

Este documento resuelve las decisiones técnicas necesarias para implementar el spec, descubiertas
al inspeccionar el estado real de `../pos-backend` (FastAPI + PostgreSQL 16, schema-per-tenant) y
`../pos-heladeria` (Angular 21, standalone components, signal stores). Ninguna de estas decisiones
reabre alcance de negocio ya cerrado en el spec (Principio XI de la
[Constitución](../../.specify/memory/constitution.md): son decisiones **técnicas** de cómo
implementar lo que el spec ya definió, no decisiones de negocio nuevas).

## D1: El ciclo de vida de la orden manual debe diferir del de `POST /orders` actual

**Decisión**: la creación de una orden manual desde la nueva Terminal de Mesas usa un modo
aditivo de `POST /orders` (campo opcional nuevo, p. ej. `hold_for_payment: bool`, default
`false`) que la crea en `status="recibida"` en vez de `"abierta"` — sin inventario descontado ni
visibilidad en cocina — hasta que el cobro (FR-011) la confirma. Toda llamada existente a
`POST /orders` que no envíe ese campo conserva exactamente el comportamiento actual (creación
directa en `abierta`, inventario descontado de inmediato). Las consultas de "pagos por confirmar"
(cola de comprobantes QR) se filtran además por `channel == "qr"`, para que una orden manual en
`recibida` nunca aparezca en el bloque de "Validación de Pago Requerida" (que exige comprobante o
revisión, algo que FR-009 dice explícitamente que un pago manual por transferencia/datáfono NO
necesita).

**Razón**: `app/api/v1/orders/service.py::create_order` hoy crea **toda** orden de staff
(`channel in {counter, waiter}`) directamente en `status="abierta"` con el inventario descontado
en el mismo paso (visible para cocina de inmediato) — es el modelo de "mesero abre una cuenta,
cobra al final" (coherente con `block_order`, que de hecho **rechaza** bloquear si quedan ítems
`pendiente`/`en_preparacion`: solo permite cobrar una orden cuya comida ya terminó o no se envió).
El spec 028 (FR-011) pide lo contrario para el flujo manual: cobrar y enviar a cocina **en la misma
acción**, con la comida sin empezar a prepararse todavía — el mismo patrón "pagar primero, cocina
después" que spec 026 ya implementó para QR (`recibida` → `_confirm_order_impl` dispara
inventario + cocina al confirmarse el pago, no al crear el pedido). Reutilizar ese patrón
comprobado, en vez de inventar uno nuevo, es la opción de menor riesgo.

**Alternativas consideradas**:
- *Reusar `create_order` + `block_order` + `pay_order` tal cual*: descartada — `block_order`
  rechaza bloquear mientras haya ítems `pendiente`/`en_preparacion`, exactamente el estado en que
  estaría una orden manual recién creada y aún no cobrada; el flujo se bloquearía a sí mismo.
- *Cambiar el comportamiento por defecto de `create_order` para todo `channel != qr`*: descartada
  — cambiaría comportamiento existente de cualquier otro caller de `POST /orders` sin decisión de
  negocio que lo autorice (Principio II); el campo opcional aditivo evita ese riesgo.

## D2: "Cerrar Mesa" (FR-016) necesita un endpoint nuevo, no reutilizar `close_session` tal cual

**Decisión**: nuevo servicio `release_paid_session` (y endpoint `POST
/table-sessions/{id}/release`) que reutiliza los mismos bloques ya existentes —
`_load(..., lock=True)` (mismo lock de fila `RN-MESA-01`), `_assert_closable` (mismos dos motivos
de rechazo: pedidos `recibida` sin confirmar, ítems de cocina sin terminar) y el helper ya
compartido `checkout.close_table_sessions` — pero **sin** exigir `_billable_orders` no vacío ni
recibir `payments`/`billing_mode`: solo libera si **no** queda nada por cobrar.

**Razón**: `close_session` (spec 010) exige `_billable_orders(db, ts.id)` no vacío y devuelve
`409 "La sesión no tiene pedidos que cobrar"` en caso contrario — exactamente el estado en que
queda una mesa después de que esta misma spec confirma sus pagos automáticamente (todas sus
órdenes ya en `pagada`). Llamar a `close_session` sobre una mesa así, tal como está hoy, siempre
falla. FR-016 pide una liberación **pura** (nada que cobrar, solo cerrar sesión y devolver la mesa
a `libre`), que es semánticamente la condición inversa a la que `close_session` verifica.

**Alternativas consideradas**:
- *Añadir un flag a `close_session` que salte el chequeo de `_billable_orders`*: descartada por
  mezclar dos contratos distintos (cobrar-y-cerrar vs. solo-liberar) en un mismo endpoint,
  reabriendo el mismo tipo de ambigüedad de UI que esta spec busca eliminar del lado del backend.
- *Confiar únicamente en el barrido automático*: descartada — es exactamente la opción que la
  clarificación del spec (sesión 2026-08-20, primera pregunta) rechazó explícitamente.

## D3: "Cobrar, Facturar y Enviar a Cocina" (FR-011) es un endpoint nuevo, no `block_order`+`pay_order`

**Decisión**: nuevo endpoint atómico (p. ej. `POST /orders/{id}/checkout-and-send`) que, en una
sola transacción, construye la venta/factura (reutilizando `build_sale`, igual que `pay_order`) y
ejecuta la misma transición `recibida → abierta` que ya usa `_confirm_order_impl` (descuento de
inventario + visibilidad en cocina), sobre una orden manual creada con `hold_for_payment=true`
(D1). Reutiliza el lock optimista por `version` ya presente en `BlockIn`/`CustomerOrder.version`
para la misma protección contra doble ejecución que ya exige spec 024 FR-018.

**Razón**: como en D1, `block_order` y `pay_order` asumen que la comida de la orden **ya** terminó
de prepararse (o nunca se envió) antes de cobrar — el orden inverso al que pide FR-011
("registrar el pago... y enviar el pedido a cocina" en un solo paso, antes de que la cocina
reciba nada). Encadenar ambos endpoints tal como existen no logra ese comportamiento sin además
violar la regla de `block_order` sobre ítems en curso.

**Alternativas consideradas**:
- *Frontend encadena `block_order` → `pay_order`*: descartada por el motivo anterior (fallaría
  siempre, dado que D1 deja la orden en `recibida`, no `abierta`, y con ítems aún no enviados a
  cocina) y porque dos llamadas HTTP separadas reintroducen la ventana de "pagado pero no
  facturado" que spec 026 FR-002 ya prohíbe explícitamente para el flujo QR — este endpoint nuevo
  la evita por construcción (una sola transacción).

## D4: "Ver Comprobante" pasa de enlace `target="_blank"` a vista ampliada en la misma pantalla

**Decisión**: nuevo componente de previsualización (modal) en el frontend que envuelve la imagen
ya servida hoy por el backend (`receipt_file_url` en `OrderPaymentAttempt`); no requiere cambios
de backend.

**Razón**: `payment-attempt-review-panel.component.ts` hoy abre el comprobante en una pestaña
nueva del navegador (`target="_blank"`); el spec (FR-002, escenario de aceptación 2) exige una
vista ampliada "sin salir de la pantalla de la mesa" — un cambio de UI puro, no de datos.

## D5: Un solo significado de "Rechazar" — el del intento de pago, no el de la orden completa

**Decisión**: el bloque central "Validación de Pago Requerida" usa exclusivamente el rechazo a
nivel de intento de pago ya implementado (`reject_payment_attempt` / endpoint
`POST /orders/payment-attempts/{id}/reject`, que exige motivo). El botón "Rechazar" que hoy vive
en `pending-orders-panel.component.ts` y cancela la orden completa (`cancelOrder`) se retira de la
UI al consolidar ambas pestañas en un único bloque (FR-001), porque haría convivir dos acciones
llamadas igual con efectos muy distintos.

**Razón**: FR-002 define "Rechazar" como una acción sobre el comprobante/intento de pago, con
motivo, que permite reintentar — no como cancelar el pedido completo; el spec no pide ni autoriza
esa segunda semántica, así que no debe quedar accesible bajo el mismo texto de botón en la interfaz
consolidada.

## D6: Impresión — se reutiliza el mecanismo cliente existente sin cambios; "Pre-cuenta" es una plantilla nueva

**Decisión**: `receipt.util.ts` / `printer-settings.store.ts` (generación de HTML + `<iframe>` +
`window.print()`, ancho configurable 48/58/80mm) se reutilizan sin modificar para FR-003
(impresión automática al confirmar) y FR-012 (reimpresión, que ya es una simple regeneración desde
`Sale`/`Invoice`, sin endpoint de backend dedicado — el modelo ya está pensado para eso,
`Sale.invoice` es `viewonly` explícitamente para reimprimir). "Imprimir Pre-cuenta" (FR-007) es una
plantilla nueva construida desde `SessionBillResponse`/`BillResponse` (no existe `Sale` todavía
antes del pago), reutilizando el mismo mecanismo de impresión.

**Razón**: el spec pide expresamente mantener el módulo de impresión "tal como funcionaba en la
versión anterior" (fuera de alcance cualquier cambio al motor de impresión); el ancho de papel por
defecto observado en código es 48mm, no 58/80mm — se preserva sin cambios, ya que el spec no pide
modificar el motor de impresión, solo mantenerlo.

## D7: Modo de la barra lateral — puramente derivado del campo `channel` ya existente

**Decisión**: FR-005 se implementa enteramente en el frontend, derivando el modo
("Resumen de Cuenta" vs. "Terminal POS / Cobro Inmediato") del `channel` (`qr | counter | waiter`)
de la(s) orden(es) activa(s) de la sesión de mesa. No requiere migración de base de datos.

**Razón**: `CustomerOrder.channel` ya existe en el modelo, con CHECK constraint
`channel IN ('qr', 'counter', 'waiter')` y ya se serializa en `OrderResponse.channel` — confirma la
suposición del spec de que el origen de la orden ya es un dato disponible.

## D8: Actualizaciones en vivo de las insignias de mesa — se reutiliza el stream de eventos ya existente

**Decisión**: FR-014 (insignias por comensal/mesa) y la actualización en vivo del listado de mesas
se apoyan en el stream SSE ya existente (`app/core/events.py`, Redis Streams por tenant,
consumido en el frontend por `realtime.service.ts`), que ya emite `payment_completed`,
`session_closed`, `table_status_changed`. Los endpoints nuevos de D2 y D3 deben emitir los mismos
eventos que ya emite `close_session`/`_confirm_order_impl`, para que el listado de mesas se
actualice sin polling adicional.

**Razón**: evita construir infraestructura de tiempo real nueva cuando ya existe una que cubre
exactamente esta necesidad.

## Resumen de Technical Context

| Aspecto | Backend (`pos-backend`) | Frontend (`pos-heladeria`) |
|---|---|---|
| Lenguaje/versión | Python 3.12, FastAPI 0.136.3, Pydantic 2.13 | TypeScript, Angular 21.1 (standalone, signals) |
| Dependencias clave | SQLAlchemy 2.0.50, Alembic 1.18.4, Redis 8 (streams + Celery), Celery 5.6.3 + APScheduler | Tailwind CSS 4.1, `@tanstack/angular-query-experimental` (no usado en esta pantalla), sin librería de componentes |
| Almacenamiento | PostgreSQL 16, schema-per-tenant | localStorage (config de impresora) |
| Testing | `unittest` stdlib, `app/characterization_tests/*.py`, SQLite en memoria, marcador `"""CONGELA comportamiento actual` | Vitest vía `ng test` (`@angular/build:unit-test`), specs co-ubicados `*.spec.ts`; Playwright presente como dependencia pero sin e2e configurado todavía |
| Target | API HTTP (Linux server) | Navegador — mismo diseño debe funcionar en tablet táctil y escritorio (spec 026, reutilizado) |
| Proyecto | Servicio web (API) | Aplicación web (SPA) |
