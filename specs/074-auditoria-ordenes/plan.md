# Implementation Plan: Auditoría del ciclo de vida de una orden

**Branch**: `074-auditoria-ordenes` | **Date**: 2026-09-03 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/074-auditoria-ordenes/spec.md`

## Summary

Instrumentar los 7 puntos del ciclo de vida de una orden (creación QR/manual, confirmación, registro de intento de pago, confirmación en efectivo, aprobación/rechazo de comprobante de transferencia, y cancelación) en `pos-backend` para que cada uno emita, tras completarse con éxito, un evento de auditoría estructurado con `order_id`, `tenant_id` y un actor explícito ({comensal, cajero, sistema}). Por decisión de alcance del spec, el único destino de esos eventos es Sentry Logs (ya habilitado en el cliente `sentry_sdk` existente, `enable_logs=True`) — este feature no introduce tabla, migración ni almacenamiento interno propio. El nombre del comensal y el comprobante de pago, cuando aparecen en un evento, se envían transformados con HMAC-SHA256 (nunca en texto plano). El envío a Sentry es desacoplado: una falla al enviarlo no bloquea ni revierte la transición de negocio.

## Technical Context

**Language/Version**: Python 3.12 (imagen `python:3.12-slim` del `Dockerfile` de `pos-backend`; sin cambio de versión)

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50 (solo para leer el contexto de tenant/actor ya resuelto por cada función de servicio, no para persistir el log), sentry-sdk 2.61.0 — ya instalado y ya inicializado con `enable_logs=True` (`app/main.py:98`); este feature no agrega ninguna dependencia nueva (Principio IX de la constitución)

**Storage**: N/A para este feature — decisión de alcance explícita del spec (Clarifications, sesión 2026-09-03, segunda ronda): no se introduce tabla ni almacenamiento interno; el único destino persistente/consultable es Sentry Logs, sujeto a su propia ventana de retención

**Testing**: `unittest` (biblioteca estándar), igual que el resto del proyecto — se ejecuta vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (no hay `pytest` instalado en `pos-backend`)

**Target Platform**: Linux server — mismo contenedor Docker existente de `pos-backend`

**Project Type**: web-service (backend FastAPI existente); este feature no toca `pos-heladeria` (frontend)

**Performance Goals**: el envío del evento de auditoría no debe añadir latencia perceptible a la respuesta HTTP del actor — se ejecuta de forma desacoplada de la transacción de negocio (FR-011); sin objetivo numérico adicional

**Constraints**: no agregar tablas ni migraciones (FR-006); no bloquear ni revertir la transición de negocio ante una falla al enviar el evento (FR-011); nunca enviar el nombre del comensal ni el comprobante de pago en texto plano (FR-005); reutilizar el cliente `sentry_sdk` ya inicializado, sin nueva dependencia; los eventos de auditoría deben quedar identificables como categoría propia dentro de Sentry, sin mezclarse indistinguiblemente con el registro operativo de errores ya existente (FR-007)

**Scale/Scope**: 7 tipos de evento de auditoría, integrados en 2 módulos existentes del backend (`app/api/v1/orders`, `app/api/v1/cart`); volumen proporcional al de órdenes del negocio (ver Assumptions del spec)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — spec aprobado en `specs/074-auditoria-ordenes/spec.md`, con clarificaciones registradas. |
| II. El Comportamiento Existente Sigue Protegido | ✅ Pass — el feature es puramente aditivo: no cambia ninguna regla de negocio de la orden, solo agrega una emisión de log tras cada transición ya existente. FR-011 exige explícitamente que una falla en esa emisión no altere el comportamiento de negocio actual. |
| III. Characterization Tests Protegen el Comportamiento Heredado | ✅ Pass — no se modifica ningún test `"CONGELA comportamiento actual:"`; se añaden tests nuevos junto a ellos. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — el nuevo comportamiento (emisión de eventos de auditoría) está definido en el spec. |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — la integración se limita a insertar una llamada al helper de auditoría en los puntos exactos listados en research.md; no se generaliza `error_middleware.py`/`RequestIdMiddleware` a otros módulos ni se refactoriza código no relacionado, aunque su docstring lo sugiera como posibilidad futura. |
| VI. Evolución Incremental | ✅ Pass — un solo tipo de cambio (observabilidad/auditoría) sobre un módulo ya identificado; no mezcla migración de datos, arquitectura ni cambio de comportamiento de negocio. |
| VII. Compatibilidad con Datos Históricos | N/A — el feature no toca facturas ni datos históricos de ventas. |
| VIII. Evolución del Modelo de Datos | N/A — no se modifica el modelo de datos persistido (no hay migración; ver Storage). El "modelo de datos" de este feature (data-model.md) describe la forma de un evento de log estructurado, no una entidad de base de datos. |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica justificación porque no hay dependencia nueva) — se reutiliza `sentry-sdk`, ya presente y ya inicializado con Sentry Logs habilitado. |
| X. Verificación Obligatoria | ✅ Pass (planificado) — research.md define tests unitarios del helper/hash y tests de integración por cada uno de los 7 puntos de emisión, con `unittest`. |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — las decisiones de negocio (Sentry como único destino, retención sujeta al plan, hash de datos sensibles, consulta solo admin vía panel externo, desacoplamiento) ya quedaron registradas en el spec (Clarifications); este plan solo resuelve el "cómo" técnico. |
| XII. Trazabilidad | ✅ Pass — Necesidad → spec.md → Clarifications → este plan → research.md/data-model.md/contracts/ → tasks.md (siguiente comando) → tests. |
| XIII. Todo en Español de Colombia | ✅ Pass — todos los artefactos de este feature se redactan en español de Colombia. |

No hay violaciones que requieran registrar en Complexity Tracking.

**Re-chequeo post Fase 1**: tras generar `research.md`, `data-model.md`, `contracts/` y `quickstart.md`, ninguna decisión de diseño introdujo una dependencia nueva, una migración, ni una desviación de lo evaluado arriba — la tabla se mantiene sin cambios.

## Project Structure

### Documentation (this feature)

```text
specs/074-auditoria-ordenes/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/            # Fase 1 (/speckit-plan)
│   └── order-audit-log-event.md
├── checklists/
│   └── requirements.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `pos-backend`, independiente de `pos-specs`)

```text
pos-backend/
├── app/
│   ├── core/
│   │   └── order_audit.py          # NUEVO — helper de emisión, tipos de actor/evento, hash
│   ├── api/v1/orders/
│   │   ├── service.py              # create_order → emite order.created (manual)
│   │   └── checkout.py             # _confirm_order_impl, confirm_cash_payment_attempt,
│   │                                #   approve_payment_attempt, reject_payment_attempt,
│   │                                #   cancel_order → emiten sus eventos respectivos
│   └── api/v1/cart/
│       └── service.py              # submit_cart → order.created (QR)
│                                    # create_payment_attempt → order.payment_attempt.created
│                                    # cancel_my_order → delega en checkout.cancel_order
│   └── characterization_tests/
│       └── test_order_audit_log.py # NUEVO — tests del helper + de los 7 puntos de integración
```

**Structure Decision**: Servicio backend único ya existente (`pos-backend`); este feature no agrega paquetes de alto nivel ni toca `pos-heladeria`. Se añade un módulo nuevo (`app/core/order_audit.py`) con el helper de emisión y los tipos de actor/evento, y se integra por llamada directa en los puntos de servicio ya identificados — sin nuevos routers, sin nuevos modelos SQLAlchemy, sin nuevas migraciones de Alembic.

## Complexity Tracking

*Sin violaciones del Constitution Check — sección no aplica.*
