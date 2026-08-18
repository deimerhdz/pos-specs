# Implementation Plan: Corrección de la cuenta de mesas fusionadas (`group_bill`)

**Branch**: `019-correccion-cuenta-mesas-fusionadas` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/019-correccion-cuenta-mesas-fusionadas/spec.md`

## Summary

`tables_advanced.group_bill` (`app/api/v1/orders/tables_advanced.py:92-114`) hoy suma
`unit_price * quantity` de todo ítem no `anulado` de **cualquier** orden del grupo, sin importar su
`status` ni ninguna promoción vigente — el camino C de la anomalía A-01. Esta spec lo corrige para
que replique el criterio de `table_sessions.compute_bill` (`app/api/v1/table_sessions/service.py:139`,
camino A, correcto y no tocado): excluir órdenes `pagada`/`cancelada` (FR-001) y descontar
promociones/combos vigentes vía `promotions.evaluate`/`combo_discount_for_lines` (FR-002).

El enfoque técnico evalúa promociones y combos **por orden** (reutilizando
`checkout.order_sale_lines(db, order.id)` sin filtrar por comensal), no replicando literalmente el
bucle por comensal de `compute_bill`. Es una decisión deliberada, no una simplificación arbitraria:
`promotions.evaluate_detailed` decide el descuento de cada línea usando solo los campos de esa
misma línea (`quantity`, `product_id`/`category_id`, `line_total`) — nunca agregando cantidades
entre líneas del lote — así que agrupar por orden en vez de por comensal produce exactamente el
mismo descuento por línea, y por tanto el mismo total, que agrupar por comensal (research.md
Decisión 1, con la prueba completa). Evaluar por orden es además el cambio de menor superficie sobre
el `group_bill` legado: conserva su estructura de iteración por orden y su invariante existente
`total == sum(orders[].subtotal)`, que agrupar por comensal habría roto sin necesidad (`GroupBillResponse`
no tiene desglose por comensal, a diferencia de `SessionBillResponse`).

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: FastAPI + SQLAlchemy (ya en uso); `app.api.v1.orders.checkout`
(`order_sale_lines`, `promo_lines_for`, `TERMINAL`) y `app.api.v1.promotions.service`
(`evaluate`, `combo_discount_for_lines`) — ambos módulos ya existentes y ya usados sin cambios por
`table_sessions.compute_bill`; ninguna dependencia nueva (Constitución, Principio IV — no aplica
justificación porque no se añade nada)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambios de esquema — `group_bill`
sigue sin persistir su resultado, cálculo puramente en tiempo de consulta, FR-004/Assumptions);
SQLite en memoria vía `app/characterization_tests/orders_fixtures.py` para los tests

**Testing**: `unittest` vía `python3 -m unittest` (verificado: no hay `pytest.ini` en el repo,
mismo patrón que el resto de `app/characterization_tests/`) — el test existente
`test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos` en
`test_orders_tables_advanced.py` (líneas 124-160) hoy **CONGELA el comportamiento defectuoso**; esta
spec lo modifica citando en el commit la decisión de `registro-de-anomalias.md` (A-01), tal como
exige el Principio II, y añade el/los test(s) nuevos que exige FR-006

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: corrección puntual dentro de un servicio backend único (no hay frontend
involucrado — `pos-heladeria` (`table.service.ts:133`, `GroupBill` en `table.interface.ts:17-20`)
consume `GET /orders/group/{group_id}/bill` sin cambio de contrato de respuesta, verificado: define
el método `groupBill()` pero no tiene ningún llamador actual en el código, así que no hay pantalla
que dependa hoy del desglose `orders[].subtotal`)

**Performance Goals**: sin objetivo nuevo — incorpora al camino C exactamente las mismas consultas
que ya paga el camino A por orden (`order_sale_lines`, `promotions.evaluate`,
`combo_discount_for_lines`); sin N+1 nuevo más allá del que ya acepta `compute_bill` hoy en
producción

**Constraints**: no se toca `table_sessions.compute_bill` (camino A) ni `orders/checkout.compute_bill`
(camino B, fuera de alcance) — Principio III, un módulo a la vez; no se modifica el contrato de
`GroupBillResponse`/`GroupBillOrderLine` (mismos campos, mismos tipos) — SC-004/Out of Scope de
`spec.md`; no se recalculan cuentas de grupo ya cobradas (FR-004)

**Scale/Scope**: 1 función (`group_bill`, ~22 líneas hoy) en 1 fichero de producción
(`tables_advanced.py`); 1 test existente a modificar con cita de decisión (T046 en
`test_orders_tables_advanced.py`) + los tests nuevos de FR-006; ningún fichero de `pos-heladeria`
ni migración de base de datos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | El cambio de comportamiento está autorizado por escrito en `registro-de-anomalias.md`, entrada A-01, "Tratamiento acordado" (2026-08-15/16), más P1 de `entrevista-negocio.md` (dueño/gerente, 2026-08-16) — citado en `spec.md` §Autorización de negocio. Ningún otro comportamiento cambia por criterio técnico. | PASS |
| **II. Los Characterization Tests son el Árbitro** | El test `test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos` (prefijo `"CONGELA comportamiento actual:"` en el docstring del módulo) congela hoy el defecto — se modifica citando A-01 en el commit, como exige el propio Principio II. FR-006 exige además un test nuevo dedicado que verifique el comportamiento corregido. Ningún otro test `CONGELA` de `pos-backend` se toca. | PASS (modificación autorizada y citada) |
| **III. Estrangulamiento antes que Reescritura** | Un solo módulo en juego: `tables_advanced.group_bill`. `table_sessions.compute_bill` (camino A) y `orders/checkout.compute_bill` (camino B) quedan explícitamente sin tocar (Out of Scope de `spec.md`). No hay otra extracción/reescritura en curso que se solape. | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia — se reutilizan `checkout.order_sale_lines`/`promo_lines_for` y `promotions.evaluate`/`combo_discount_for_lines`, ya presentes y ya usados por el camino A. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | FR-004 lo exige explícitamente y `group_bill` no persiste su resultado (no hay fila que recalcular) — el cambio solo afecta cálculos ejecutados **después** del despliegue. Ninguna factura ni cuenta ya emitida se toca. | PASS |
| **VI. Todo en Español de Colombia** | Esta spec, plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que la spec de origen y el resto de `pos-specs`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/019-correccion-cuenta-mesas-fusionadas/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — prueba de equivalencia por-orden vs. por-comensal
├── data-model.md        # Fase 1 (/speckit-plan)
├── quickstart.md        # Fase 1 (/speckit-plan)
├── contracts/
│   └── group-bill-endpoint.md  # Fase 1 (/speckit-plan) — contrato HTTP (sin cambios de forma)
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en el repositorio sibling
`pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance). Rutas
listadas relativas a la raíz de `pos-backend`.

```text
app/
├── api/v1/orders/
│   ├── tables_advanced.py    # ÚNICO fichero de producción que cambia — group_bill:92-114
│   │                          # reescribe su bucle interno para excluir TERMINAL (FR-001) y
│   │                          # aplicar promotions.evaluate/combo_discount_for_lines por orden
│   │                          # (FR-002); merge_orders, move_order, set_table_status: SIN CAMBIOS
│   ├── checkout.py            # SIN CAMBIOS — solo se importa TERMINAL, order_sale_lines,
│   │                          # promo_lines_for (ya públicos, ya usados por table_sessions/service.py)
│   ├── router.py               # SIN CAMBIOS — mismo endpoint, mismo response_model
│   └── schemas.py               # SIN CAMBIOS — GroupBillResponse/GroupBillOrderLine ya tienen
│                                 # los campos necesarios (subtotal neto ya cabe en el tipo Decimal
│                                 # existente)
│
├── api/v1/promotions/service.py   # SIN CAMBIOS — solo se consume evaluate/combo_discount_for_lines
├── api/v1/table_sessions/service.py  # SIN CAMBIOS — camino A, referencia, out of scope
│
└── characterization_tests/
    ├── test_orders_tables_advanced.py  # Se MODIFICA el test T046 existente (cita A-01 en el
    │                                    # commit, Principio II) para verificar el comportamiento
    │                                    # corregido en vez del defectuoso, y se añaden los casos
    │                                    # de FR-006 (exclusión de pagada/cancelada, aplicación de
    │                                    # promoción, y el caso combinado del ejemplo de A-01)
    └── orders_fixtures.py               # SIN CAMBIOS — make_promotion/make_promotion_target ya
                                          # existentes cubren lo que necesitan los tests nuevos
```

**Structure Decision**: cambio de un solo fichero de producción
(`app/api/v1/orders/tables_advanced.py`), sin paquete ni módulo nuevo — no es una extracción
(Principios III no exige "estrangulamiento" para una corrección puntual de ~15 líneas dentro de un
módulo que ya existe), es una corrección de lógica interna a una función ya extraída (spec 017 ya
caracterizó `tables_advanced.py` como módulo propio). `group_bill` pasa a importar
`app.api.v1.orders.checkout` (para `TERMINAL`, `order_sale_lines`, `promo_lines_for`) y
`app.api.v1.promotions.service` (para `evaluate`, `combo_discount_for_lines`) — las mismas
dependencias que ya declara `table_sessions/service.py:27,29`, verificado sin import circular
(`tables_advanced.py` no es importado por `checkout.py` ni por `promotions/service.py`).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
