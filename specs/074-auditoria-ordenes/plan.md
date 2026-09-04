# Implementation Plan: Auditoría del ciclo de vida de una orden

**Branch**: `074-auditoria-ordenes` | **Date**: 2026-09-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/074-auditoria-ordenes/spec.md`

## Summary

Instrumentar los 8 puntos del ciclo de vida de una orden (creación QR/manual, confirmación, registro de intento de pago, confirmación en efectivo, aprobación/rechazo de comprobante de transferencia, cancelación, y cobro-y-envío en un solo paso) en `pos-backend` para que cada uno emita, tras completarse con éxito, un evento de auditoría estructurado con `order_id`, `tenant_id` y un actor explícito ({comensal, cajero, sistema}). Por decisión de alcance del spec, el único destino de esos eventos es Sentry Logs (ya habilitado en el cliente `sentry_sdk` existente, `enable_logs=True`) — este feature no introduce tabla, migración ni almacenamiento interno propio. El nombre del comensal y el comprobante de pago, cuando aparecen en un evento, se envían transformados con HMAC-SHA256 (nunca en texto plano). El envío a Sentry es desacoplado: una falla al enviarlo no bloquea ni revierte la transición de negocio. **Ya implementado y en producción.**

**Extensión (FR-015–FR-021, planificada en este documento)**: con el feature anterior ya en producción, se añade una segunda capa, de naturaleza distinta — logging operativo general con el patrón `request_id` (hoy exclusivo de `/api/v1/super-admin`, spec 068), extendido a **todo el backend de negocio excepto super-admin**, para toda petición HTTP mutativa (`POST`/`PUT`/`PATCH`/`DELETE`). Registra metadatos estructurados (método, ruta, status, duración, actor, tenant, `request_id`) — nunca el cuerpo de la petición/respuesta — tanto en éxito como en error. Los 8 eventos de auditoría de orden existentes se mantienen intactos como capa adicional; además, cada uno pasa a incluir el `request_id` de la petición que lo originó (FR-021), correlacionando ambas capas.

## Technical Context

**Language/Version**: Python 3.12 (imagen `python:3.12-slim` del `Dockerfile` de `pos-backend`; sin cambio de versión)

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50 (solo para leer el contexto de tenant/actor ya resuelto por cada función de servicio, no para persistir el log), sentry-sdk 2.61.0 — ya instalado y ya inicializado con `enable_logs=True` (`app/main.py:98`); este feature no agrega ninguna dependencia nueva (Principio IX de la constitución)

**Storage**: N/A para este feature — decisión de alcance explícita del spec (Clarifications, sesión 2026-09-03, segunda ronda): no se introduce tabla ni almacenamiento interno; el único destino persistente/consultable es Sentry Logs, sujeto a su propia ventana de retención

**Testing**: `unittest` (biblioteca estándar), igual que el resto del proyecto — se ejecuta vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (no hay `pytest` instalado en `pos-backend`)

**Target Platform**: Linux server — mismo contenedor Docker existente de `pos-backend`

**Project Type**: web-service (backend FastAPI existente); este feature no toca `pos-heladeria` (frontend)

**Performance Goals**: el envío del evento de auditoría no debe añadir latencia perceptible a la respuesta HTTP del actor — se ejecuta de forma desacoplada de la transacción de negocio (FR-011); sin objetivo numérico adicional. Para la extensión de logging operativo (FR-015 en adelante): el middleware nuevo corre en el camino de **toda** petición mutativa del backend (salvo super-admin), así que su propio código debe ser trivial (armar un diccionario de atributos y una llamada a `sentry_sdk.logger.*`) y nunca debe poder fallar de forma que rompa la petición real — mismo principio de desacoplamiento que FR-011, aplicado aquí a nivel de middleware en vez de a nivel de llamada a un helper.

**Constraints**: no agregar tablas ni migraciones (FR-006); no bloquear ni revertir la transición de negocio ante una falla al enviar el evento (FR-011); nunca enviar el nombre del comensal ni el comprobante de pago en texto plano (FR-005); reutilizar el cliente `sentry_sdk` ya inicializado, sin nueva dependencia; los eventos de auditoría deben quedar identificables como categoría propia dentro de Sentry, sin mezclarse indistinguiblemente con el registro operativo de errores ya existente (FR-007). Para FR-015 en adelante: nunca registrar el cuerpo de la petición/respuesta, bajo ninguna circunstancia (FR-018); no tocar `/api/v1/super-admin`, que mantiene su propio mecanismo ya existente (FR-015); una falla al registrar la entrada de log operativo NUNCA debe propagarse ni alterar la respuesta de la petición real que la originó (mismo principio de FR-011, aplicado a esta capa).

**Scale/Scope**: 8 tipos de evento de auditoría de orden, integrados en 2 módulos existentes del backend (`app/api/v1/orders`, `app/api/v1/cart`); volumen proporcional al de órdenes del negocio (ver Assumptions del spec). Además (FR-015 en adelante): un middleware nuevo que cubre **todas** las rutas mutativas de **todos** los routers del backend salvo `/api/v1/super-admin` (~20 routers, ver `app/main.py`) — volumen proporcional al de peticiones mutativas de todo el negocio, no solo de órdenes.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — spec aprobado en `specs/074-auditoria-ordenes/spec.md`, con clarificaciones registradas. |
| II. El Comportamiento Existente Sigue Protegido | ✅ Pass — el feature es puramente aditivo: no cambia ninguna regla de negocio de la orden, solo agrega una emisión de log tras cada transición ya existente. FR-011 exige explícitamente que una falla en esa emisión no altere el comportamiento de negocio actual. |
| III. Characterization Tests Protegen el Comportamiento Heredado | ✅ Pass — no se modifica ningún test `"CONGELA comportamiento actual:"`; se añaden tests nuevos junto a ellos. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — el nuevo comportamiento (emisión de eventos de auditoría) está definido en el spec. |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — **actualizado**: la evaluación original decía que este feature NO generalizaría `error_middleware.py`/`RequestIdMiddleware` a otros módulos. Eso cambió: FR-015–FR-021 (extensión aprobada explícitamente en Clarifications, no una refactorización oportunista colada aparte) piden exactamente esa generalización — el propio docstring de `error_middleware.py` ya la anticipaba como "el cambio... cuando de verdad haga falta un segundo módulo real que lo necesite". Sigue sin ser una refactorización oportunista porque: (a) tiene spec propio (FR-015–FR-021), (b) no se aprovecha para tocar nada más de `error_middleware.py` que no pida el spec (el envelope de errores de super-admin, `DomainError`, etc. quedan intactos). |
| VI. Evolución Incremental | ✅ Pass — sigue siendo un solo tipo de cambio (observabilidad), aunque FR-015 en adelante lo aplique a un alcance mucho más amplio (todo el backend salvo super-admin, no solo órdenes). El radio de alcance es grande, pero la clase de cambio es una sola — no mezcla esto con una migración de datos, un cambio de arquitectura, ni una modificación de comportamiento de negocio de ningún otro módulo. |
| VII. Compatibilidad con Datos Históricos | N/A — el feature no toca facturas ni datos históricos de ventas. |
| VIII. Evolución del Modelo de Datos | N/A — no se modifica el modelo de datos persistido (no hay migración; ver Storage). El "modelo de datos" de este feature (data-model.md) describe la forma de un evento de log estructurado, no una entidad de base de datos. |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica justificación porque no hay dependencia nueva) — se reutiliza `sentry-sdk`, ya presente y ya inicializado con Sentry Logs habilitado. |
| X. Verificación Obligatoria | ✅ Pass (planificado) — research.md define tests unitarios del helper/hash y tests de integración por cada uno de los 8 puntos de emisión de orden, con `unittest`. Para FR-015 en adelante: tests del middleware nuevo (mutativo vs. lectura, éxito vs. error, ausencia de cuerpo) y una verificación de que la suite completa de characterization tests sigue en verde — dado que este middleware se monta sobre prácticamente todas las rutas del backend, es el punto donde una regresión tendría el radio de impacto más amplio de todo este spec. |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — las decisiones de negocio (Sentry como único destino, retención sujeta al plan, hash de datos sensibles, consulta solo admin vía panel externo, desacoplamiento) ya quedaron registradas en el spec (Clarifications); este plan solo resuelve el "cómo" técnico. |
| XII. Trazabilidad | ✅ Pass — Necesidad → spec.md → Clarifications → este plan → research.md/data-model.md/contracts/ → tasks.md (siguiente comando) → tests. |
| XIII. Todo en Español de Colombia | ✅ Pass — todos los artefactos de este feature se redactan en español de Colombia. |

No hay violaciones que requieran registrar en Complexity Tracking.

**Re-chequeo post Fase 1 (auditoría de orden, FR-001-014)**: tras generar `research.md`, `data-model.md`, `contracts/` y `quickstart.md`, ninguna decisión de diseño introdujo una dependencia nueva, una migración, ni una desviación de lo evaluado arriba — la tabla se mantiene sin cambios.

**Re-chequeo post Fase 1 (extensión de logging operativo, FR-015-021)**: tras generar `research.md` §7-11, `data-model.md` (extensión), `contracts/operational-log-entry.md` y `quickstart.md` §3-4, tampoco se introdujo ninguna dependencia nueva (sigue siendo `sentry_sdk`, ya presente) ni ninguna migración — el único ajuste real a la tabla de arriba es la fila V (generalizar `error_middleware.py`, ya corregida) y las filas VI/X (alcance/radio de impacto más amplio, ya anotado). El diseño elegido (middleware nuevo separado, side-effect de una línea en 3 dependencias ya existentes) confirma que no hace falta tocar `RequestIdMiddleware`/`register_error_handlers` de super-admin en absoluto.

## Project Structure

### Documentation (this feature)

```text
specs/074-auditoria-ordenes/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — §1-6 auditoría de orden, §7-11 extensión
├── data-model.md         # Fase 1 (/speckit-plan) — íd.
├── quickstart.md         # Fase 1 (/speckit-plan) — íd.
├── contracts/            # Fase 1 (/speckit-plan)
│   ├── order-audit-log-event.md
│   └── operational-log-entry.md   # NUEVO — extensión FR-015-FR-021
├── checklists/
│   └── requirements.md
└── tasks.md              # Fase 2 (/speckit-tasks) — fases T001-T043 ya implementadas;
                           #   la extensión necesita su propia ronda de /speckit-tasks
```

### Source Code (repositorio `pos-backend`, independiente de `pos-specs`)

```text
pos-backend/
├── app/
│   ├── core/
│   │   ├── order_audit.py          # helper de emisión de eventos de orden, tipos, hash
│   │   │                            #   (ya existente — se le añade request_id opcional, FR-021)
│   │   ├── error_middleware.py     # NUEVO en esta extensión: middleware de logging operativo
│   │   │                            #   (FR-015-FR-020), separado del RequestIdMiddleware/
│   │   │                            #   register_error_handlers de super-admin (no se tocan)
│   │   ├── dependencies.py         # get_current_user → estampa actor en request.state
│   │   ├── qr_context.py           # get_session_context → estampa actor en request.state
│   │   └── db.py                   # get_tenant → estampa tenant_id en request.state
│   ├── main.py                     # monta el middleware nuevo (todas las rutas salvo super-admin)
│   ├── api/v1/orders/
│   │   ├── service.py              # create_order → emite order.created (manual)
│   │   └── checkout.py             # _confirm_order_impl, confirm_cash_payment_attempt,
│   │                                #   approve_payment_attempt, reject_payment_attempt,
│   │                                #   cancel_order, checkout_and_send → emiten sus eventos
│   └── api/v1/cart/
│       └── service.py              # submit_cart → order.created (QR)
│                                    # create_payment_attempt → order.payment_attempt.created
│                                    # cancel_my_order → delega en checkout.cancel_order
│   └── characterization_tests/
│       ├── test_order_audit_log.py       # tests del helper + de los 8 puntos de integración
│       └── test_operational_log.py       # NUEVO — tests del middleware de logging operativo
```

**Structure Decision**: Servicio backend único ya existente (`pos-backend`); este feature no agrega paquetes de alto nivel ni toca `pos-heladeria`. La auditoría de orden (`app/core/order_audit.py`) ya está implementada. La extensión de logging operativo (FR-015 en adelante) añade una clase de middleware nueva en `app/core/error_middleware.py` (junto al `RequestIdMiddleware` de super-admin, sin modificarlo) y una modificación de una línea en cada una de las 3 dependencias compartidas que ya resuelven actor/tenant (`get_tenant`, `get_current_user`, `get_session_context`) para dejarlos disponibles en `request.state` — sin nuevos routers, sin nuevos modelos SQLAlchemy, sin nuevas migraciones de Alembic.

## Complexity Tracking

*Sin violaciones del Constitution Check — sección no aplica.*
