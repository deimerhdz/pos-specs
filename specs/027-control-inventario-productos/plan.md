# Implementation Plan: Control de Inventario por Producto (Switch de Insumos)

**Branch**: `027-control-inventario-productos` | **Date**: 2026-08-19 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/027-control-inventario-productos/spec.md`

## Summary

Hoy todo producto vendible necesita, sí o sí, una receta fija o un grupo de opciones que descuente
inventario, o la venta se rechaza con `409` (`ensure_lines_consume_inventory`,
`app/catalog_engine/consumption.py:156-217`, RN-CAT-34 de spec 003) — no existe forma de crear un
producto legítimamente sin inventario (domicilios, propinas, servicios). Este plan agrega una
columna `tracks_inventory` a `products` (migración con backfill dirigido, research.md Decisión 4) y
un switch en el formulario de producto que la controla. La exención de venta se inyecta en el único
punto del backend que decide qué se descuenta —`plan_line_consumption`— y se repite en el guard que
decide si rechazar la venta, porque son dos caminos de código independientes que ambos necesitan el
mismo chequeo (research.md Decisión 1/2); un parche que solo tocara el guard dejaría productos sin
inventario generando movimientos reales si conservan insumos de una configuración anterior (FR-008).
El default a nivel de modelo ORM se mantiene en `True` — no en `False` como pide el formulario nuevo
— para no romper en silencio la suite completa de characterization tests de spec 003, que construye
`Product(...)` sin saber nada de este campo (research.md Decisión 3). En el frontend, el switch y la
sección de insumos reutilizan patrones ya existentes en el mismo componente (el switch real de
`hasSizes`, el banner ámbar de advertencia, y `ConfirmService` para la confirmación de FR-014) — no
se introduce ningún componente nuevo.

## Technical Context

**Language/Version**: Backend — Python 3.14 (venv `pos-backend/env`). Frontend — TypeScript 5.9.2
(Angular 21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI, SQLAlchemy 2.0 (sync), Pydantic 2, Alembic. Ninguna dependencia nueva — el
  cambio es una columna, una migración y dos funciones ya existentes de `catalog_engine/consumption.py`
  (Principio IX no aplica, nada se añade).
- Frontend: Angular 21 (standalone + signals), Tailwind CSS v4 (sin Angular Material). Ninguna
  dependencia nueva — el switch reutiliza el patrón ya usado por `hasSizes`
  (`product-form.component.ts:145-150`) y la confirmación reutiliza `ConfirmService`
  (`shared/feedback/confirm.service.ts`), ambos ya existentes.

**Storage**: PostgreSQL 16 schema-per-tenant. **Con migración** — nueva columna
`products.tracks_inventory` (`Boolean NOT NULL`, `server_default='true'`), agregada vía
`@for_each_tenant_schema` siguiendo el patrón exacto de `f5a6b7c8d9e0_availability_change_partial_count.py`
(`products.available`), con backfill dirigido por `op.execute(UPDATE ... EXISTS ...)` que replica la
lógica de `load_recipe`/`group_discounts` (data-model.md, research.md Decisión 4). `down_revision`
apunta al head actual, `d2e3f4a5b6c7` (`d2e3f4a5b6c7_active_order_per_participant.py`).

**Testing**: Backend — `unittest` vía `python -m unittest` (sin pytest en el repo). Se extiende
`app/characterization_tests/test_catalog_consumption_plan.py` (CONGELA de
`app/catalog_engine/consumption.py`, clase `EnsureLinesConsumeInventoryTests`) con casos nuevos para
`tracks_inventory=False`, citando la autorización de esta spec (FR-005) — los 4 casos ya existentes
no se modifican (research.md Decisión 5). Frontend — Vitest + `@angular/build:unit-test`; se crea
`product-form.component.spec.ts` (no existe hoy).

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`), usado por
administradores de catálogo en el formulario de producto.

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. `plan_line_consumption` y
`ensure_lines_consume_inventory` ya hacen una consulta por línea por cada chequeo existente
(`load_recipe`, `load_variant_groups`, `variant_label`); agregar una consulta más
(`_tracks_inventory`) es consistente con ese estilo ya existente, no una regresión de performance
nueva (research.md Decisión 1).

**Constraints**:
- El default del campo a nivel de modelo ORM (`Product.tracks_inventory`) DEBE ser `True`, no
  `False` — invertirlo rompe en silencio la suite completa de characterization tests de spec 003
  (research.md Decisión 3). El default `False` que pide FR-001 vive únicamente en
  `ProductCreate.tracks_inventory` (Pydantic).
- La exención de FR-005 DEBE vivir en `plan_line_consumption`, no solo en el guard — es el único
  punto que consumen directamente `deduct_order_item`, `reverse_order_item` (`orders/consumption.py`)
  y `deduct_sale` (`sales/consumption.py`) para el descuento/reversa real (research.md Decisión 1).
- La migración NO reutiliza código de `app/` dentro de su `op.execute` — sigue el precedente del
  proyecto de migraciones autocontenidas en SQL puro (research.md Decisión 4).
- Fuera de alcance explícito de `spec.md`: cambiar el cálculo de consumo cuando el switch está
  activado, configurar el switch por presentación, reportes/filtros por este atributo, y el diseño
  visual concreto del switch (se resuelve reutilizando un patrón ya existente, research.md Decisión
  6, no diseñando uno nuevo).

**Scale/Scope**: 1 tabla modificada (`products`, +1 columna), 1 migración nueva, 0 endpoints
nuevos (3 endpoints existentes ganan un campo: `POST /products`, `PATCH /products/{id}`,
`GET /products*`); 2 funciones de backend modificadas (`plan_line_consumption`,
`ensure_lines_consume_inventory`) + 1 función auxiliar nueva (`_tracks_inventory`), todas en el mismo
archivo; 3 schemas Pydantic modificados (`ProductCreate`, `ProductUpdate`, `ProductResponse`); en
`pos-heladeria`, 4 interfaces TypeScript modificadas, ~1 componente modificado
(`product-form.component.ts`, sin componentes nuevos) y su servicio (`product.service.ts`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 14 FRs y 2 clarificaciones ya resueltas (sesiones `/speckit-specify` y `/speckit-clarify`, 2026-08-19) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento existente que cambia es la exención de `RN-CAT-34`/FR-013 de spec 003, acotada explícitamente a productos con `tracks_inventory=false` (spec, "Naturaleza de esta spec") — decisión de negocio documentada con quién (usuario, vía sesión interactiva), cuándo (2026-08-19), qué cambia (el guard de venta), por qué (permitir productos sin inventario), y qué se ve afectado (`plan_line_consumption`, `ensure_lines_consume_inventory`). Todo producto con `tracks_inventory=true` (el default de migración para quien ya tenía receta configurada, research.md Decisión 4) conserva su comportamiento exacto, verificado por los 4 tests ya existentes de `EnsureLinesConsumeInventoryTests` sin modificar (research.md Decisión 5). | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_catalog_consumption_plan.py` (CONGELA) se extiende con casos nuevos citando esta spec (FR-005) como autorización — los 4 tests ya existentes no se tocan, y su paso sin modificación es la evidencia misma de que Decisión 3 (default `True` a nivel de modelo) no afecta el comportamiento protegido. `golden_master_core.py` se verifica sin modificar (research.md Decisión 5). | PASS (condicionado a verificar en la fase de implementación, no en este plan) |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | La exención de FR-005 es comportamiento nuevo explícito, acotado a `tracks_inventory=false`; el resto (FR-001 a FR-004, FR-006 a FR-014) es un atributo nuevo y su presentación sobre reglas ya existentes de spec 002/003 — no se exige equivalencia con el pasado para el caso nuevo, solo conformidad con `spec.md`. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | El switch, el banner de advertencia y la confirmación reutilizan patrones ya existentes en el mismo componente/proyecto (`hasSizes`, banner ámbar, `ConfirmService`) en vez de extraer componentes nuevos o refactorizar `product-form.component.ts` más allá de lo que el spec pide (research.md Decisiones 6, 8, 9). | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 crear sin inventario → US2 activar y configurar → US3 cambiar el switch sin perder datos → US4 migración), cada una verificable por separado (quickstart.md). No se mezcla con ninguna refactorización del motor de consumo ni cambio de arquitectura — el cambio de backend se acota a un archivo (`catalog_engine/consumption.py`). | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ningún movimiento de inventario ya generado — la migración solo agrega una columna nueva a `products` y no recalcula ni borra ningún `InventoryMovement` histórico. | PASS |
| **VIII. Evolución del Modelo de Datos** | Migración especificada completa en data-model.md/research.md Decisión 4: columna nueva, tipo, nullable, default de ORM y de BD (con la asimetría justificada en Decisión 3), estrategia de backfill (`UPDATE ... EXISTS`, replicando `load_recipe`/`group_discounts`) y estrategia de rollback (`drop_column`, reversible sin manejo especial — perder la columna no destruye otro dato). Se recomienda además una verificación de un solo uso antes de producción (research.md Decisión 4, nota de verificación). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva en ningún repositorio. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest`/Vitest ejecutables, más la suite completa de characterization tests como red de no-regresión (Historia 4). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La existencia del switch, su default apagado, la persistencia de insumos al apagarlo, y el aviso/confirmación de FR-013/FR-014 son las decisiones de negocio (spec, Clarifications). *Cómo* lograrlas sin romper el resto del sistema — dónde inyectar la exención, qué default de ORM usar, cómo migrar datos existentes — son las decisiones técnicas correspondientes, documentadas en research.md sin mezclarse con las de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, incluida la sesión de clarificación) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica) → tareas de `tasks.md` (Fase 2, no generada por este comando) → implementación → characterization tests extendidos + tests nuevos → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/027-control-inventario-productos/
├── plan.md                                    # Este fichero (/speckit-plan)
├── research.md                                 # Fase 0 — decisiones técnicas y alternativas descartadas
├── data-model.md                               # Fase 1 — columna nueva, migración, transiciones
├── quickstart.md                               # Fase 1 — validación ejecutable por historia de usuario
├── contracts/                                  # Fase 1 — contratos HTTP (forma y efecto)
│   ├── product-tracks-inventory-field.md
│   └── sale-consumption-guard.md
└── tasks.md                                    # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
alembic/versions/
└── <rev>_products_tracks_inventory.py   # NUEVO — down_revision='d2e3f4a5b6c7' (head actual);
                                            op.add_column(products.tracks_inventory, server_default
                                            'true') + op.execute(UPDATE ... EXISTS ...) de backfill
                                            dirigido (data-model.md, research.md Decisión 4)

app/models/
└── product.py                            # MODIFICADO — nueva columna tracks_inventory: Mapped[bool]
                                            = mapped_column(Boolean, default=True,
                                            server_default="true"), junto a `available` (línea ~37-39)

app/api/v1/products/
├── schemas.py                            # MODIFICADO — ProductCreate.tracks_inventory: bool = False
                                            (líneas 14-23); ProductUpdate.tracks_inventory: bool | None
                                            = None (líneas 26-33); ProductResponse.tracks_inventory:
                                            bool (líneas 36-48)
└── service.py                            # MODIFICADO — create_product (líneas 41-65) pasa
                                            tracks_inventory=data.tracks_inventory al constructor;
                                            update_product (líneas 67-92) agrega el `if data.
                                            tracks_inventory is not None: product.tracks_inventory = ...`
                                            junto a los de `active`/`available`

app/catalog_engine/
└── consumption.py                        # MODIFICADO — nueva función _tracks_inventory(db,
                                            variant_id) -> bool junto a variant_label (línea ~146);
                                            plan_line_consumption (líneas 89-129) la consulta primero y
                                            devuelve [] si es False; ensure_lines_consume_inventory
                                            (líneas 156-217) la consulta al inicio de cada iteración del
                                            bucle y hace `continue` si es False, antes de
                                            required_consumption y de la clasificación sin_receta/
                                            sin_eleccion (research.md Decisión 1/2)

app/characterization_tests/
└── test_catalog_consumption_plan.py      # MODIFICADO (no se tocan los 4 tests existentes) — se
                                            agregan casos nuevos a EnsureLinesConsumeInventoryTests
                                            (línea ~234+) para tracks_inventory=False: sin insumos (no
                                            bloquea), con insumos guardados de antes (tampoco bloquea ni
                                            genera ConsumptionLine) — cita spec 027 FR-005 como
                                            autorización

# pos-heladeria
src/app/modules/products/interfaces/
└── product.interface.ts                  # MODIFICADO — tracks_inventory: boolean agregado a
                                            Product (líneas 15-27), ProductCreatePayload (39-45),
                                            ProductUpdatePayload (48-56) y ProductDraft (268-281)

src/app/modules/products/services/
└── product.service.ts                    # MODIFICADO — toProductPayload (líneas 519-527) y
                                            toProduct (644-657) mapean el campo nuevo; sin método
                                            toggle dedicado en el servicio (el toggle vive en el
                                            componente, ver abajo, porque requiere la confirmación de
                                            FR-014 antes de llamar updateProduct)

src/app/modules/products/pages/
├── product-form.component.ts             # MODIFICADO — switch nuevo (copia del patrón de
                                            toggleHasSizes(), línea 734) con toggleTracksInventory()
                                            que llama ConfirmService.ask(...) solo al apagar con
                                            insumos configurados (research.md Decisión 6/9); bloques
                                            "Insumos fijos" (218-242) y "Sabores a elegir" (244-350)
                                            envueltos en @if (draft().tracks_inventory) (Decisión 7);
                                            banner ámbar nuevo de advertencia al guardar sin insumos
                                            con switch activado (Decisión 8, FR-013)
└── product-form.component.spec.ts        # NUEVO — no existe hoy; cubre los 4 escenarios de
                                            quickstart.md (crear sin inventario, activar sin insumos
                                            muestra aviso, apagar con insumos pide confirmación,
                                            reactivar no pide confirmación y conserva insumos)
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de los
ficheros de arriba (US1 → `schemas.py`+`service.py`+`consumption.py` (rama `tracks_inventory=false`
sin insumos) + `product-form.component.ts` (switch apagado); US2 → `consumption.py` (rama
`tracks_inventory=true`, sin cambios) + banner de advertencia; US3 → `consumption.py` (el caso más
delicado: insumos guardados con switch apagado, research.md Decisión 2) + confirmación de
`product-form.component.ts`; US4 → la migración), consistente con Principio VI. No se crea ningún
componente, servicio ni módulo nuevo en el frontend — toda la funcionalidad vive en el mismo
componente que ya implementa la sección de insumos, y reutiliza servicios ya existentes
(`ConfirmService`). El único archivo de backend con lógica de negocio nueva es
`app/catalog_engine/consumption.py` — el resto son campos nuevos propagados por schemas/modelo/
servicio ya existentes.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
