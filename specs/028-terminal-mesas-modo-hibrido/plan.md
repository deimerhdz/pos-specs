# Implementation Plan: Rediseño Híbrido de la Terminal de Mesas — Validación QR y Cobro Manual

**Branch**: `028-terminal-mesas-modo-hibrido` | **Date**: 2026-08-20 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/028-terminal-mesas-modo-hibrido/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

La Terminal de Mesas (pantalla existente en `pos-heladeria`, consumidora de `pos-backend`) hoy
mezcla dos flujos con una sola interfaz: pedidos QR pre-pagados que solo necesitan validación de
comprobante, y órdenes manuales de caja que sí necesitan cobro. Esa mezcla es la causa raíz del bug
reportado (botón genérico "Cobrar y cerrar mesa" fallando sobre órdenes QR ya pagadas). El enfoque
técnico: (1) consolidar las dos pestañas actuales en un único bloque central de validación,
reutilizando sin cambios los endpoints de aprobación/rechazo/cobro-en-efectivo ya implementados
(spec 024/026); (2) hacer que la barra lateral derecha derive su modo ("Resumen de Cuenta" vs.
"Terminal POS") del campo `channel` que la orden ya tiene en el modelo de datos — sin migración; y
(3) agregar dos piezas de servicio nuevas en el backend, ambas aditivas y sin tocar el
comportamiento de ningún endpoint existente: un endpoint de cobro atómico "pagar y enviar a cocina"
para la orden manual (que no encaja en el `block_order`/`pay_order` actual, pensado para el patrón
opuesto de "cocina primero, cobro al final"), y un endpoint de liberación pura de mesa ya pagada
(que tampoco encaja en `close_session`, pensado para cobrar Y cerrar a la vez). Ver
[research.md](./research.md) para el detalle de cada decisión técnica (D1-D8).

## Technical Context

**Language/Version**: Backend: Python 3.12 (FastAPI 0.136.3, Pydantic 2.13). Frontend: TypeScript
sobre Angular 21.1 (standalone components, `ChangeDetectionStrategy.OnPush`, signal stores — sin
NgRx).

**Primary Dependencies**: Backend: SQLAlchemy 2.0.50, Alembic 1.18.4, Redis 8 (event bus por
tenant vía Streams + Celery), Celery 5.6.3 + APScheduler 3.11.3 (barrido de sesiones huérfanas,
spec 010). Frontend: Tailwind CSS 4.1 (sin librería de componentes UI), sin dependencias nuevas
requeridas por esta spec.

**Storage**: PostgreSQL 16, schema-per-tenant. Sin migración: el campo `channel` que decide el modo
de la barra lateral (FR-005) ya existe en `CustomerOrder` (ver [data-model.md](./data-model.md)).

**Testing**: Backend: `unittest` (stdlib) sobre `app/characterization_tests/*.py`, contra SQLite en
memoria, sin mocks; tests nuevos deben citar el spec/FR que verifican, igual que los existentes
citan `RN-MESA-*`. Frontend: Vitest vía `ng test` (`@angular/build:unit-test`), specs co-ubicados
`*.component.spec.ts` con `TestBed`/`HttpTestingController`.

**Target Platform**: Aplicación web servida por el backend FastAPI (Linux) y consumida desde
navegador — el mismo diseño debe operar igual de bien en tablet táctil y en escritorio con
mouse/teclado (reutilizado de spec 026, sin excepción para esta spec).

**Project Type**: Aplicación web de dos repositorios independientes en producción — API
(`pos-backend`) + SPA (`pos-heladeria`) — según el alcance fijado por la
[Constitución](../../.specify/memory/constitution.md).

**Performance Goals**: Ninguno nuevo — se reutilizan los SLA implícitos ya vigentes de la Terminal
de Mesas (confirmación de pago y actualización de insignias percibidas como instantáneas vía el
stream SSE ya existente, sin polling adicional).

**Constraints**: Ningún endpoint reutilizado cambia de contrato ni de comportamiento (Principio VI
— evolución incremental, sin mezclar refactorización con funcionalidad nueva); el motor de
impresión térmica (48/58/80mm, generación HTML + `iframe` + `window.print()`) se mantiene sin
modificar (fuera de alcance del spec); texto legible ≥16px y controles táctiles ≥44×44pt
(reutilizado de spec 026, FR-010/FR-011 de esa spec).

**Scale/Scope**: Una pantalla (Terminal de Mesas) en un frontend multi-tenant ya en producción; 2
endpoints nuevos aditivos + 1 campo opcional aditivo en el backend; sin cambios de esquema de base
de datos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra los trece principios de la [Constitución](../../.specify/memory/constitution.md)
(v3.0.0, fase de evolución funcional):

| Principio | Estado | Evidencia |
|---|---|---|
| I. Nace de un spec | ✅ PASA | [spec.md](./spec.md), aprobado y clarificado (sesión 2026-08-20) antes de este plan. |
| II. Comportamiento existente protegido | ✅ PASA | Los cambios de comportamiento (ocultar "Cobrar y cerrar mesa" para QR, bloqueo bidireccional de orígenes mixtos, envío a cocina por comensal, nueva acción "Cerrar Mesa") están documentados explícitamente en el spec (Naturaleza de esta spec + sección Clarifications, con quién —el usuario, vía `/speckit-clarify`— y cuándo —2026-08-20—), siguiendo el mismo precedente ya usado por spec 026 para su propio cambio de FR-017: la autorización vive en el spec funcional aprobado, no requiere una entrada adicional en `registro-de-anomalias.md` (ese registro es el libro de anomalías de la fase de reconocimiento/modernización, ya cerrada). Los endpoints nuevos (D2/D3 de research.md) son **aditivos** — ningún caller existente de `POST /orders`, `block_order`, `pay_order` o `close_session` cambia de comportamiento. |
| III. Characterization tests protegidos | ✅ PASA (con acción de seguimiento) | Ningún endpoint con characterization tests existentes (`close_session`, `block_order`, `pay_order`, `approve/reject/confirm-cash_payment_attempt`) cambia de firma ni de comportamiento. Los tests nuevos (para `checkout-and-send` y `release`) se agregan junto a los existentes, citando este spec, sin tocar los ya congelados. |
| IV. Nuevos specs pueden introducir nuevo comportamiento | ✅ PASA | El spec documenta explícitamente qué cambia y por qué (ver II). |
| V. Nueva funcionalidad antes que refactor oportunista | ✅ PASA | El plan reutiliza deliberadamente `build_sale`, `_confirm_order_impl`, `close_table_sessions`, `_assert_closable` tal cual existen — no se propone ninguna refactorización de código no relacionada con esta funcionalidad. |
| VI. Evolución incremental | ✅ PASA | Los dos endpoints nuevos son estrictamente aditivos (campo opcional con default que preserva el comportamiento actual; endpoints nuevos en rutas nuevas). No se mezcla con migración de datos ni cambio de arquitectura. |
| VII. Compatibilidad con datos históricos | ✅ PASA | Ninguna venta/factura ya emitida se recalcula, reemite ni cambia de representación; "Reimprimir Factura POS" es una regeneración de solo lectura del mismo documento (D6). |
| VIII. Evolución del modelo de datos | ✅ PASA — N/A | Sin cambios de esquema (ver [data-model.md](./data-model.md)): un campo opcional de request (`hold_for_payment`), no una columna nueva. Sin migración, sin estrategia de rollback de esquema que declarar. |
| IX. Dependencias nuevas justificadas | ✅ PASA — N/A | No se introduce ninguna dependencia nueva en ningún repo. |
| X. Verificación obligatoria | ⚠️ PENDIENTE (se resuelve en `/speckit-tasks`) | Este plan identifica qué tests deben agregarse/actualizarse (ver [quickstart.md](./quickstart.md)); la ejecución y verificación ocurre en la fase de implementación, no en la de planeación. |
| XI. Decisiones de negocio vs. técnicas | ✅ PASA | Las decisiones D1-D8 de [research.md](./research.md) son técnicas (cómo implementar lo que el spec ya decidió), no reabren ninguna decisión de negocio del spec. |
| XII. Trazabilidad | ✅ PASA | Cadena completa: Necesidad (bug reportado + modelo híbrido) → spec 028 → Clarifications (2026-08-20) → este plan (research/data-model/contracts) → tareas (`/speckit-tasks`, pendiente) → tests. |
| XIII. Español de Colombia | ✅ PASA | Todo el plan, research, data-model, contratos y quickstart están en español de Colombia, consistente con el resto del repositorio de specs. |

**Sin violaciones que requieran justificación** — no aplica la tabla de Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/028-terminal-mesas-modo-hibrido/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── api-contracts.md # Phase 1 output (/speckit-plan command)
├── checklists/
│   └── requirements.md  # /speckit-specify output, re-validated by /speckit-clarify
└── tasks.md              # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

Esta feature vive en dos repositorios hermanos a `pos-specs`, ambos ya en producción (ver
"Alcance del Proyecto" de la [Constitución](../../.specify/memory/constitution.md)); no hay una
opción de estructura "a elegir" — ya existe y se reutiliza tal cual:

```text
../pos-backend/                              # FastAPI + PostgreSQL 16 (schema-per-tenant)
├── app/
│   ├── models/
│   │   ├── customer_order.py                # channel, status — sin cambios de esquema
│   │   ├── order_payment_attempt.py          # sin cambios
│   │   ├── table_session.py                  # sin cambios
│   │   └── dining_table.py                   # sin cambios
│   ├── api/v1/
│   │   ├── orders/
│   │   │   ├── service.py                    # create_order: + hold_for_payment (D1)
│   │   │   ├── checkout.py                   # + checkout_and_send (D3); approve/reject/
│   │   │   │                                 #   confirm_cash/block_order/pay_order sin cambios
│   │   │   ├── schemas.py                    # OrderCreate.hold_for_payment (nuevo, opcional)
│   │   │   └── router.py                     # + POST /orders/{id}/checkout-and-send
│   │   └── table_sessions/
│   │       ├── service.py                    # + release_paid_session (D2)
│   │       ├── schemas.py                    # + ReleaseSessionResponse
│   │       └── router.py                     # + POST /table-sessions/{id}/release
│   └── characterization_tests/               # tests nuevos junto a los existentes
│       ├── test_orders_checkout.py           # (o el fichero de checkout.py existente)
│       └── test_table_sessions_service.py
│
../pos-heladeria/                             # Angular 21 (standalone, signal stores)
├── src/app/modules/tables/
│   ├── pages/
│   │   └── table-sessions.component.ts       # consolida tabs "pedido"/"pendientes" (FR-001)
│   ├── components/
│   │   ├── payment-attempt-review-panel.component.ts   # + modal "Ver Comprobante" (D4)
│   │   ├── pending-orders-panel.component.ts            # retirado / fusionado al bloque único
│   │   ├── pos-checkout-panel.component.ts               # + modo dual Resumen/Terminal POS
│   │   ├── session-bill-panel.component.ts                # retira "Cobrar y cerrar mesa" para QR
│   │   └── manual-order-panel.component.ts (nuevo)         # "+ Crear Orden Manual" (FR-004)
│   ├── services/
│   │   ├── table-session.service.ts          # + release()
│   │   └── dining-session.service.ts          # + checkoutAndSend()
│   ├── interfaces/dining.interface.ts         # tipos de modo de barra lateral (nuevo, solo tipos)
│   └── services/pos-terminal.store.ts         # + lógica de insignia por mesa (FR-014)
└── src/app/core/printing/
    └── receipt.util.ts                        # + sessionBillToReceipt() para "Pre-cuenta" (D6)
```

**Structure Decision**: se reutiliza la estructura ya existente de ambos repos (módulo `tables` en
el frontend, routers `orders`/`table_sessions` en el backend); no se crea ningún módulo/paquete
nuevo. Los únicos archivos nuevos son un componente de UI (panel de orden manual), un servicio de
recibo para pre-cuenta, y los dos endpoints nuevos descritos en
[contracts/api-contracts.md](./contracts/api-contracts.md) — todo el resto son modificaciones
localizadas a archivos ya existentes.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No aplica — el Constitution Check no registró ninguna violación (ver tabla arriba).
