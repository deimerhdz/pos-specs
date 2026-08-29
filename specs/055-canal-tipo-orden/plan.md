# Implementation Plan: Estandarización de canal y tipo de orden — habilitación de pedidos "Para Llevar"

**Branch**: `055-canal-tipo-orden` | **Date**: 2026-08-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/055-canal-tipo-orden/spec.md`

## Summary

Estandariza el canal de origen de `CustomerOrder` (`qr`/`counter`/`waiter` → `QR_MENU`/`POS`,
`WHATSAPP`/`API` nuevos sin punto de creación aún) y agrega un tipo de orden nuevo (`DINE_IN`/
`TAKEAWAY`/`DELIVERY`), ambos como `String` + `CheckConstraint` + índice dedicado (mismo patrón que
`status`, no un `ENUM` nativo — research.md D1). Valida al crear una orden que la combinación
canal×tipo de orden tenga sentido de negocio (FR-006), únicamente en el único punto donde ambos
valores son realmente variables (`orders/service.py::create_order`). Como consecuencia, habilita la
pestaña "Para Llevar" (hoy deshabilitada) en `manual-order-page.component.ts`: sin selección de
mesa, reusando el campo "Cliente" con "Consumidor final" por defecto que ya construyó spec 054.

Hallazgo central de la investigación técnica (research.md D2): fusionar `counter` y `waiter` en un
único valor `POS` — tal como pide la spec — rompería en silencio la segregación de la que depende
`orders/consolidation.py::get_or_create_open_order` (Terminal de Mesas, modo híbrido, spec 028)
para no reabrir una comanda de mostrador **ya cobrada** (`checkout_and_send` deja las órdenes
pagadas en `status='abierta'` a propósito). Se preserva con una columna técnica nueva,
`is_consolidation_order` (no expuesta en ningún schema de API, no forma parte del catálogo de
canal que ve el negocio) — ver Complexity Tracking.

## Technical Context

**Language/Version**: Backend: Python 3.11+, FastAPI, SQLAlchemy 2.0 (`Mapped`/`mapped_column`),
Alembic. Frontend: TypeScript 5.9.2, Angular 21.1 (componentes standalone, signals, control flow
`@if`/`@for`).

**Primary Dependencies**: Backend: `fastapi`, `sqlalchemy`, `alembic`, `pydantic` — sin
dependencias nuevas. Frontend: `@angular/core`, `@angular/forms` (ya en uso) — sin dependencias
nuevas.

**Storage**: PostgreSQL, multi-tenant por schema (`for_each_tenant_schema`). Modifica
`customer_orders` (schema `tenant`): reemplaza los valores de `channel`, agrega `order_type` e
`is_consolidation_order` — ver data-model.md.

**Testing**: Backend: `pytest` sobre `app/characterization_tests/` (estilo `unittest.TestCase`).
Frontend: Vitest vía `@angular/build:unit-test`, specs colocados `*.component.spec.ts`/
`*.store.spec.ts`.

**Target Platform**: Web — API REST (`pos-backend`) consumida por la SPA Angular de la terminal POS
(`pos-heladeria`), navegador de escritorio/pantalla ancha.

**Project Type**: Aplicación web existente de dos repositorios hermanos a este (`pos-specs`):
`../pos-backend` (API) y `../pos-heladeria` (frontend). Cambia ambos.

**Performance Goals**: Sin objetivos numéricos nuevos. Los índices nuevos (FR-002, FR-004) son
para que un filtro por canal/tipo de orden sobre `customer_orders` no dependa de un `seq scan` a
medida que la tabla crece — sin un volumen objetivo específico más allá de "eficiente" (spec.md).

**Constraints**: Cero cambio de comportamiento observable en los flujos QR y de consolidación
más allá de un *rename* verificado 1:1 de sus valores de canal (research.md D2, D4) — 0 tests con
prefijo `"CONGELA comportamiento actual:"` deben quedar en rojo tras el rename de literales
(`test_orders_consolidation.py`, y los demás archivos listados en research.md D2 que usan
`"waiter"`/`"counter"` como valor de setup); "Domicilio" sigue deshabilitado en
`manual-order-page.component.ts` (FR-012, fuera de alcance); no se agregan dependencias nuevas
(Principio IX).

**Scale/Scope**: Backend: 1 migración Alembic nueva, `app/models/customer_order.py` (3 columnas:
1 modificada, 2 nuevas), `app/api/v1/orders/schemas.py` (`OrderChannel` con valores nuevos,
`OrderType` nuevo, `OrderCreate`/`OrderResponse`), `app/api/v1/orders/service.py` (validación de
combinaciones + rechazo mesa+TAKEAWAY/DELIVERY + rename `QR`→`QR_MENU`),
`app/api/v1/orders/consolidation.py` (rename `channel=='waiter'` → `is_consolidation_order`),
`app/api/v1/cart/service.py` (rename de literales de canal). Frontend: `dining.interface.ts`
(tipos), `pos-terminal.store.ts` (`createManualOrderFromDraft()` + 3 literales `'qr'`),
`manual-order-page.component.ts` (pestaña "Para Llevar" habilitada, ocultar bloque "Mesas",
condición del botón "Confirmar"). Tests: characterization tests de backend actualizados +
casos nuevos de validación; specs de frontend nuevos para la pestaña "Para Llevar".

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Principio I (Nace de un spec)** ✅ — `specs/055-canal-tipo-orden/spec.md` existe, sin
  `[NEEDS CLARIFICATION]` pendiente (las dos ambigüedades reales se resolvieron con el usuario
  antes de escribir la spec).
- **Principio II (Comportamiento existente protegido)** ✅ — spec.md documenta explícitamente los
  dos cambios de comportamiento observable (catálogo de canal estandarizado; habilitación real de
  "Para Llevar") con autorización directa del dueño/desarrollador el 2026-08-29. El hallazgo
  técnico adicional de este plan (research.md D2, la colisión `counter`/`waiter`) es una
  **preservación** de comportamiento, no un cambio nuevo — verificado end to end (research.md D2,
  Escenario 5 de quickstart.md).
- **Principio III (Characterization tests)** ✅ — identificados con precisión los tests que deben
  actualizarse (research.md D2: `test_orders_consolidation.py:241,263` y los fixtures que usan
  `"waiter"`/`"counter"` como literal) y por qué el cambio es un *rename* comportamiento-preservado,
  no una modificación de fondo — satisface la exigencia de Principio III de justificar con spec +
  evidencia de que otros comportamientos protegidos no se ven afectados.
- **Principio IV (Nuevo comportamiento vía spec)** ✅ — todo el comportamiento nuevo (catálogo
  estandarizado, validación de combinaciones, habilitación de "Para Llevar") está definido en
  spec.md, FR-001 a FR-015.
- **Principio V (No refactors oportunistas)** ✅ — no se toca `pos-tables-panel.component.ts`
  (mismo tipo `OrderTypeTab`, pero instancia de store aparte — research.md D6) ni se introduce
  ningún componente/servicio reutilizable nuevo sin un segundo consumidor real.
- **Principio VI (Evolución incremental)** — vigilar: esta spec mezcla, en un mismo incremento,
  una migración de esquema, un rename de comportamiento interno (D2) y una habilitación de UI. Se
  acepta porque los tres están genuinamente acoplados (no se puede habilitar "Para Llevar" sin el
  campo `order_type`, y no se puede introducir el campo `channel` estandarizado sin resolver la
  colisión D2 que ese mismo cambio destapa) — no son cambios independientes agrupados por
  conveniencia. "Domicilio" se deja explícitamente fuera (FR-012) para no sumar un cuarto frente.
- **Principio VII (Datos históricos)** ✅ — no se toca ninguna `Sale`/factura ya emitida; el
  backfill de `channel`/`order_type` reclasifica metadatos de clasificación de `CustomerOrder`, no
  importes ni documentos fiscales ya emitidos (data-model.md).
- **Principio VIII (Evolución del modelo de datos)** ✅ — data-model.md documenta explícitamente
  las 3 columnas (1 modificada, 2 nuevas), sus defaults, compatibilidad con datos existentes,
  estrategia de migración (research.md D7) y estrategia de rollback (`downgrade()`, D7).
- **Principio IX (Dependencias nuevas)** ✅ — ninguna.
- **Principio X (Verificación obligatoria)** — pendiente de ejecución en `/speckit-tasks` +
  `/speckit-implement`: mantener en verde `test_orders_service.py`, `test_orders_consolidation.py`,
  `test_cart_single_active_order.py`, `test_orders_checkout.py`, `test_orders_kitchen.py`,
  `test_scheduler.py`, `test_orders_timezone.py`, `test_split_blindaje.py` (todos citados en
  research.md D2 por usar literales de canal), `manual-order-page.component.spec.ts` y
  `pos-terminal.store.spec.ts`; agregar cobertura nueva para FR-006/FR-007 (combinaciones),
  Decisión 5 (mesa+TAKEAWAY), y la pestaña "Para Llevar" (quickstart.md, Escenarios 1-5).
- **Principio XI (Negocio vs. técnico)** ✅ — la necesidad de negocio (catálogo estandarizado,
  validación de combinaciones, "Para Llevar" habilitado) viene del dueño/desarrollador en spec.md;
  las decisiones de este plan (D1-D7) son todas técnicas — en particular, `is_consolidation_order`
  es una decisión técnica para no romper una regla de negocio ya vigente (no reabrir una comanda
  cobrada), no una funcionalidad de negocio nueva.
- **Principio XII (Trazabilidad)** ✅ — Necesidad (pedido directo del dueño/desarrollador) → Spec
  055 → este Plan (research.md D1-D7) → Tasks/Implementación/Tests (siguientes comandos).
- **Principio XIII (Español de Colombia)** ✅ — todo este documento y los artefactos generados se
  redactan en español de Colombia.

**Resultado**: Gate PASA. Una desviación respecto a una lectura literal de spec.md requiere
justificación explícita — ver Complexity Tracking (columna técnica `is_consolidation_order`, no
pedida por el usuario pero necesaria para no romper Principio II).

**Re-chequeo post-diseño (tras Fase 1)**: data-model.md y el contrato (`contracts/orders-create.md`)
confirman que el diseño no agrega dependencias nuevas (Principio IX), documenta migración y
rollback completos (Principio VIII), no afecta ningún dato histórico de facturación (Principio VII),
y que cada test de characterization identificado en research.md D2 tiene un reemplazo señalado que
preserva el comportamiento que protegía (Principio III). Gate sigue PASANDO.

## Project Structure

### Documentation (this feature)

```text
specs/055-canal-tipo-orden/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan) — decisiones D1-D7
├── data-model.md                  # Fase 1 (/speckit-plan)
├── contracts/
│   └── orders-create.md           # Fase 1 (/speckit-plan) — POST /orders
├── quickstart.md                  # Fase 1 (/speckit-plan) — 5 escenarios de validación
└── tasks.md                       # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repositorios de la aplicación)

El código vive en los repositorios hermanos `../pos-backend` (FastAPI) y `../pos-heladeria`
(Angular), no en este repositorio de specs (`pos-specs`).

```text
pos-backend/
├── app/models/customer_order.py                     # channel: valores nuevos + índice;
│                                                       # order_type nuevo + índice;
│                                                       # is_consolidation_order nuevo (data-model.md)
├── app/api/v1/orders/schemas.py                      # OrderChannel: 4 valores nuevos;
│                                                       # OrderType nuevo; OrderCreate/OrderResponse
├── app/api/v1/orders/service.py                      # validación de combinaciones (FR-006/007);
│                                                       # rechazo mesa+TAKEAWAY/DELIVERY (research.md D5);
│                                                       # rename OrderChannel.QR → QR_MENU
├── app/api/v1/orders/consolidation.py                # get_or_create_open_order: channel=='waiter'
│                                                       # → is_consolidation_order (research.md D2)
├── app/api/v1/cart/service.py                        # rename de literales de canal (línea 589)
├── alembic/versions/<nueva>_estandariza_canal_tipo_orden.py   # migración (research.md D7)
└── app/characterization_tests/
    ├── test_orders_consolidation.py                  # assertEqual(channel, "waiter") → "POS" +
    │                                                   # is_consolidation_order (research.md D2)
    ├── orders_fixtures.py, table_sessions_fixtures.py # default "waiter" → "POS"
    ├── test_orders_service.py, test_orders_checkout.py, test_orders_kitchen.py,
    │   test_cart_single_active_order.py, test_scheduler.py, test_orders_timezone.py,
    │   test_split_blindaje.py                        # literales de canal, sin cambio de aserción
    └── (nuevos casos) validación de combinaciones, rechazo mesa+TAKEAWAY

pos-heladeria/
├── src/app/modules/tables/interfaces/dining.interface.ts     # OrderChannel/OrderType, payload,
│                                                               # 1 literal 'qr' → 'QR_MENU' (línea 234)
├── src/app/modules/tables/services/pos-terminal.store.ts     # createManualOrderFromDraft(): guard
│                                                               # de mesa condicionado a orderTypeTab;
│                                                               # payload channel/order_type/dining_table_id;
│                                                               # 3 literales 'qr' → 'QR_MENU' (líneas 162,425,450)
├── src/app/modules/tables/pages/manual-order-page.component.ts  # pestaña "Para Llevar" habilitada;
│                                                               # bloque "Mesas" condicionado; botón
│                                                               # "Confirmar" sin exigir mesa en Para Llevar;
│                                                               # applyDefaultCustomerName() también al
│                                                               # cambiar de pestaña (research.md D6)
└── src/app/modules/tables/pages/manual-order-page.component.spec.ts,
    src/app/modules/tables/services/pos-terminal.store.spec.ts   # casos nuevos para "Para Llevar"
```

**Structure Decision**: sin componentes/servicios nuevos — todos los cambios caen dentro de
archivos ya existentes y ya referenciados por specs 028/036/054. `pos-tables-panel.component.ts`
(Terminal de Mesas general) no se toca: usa una instancia distinta de `PosTerminalStore` y esta
spec no le pide ningún cambio (research.md D6).

## Complexity Tracking

> Desviación respecto a una lectura literal de spec.md (que solo pide dos campos: canal y tipo de
> orden) — se documenta explícitamente porque introduce una tercera columna no solicitada.

| Desviación | Por qué es necesaria | Alternativa más simple descartada |
|---|---|---|
| Columna técnica `is_consolidation_order` (no expuesta en ningún schema de API, no forma parte del catálogo de canal de negocio) | Fusionar `counter`/`waiter` en `POS` (pedido explícito de la spec) elimina la única señal que hoy usa `get_or_create_open_order` para no reabrir una comanda de mostrador ya cobrada (`status='abierta'` + venta ya emitida) — sin esta columna, el cambio de canal introduciría un bug de facturación real (research.md D2) | (a) No fusionar `counter`/`waiter` y exponer 5 valores de canal en la API — descartado, contradice el catálogo de 4 valores pedido explícitamente por el usuario. (b) Resolver la colisión con una consulta más permisiva (excluir órdenes con venta ya emitida) en vez de una columna — descartado, cambia el criterio real de "qué comanda es mía" a algo distinto de lo que es hoy, un comportamiento nuevo no autorizado por spec.md |
