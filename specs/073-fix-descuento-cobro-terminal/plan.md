# Implementation Plan: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Branch**: `073-fix-descuento-cobro-terminal` | **Date**: 2026-09-02 (rev. 2026-09-03: quinta superficie US7) | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/073-fix-descuento-cobro-terminal/spec.md`

> **Revisión 2026-09-03** — tras implementar las Historias 1-6, el dueño reprodujo el mismo
> defecto en el panel **"Pagos por confirmar"** (revisión de pago del cajero para un pedido de
> comensal por QR). La spec lo incorpora como **quinta superficie** (Historia 7, FR-021 a FR-024).
> Este plan añade una **cuarta pieza** al Summary y las decisiones D13-D15 a research.md. **No
> abre anomalía nueva**: `A-70` ya cubre el cambio de vigencia en `confirm_cash_payment_attempt` /
> `approve_payment_attempt`. Sin endpoint nuevo, sin migración, sin helper nuevo — reusa
> `compute_checkout_preview` (D4) y el instante congelado (D7).

## Summary

El reconocimiento de código (spec.md, sección "Contexto del defecto") ya aisló la causa raíz: la
Terminal valida el pago contra un total que arma el propio navegador
(`PosTerminalStore.totals()`, `pos-terminal.store.ts:883-896`, con `const discount = 0` fijo),
mientras el backend sí aplica la promoción al cobrar — dos cálculos independientes que solo
coinciden por casualidad. El mismo patrón (un total sin descuento calculado fuera de la autoridad
de la venta) reaparece en el chequeo previo del efectivo del panel "Pagos por confirmar"
(`_order_total`, `checkout.py:939-955`) — la quinta superficie que la revisión del 2026-09-03
añadió. La corrección tiene **cuatro** piezas independientes y verificables por separado:

1. **Autoridad del monto (US1-US3, FR-001 a FR-007a)** — dos endpoints de solo lectura nuevos en
   `pos-backend`: `GET /orders/{order_id}/checkout-preview` (pedido ya creado: mesa, para llevar,
   domicilio) y `POST /orders/draft-preview` (borrador sin guardar de la pantalla de armado
   manual). Ambos devuelven `{subtotal, discount, delivery_fee, total}` calculados con el mismo
   motor que ya usa el cobro real (`order_sale_lines` + `auto_discount`, reuso literal). En
   `pos-heladeria`, `PosTerminalStore.totals()` deja de calcular localmente y pasa a consumir
   estos endpoints con el mismo molde señal-loading-error que ya usa `loadSessionBill()`
   ([contracts/preview-cobro-pedido.md](./contracts/preview-cobro-pedido.md),
   [contracts/preview-borrador-orden-manual.md](./contracts/preview-borrador-orden-manual.md)).

2. **Vigencia congelada (US4, FR-008 a FR-012a, FR-018/FR-018a)** — columna nueva
   `promotion_evaluated_at` (nulable, sin backfill, **`DateTime(timezone=True)` — aware UTC**, porque
   el valor se pasa a `local_now()`, que trataría un naive como hora local del tenant) en
   `customer_orders` y en `sales`, poblada una sola vez al crear el pedido. Un único helper
   (`promotion_evaluation_instant`) decide, en cada punto donde hoy se calcula "la hora del cobro",
   si usa ese instante congelado (o el más antiguo entre varias rondas de una misma mesa que se
   cobran juntas; **por pedido** en las mesas fusionadas, que se cobran individualmente — FR-018a)
   o cae a la hora actual para pedidos anteriores a esta spec. El instante que efectivamente se usó
   queda en `sales.promotion_evaluated_at` y se expone en `SaleResponse` + el detalle de venta del
   frontend (FR-011a/SC-009). El **estado** de la promoción nunca se congela — ya se lee vivo en
   cada llamada, sin cambio necesario
   ([contracts/vigencia-congelada-promocion.md](./contracts/vigencia-congelada-promocion.md)).
   Deroga el comportamiento actual — requiere `A-70` en `registro-de-anomalias.md` antes de
   implementar (Principio II).

3. **Catálogo de la Terminal (US6, FR-016/FR-017)** — cero cambios de backend: la condición
   legible ya publicada para el menú QR (spec 066, `MenuVariantPromotion`) ya llega intacta al
   store de la Terminal; solo se reemplaza el cálculo local de la insignia
   (`productDiscountBadge()`, `pos-terminal.store.ts:404-441`) por leer ese dato directo
   ([contracts/catalogo-condicion-legible.md](./contracts/catalogo-condicion-legible.md)).

4. **Revisión de pago del cajero para pedidos QR (US7, FR-021 a FR-024)** — quinta superficie,
   añadida el 2026-09-03. **Cero endpoint nuevo, cero migración, cero helper nuevo**: el chequeo
   previo del efectivo de `confirm_cash_payment_attempt` deja de usar `_order_total` (suma sin
   descuento) y pasa a `compute_checkout_preview(...).total` (D4) — `_order_total` se elimina
   (D13). El panel `PaymentAttemptReviewPanelComponent` consume `checkoutPreview(order.id)` (método
   de `DiningSessionService` ya existente), muestra el desglose `Subtotal / Descuento / Domicilio /
   Total`, calcula el vuelto sobre el total real y aplica la reconfirmación de FR-024 cuando el
   total cambió (D14/D15). `approve_payment_attempt` (transferencia) ya emitía la venta correcta
   desde T028 — solo se le añade la vista del total antes de "Aprobar", en el frontend.
   [contracts/revision-pago-cajero-qr.md](./contracts/revision-pago-cajero-qr.md).

Las cuatro piezas comparten una sola pieza de negocio: **nunca duplicar el cálculo de descuento en
el navegador ni en un chequeo previo del backend** (spec 063, FR-023 — research.md D9 confirma que
esta decisión ya existía y simplemente no llegó a cablearse para los pedidos que no vienen del
carrito QR ni para el chequeo del "monto recibido"). El detalle completo de cada decisión técnica
está en [research.md](./research.md) (D1-D12; D13-D15 para US7).

## Technical Context

**Language/Version**: Python 3.12 / FastAPI 0.136 (`pos-backend`); TypeScript 5.9 / Angular 21
standalone, signals (`pos-heladeria`) — sin cambio de versión en ninguno de los dos repos.

**Primary Dependencies**: sin dependencia nueva en ningún repo (Principio IX: nada que
justificar). Backend: SQLAlchemy 2.0, Alembic 1.18, Pydantic 2.13. Frontend: `@angular/*` 21,
RxJS, `@tanstack/angular-query-experimental` (no se usa aquí — el store ya tiene su propio patrón
señal+servicio HTTP, research.md D10).

**Storage**: PostgreSQL 16, schema-per-tenant. **Una migración nueva, aditiva y no retroactiva**
(ver [data-model.md](./data-model.md)): `promotion_evaluated_at` nulable, `DateTime(timezone=True)`
(aware UTC — ver Constitution Check §VIII), en `customer_orders` y en `sales`, sin backfill,
siguiendo el patrón de `d427cd419e79_domicilio_orden_manual.py` (spec 056) salvo por el tipo de
columna con zona.

**Testing**: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
(backend) y `npm test` / Vitest (`*.spec.ts`, frontend) — sin cambio de framework en ninguno de
los dos. Ningún test con prefijo `"CONGELA comportamiento actual:"` cubre hoy directamente el
cálculo del total en `pos-terminal.store.spec.ts`/`pos-checkout-panel.component.spec.ts` ni el
`now` de `auto_discount` en `test_orders_checkout.py`/`test_table_sessions_service.py` (confirmado
por lectura de ambas suites) — los que sí toquen ese comportamiento se actualizan citando `A-70`
explícitamente (Principio III), nunca en silencio. Para US7: los tests de `confirm_cash_payment_attempt`
viven en `test_orders_payment_gate.py` (specs 024/026/028/056, **no** `"CONGELA"`) y hoy usan
pedidos sin promoción, así que `discount = 0` los deja verdes tras el cambio de `_order_total`; se
añaden casos con promoción vigente. En el frontend, `payment-attempt-review-panel.component.spec.ts`
y `payment-validation-block.component.spec.ts`. Ver
[quickstart.md](./quickstart.md) para el detalle de validación por historia.

**Target Platform**: web, Terminal de mesas y back-office (`pos-heladeria`) consumiendo
`pos-backend` vía HTTP/JSON. Sin cambios de infraestructura ni despliegue.

**Project Type**: Web application, dos repositorios independientes ya en producción — esta spec
toca **ambos** (a diferencia de la 072, que era 100% frontend): backend gana 2 endpoints + 1
migración + 1 helper reutilizado en 8 call sites + 1 campo en `SaleResponse` + (US7) el chequeo
previo de `confirm_cash_payment_attempt` pasa a `compute_checkout_preview` y `_order_total` se
elimina; frontend reemplaza un cálculo local por consumo de esos endpoints en 2 pantallas +
simplifica el catálogo en 2 componentes + agrega una fila al detalle de venta del módulo `sales` +
(US7) cablea el preview autoritativo en `PaymentAttemptReviewPanelComponent` /
`PaymentValidationBlockComponent`.

**Performance Goals**: `SC-008` — el desglose con descuento aparece en ≤ 1 segundo en el 95% de
las aperturas del panel de cobro; por encima de eso, estado "calculando" visible y "Cobrar"
deshabilitado hasta recibir el total (FR-007a) — nunca un total provisional. Los endpoints nuevos
reusan exactamente las mismas consultas que ya ejecuta el cobro real (`order_sale_lines`,
`auto_discount`), sin agregar ninguna consulta N+1 nueva: mismo costo que el cobro que ya corre
hoy, solo que antes en vez de en el momento de pagar.

**Constraints**: FR-019/FR-020 (spec 029, spec 046) — sin descuento manual, un único método de
pago por cobro — se conservan sin tocar. FR-011/Principio VII — ninguna venta ya emitida cambia de
importe, desglose ni representación; la columna nueva en `sales` queda `NULL` en todo lo histórico
sin ningún `UPDATE`.

**Scale/Scope**: 2 repositorios. Backend: 1 migración (2 tablas, `DateTime(timezone=True)`), 2
endpoints nuevos (`checkout-preview`, `draft-preview`), 1 helper nuevo
(`promotion_evaluation_instant`) aplicado en 8 call sites existentes de
`auto_discount`/`evaluate_variant_sets` (uno de ellos, `group_bill`, por pedido dentro del bucle —
FR-018a), 1 kwarg nuevo en `build_sale`, 1 campo nuevo en `SaleResponse`, y (US7) 1 función
eliminada (`_order_total`) con su única llamada repuntada a `compute_checkout_preview`. Frontend:
`PosTerminalStore` gana 2 pares de signals (preview + draft-preview) con su loading/stale,
`PosCheckoutPanelComponent`/`manual-order-page.component.ts` cambian de dónde leen
`total`/`discount`, `pos-catalog-drawer.component.ts` y `manual-order-page.component.ts` cambian
la fuente de la insignia de promoción, `sales-page.component.ts` gana una fila con el instante
congelado en el detalle de venta, y (US7) `PaymentAttemptReviewPanelComponent` gana un trío de
signals locales para el preview + desglose + reconfirmación mientras
`PaymentValidationBlockComponent` pierde su fila de total local. Cero archivos nuevos de
componente — todo son extensiones de servicios/componentes ya existentes.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra la [Constitución v3.0.0](../../.specify/memory/constitution.md).

| Principio | Estado | Evidencia |
|---|---|---|
| I. Las nuevas funcionalidades nacen de un spec | ✅ | [spec.md](./spec.md) aprobada, autorizada por el dueño el 2026-09-02; Historia 7 (quinta superficie) autorizada el 2026-09-03 (encabezado "Autorización adicional" + Clarifications sesión 2026-09-03). |
| II. El comportamiento existente sigue protegido | ⚠️→✅ | Es mayormente una corrección (US1-US3, US5, US6, **US7** restauran comportamiento pretendido — spec.md lo declara explícito, no abren anomalía), **salvo** US4/FR-009 que sí deroga una regla de negocio vigente (evaluar siempre con la hora del cobro). Requiere `A-70` en `registro-de-anomalias.md` **antes** de implementar el paso 3 del flujo de [contracts/vigencia-congelada-promocion.md](./contracts/vigencia-congelada-promocion.md) — **ya registrada** (2026-09-02, entrada `### A-70 — [DECISIÓN DE NEGOCIO — spec 073]`, tarea T001 completada). `A-70` ya enumera `confirm_cash_payment_attempt` y `approve_payment_attempt` entre sus 8 call sites, así que **US7 no abre anomalía nueva** — spec.md §"Autorización adicional (2026-09-03)" lo declara explícito ("es el mismo defecto de FR-001/FR-002"). Bloqueante para cualquier tarea que toque `promotion_evaluation_instant` — ya desbloqueado. |
| III. Los characterization tests protegen el comportamiento heredado | ✅ | research.md (Technical Context) confirma que ningún test hoy lleva el prefijo `"CONGELA comportamiento actual:"` sobre el cálculo de total del frontend ni sobre el `now` de `auto_discount` en las suites relevantes de `pos-backend`. Cualquier test que SÍ deba actualizarse por el cambio de vigencia (US4) lo hace citando `A-70` en el mismo commit, con evidencia de que FR-009a (estado vivo) y FR-011/FR-012 (nada retroactivo) siguen intactos — ambos verificados en quickstart.md. |
| IV. Los nuevos specs pueden introducir nuevo comportamiento | ✅ | US4 (FR-009) es exactamente eso — comportamiento nuevo autorizado, no equivalencia con el pasado. |
| V. Nuevas funcionalidades antes que refactorizaciones oportunistas | ✅ | `checkout.compute_bill`/`BillResponse` (`GET /orders/tables/{table_id}/bill`) está confirmado roto (no aplica `auto_discount`) pero sin ningún consumidor en `pos-heladeria` (research.md D4) — **no se toca**, aunque comparte síntoma, porque ninguna de las 5 superficies en alcance lo usa. `promotion_evaluation_instant` se aplica a `table_sessions`/`tables_advanced` no como refactor libre sino porque FR-018/FR-018a lo piden explícitamente. En `tables_advanced.py::group_bill` es además **obligado**: sus pedidos se cobran individualmente por `checkout_and_send`/`pay_order` (que sí cambian), así que sin el mismo cambio el preview consolidado dejaría de coincidir con lo cobrado (research.md D3). **US7**: eliminar `_order_total` no es refactor oportunista — FR-023 exige que el chequeo previo compare contra el `Total` real, y `_order_total` es justo la aritmética sin descuento que hay que quitar; se elimina en vez de dejarla latente porque no tiene ningún otro consumidor (research.md D13). |
| VI. Evolución incremental | ✅ | Cuatro piezas (autoridad del monto / vigencia congelada / catálogo / revisión de pago del cajero QR) verificables por separado — cada una con su propio contrato y su propia sección de quickstart.md, sin depender entre sí para poder probarse (los endpoints de preview funcionan aunque `promotion_evaluated_at` sea `NULL`; el catálogo no depende de ninguna de las otras; US7 reusa el endpoint de US1 sin tocarlo). Una sola migración, aditiva, sin mezclarse con cambios de arquitectura. US7 llegó como incremento posterior (2026-09-03) y se mantiene como unidad aparte, no reabre las piezas ya entregadas. |
| VII. Compatibilidad con datos históricos | ✅ | Ninguna venta ni pedido existente se recalcula: `promotion_evaluated_at` nace `NULL` en el 100% de las filas actuales, sin backfill (data-model.md), y el helper D3 cae a "hora del cobro" exactamente cuando el campo es `NULL` — comportamiento idéntico al actual para todo lo histórico (`SC-007`, verificado en quickstart.md Historia 4, punto 4). |
| VIII. Evolución del modelo de datos | ✅ | [data-model.md](./data-model.md) especifica columnas, nulabilidad, tipo (`DateTime(timezone=True)` — aware UTC, con el motivo: `local_now()` trataría un naive como hora local del tenant), quién escribe cada una, compatibilidad con datos existentes, y el `upgrade`/`downgrade` completo de la migración (rollback = `drop_column`, sin pérdida de nada preexistente). |
| IX. Dependencias nuevas justificadas | ✅ N/A | Ninguna dependencia nueva en ningún repo. |
| X. Verificación obligatoria | ✅ | [quickstart.md](./quickstart.md) define escenarios ejecutables por las 7 historias más verificación cruzada de alcance protegido (sesión de mesa, QR en menú/carrito, sin descuento manual, único método de pago, `_order_total` eliminado), y exige correr ambas suites de characterization tests antes/después como red de seguridad. |
| XI. Decisiones de negocio frente a técnicas | ✅ | La decisión de negocio (vigencia = instante de creación, no de cobro; quinta superficie en alcance) ya está tomada en spec.md por el dueño; las decisiones técnicas de este plan (dónde vive el campo, cómo se agrega entre rondas, forma de los endpoints, reusar `compute_checkout_preview` para el chequeo previo de US7) están en research.md D1-D7, D12-D15, sin mezclarse con la decisión de negocio misma. |
| XII. Trazabilidad | ✅ | Necesidad (bug reportado, reproducido con un pedido real; US7 reproducido de nuevo el 2026-09-03) → spec 073 (FR-001…FR-024, FR-018a) → Clarifications (3 sesiones) → este plan → research.md (15 decisiones) → data-model.md → 5 contratos → quickstart.md → `A-70` (registrada 2026-09-02, ya cubre US7) → tasks.md → tests → verificación. FR-021 a FR-024 (quinta superficie) quedan enumerados en el spec y en [contracts/revision-pago-cajero-qr.md](./contracts/revision-pago-cajero-qr.md). |
| XIII. Todo en español de Colombia | ✅ | Los textos de UI nuevos que introduce esta spec — el estado "calculando" (FR-007a/FR-024), el aviso de descuento no confirmado (FR-015) y el aviso de "el total cambió respecto al declarado por el comensal" (FR-024) — se redactan en español de Colombia al implementarse, mismo tono que el resto de mensajes ya existentes en `pos-heladeria` (p. ej. "Faltan $X para cubrir la cuenta."). |

### Re-evaluación post-diseño (Phase 1)

Repasada tras generar data-model.md, los cuatro contratos y quickstart.md; **revisada de nuevo el
2026-09-03** tras añadir el quinto contrato (revisión de pago del cajero QR) y las decisiones
D13-D15. **Sin violaciones nuevas.**

- **II confirmado con más precisión**: el diseño de Phase 1 no amplió el alcance de lo que deroga
  FR-009 — sigue siendo exclusivamente "qué instante se usa para la vigencia temporal", nunca el
  estado de la promoción (FR-009a, invariante 2 de
  [contracts/vigencia-congelada-promocion.md](./contracts/vigencia-congelada-promocion.md)). La
  entrada `A-70` **ya está registrada** (2026-09-02) — implementación de la Fase 6 desbloqueada.
- **II + XII — alcance de A-70 alineado con el spec**: la cuenta consolidada de mesas fusionadas
  (`tables_advanced.py::group_bill`) estaba en el alcance de A-70 y de `tasks.md` pero no en el
  texto de FR-018; se agregó **FR-018a** al spec para cerrar la cadena Necesidad→Spec→Decisión.
  El instante en mesas fusionadas se decide **por pedido** (no `MIN` de grupo), porque esos
  pedidos se cobran individualmente y el preview debe coincidir con el cobro.
- **VIII — tipo de columna**: `promotion_evaluated_at` es `DateTime(timezone=True)` (aware UTC),
  no `DateTime` naive como `created_at`: el valor se pasa a `local_now()`, que interpreta un naive
  como hora local del tenant y desplazaría la evaluación de la franja por el offset (research.md
  D1). La migración se desvía a propósito del esqueleto de `d427cd419e79` en este punto.
- **X + FR-011a — SC-009 ahora sí verificable**: `sales.promotion_evaluated_at` se expone en
  `SaleResponse` y en el detalle de venta del frontend (`tasks.md` T032a-T032c); antes solo se
  persistía, dejando SC-009 sin ruta de lectura.
- **V confirmado con evidencia adicional**: el diseño explícitamente **descarta** tocar
  `checkout.compute_bill` (endpoint sin consumidor) pese a compartir el mismo síntoma — la
  tentación de "ya que estamos" quedó registrada y rechazada en research.md D4, no solo evitada
  por omisión.
- **VI confirmado**: los 5 contratos de Phase 1 mapean 1:1 a las cuatro piezas del Summary
  (`preview-cobro-pedido` + `preview-borrador-orden-manual` = pieza 1; `vigencia-congelada` =
  pieza 2; `catalogo-condicion-legible` = pieza 3; `revision-pago-cajero-qr` = pieza 4) — ningún
  contrato mezcla piezas independientes.
- **II + V + XII — US7 (quinta superficie, 2026-09-03)**: es el mismo defecto de FR-001/FR-002 en
  el chequeo previo del efectivo (`_order_total`) y en el frontend del panel "Pagos por confirmar";
  **no abre anomalía nueva** (spec.md lo declara; `A-70` ya cubre el cambio de vigencia en
  `confirm_cash_payment_attempt`/`approve_payment_attempt`). El diseño **no** añade endpoint,
  migración ni helper — reusa `compute_checkout_preview` (D4) y el instante congelado (D7).
  Eliminar `_order_total` es exactamente lo que pide FR-023, no un refactor oportunista (research.md
  D13). `approve_payment_attempt` no se toca en backend (ya emitía la venta correcta desde T028) —
  se deja constancia de que se revisó.

Ningún gate queda en rojo.

## Project Structure

### Documentation (this feature)

```text
specs/073-fix-descuento-cobro-terminal/
├── plan.md                                    # Este archivo
├── research.md                                # Decisiones D1-D12; D13-D15 (US7)
├── data-model.md                               # Columnas nuevas + migración + CheckoutPreview
├── contracts/
│   ├── preview-cobro-pedido.md                 # FR-001 a FR-007a (US1-US3); reusado por US7
│   ├── preview-borrador-orden-manual.md        # FR-013 a FR-015a (US5)
│   ├── vigencia-congelada-promocion.md         # FR-008 a FR-012a (US4) + A-70
│   ├── catalogo-condicion-legible.md           # FR-016/FR-017 (US6)
│   └── revision-pago-cajero-qr.md              # FR-021 a FR-024 (US7) — quinta superficie
├── quickstart.md                               # Validación ejecutable por historia (1-7)
├── checklists/                                 # (ya generadas por /speckit-clarify o previas)
└── tasks.md                                    # Salida de /speckit-tasks (no de este comando)
```

### Source Code (repository root)

```text
../pos-backend/
├── alembic/versions/
│   └── <rev>_congela_instante_vigencia_promociones.py   # data-model.md — 2 columnas DateTime(timezone=True), sin backfill
├── app/
│   ├── models/
│   │   ├── customer_order.py       # + promotion_evaluated_at (nullable, DateTime(timezone=True))
│   │   └── sale.py                 # + promotion_evaluated_at (nullable, DateTime(timezone=True))
│   └── api/v1/
│       ├── orders/
│       │   ├── checkout.py         # + promotion_evaluation_instant(); + compute_checkout_preview()
│       │   │                       #   8 call sites existentes pasan a usar el helper (research.md D3)
│       │   │                       #   group_bill: helper POR PEDIDO dentro del bucle (FR-018a)
│       │   │                       #   US7: confirm_cash_payment_attempt usa compute_checkout_preview
│       │   │                       #        para el chequeo previo; _order_total ELIMINADO (D13)
│       │   ├── service.py          # create_order: fija promotion_evaluated_at (aware UTC) al crear
│       │   ├── schemas.py          # + CheckoutPreviewResponse, + DraftPreviewIn
│       │   └── router.py           # + GET /{order_id}/checkout-preview, + POST /draft-preview
│       ├── cart/service.py         # segundo punto de creación: fija promotion_evaluated_at
│       ├── sales/builder.py        # build_sale: + kwarg promotion_evaluated_at; compute_total() extraída
│       ├── sales/schemas.py        # + promotion_evaluated_at en SaleResponse (FR-011a/SC-009)
│       └── table_sessions/service.py  # compute_bill/_close_unified/_close_split usan el helper
│
└── app/characterization_tests/
    ├── test_orders_checkout.py       # + tests de compute_checkout_preview, promotion_evaluation_instant, SaleResponse
    ├── test_orders_payment_gate.py   # US7: + tests de confirm_cash con promoción (chequeo previo = total real)
    ├── test_table_sessions_service.py # + tests de FR-012a (rondas sucesivas)
    ├── test_orders_tables_advanced.py # + tests de FR-018a (mesas fusionadas, por pedido)
    ├── test_cart_service.py           # + test de FR-018 (QR congela el instante)
    └── test_promotions_service.py     # sin cambios (el motor no cambia de firma)

../pos-heladeria/
└── src/app/modules/
    ├── tables/
    │   ├── services/
    │   │   ├── pos-terminal.store.ts             # totals() → checkoutPreview()/draftPreview();
    │   │   │                                      #   elimina productDiscountBadge(s)
    │   │   ├── pos-terminal.store.spec.ts         # + tests de los 2 preview, catálogo sin badge local
    │   │   ├── dining-session.service.ts          # (sin cambios — checkoutPreview() ya existe, US7 lo reusa)
    │   │   └── table-session.service.ts           # (sin cambios — ya sirve de referencia de patrón)
    │   ├── components/
    │   │   ├── pos-checkout-panel.component.ts    # [total] → checkoutPreview()?.total; estado "calculando"
    │   │   ├── pos-checkout-panel.component.spec.ts
    │   │   ├── payment-input.component.ts         # sin cambios (ya recibe total genérico)
    │   │   ├── pos-catalog-drawer.component.ts    # badge local → variant.promotion.short_condition
    │   │   ├── payment-attempt-review-panel.component.ts   # US7: + signals preview local, desglose,
    │   │   │                                               #      vuelto sobre total real, reconfirmación FR-024
    │   │   ├── payment-attempt-review-panel.component.spec.ts
    │   │   ├── payment-validation-block.component.ts       # US7: elimina fila $ total local + método total()
    │   │   └── payment-validation-block.component.spec.ts
    │   └── pages/
    │       ├── manual-order-page.component.ts     # resumen + fila Descuento + reconfirmación FR-015a
    │       └── manual-order-page.component.spec.ts
    └── sales/                                     # detalle de venta — FR-011a/SC-009
        ├── interfaces/sales.interface.ts          # + promotion_evaluated_at en Sale
        ├── pages/sales-page.component.ts          # detalle: fila "promociones evaluadas con vigencia de {fecha}"
        └── pages/sales-page.component.spec.ts
```

**Structure Decision**: sin archivos nuevos de componente en ningún repo — todo son extensiones de
módulos ya existentes (`checkout.py`, `pos-terminal.store.ts`,
`payment-attempt-review-panel.component.ts`) o archivos nuevos estrictamente de infraestructura (la
migración) y de contrato (`schemas.py`). US7 no añade ningún archivo — reusa el endpoint
`checkout-preview`, el método `DiningSessionService.checkoutPreview()` y el instante congelado, ya
existentes. El árbol completo de tareas concretas (qué línea exacta cambia en cada archivo) es
responsabilidad de `tasks.md` (`/speckit-tasks`), no de este plan — la revisión del 2026-09-03
implica que `tasks.md` gana una **Fase nueva para US7** (P1), sin tocar las fases 1-9 ya
completadas.

## Complexity Tracking

> Sin violaciones del Constitution Check que justificar — tabla omitida. El único punto marcado
> ⚠️ (Principio II) se resuelve con un ítem de proceso (`A-70` antes de implementar), no con una
> excepción a la Constitución.
