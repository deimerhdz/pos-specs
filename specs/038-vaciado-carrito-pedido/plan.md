# Implementation Plan: Vaciado del Carrito del Participante al Crear el Pedido (Menú QR)

**Branch**: `038-vaciado-carrito-pedido` | **Date**: 2026-08-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/038-vaciado-carrito-pedido/spec.md`

## Summary

Hoy `submit_cart` (`app/api/v1/cart/service.py:491-633`) marca el carrito del comensal como
`status='confirmado'` al crear el pedido — la fila y sus líneas sobreviven para siempre, sin que
nada las vuelva a leer salvo dos characterization tests. Esta spec cambia ese comportamiento
deliberadamente (FR-003, Principio III): al confirmar con éxito, el carrito y sus líneas se
**eliminan físicamente** (`db.delete(cart)`, aprovechando el `cascade="all, delete-orphan"` ya
declarado en `Cart.items` y el `ondelete="CASCADE"` de `cart_items`/`cart_item_options`) dentro de
la misma transacción que crea el pedido — sin migración de esquema para `Cart`/`CartItem`, ver
[research.md](./research.md) Decisión 1.

La segunda pieza es un snapshot de descuento por línea que hoy no existe en `OrderItem`: dos
columnas nuevas nullable (`discounted_unit_price`, `discounted_line_total`, mismos nombres que
`CartItemResponse` ya usa) pobladas en `submit_cart` con el mismo motor que ya usa
`serialize_cart` (`promotions.active_discount_promotions` + `best_line_discount`) — lógica que se
extrae a una función compartida en vez de duplicarse (research.md Decisión 4), para garantizar que
el pedido recién confirmado muestre exactamente lo que el carrito mostraba (FR-013/CA-6/SC-005).

La tercera pieza es reordenar las validaciones existentes de `submit_cart` para que, cuando el
carrito esté vacío o no exista **y** el comensal ya tenga un pedido no terminal, la respuesta sea
un `409` explícito de "ya fue enviado" en vez del `409`/`404` genérico de hoy (FR-007) — sin tocar
la garantía de última instancia contra duplicados (`idx_active_order_per_participant`, spec 025,
sin cambios, FR-008).

Una investigación de planeación (research.md Decisión 2) encontró que la Assumption del spec sobre
`Cart.status='confirmado'` quedando "sin ningún camino que lo produzca" es técnicamente inexacta:
`orders/consolidation.py::consolidate_table` (vía del mesero, explícitamente fuera de alcance de
esta spec) también asigna ese valor y sigue haciéndolo después de este cambio, protegido por un
tercer test `CONGELA` no citado por la spec (`test_orders_consolidation.py:269-315`) que esta spec
no toca ni afecta. Esto hace innecesaria cualquier alteración del `CheckConstraint` de
`Cart.status` — se documenta como decisión técnica, no como cambio a `spec.md` (la conclusión de
la Assumption, "no hace falta migrar el constraint", sigue siendo correcta; solo su premisa se
corrige aquí).

## Technical Context

**Language/Version**: Backend — Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`,
confirmado con `python3 --version`). Frontend — TypeScript 5.9.2 (Angular 21.1, standalone
components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync, `Mapped`/`mapped_column`), Alembic 1.18.4,
  Pydantic 2.13.4. **Ninguna dependencia nueva** (Principio IX no aplica) — todo lo que esta spec
  necesita (cascada ORM, motor de promociones, Alembic `@for_each_tenant_schema`) ya existe.
- Frontend: Angular 21 (standalone + signals), Tailwind CSS 4. Ninguna dependencia nueva.

**Storage**: PostgreSQL 16, **schema por tenant** (`{"schema": "tenant"}` en `Cart`, `CartItem`,
`CustomerOrder`, `OrderItem` — `app/models/*.py`), vía Alembic con `@for_each_tenant_schema`
(`app/scripts/tenant.py`, recorre `shared.tenants` + `tenant_default`). Una sola migración nueva:
dos columnas nullable en `order_items` (`discounted_unit_price`, `discounted_line_total`,
`Numeric(12,2)`). **Sin migración para `Cart`/`CartItem`** — el borrado físico es un cambio de
código (`db.delete(cart)`), no de esquema; ver [data-model.md](./data-model.md).

**Testing**: `unittest` vía `python -m unittest` (sin pytest en el repo, sin `conftest.py`). DB de
characterization tests: SQLite en memoria (`app/characterization_tests/cart_fixtures.py::new_session`,
con `_remove_partial_unique_indexes()` para los índices `postgresql_where`). Se reescriben, citando
esta spec, los dos tests `CONGELA` identificados en `spec.md` (`test_cart_service.py:484-522`,
`test_cart_router.py:216-252`) y se agregan tests nuevos por historia de usuario en los mismos
ficheros (sin fichero nuevo — ningún endpoint nace ni desaparece). Frontend: sin `.spec.ts` propio
para los componentes tocados (mismo criterio que el resto de `tables/pages/checkout/`) —
validación manual vía `ng serve` (ver [quickstart.md](./quickstart.md)).

**Target Platform**: Linux server (`pos-backend` en producción) + navegador (`pos-heladeria`, menú
público QR).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: Sin objetivo de throughput nuevo. El snapshot de descuento agrega, por línea
del carrito confirmado, el mismo cómputo que `serialize_cart` ya hace en cada `GET /cart` — sin
I/O adicional (las promociones activas ya se cargan una sola vez por llamada), así que el costo
marginal de `submit_cart` es el mismo orden de magnitud que hoy.

**Constraints**:
- FR-004 exige que el borrado del carrito y la creación del pedido compartan una única transacción
  — si la creación falla por cualquier motivo, ninguno de los dos cambios se aplica (research.md
  Decisión 1, mismo bloque `try/except` que ya existe).
- FR-006/FR-010 exigen que el borrado sea estrictamente por `participant_id` — sin cambios, ya es
  así (`_load_open_cart(db, participant.id)`).
- FR-014 prohíbe que el snapshot de descuento alimente o sustituya el cálculo de
  `orders/checkout.py` (`promotions.evaluate`/`combo_discount_for_lines`) — son caminos de código
  ya independientes hoy (research.md, verificación de Fase 0); esta spec no los toca.
- FR-015 exige que el campo nuevo sea `NULL` para `OrderItem` histórico, sin backfill — la
  migración no lleva `server_default` (Principio VII/VIII).
- Fuera de alcance explícito de `spec.md`: purga de carritos abandonados, carrito del terminal de
  staff, cambios al motor de promociones, modificar/cancelar un pedido ya enviado, la excepción al
  índice único de "una orden activa por participante", y registrar la decisión en
  `registro-de-anomalias.md` (tarea administrativa aparte).

**Scale/Scope**: 1 función modificada en profundidad (`submit_cart`), 1 función interna extraída y
compartida (cómputo de descuento por línea, hoy solo en `serialize_cart`), 1 migración nueva
(2 columnas nullable en `order_items`, sin tabla nueva), 2 characterization tests reescritos +
tests nuevos por historia de usuario en los mismos 2 ficheros, 1 schema Pydantic extendido
(`OrderItemResponse`) en `pos-backend`; en `pos-heladeria`: 1 interfaz TypeScript extendida
(`DiningOrderItem`, campos opcionales espejo del backend) — **sin cambio de plantilla/UI**, ver
research.md Decisión 8 (ninguna pantalla del comensal muestra hoy precio de línea de pedido).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 15 FRs y 4 clarificaciones ya resueltas (sesión 2026-08-26) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El cambio de comportamiento (borrado físico en vez de `'confirmado'`) está explícitamente documentado en `spec.md` §"Naturaleza de esta spec" y en las Clarifications, con la decisión de negocio ya tomada ahí ("quien encarga esta spec... ejerciendo el rol de negocio"). Registrada en `specs/000-reconocimiento/registro-de-anomalias.md` como A-53, con quién/cuándo/qué cambia/por qué/funcionalidades afectadas, tal como exige el Principio II. Todo lo demás es aditivo o de alcance ya acordado: FR-001/FR-010/FR-011 documentan explícitamente comportamiento **sin cambio**. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Los 2 tests `CONGELA` que `spec.md` identifica (`test_submit_cart_confirma_pedido_y_abre_carrito_nuevo`, `test_submit_cart_endpoint_evento_tras_commit`) se actualizan citando esta spec, siguiendo la convención ya usada por spec 020 (`quickstart.md` Paso 3: renombrar reflejando el comportamiento corregido, mantener el prefijo `CONGELA`). research.md Decisión 2 documenta evidencia adicional a la que trae `spec.md`: existe un **tercer** test `CONGELA` que también asegura `Cart.status == 'confirmado'` (`test_orders_consolidation.py:269-315`, sobre `consolidate_table`, vía del mesero) — no citado por `spec.md` porque protege un código que esta spec no toca (`orders/consolidation.py`, fuera de alcance, "carrito del terminal de staff"); se verifica que sigue en verde sin ninguna modificación, ya que el código que protege no cambia. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El borrado físico, el mensaje de "ya fue enviado" y el snapshot de descuento son comportamiento nuevo definido por `spec.md` — no se exige equivalencia con el pasado en esos tres puntos, solo conformidad con la spec y ausencia de regresiones no autorizadas en el resto de `submit_cart`/`orders`. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | La única extracción de código (mover el cómputo de descuento por línea de `serialize_cart` a una función compartida, research.md Decisión 4) es una consecuencia directa de FR-002/FR-013 — sin ella, `submit_cart` duplicaría la lógica de `serialize_cart` o el snapshot podría divergir del carrito que el comensal vio, violando SC-005. No se toca ningún otro código de `cart`/`orders` que no participe de esta funcionalidad. En frontend, research.md Decisión 8 explícitamente **rechaza** agregar una vista de precio en la pantalla "Mis pedidos" del comensal — ninguna FR/CA la pide, y esa pantalla no muestra ningún precio hoy; agregarla sería alcance no pedido. | PASS |
| **VI. Evolución Incremental** | El alcance se divide igual que las historias de `spec.md`: US1 (borrado + snapshot + respuesta sin error), US2 (mensaje de duplicado), US3 (segunda ronda, ya cubierta por `_get_or_create_open_cart` sin cambios), US4 (aislamiento por participante, ya cubierto por el filtro existente por `participant_id`). No se mezcla la migración de datos con ningún cambio de arquitectura: es una única migración aditiva y reversible, en el mismo incremento que el código que la necesita (Principio VIII lo exige explícito, no lo prohíbe). | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta o factura ya emitida. Las filas `OrderItem` anteriores a esta spec quedan con `discounted_unit_price`/`discounted_line_total` en `NULL` (FR-015), sin backfill ni recálculo — el frontend los trata como "sin descuento". Las filas de `Cart` con `status='confirmado'` que ya existen en producción (de antes de esta spec) no se tocan ni se purgan — la migración no las toca; el `CheckConstraint` sigue aceptando ese valor sin alteración (research.md Decisión 2). | PASS |
| **VIII. Evolución del Modelo de Datos** | data-model.md especifica las 2 columnas nuevas de `order_items` (tipo, nulabilidad, sin default), su compatibilidad con filas existentes (NULL, FR-015) y su estrategia de rollback (`op.drop_column` × 2, sin pérdida de datos porque son puramente aditivas). `Cart`/`CartItem` no cambian de esquema — el borrado es comportamiento de aplicación, documentado igual de explícito en data-model.md aunque no requiera migración. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario de `spec.md` tiene su "Independent Test"; quickstart.md los traduce a comandos `unittest` ejecutables sobre los ficheros de test existentes, más una verificación explícita de que el tercer test `CONGELA` (`test_orders_consolidation.py`) y el resto de la suite de `pos-backend` siguen en verde. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las decisiones de negocio (borrado físico en vez de archivado, snapshot de descuento nuevo, mensaje explícito de duplicado) están en `spec.md`/Clarifications. Cómo implementarlas (extraer la función de descuento compartida, reordenar las validaciones de `submit_cart`, nombres/tipos de columna, no tocar el `CheckConstraint` de `Cart.status`, no modificar la UI de "Mis pedidos") son decisiones técnicas documentadas en research.md, cada una con su alternativa descartada. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, sesión de clarificación 2026-08-26, citando `submit_cart` y los 2 tests CONGELA) → este `plan.md`/`research.md` (decisión técnica + hallazgo del tercer CONGELA) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests reescritos/nuevos + verificación explícita de no-regresión sobre `consolidation.py`/`checkout.py`/`qr_context.py` → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/038-vaciado-carrito-pedido/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — columnas, restricciones, ciclo de vida
├── quickstart.md          # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/              # Fase 1 (/speckit-plan) — contratos HTTP modificados
│   ├── cart-submit.md
│   ├── cart-get.md
│   └── cart-orders.md
└── tasks.md                # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── api/v1/cart/
│   ├── service.py                    # MODIFICADO — submit_cart(): borrado físico del carrito
│   │                                    (research.md D1), snapshot de descuento por línea (D4),
│   │                                    validaciones reordenadas para el mensaje de "ya fue
│   │                                    enviado" (D3). serialize_cart(): su bloque de descuento
│   │                                    por línea se extrae a una función compartida (D4), sin
│   │                                    cambiar su resultado observable.
│   └── router.py                     # SIN CAMBIOS — submit_cart() del router sigue publicando
│                                        events.order_created tras el commit; su cálculo de
│                                        `total` no se actualiza (research.md D6, fuera de alcance)
│
├── api/v1/orders/
│   └── schemas.py                    # MODIFICADO — OrderItemResponse (líneas 139-157) gana
│                                        discounted_unit_price/discounted_line_total (Decimal |
│                                        None), mismos nombres que CartItemResponse
│
├── models/
│   └── order_item.py                 # MODIFICADO — OrderItem gana discounted_unit_price/
│                                        discounted_line_total (Numeric(12,2), nullable, sin
│                                        default)
│                                        # cart.py / cart_item.py / customer_order.py: SIN CAMBIOS
│                                        de esquema (ver data-model.md)
│
├── characterization_tests/
│   ├── test_cart_service.py          # MODIFICADO — test_submit_cart_confirma_pedido_y_abre_
│   │                                    carrito_nuevo (líneas 484-522) reescrito y renombrado
│   │                                    (research.md D2/quickstart.md), citando esta spec; tests
│   │                                    nuevos para US1 (snapshot de descuento, atomicidad ante
│   │                                    fallo) y US3 (segunda ronda)
│   └── test_cart_router.py           # MODIFICADO — test_submit_cart_endpoint_evento_tras_commit
│                                        (líneas 216-252) con su aserción de ausencia de fila
│                                        reescrita, mismo nombre (la garantía que protege no
│                                        cambia); tests nuevos para US2 (mensaje de "ya fue
│                                        enviado") y US4 (aislamiento por participante)
│                                        # test_orders_consolidation.py: SIN CAMBIOS — se verifica
│                                        que test_consolidate_table_consolida_carritos_en_orden_
│                                        existente sigue en verde (research.md D2)
│
└── alembic/versions/
    └── {rev}_order_item_discount_snapshot.py  # NUEVO — 2 columnas nullable en order_items,
                                                  schema tenant, @for_each_tenant_schema
                                                  (down_revision='205f518df786', head actual)

# pos-heladeria
src/app/modules/tables/
└── interfaces/
    └── dining.interface.ts           # MODIFICADO — DiningOrderItem gana discounted_unit_price?/
                                         discounted_line_total? (string | null), espejo de
                                         CartItemResponse en diner.interface.ts. SIN cambios en
                                         ningún componente/plantilla (research.md D8) — payment-
                                         method-step.component.ts, transfer-details-step.
                                         component.ts, dining-cart.service.ts, checkout-hydration.
                                         guard.ts, confirmation-step.component.ts y public-menu.
                                         component.ts ya implementan FR-010/FR-011/FR-012 tal cual
                                         los exige la spec, sin ningún cambio de código requerido
```

**Structure Decision**: el cambio backend se concentra casi por completo en una sola función
(`submit_cart`) más una extracción interna que también usa `serialize_cart` — no se crea ningún
módulo/paquete nuevo, ni se toca `orders/consolidation.py`, `orders/checkout.py` ni
`core/qr_context.py` (los tres verificados en Fase 0: solo leen carritos `status='abierto'`, ajenos
al borrado de uno `'confirmado'`/eliminado). El frontend no requiere ningún cambio de componente:
las tres funciones que `spec.md` cita como "sin cambio" (FR-010/FR-011/FR-012) ya están
implementadas exactamente así hoy (verificado leyendo el código, no solo asumido), y la única
adición (`DiningOrderItem`) es un campo opcional sin consumidor visual — no hay UI que dividir por
historia de usuario porque no hay UI nueva.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
