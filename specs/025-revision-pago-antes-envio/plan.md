# Implementation Plan: Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

**Branch**: `025-revision-pago-antes-envio` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/025-revision-pago-antes-envio/spec.md`

## Summary

Hoy, `POST /cart/submit` (`app/api/v1/cart/service.py:490-568`, `pos-backend`) crea la
`CustomerOrder` de inmediato, sin ningún dato de pago — el comensal elige método y (si aplica)
sube su comprobante *después*, sobre una orden que ya existe, vía `POST
/cart/orders/{order_id}/payment-attempts` + `POST /cart/payment-attempts/{id}/receipt/presign` +
`POST /cart/payment-attempts/{id}/receipt` (spec 024, ya implementado). Esta spec invierte ese
orden: el pedido deja de crearse al enviar el carrito y pasa a crearse **junto con** su primer
intento de pago, en una sola operación atómica, solo cuando el comensal completa el paso de pago
que le corresponde (confirmar efectivo, o cargar el comprobante de una transferencia).

El cambio central es fusionar "crear la orden" + "crear su primer intento de pago" en
`POST /cart/submit`, que gana un cuerpo obligatorio (`payment_method_id`, y `receipt_file_url`
para transferencia). Para que el comensal pueda subir un comprobante *antes* de que exista
cualquier orden o intento al que asociarlo, se agrega un presign genérico
(`POST /cart/payment-receipt/presign`) que no depende de ningún recurso previo — a diferencia del
presign actual, ligado a un `attempt_id` ya existente (que se conserva, sin cambios, para el
reintento tras un rechazo, spec 024 Historia 5, el único caso donde sí existe ya una orden). La
garantía de "nunca dos pedidos por una misma confirmación" (FR-013) se obtiene subiendo el chequeo
ya existente de "una orden activa por comensal" (spec 024, hoy solo una validación de aplicación)
a un índice único parcial real en la base de datos — la misma garantía cubre ambos requisitos con
un solo mecanismo, sin inventar una clave de idempotencia nueva.

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21.1.x, standalone components).

**Primary Dependencies**: Backend: FastAPI, SQLAlchemy 2.0, Alembic, boto3 (Cloudflare R2, ya en
uso). Frontend: Angular 21, `@tanstack/angular-query-experimental`, Tailwind 4. Ninguna
dependencia nueva — todo lo que requiere esta spec (índice único parcial de Postgres, un presign
más sin ligar a un recurso) ya está disponible con lo que el proyecto usa hoy (Principio IX: no
aplica justificación).

**Storage**: PostgreSQL 16 schema-per-tenant. Cambio de esquema acotado a un índice único parcial
nuevo sobre `customer_orders(participant_id)` — ninguna tabla ni columna nueva; `OrderPaymentAttempt`
(spec 024) no cambia de forma, solo cambia *cuándo* se crea su primera fila. Cloudflare R2 para el
archivo del comprobante, reutilizando `app/core/storage.py` sin cambios.

**Testing**: `unittest` vía `python -m unittest`, extendiendo `app/characterization_tests/
cart_fixtures.py` y los tests nuevos de spec 024 (`test_cart_payment_attempts.py`,
`test_cart_single_active_order.py`) — no se crean ficheros de test nuevos, se amplían los ya
existentes porque ejercitan exactamente las mismas funciones (`submit_cart`, la garantía de una
orden activa) que esta spec modifica. Frontend: Vitest (mismo patrón ya en uso, aunque spec 024 no
llegó a escribir specs de los componentes nuevos — sigue pendiente, no se agrava aquí).

**Target Platform**: Linux server (`pos-backend`) + navegador móvil del comensal (`pos-heladeria`).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo nuevo — se fusionan dos peticiones HTTP secuenciales
(`submit_cart` + `create_payment_attempt`) en una sola; si acaso reduce una llamada de red por
pedido enviado en efectivo respecto a spec 024.

**Constraints**:
- No se agrega ningún valor nuevo al `CHECK` de `CustomerOrder.status` — no existe un estado
  "borrador"/"draft" para el pedido antes de pagar; simplemente el pedido no existe todavía
  (research.md, Decisión 3). Mantiene intacto el mismo invariante que ya protegió spec 024.
- Los endpoints ya existentes para el reintento tras rechazo (`POST
  /cart/orders/{order_id}/payment-attempts`, `POST /cart/payment-attempts/{id}/receipt/presign`,
  `POST /cart/payment-attempts/{id}/receipt`) NO cambian — siguen siendo el único camino para un
  segundo intento sobre una orden que ya existe (spec 024, Historia 5).
- El contexto de tenant/mesa/comensal en el presign genérico nuevo viene siempre del
  `x-session-token` firmado (`get_session_context`), igual que todo el resto de `cart/router.py`
  (invariante A-24, spec 007).
- Ningún cambio toca `checkout.py` (aprobar/rechazar/confirmar-efectivo/`confirm_order`) — esa
  parte del flujo, ya construida por spec 024, permanece exactamente igual; el pedido que llega
  ahí ya viene siempre con su primer intento de pago adjunto.

**Scale/Scope**: 1 migración (índice único parcial), 1 endpoint nuevo (presign genérico), 1
endpoint modificado (`POST /cart/submit` gana cuerpo obligatorio), ~4 funciones de servicio
tocadas en `pos-backend`; en `pos-heladeria`, la pantalla de "Enviar pedido" se reemplaza por un
flujo de revisión + pago (reutilizando casi todo el UI que spec 024 ya construyó para el
modal de pago, solo movido antes de la creación del pedido en vez de después).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 13 FRs y 3 clarificaciones resueltas (sesión 2026-08-18) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento que cambia es cuándo se crea la `CustomerOrder` (antes: al enviar el carrito, sin pago; ahora: junto con su primer intento de pago) — cambio de negocio explícito, documentado en Assumptions de `spec.md` como autorizado por esta spec de evolución funcional (Principio IV), no una corrección de anomalía heredada. `checkout.py` (todo lo posterior a que el pedido existe) no se toca. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún test `"CONGELA comportamiento actual:"` cubre `submit_cart`/la garantía de una orden activa (son tests nuevos de spec 024, sin ese prefijo). Los tests de spec 024 que sí ejercitan la forma actual de `submit_cart()` (sin pago) y de `create_payment_attempt` como primer intento se actualizan explícitamente en esta spec, citándola, con evidencia de que el resto de la suite sigue en verde (research.md, Decisión 5). | PASS (condicionado a verificar en implementación) |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Reordenar cuándo nace el pedido es comportamiento nuevo explícito, no una corrección — conforme a `spec.md`, sin exigir equivalencia con el orden anterior. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | No se toca `checkout.py`, `orders/router.py`, el panel de revisión del cajero (`payment-attempt-review-panel.component.ts`) ni ningún endpoint de aprobar/rechazar/confirmar-efectivo — todo eso ya construido por spec 024 se reutiliza tal cual. Los endpoints de reintento tras rechazo tampoco se tocan. | PASS |
| **VI. Evolución Incremental** | Un solo cambio de comportamiento (momento de creación del pedido) con un mecanismo (índice único parcial) que resuelve dos requisitos relacionados (FR-011, FR-013) sin mezclar con ninguna migración de datos históricos ni cambio de arquitectura. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`Invoice` ni ninguna venta ya facturada. Pedidos `recibida` ya existentes antes de este cambio (sin intento de pago, creados bajo el flujo viejo) no se recalculan ni se les fuerza un intento retroactivo — quedan como están, fuera de alcance (ver Edge Cases). | PASS |
| **VIII. Evolución del Modelo de Datos** | Índice único parcial nuevo sobre `customer_orders(participant_id) WHERE status NOT IN ('pagada', 'cancelada')` — ninguna tabla/columna nueva, ninguna fila existente cambia de significado (los `NULL` de `participant_id`, órdenes de mostrador/mesero, quedan excluidos automáticamente por semántica de índice único en Postgres). Migración vía `@for_each_tenant_schema`, con `downgrade()` que solo elimina el índice — ver research.md y data-model.md. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables sobre la suite ya existente de spec 024, extendida. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La decisión de negocio (el pedido nace con el pago resuelto, no antes) está en `spec.md`. La decisión técnica de *cómo* garantizarlo (índice único parcial en vez de una clave de idempotencia nueva) es de este plan, justificada en research.md sin inventar ninguna regla de negocio nueva. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` → este `plan.md`/`research.md` → `tasks.md` (Fase 2, no generada aquí) → implementación → tests actualizados/nuevos → `quickstart.md`. | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/025-revision-pago-antes-envio/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md        # Fase 1 (/speckit-plan) — el único cambio de esquema (índice único parcial)
├── quickstart.md        # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados
│   ├── submit-cart-with-payment.md
│   └── payment-receipt-presign.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Rutas relativas a la raíz de cada repo (`../pos-backend`, `../pos-heladeria`).

```text
# pos-backend
app/
├── models/
│   └── customer_order.py             # MODIFICADO — nuevo Index() único parcial en __table_args__
│
├── api/v1/cart/
│   ├── schemas.py                    # MODIFICADO — SubmitCartIn (payment_method_id,
│   │                                    receipt_file_url?); PaymentReceiptPresignIn reexporta
│   │                                    ReceiptPresignIn (mismo shape, sin campos nuevos)
│   ├── service.py                    # MODIFICADO — submit_cart(db, participant, payment_method_id,
│   │                                    receipt_file_url=None) crea Orden + primer
│   │                                    OrderPaymentAttempt en una sola transacción, capturando
│   │                                    IntegrityError del índice nuevo; presign_payment_receipt()
│   │                                    NUEVO (sin attempt_id); create_payment_attempt/
│   │                                    presign_receipt/attach_receipt SIN CAMBIOS (solo reintento)
│   └── router.py                     # MODIFICADO — POST /cart/submit exige body; POST
│                                        /cart/payment-receipt/presign NUEVO
│
├── characterization_tests/
│   ├── cart_fixtures.py              # MODIFICADO — helper para sembrar el índice nuevo si algún
│   │                                    test necesita dos participantes/dos órdenes a propósito
│   ├── test_cart_payment_attempts.py  # MODIFICADO (spec 024) — los tests que asumían
│   │                                    submit_cart()+create_payment_attempt() como dos pasos se
│   │                                    actualizan al nuevo submit_cart(method_id, receipt_url?)
│   └── test_cart_single_active_order.py # MODIFICADO (spec 024) — agrega el caso de violación
│                                          concurrente del índice (dos submits casi simultáneos)
│
└── alembic/versions/
    └── {rev}_active_order_per_participant.py  # NUEVO — índice único parcial vía
                                                  @for_each_tenant_schema

# pos-heladeria
src/app/modules/tables/
├── interfaces/diner.interface.ts      # MODIFICADO — SubmitCartPayload nuevo; tipos de intento de
│                                         pago existentes (spec 024) sin cambios de forma
├── services/diner.service.ts          # MODIFICADO — submitCart(methodId, receiptUrl?);
│                                         presignPaymentReceipt() nuevo; resto sin cambios
└── pages/public-menu.component.ts     # MODIFICADO — "Enviar pedido" abre la pantalla de
                                          revisión+pago (reutiliza el modal de selección de método
                                          y carga de comprobante que spec 024 ya construyó, ahora
                                          antes de crear el pedido); se retira el botón "Elegir
                                          cómo pagar" del historial (ya no aplica: todo pedido nace
                                          con su intento adjunto); el flujo de "Reintentar pago"
                                          tras un rechazo (spec 024 Historia 5) permanece igual
```

**Structure Decision**: todo el cambio vive dentro de los mismos tres ficheros de `cart` que ya
tocó spec 024 (`schemas.py`, `service.py`, `router.py`) más una migración — no se crea ningún
paquete nuevo. `checkout.py`, `orders/router.py` y el panel del cajero quedan explícitamente sin
tocar (Principio V), porque el pedido que llega ahí ya trae su intento de pago por construcción.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
