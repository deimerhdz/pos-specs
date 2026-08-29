# Implementation Plan: Habilitación del tipo de orden "Domicilio" en la creación manual de pedidos

**Branch**: `056-domicilio-orden-manual` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/056-domicilio-orden-manual/spec.md`

## Summary

Habilita la pestaña "🛵 Domicilio" (hoy deshabilitada) en `manual-order-page.component.ts`,
reusando el tipo de orden `DELIVERY` que spec 055 ya dejó modelado pero sin ningún punto de
creación real. Agrega tres campos nuevos a `CustomerOrder` (`delivery_address`, `delivery_phone`,
`delivery_fee`), obligatorios los dos primeros salvo el teléfono, únicamente cuando `order_type =
DELIVERY` (FR-003 a FR-007) — a diferencia del campo "Cliente" para "En Mesa"/"Para Llevar" (spec
054/055), que sigue defaulteando a "Consumidor final", para "Domicilio" ese mismo campo pasa a ser
obligatorio y sin ningún valor por defecto (decisión de negocio explícita de esta spec).

Hallazgo central de la investigación técnica (research.md Decisión 5): el pedido de la spec de que
"el valor del domicilio se sume al total de la orden" no es un cambio de una sola fórmula —
`Sale.total` se calcula hoy en 3 puntos independientes de `checkout.py` (`build_sale`, el chequeo
previo de efectivo `_order_total()`, y el pago autogenerado de `approve_payment_attempt`), y los
tres deben aprender sobre `delivery_fee` o el flujo de pago por transferencia de una orden
`DELIVERY` queda roto con un `422` real, no solo con un total mal mostrado. Los tres se actualizan
juntos, sin cambiar la forma de ningún endpoint de checkout (contracts/orders-checkout-total.md).

## Technical Context

**Language/Version**: Backend: Python 3.11+, FastAPI, SQLAlchemy 2.0 (`Mapped`/`mapped_column`),
Alembic. Frontend: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals, control flow
`@if`/`@for`).

**Primary Dependencies**: Backend: `fastapi`, `sqlalchemy`, `alembic`, `pydantic` — sin
dependencias nuevas. Frontend: `@angular/core`, `@angular/forms` (ya en uso) — sin dependencias
nuevas.

**Storage**: PostgreSQL, multi-tenant por schema (`for_each_tenant_schema`). Modifica
`customer_orders` (3 columnas nuevas + 1 `CheckConstraint`) y `sales` (1 columna nueva), ambas
schema `tenant` — ver data-model.md.

**Testing**: Backend: `pytest` sobre `app/characterization_tests/` (estilo `unittest.TestCase`).
Frontend: Vitest vía `@angular/build:unit-test`, specs colocados `*.component.spec.ts`/
`*.store.spec.ts`.

**Target Platform**: Web — API REST (`pos-backend`) consumida por la SPA Angular de la terminal POS
(`pos-heladeria`), navegador de escritorio/pantalla ancha.

**Project Type**: Aplicación web existente de dos repositorios hermanos a este (`pos-specs`):
`../pos-backend` (API) y `../pos-heladeria` (frontend). Cambia ambos.

**Performance Goals**: Sin objetivos numéricos nuevos — sin índice nuevo (data-model.md: estos tres
campos no son catálogo de filtrado, a diferencia de `channel`/`order_type` en spec 055).

**Constraints**: Cero cambio de comportamiento observable en órdenes que no sean `DELIVERY` (nuevas
o históricas) — el término `delivery_fee` vale `0`/`NULL` para cualquier otro `order_type`
(FR-012); ninguna `Sale` ya emitida se recalcula (FR-013, Principio VII); no se agregan
dependencias nuevas (Principio IX).

**Scale/Scope**: Backend: 1 migración Alembic nueva (sin backfill — columnas puramente aditivas),
`app/models/customer_order.py` (3 columnas nuevas + 1 constraint), `app/models/sale.py` (1 columna
nueva), `app/api/v1/orders/schemas.py` (`OrderCreate`/`OrderResponse`, 3 campos nuevos cada uno),
`app/api/v1/orders/service.py` (validación de campos obligatorios para `DELIVERY`),
`app/api/v1/sales/builder.py` (`build_sale`: parámetro + fórmula de total), `app/api/v1/orders/
checkout.py` (4 sitios que llaman a `build_sale`, más `_order_total()` y el `total` local de
`approve_payment_attempt` — research.md Decisión 5). Frontend: `dining.interface.ts` (3 campos
nuevos en 2 interfaces), `pos-terminal.store.ts` (3 signals nuevos, `totals()` extendido,
`createManualOrderFromDraft()` con tercera rama `esDomicilio`), `manual-order-page.component.ts`
(pestaña "Domicilio" habilitada, 4 campos nuevos condicionales, fila de total nueva,
`applyDefaultCustomerName()` con excepción para Domicilio, botón "Confirmar" con validación
nueva). Tests: characterization tests de backend actualizados (uno reescrito, spec 055's caso de
"Domicilio deshabilitado") + casos nuevos de validación y de total; specs de frontend nuevos para
la pestaña "Domicilio".

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/056-domicilio-orden-manual/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (la única ambigüedad real — alcance de "sumarse al total" —
  se resolvió con el usuario antes de finalizar la spec).
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente los
  tres cambios de comportamiento observable (habilitación real de "Domicilio"; campo "Cliente" sin
  valor por defecto solo para este tipo de orden; extensión de la fórmula de `Sale.total`) con
  autorización directa del dueño/desarrollador el 2026-08-29.
- **Principio III (Characterization tests)** ✅ — identificado con precisión el único test
  protegido que este spec contradice y por qué (research.md Decisión 11:
  `manual-order-page.component.spec.ts:148-157`, "Domicilio sigue deshabilitado" — spec 056 lo
  autoriza explícitamente vía FR-001) — satisface la exigencia de spec + decisión de negocio que
  justifique el cambio, sin dejar ningún otro test protegido en rojo (research.md Decisión 5
  documenta exactamente qué otros cálculos de total deben actualizarse para que
  `test_orders_checkout.py` no quede roto).
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (campos de
  domicilio, validación de obligatoriedad, extensión del total) está definido en spec.md, FR-001 a
  FR-014.
- **Principio V (No refactors oportunistas)** ✅ — no se toca `pos-tables-panel.component.ts` (su
  propia pestaña "Domicilios", FR-014, fuera de alcance — research.md Decisión 6 confirma el
  aislamiento por instancia de store); no se introduce ningún componente/servicio reutilizable
  nuevo sin un segundo consumidor real.
- **Principio VI (Evolución incremental)** — vigilar: esta spec mezcla, en un mismo incremento, una
  migración de esquema (2 tablas), una habilitación de UI, y una extensión de una fórmula de
  cálculo ya en producción (`Sale.total`). Se acepta porque los tres están genuinamente acoplados
  (no se puede habilitar "Domicilio" de forma útil sin que el valor del domicilio efectivamente se
  cobre, y no se puede cobrar sin tocar los 3 puntos de cálculo de total identificados en
  research.md Decisión 5) — no son cambios independientes agrupados por conveniencia.
- **Principio VII (Datos históricos)** ✅ — ninguna `Sale` ya emitida se recalcula; las columnas
  nuevas son puramente aditivas y nulables, sin backfill (data-model.md) — a diferencia de spec
  055, no existe ningún dato histórico `DELIVERY` que reclasificar.
- **Principio VIII (Evolución del modelo de datos)** ✅ — data-model.md documenta explícitamente
  las 4 columnas nuevas (3 en `customer_orders`, 1 en `sales`), su nulabilidad, ausencia de
  default, compatibilidad con datos existentes, estrategia de migración (research.md Decisión 10) y
  estrategia de rollback (`downgrade()` simétrico, sin remapeo de datos).
- **Principio IX (Dependencias nuevas)** ✅ — ninguna.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde `test_orders_service.py`, `test_orders_checkout.py`,
  `test_orders_kitchen.py`, `test_cart_single_active_order.py`, `test_orders_timezone.py` (todos
  candidatos a usar literales de canal/tipo de orden de spec 055 en su setup);
  `manual-order-page.component.spec.ts` y `pos-terminal.store.spec.ts`; reescribir el caso de
  "Domicilio deshabilitado" (Principio III) y agregar cobertura nueva para FR-003/FR-007
  (obligatoriedad) y FR-011/FR-012/FR-013 (total, quickstart.md Escenarios 1-4).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (habilitar "Domicilio", los
  tres campos nuevos, el valor sumado al total) viene del dueño/desarrollador en spec.md; las
  decisiones de este plan (research.md D1-D11) son todas técnicas — en particular, el hallazgo de
  los 3 puntos de cálculo de total (D5) es una decisión técnica para cumplir correctamente una
  regla de negocio ya autorizada (FR-011), no una funcionalidad de negocio nueva no pedida.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedido directo del dueño/desarrollador) → Spec
  056 → este Plan (research.md D1-D11) → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Sin desviaciones respecto a una lectura literal de spec.md que requieran
justificación — a diferencia de spec 055 (columna técnica `is_consolidation_order` no pedida por el
usuario), todas las columnas/cambios de este plan están directamente pedidos o son consecuencia
necesaria y documentada de FR-011 (research.md Decisión 5). Sin Complexity Tracking.

**Re-chequeo post-diseño (tras Fase 1)**: data-model.md y los contratos
(`contracts/orders-create.md`, `contracts/orders-checkout-total.md`) confirman que el diseño no
agrega dependencias nuevas (Principio IX), documenta migración y rollback completos (Principio
VIII), no afecta ningún dato histórico de facturación (Principio VII), y que el único test de
characterization contradicho (research.md Decisión 11) tiene una razón explícita amparada en
spec.md (Principio III). Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/056-domicilio-orden-manual/
├── plan.md                            # Este archivo (/speckit-plan)
├── research.md                        # Fase 0 (/speckit-plan) — decisiones D1-D11
├── data-model.md                      # Fase 1 (/speckit-plan)
├── contracts/
│   ├── orders-create.md               # Fase 1 (/speckit-plan) — POST /orders
│   └── orders-checkout-total.md       # Fase 1 (/speckit-plan) — impacto en Sale.total
├── quickstart.md                      # Fase 1 (/speckit-plan) — 4 escenarios de validación
└── tasks.md                           # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repositorios de la aplicación)

El código vive en los repositorios hermanos `../pos-backend` (FastAPI) y `../pos-heladeria`
(Angular), no en este repositorio de specs (`pos-specs`).

```text
pos-backend/
├── app/models/customer_order.py                # delivery_address, delivery_phone, delivery_fee
│                                                  # nuevos + CheckConstraint no-negativo (data-model.md)
├── app/models/sale.py                          # delivery_fee nuevo (mismo patrón que discount/tax/tip)
├── app/api/v1/orders/schemas.py                # OrderCreate/OrderResponse: 3 campos nuevos c/u
├── app/api/v1/orders/service.py                # create_order: validación de obligatoriedad para
│                                                  # DELIVERY (research.md Decisión 4)
├── app/api/v1/sales/builder.py                 # build_sale: parámetro delivery_fee + fórmula de
│                                                  # total extendida (research.md Decisión 5)
├── app/api/v1/orders/checkout.py               # 4 llamadas a build_sale (pay_order,
│                                                  # checkout_and_send, approve_payment_attempt,
│                                                  # confirm_cash_payment_attempt) + _order_total() +
│                                                  # total local de approve_payment_attempt
│                                                  # (research.md Decisión 5 — los 3 puntos)
├── alembic/versions/<nueva>_domicilio_orden_manual.py   # migración (research.md Decisión 10)
└── app/characterization_tests/
    ├── test_orders_service.py                  # casos nuevos: validación de campos obligatorios
    │                                              # DELIVERY (422 sin customer_name/address/fee)
    ├── test_orders_checkout.py                 # casos nuevos: Sale.total/delivery_fee en los 4
    │                                              # caminos de checkout, foco en approve_payment_attempt
    └── (sin characterization tests contradichos en backend — solo frontend, ver abajo)

pos-heladeria/
├── src/app/modules/tables/interfaces/dining.interface.ts     # OrderCreatePayload/DiningOrder:
│                                                               # 3 campos nuevos c/u
├── src/app/modules/tables/services/pos-terminal.store.ts     # deliveryAddress/deliveryPhone/
│                                                               # deliveryFee signals nuevos;
│                                                               # totals() extendido;
│                                                               # createManualOrderFromDraft(): rama
│                                                               # esDomicilio (research.md Decisión 7)
├── src/app/modules/tables/pages/manual-order-page.component.ts  # pestaña "Domicilio" habilitada;
│                                                               # 4 campos nuevos condicionales;
│                                                               # applyDefaultCustomerName() con
│                                                               # excepción Domicilio; botón "Confirmar"
│                                                               # con validación nueva; fila de total
│                                                               # nueva (research.md Decisión 8)
└── src/app/modules/tables/pages/manual-order-page.component.spec.ts,
    src/app/modules/tables/services/pos-terminal.store.spec.ts   # reescribe el caso "Domicilio
                                                                   # deshabilitado" (Principio III) +
                                                                   # casos nuevos para la pestaña
```

**Structure Decision**: sin componentes/servicios nuevos — todos los cambios caen dentro de
archivos ya existentes y ya referenciados por specs 036/054/055. `pos-tables-panel.component.ts`
(Terminal de Mesas general) no se toca: usa una instancia distinta de `PosTerminalStore` y esta
spec no le pide ningún cambio (research.md Decisión 6).

## Complexity Tracking

*Sin violaciones que justificar — ver "Resultado" en Constitution Check.*
