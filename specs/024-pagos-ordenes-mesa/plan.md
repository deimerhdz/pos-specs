# Implementation Plan: Pagos de Órdenes en Mesa (Skeilopos)

**Branch**: `024-pagos-ordenes-mesa` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/024-pagos-ordenes-mesa/spec.md`

## Summary

El flujo QR existente (spec 007) hoy no tiene ningún paso de pago del lado del comensal: `POST
/cart/submit` crea la `CustomerOrder` en `recibida` y el único punto que hace avanzar una orden a
producción/cocina es `confirm_order` (`recibida → abierta`, `app/api/v1/orders/checkout.py`), que
hoy solo valida `status == "recibida"` y stock. La verificación "sin evidencia" que motiva esta
spec ocurre del lado del **cajero**, en el cobro de mesa (`POST
/table-sessions/{id}/close`/`checkout.pay_order`), donde el cajero teclea método+monto sin ningún
comprobante — no en una pantalla del comensal que haya que "quitar".

Esta spec agrega, delante de `confirm_order`, un flujo de pago por orden: (1) el tenant configura
sus métodos de pago (`payment_methods`, ya existe, se extiende con datos de transferencia); (2) el
comensal elige un método y, si es transferencia, sube un comprobante (un archivo, vía presign a R2)
que crea un **Intento de Pago** (`order_payment_attempts`, entidad nueva); (3) el cajero aprueba o
rechaza el comprobante, o confirma el efectivo con el monto recibido; (4) `confirm_order` exige un
intento de pago `confirmado` antes de dejar avanzar la orden a comanda (FR-017) — sin agregar un
estado nuevo a `CustomerOrder.status` ni un setter de estado genérico (invariante A-25, spec 008).

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21.1.x, standalone components, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync), Alembic 1.18.4, Pydantic 2.13.4, boto3
  1.43.48 (Cloudflare R2, ya en uso vía `app/core/storage.py`), PyJWT 2.13.0 (ya en uso para el
  token de sesión del comensal, `app/core/qr_context.py`). Ninguna dependencia nueva — todo lo que
  requiere esta spec (presign a R2, JSONB, índices parciales de Postgres) ya está disponible con lo
  que el proyecto usa hoy (Principio IX: no aplica justificación porque no se añade nada).
- Frontend: Angular 21 (standalone + signals), `@tanstack/angular-query-experimental` 5.101.4 para
  estado de servidor, Tailwind 4.1.12. Ninguna dependencia nueva.

**Storage**: PostgreSQL 16 schema-per-tenant (tabla nueva `order_payment_attempts` y columna nueva
`payment_methods.payment_info` en el esquema `tenant`, vía Alembic con `@for_each_tenant_schema`,
mismo patrón que `business_hours.py`/`payment.py`). Cloudflare R2 (S3-compatible vía `boto3`) para
el archivo del comprobante, reutilizando `app/core/storage.py` (`generate_presigned_put_url`,
`build_object_key`, `public_url_for`) con un folder nuevo (`"comprobantes"`) en la whitelist de
`PresignRequest.folder`.

**Testing**: `unittest` vía `python -m unittest` (sin pytest en el repo, mismo patrón que el resto
de `app/characterization_tests/`). Se agregan fixtures nuevas (`payment_attempts_fixtures.py` o
extensión de `orders_fixtures.py`) y módulos de test nuevos por historia de usuario, ejecutados en
SQLite en memoria con el `schema_translate_map` ya establecido en `fixtures.py`. Frontend: Vitest +
`@angular/build:unit-test`, specs colocados (`*.spec.ts`) junto a cada componente/servicio nuevo,
mismo patrón que `payment-draft.util.spec.ts`.

**Target Platform**: Linux server (API `pos-backend` en producción) + navegador (SPA `pos-heladeria`
en producción, incluyendo el navegador móvil del comensal que escanea el QR).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo — esta spec agrega escrituras puntuales
(un `INSERT` por intento de pago, un `PATCH` por aprobación/rechazo/confirmación) sobre un flujo ya
de baja concurrencia por mesa; el único requisito de rendimiento explícito es de UX (SC-002: alta de
un método nuevo visible en <2 min, ya cubierto por no requerir despliegue).

**Constraints**:
- No se agrega un valor nuevo al `CHECK` de `CustomerOrder.status` (protegido por characterization
  spec 017/008) — "pendiente de pago" se deriva de `status == "recibida"` sin intento confirmado,
  nunca se persiste como estado literal (ver research.md Decisión 1).
- No se agrega un endpoint de transición de estado genérica (invariante A-25) — el único punto que
  hace avanzar una orden a comanda sigue siendo `confirm_order`, extendido con una precondición
  nueva, no reemplazado.
- El contexto de tenant/mesa/comensal en cualquier endpoint nuevo del comensal viene siempre del
  `x-session-token` firmado (`get_session_context`), nunca de un parámetro de body/query (invariante
  A-24, spec 007).
- Ninguna factura ya emitida ni venta ya cobrada (`Sale`/`Payment`, `app/models/payment.py`) se
  toca — esta spec vive completamente *antes* de `checkout.pay_order`/`close_session`.
- Fuera de alcance explícito de `spec.md`: integración con pasarelas/bancos, cuenta consolidada por
  mesa, cancelación de órdenes, expiración de pagos pendientes, propinas/descuentos/pagos mixtos,
  restricciones de tamaño/formato de archivo.

**Scale/Scope**: 1 tabla nueva (`order_payment_attempts`), 1 columna nueva
(`payment_methods.payment_info`), ~8 endpoints nuevos + 2 modificados (`confirm_order`,
`submit_cart`) en `pos-backend`; 1 pantalla nueva del comensal (selección de método + comprobante),
1 panel nuevo del cajero (revisión de comprobantes + confirmación de efectivo) y 1 extensión de la
configuración de métodos de pago del tenant en `pos-heladeria`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 6 historias priorizadas, 18 FRs y 4 clarificaciones ya resueltas (sesión 2026-08-18) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento existente que cambia es `confirm_order`: gana una precondición nueva (intento de pago confirmado). Es un cambio de negocio explícito, definido en FR-017 del propio spec — no requiere entrada nueva en `registro-de-anomalias.md` porque no es una corrección de una anomalía heredada, es comportamiento nuevo autorizado por un spec de evolución funcional (Principio IV). `submit_cart` gana la restricción de una orden activa por comensal (FR-005/FR-006), también nueva y definida en el spec. Ningún otro camino de `checkout.py`/`table_sessions` cambia. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_orders_*`/`test_cart_*` con prefijo `"CONGELA comportamiento actual:"` (specs 015/016/017) no se modifican — se leen para confirmar que `confirm_order` solo valida `status`+stock hoy (research.md) y que `submit_cart` no valida "una orden activa" hoy. Si algún test CONGELA asume implícitamente "cualquier `recibida` con stock se confirma", ese test se actualiza citando esta spec + FR-017 en el mismo commit, con evidencia de que el resto de casos protegidos sigue en verde. | PASS (condicionado a verificar en Fase de implementación, no en este plan) |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Toda la spec es comportamiento nuevo (gating de comanda por pago, comprobantes, métodos configurables) — no se exige equivalencia con el pasado, solo conformidad con `spec.md` y ausencia de regresión en lo no tocado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | No se toca `payment-input.component.ts`/`payment-draft.util.ts` (cobro de mesa) ni `checkout.pay_order`/`close_session` — comparten el nombre "pago" pero son un flujo distinto (cobro de mesa consolidado, fuera de alcance de `spec.md`) y no se refactorizan para "unificar" con el intento de pago nuevo. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias de usuario del spec (config de métodos → intento+comprobante → efectivo → gate de comanda → reintento → una-orden-activa), cada una verificable por separado (ver Project Structure). No se mezcla con ninguna migración de datos histórica ni cambio de arquitectura. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta ya facturada — el flujo nuevo termina antes de que exista una `Sale`. | PASS |
| **VIII. Evolución del Modelo de Datos** | Ver data-model.md: tabla nueva `order_payment_attempts` (sin FK a datos existentes que cambien de significado), columna nueva `payment_methods.payment_info` (nullable, no rompe filas existentes), migraciones vía `@for_each_tenant_schema` con estrategia de rollback (`op.drop_table`/`op.drop_column`) explícita en research.md. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables, más characterization tests existentes de `cart`/`orders` como red de no-regresión. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Dos decisiones técnicas quedan explícitas en research.md para no confundirse con decisiones de negocio: (a) "pendiente de pago" es un estado derivado, no una columna nueva; (b) "cajero" se resuelve como cualquier usuario staff autenticado (`get_current_user`), no un rol nuevo — el spec no exige una restricción de rol distinta a "no es el comensal". | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec) → este `plan.md`/`research.md` (Decisión técnica) → tareas de `tasks.md` (Fase 2, no generada por este comando) → implementación → characterization tests + tests nuevos → quickstart.md (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/024-pagos-ordenes-mesa/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md        # Fase 1 (/speckit-plan) — entidades, columnas, transiciones, migraciones
├── quickstart.md        # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados
│   ├── tenant-payment-methods.md
│   ├── diner-payment-flow.md
│   ├── cashier-payment-review.md
│   └── order-confirm-gate.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── models/
│   ├── payment.py                    # MODIFICADO — PaymentMethod gana payment_info (JSONB null)
│   └── order_payment_attempt.py      # NUEVO — OrderPaymentAttempt (order_payment_attempts)
│
├── api/v1/
│   ├── sales/
│   │   ├── router.py                 # MODIFICADO — PATCH /payment-methods/{id} nuevo (editar/
│   │   │                               activar/desactivar con guardia "al menos uno activo")
│   │   ├── service.py                # MODIFICADO — validación de método activo mínimo
│   │   └── schemas.py                # MODIFICADO — payment_info en Create/Update/Response
│   │
│   ├── cart/
│   │   ├── router.py                 # MODIFICADO — endpoints nuevos del comensal (métodos
│   │   │                               disponibles, crear intento, presign+adjuntar comprobante),
│   │   │                               todos vía Depends(get_session_context) (invariante A-24)
│   │   ├── service.py                # MODIFICADO — submit_cart valida una orden activa por
│   │   │                               comensal (FR-005); nuevas funciones de intento de pago
│   │   └── schemas.py                # MODIFICADO — payloads/responses de intento y comprobante
│   │
│   ├── orders/
│   │   ├── router.py                 # MODIFICADO — endpoints nuevos del cajero (listar
│   │   │                               intentos, aprobar, rechazar, confirmar efectivo)
│   │   └── checkout.py                # MODIFICADO — confirm_order exige intento confirmado
│   │                                    (FR-017); nuevas funciones approve/reject/confirm_cash
│   │
│   └── uploads/
│       └── schemas.py                 # MODIFICADO — folder Literal gana "comprobantes"
│
├── core/
│   └── storage.py                     # SIN CAMBIOS — generate_presigned_put_url/build_object_key
│                                        ya cubren el caso; solo cambia el folder permitido
│
├── characterization_tests/
│   ├── fixtures.py                     # SIN CAMBIOS estructurales — se reutiliza para crear
│   │                                     tenant/producto/orden de prueba
│   ├── payment_attempts_fixtures.py    # NUEVO — helpers para PaymentMethod (transfer) y
│   │                                     OrderPaymentAttempt en distintos estados
│   ├── test_orders_payment_gate.py     # NUEVO — FR-017/FR-018 (US4): confirm_order exige
│   │                                     intento confirmado; doble confirmación no duplica efecto
│   ├── test_cart_payment_attempts.py   # NUEVO — FR-012/FR-015/FR-015a (US2/US5): un solo
│   │                                     intento pendiente a la vez, reintento tras rechazo
│   ├── test_cart_single_active_order.py # NUEVO — FR-005/FR-006 (US6)
│   └── test_sales_payment_methods.py   # NUEVO — FR-001/FR-002/FR-003 (US1): al menos un método
│                                          activo, edición no altera pagos ya confirmados
│
└── alembic/versions/
    └── {rev}_order_payment_attempts.py  # NUEVO — tabla + índice único parcial + columna
                                            payment_methods.payment_info, vía
                                            @for_each_tenant_schema

# pos-heladeria
src/app/modules/
├── sales/
│   ├── interfaces/sales.interface.ts        # MODIFICADO — PaymentMethod gana payment_info;
│   │                                           PaymentMethodUpdatePayload nuevo
│   ├── services/payment-method.service.ts   # MODIFICADO — update()/activar-desactivar
│   └── pages/payment-methods-page.component.ts # MODIFICADO — formulario de datos de
│                                                 transferencia + guardia de "al menos uno activo"
│
└── tables/
    ├── interfaces/diner.interface.ts         # MODIFICADO — PaymentMethod del comensal,
    │                                            OrderPaymentAttempt, payloads de intento/comprobante
    ├── services/
    │   ├── dining-cart.service.ts             # MODIFICADO — listar métodos, crear intento,
    │   │                                        presign+subir comprobante
    │   └── dining-session.service.ts          # SIN CAMBIOS estructurales (mismo session_token)
    ├── pages/
    │   └── public-menu.component.ts           # MODIFICADO — pantalla de pago del comensal
    │                                             (selección de método, datos de transferencia,
    │                                             carga de comprobante, estado pendiente/rechazado)
    └── components/
        ├── pending-orders-panel.component.ts  # MODIFICADO — el botón "Confirmar" refleja el
        │                                        gate nuevo (deshabilitado sin intento confirmado)
        └── payment-attempt-review-panel.component.ts # NUEVO — panel del cajero: comprobantes
                                                          pendientes (aprobar/rechazar con motivo),
                                                          confirmación de efectivo (monto→cambio)
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de los
ficheros de arriba (US1 → `sales/*`; US2/US5 → `cart/*` + `payment-attempt-review-panel`; US3 →
`orders/*` (confirm-cash) + el mismo panel; US4 → `orders/checkout.py::confirm_order`; US6 →
`cart/service.py::submit_cart`), consistente con Principio VI (evolución incremental, unidades
verificables). No se crea ningún módulo/paquete nuevo en `pos-backend` más allá de un modelo y sus
fixtures/tests — se extienden los tres routers ya existentes (`sales`, `cart`, `orders`) porque cada
endpoint nuevo pertenece naturalmente al dominio que ya agrupa ese router. En `pos-heladeria` se
extiende el módulo `tables` (donde ya vive todo el flujo QR + staff de mesas) y `sales` (donde ya
vive la configuración de métodos de pago) — sin módulo nuevo.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
