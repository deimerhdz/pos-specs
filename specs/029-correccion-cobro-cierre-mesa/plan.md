# Implementation Plan: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas

**Branch**: `029-correccion-cobro-cierre-mesa` | **Date**: 2026-08-21 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/029-correccion-cobro-cierre-mesa/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

La Terminal de Mesas (pantalla existente en `pos-heladeria`, consumidora de `pos-backend`, spec
028 ya en producción) tiene cuatro defectos puntuales, los cuatro con la misma causa de fondo: en
varios puntos, el sistema decide qué mostrar o qué permitir mirando solo el estado de cocina, sin
verificar si el pedido ya tiene un pago registrado. El enfoque técnico: (1) introducir una única
señal nueva y confiable de "este pedido ya está pagado" —la existencia de una `Sale` con
`customer_order_id` igual al del pedido, no `CustomerOrder.status`, porque los dos caminos de pago
vigentes (QR y mostrador) nunca alcanzan `status == "pagada"`— expuesta como un campo computado
`paid` en `OrderResponse`; (2) reutilizar esa misma señal en las tres correcciones que la
necesitan: bloquear `void_item` cuando ya existe pago (backend, con guard nuevo), corregir la
insignia "Listo" para que exija pago además de cocina lista (frontend, `deriveTableStatus`), y
ocultar la acción "Anular" en la interfaz; (3) endurecer el único campo de request que hoy acepta
un descuento manual (`CheckoutAndSendIn.discount`, exclusivo de esta pantalla) para que solo admita
`0`, y retirar del frontend el atajo/popover que lo alimentaba; y (4) consolidar en una sola
implementación (la ya verificada contra el backend) las dos rutas de código que hoy reimprimen la
misma factura. Ver [research.md](./research.md) para el detalle de cada decisión técnica (D1-D4).

## Technical Context

**Language/Version**: Backend: Python 3.12 (FastAPI 0.136.3, Pydantic 2.13). Frontend: TypeScript
sobre Angular 21.1 (standalone components, `ChangeDetectionStrategy.OnPush`, signal stores — sin
NgRx).

**Primary Dependencies**: Backend: SQLAlchemy 2.0.50 (subconsulta `EXISTS` sobre `sales`, sin
dependencia nueva). Frontend: Tailwind CSS 4.1, sin dependencias nuevas requeridas por esta spec.

**Storage**: PostgreSQL 16, schema-per-tenant. Sin migración: el nuevo campo `paid` de
`OrderResponse` es computado en el momento de servir la respuesta (subconsulta sobre
`sales.customer_order_id`, ya indexada), no una columna nueva (ver [data-model.md](./data-model.md)).

**Testing**: Backend: `unittest` (stdlib) sobre `app/characterization_tests/*.py`, contra SQLite en
memoria, sin mocks; el characterization test existente `test_orders_kitchen.py` requiere
actualización explícita (Principio III, ver Constitution Check). Frontend: Vitest vía `ng test`
(`@angular/build:unit-test`), specs co-ubicados `*.component.spec.ts`/`*.store.spec.ts` con
`TestBed`/`HttpTestingController`.

**Target Platform**: Aplicación web servida por el backend FastAPI (Linux) y consumida desde
navegador — mismo diseño ya vigente en tablet táctil y en escritorio (spec 026), sin cambios de
layout en esta spec.

**Project Type**: Aplicación web de dos repositorios independientes en producción — API
(`pos-backend`) + SPA (`pos-heladeria`) — según el alcance fijado por la
[Constitución](../../.specify/memory/constitution.md).

**Performance Goals**: Ninguno nuevo — el campo `paid` se calcula con una subconsulta `EXISTS`
sobre una columna ya indexada (`sales.customer_order_id`), sin impacto perceptible sobre el tiempo
de respuesta ya vigente de los endpoints que serializan `OrderResponse`.

**Constraints**: Ningún endpoint reutilizado sin cambios (ver tabla en
[contracts/api-contracts.md](./contracts/api-contracts.md)) cambia de contrato; los dos que sí
cambian (`void_item`, `checkout_and_send`) solo **restringen** casos que hoy son errores de
negocio no capturados (Principio VI — corrección, no rediseño). El motor de impresión térmica y el
motor de promociones/combos se mantienen sin modificar (fuera de alcance del spec).

**Scale/Scope**: Una pantalla (Terminal de Mesas) en un frontend multi-tenant ya en producción; 0
endpoints nuevos, 1 campo de respuesta nuevo (computado) + 1 restricción endurecida sobre un campo
de request ya existente + 1 guard nuevo en un endpoint ya existente; sin cambios de esquema de base
de datos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra los trece principios de la [Constitución](../../.specify/memory/constitution.md)
(v3.0.0, fase de evolución funcional):

| Principio | Estado | Evidencia |
|---|---|---|
| I. Nace de un spec | ✅ PASA | [spec.md](./spec.md), aprobado y clarificado (sesión 2026-08-21) antes de este plan. |
| II. Comportamiento existente protegido | ✅ PASA | Los cuatro cambios de comportamiento (bloqueo de anulación tras pago, eliminación absoluta del descuento manual sin excepción por rol, insignia "Listo" exige pago, consolidación de impresión) están documentados explícitamente en el spec ("Naturaleza de esta spec" + sección Clarifications, con quién —el usuario, vía `/speckit-clarify`— y cuándo —2026-08-21—), siguiendo el mismo precedente ya usado por spec 025/028: la autorización vive en el spec funcional aprobado. La corrección de anulación convierte además la anomalía ya registrada **A-16** (`registro-de-anomalias.md`, "tratamiento acordado: corregir") en una decisión de negocio explícita y acotada (solo `void_item`, no `transition_kitchen` — ver research.md D3 para la justificación de por qué esa distinción es intencional, no un olvido). |
| III. Characterization tests protegidos | ⚠️ ACCIÓN REQUERIDA (identificada, se ejecuta en `/speckit-tasks`/`/speckit-implement`) | `test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16` (`app/characterization_tests/test_orders_kitchen.py:71-89`) congela el comportamiento **actual** de `void_item` que esta spec cambia deliberadamente. Research.md D3 documenta la actualización exacta requerida: la aserción sobre `void_item` pasa a esperar `409` (citando esta spec y A-16 en el docstring), la aserción sobre `transition_kitchen` se mantiene sin cambios, y `test_mark_order_ready_409_si_orden_pagada_contraste_a16` no se toca. Ningún otro test `"CONGELA comportamiento actual:"` se ve afectado. |
| IV. Nuevos specs pueden introducir nuevo comportamiento | ✅ PASA | El spec documenta explícitamente qué cambia y por qué (ver II). |
| V. Nueva funcionalidad antes que refactor oportunista | ✅ PASA | El plan reutiliza deliberadamente `build_sale`, `void_item`, `deriveTableStatus`, `printOrderInvoice`/`resolveSaleForOrder` tal cual existen — ninguna refactorización no relacionada. Se decide explícitamente **no** tocar el campo compartido `discount` de `sales/schemas.py` (alcance de spec 011) ni `transition_kitchen`/`mark_order_ready`, precisamente para no mezclar esta corrección con alcance de otras specs (research.md D3/D4). |
| VI. Evolución incremental | ✅ PASA | Los cuatro cambios son correcciones puntuales y aditivas: un campo de respuesta computado (sin migración), una restricción de esquema sobre un campo de request ya existente, un guard nuevo en un endpoint ya existente, y una consolidación de código de UI ya existente. No se mezcla con migración de datos ni cambio de arquitectura. |
| VII. Compatibilidad con datos históricos | ✅ PASA | Ninguna venta/factura ya emitida se recalcula, reemite ni cambia de representación; "Imprimir Factura" sigue siendo una regeneración de solo lectura del mismo documento (`Sale.invoice`, `viewonly`, sin cambios). |
| VIII. Evolución del modelo de datos | ✅ PASA — N/A | Sin cambios de esquema (ver [data-model.md](./data-model.md)): `paid` es un campo computado de respuesta, no una columna nueva; la restricción de `discount` es un cambio de validación (`le=0`) sobre un campo ya existente, no un cambio de esquema de base de datos. Sin migración, sin estrategia de rollback de esquema que declarar. |
| IX. Dependencias nuevas justificadas | ✅ PASA — N/A | No se introduce ninguna dependencia nueva en ningún repo. |
| X. Verificación obligatoria | ⚠️ PENDIENTE (se resuelve en `/speckit-tasks`) | Este plan identifica qué tests deben agregarse/actualizarse (ver [quickstart.md](./quickstart.md) y research.md D3); la ejecución y verificación ocurre en la fase de implementación, no en la de planeación. |
| XI. Decisiones de negocio vs. técnicas | ✅ PASA | Las decisiones D1-D4 de [research.md](./research.md) son técnicas (cómo implementar lo que el spec ya decidió — incluida la señal `Sale.customer_order_id` como definición operativa de "pagado"), no reabren ninguna decisión de negocio del spec. |
| XII. Trazabilidad | ✅ PASA | Cadena completa: Necesidad (cuatro defectos reportados por el usuario, con captura de evidencia) → spec 029 → Clarifications (2026-08-21) → este plan (research/data-model/contracts) → tareas (`/speckit-tasks`, pendiente) → tests. |
| XIII. Español de Colombia | ✅ PASA | Todo el plan, research, data-model, contratos y quickstart están en español de Colombia, consistente con el resto del repositorio de specs. |

**Sin violaciones que requieran justificación** — no aplica la tabla de Complexity Tracking. La
única fila en estado distinto de "PASA" (Principio III) es una acción de seguimiento ya
identificada con su alcance exacto documentado, no una violación sin resolver.

## Project Structure

### Documentation (this feature)

```text
specs/029-correccion-cobro-cierre-mesa/
├── plan.md               # This file (/speckit-plan command output)
├── research.md            # Phase 0 output (/speckit-plan command)
├── data-model.md           # Phase 1 output (/speckit-plan command)
├── quickstart.md            # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── api-contracts.md      # Phase 1 output (/speckit-plan command)
├── checklists/
│   └── requirements.md         # /speckit-specify output, re-validated by /speckit-clarify
└── tasks.md                      # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

Esta feature vive en dos repositorios hermanos a `pos-specs`, ambos ya en producción (ver
"Alcance del Proyecto" de la [Constitución](../../.specify/memory/constitution.md)); no hay una
opción de estructura "a elegir" — ya existe y se reutiliza tal cual:

```text
../pos-backend/                              # FastAPI + PostgreSQL 16 (schema-per-tenant)
├── app/
│   ├── models/
│   │   └── sale.py                          # Sale.customer_order_id — sin cambios de esquema, ya indexado
│   ├── api/v1/orders/
│   │   ├── kitchen.py                       # void_item: + guard "ya pagado" (D3); transition_kitchen/
│   │   │                                    #   mark_order_ready sin cambios
│   │   ├── schemas.py                       # OrderResponse: + paid (D2); CheckoutAndSendIn.discount:
│   │   │                                    #   ge=0 → ge=0,le=0 (D4)
│   │   ├── service.py                       # + order_has_sale(db, order_id) / paid_order_ids(db, ids):
│   │   │                                    #   mismo patrón de consulta que has_billable_orders
│   │   │                                    #   (table_sessions/service.py:65-83, sin tocar), reutilizado
│   │   │                                    #   por void_item y por la serialización de paid
│   │   └── router.py                        # sin endpoints nuevos; void_item, list_orders/get_order y
│   │                                        #   checkout_and_send existentes, con el nuevo comportamiento
│   │                                        #   de arriba (list_orders/get_order adjuntan `paid` en bloque
│   │                                        #   antes de serializar, sin N+1 por pedido)
│   └── characterization_tests/
│       └── test_orders_kitchen.py           # actualización explícita de la aserción sobre void_item
│                                            #   (Principio III, ver Constitution Check); tests nuevos
│                                            #   para el guard sobre orden 'abierta' con Sale asociada
│                                            #   y para el rechazo 422 de discount
│
../pos-heladeria/                             # Angular 21 (standalone, signal stores)
├── src/app/modules/tables/
│   ├── pages/
│   │   └── table-sessions.component.ts       # retira el atajo F4 (D4); diálogo de éxito retira su
│   │                                         #   botón de impresión para el caso de un solo comprobante (D1)
│   ├── components/
│   │   ├── pos-order-panel.component.ts      # retira popover/botón de descuento (D4); botón "Anular"
│   │   │                                     #   condicionado a !paid (D3); texto de cabecera de tres
│   │   │                                     #   estados (D2)
│   │   └── pos-checkout-panel.component.ts   # botón "Reimprimir Factura POS" → "Imprimir Factura" (D1)
│   ├── services/
│   │   └── pos-terminal.store.ts             # deriveTableStatus: + condición paid (D2); retira señales/
│   │                                         #   métodos de descuento manual (D4); printReceipt/
│   │                                         #   lastReceipts retirado del camino de un solo comprobante (D1)
│   └── interfaces/dining.interface.ts        # DiningOrder: + paid: boolean (espejo de OrderResponse.paid)
└── src/app/modules/tables/services/
    └── pos-terminal.store.spec.ts            # actualización de describe('deriveTableStatus', ...) +
                                              #   tests nuevos para la ocultación de "Anular" y el
                                              #   texto de cabecera de tres estados
```

**Structure Decision**: se reutiliza la estructura ya existente de ambos repos (módulo `tables` en
el frontend, router `orders` en el backend); no se crea ningún módulo/paquete nuevo ni componente
nuevo. Todos los archivos listados ya existen — esta spec es exclusivamente una corrección
localizada sobre código ya en producción (spec 028), sin superficie nueva de UI.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No aplica — el Constitution Check no registró ninguna violación (ver tabla arriba); la única fila
en estado distinto de "PASA" es una acción de seguimiento con alcance ya documentado, no una
violación que requiera justificación.
