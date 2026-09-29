# Implementation Plan: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Branch**: `087-fix-caja-mesas-menu-ventas` | **Date**: 2026-09-28 (actualizado 2026-09-29: US8–US11) | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/087-fix-caja-mesas-menu-ventas/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Corrige 7 defectos/carencias de usabilidad y de integridad de datos en `pos-backend` +
`pos-heladeria`, repartidos en 4 módulos: (1) el cálculo de precio se rompe cuando un
producto en promoción lleva adicionales (fix centralizado, sin recalcular ventas históricas
por el Principio VII); (2) "Agregar producto" sobre un pedido abierto crea duplicados en vez
de anexar al mismo `order_id`; (3) el número de pedido de mesa se recalcula por posición de
arreglo en el frontend en vez de asignarse una sola vez en el backend (se persiste, se
reinicia por turno de caja); (4) se elimina por completo "Arqueo Parcial" (UI + endpoint +
tabla, con borrado físico del historial); (5) el cierre de turno imprime hoy sidebar/header
junto al resumen y sin nombre de archivo controlado; (6) el nombre de cliente pasa de opcional
a obligatorio en la creación de pedidos, mostrado de forma destacada; (7) se exponen
presentación/adicionales/notas de forma consistente en todas las superficies de pedido, y el
campo Cambio/Devuelta (ya existente en el modelo `Sale`) se muestra en el detalle de venta
cuando el pago incluyó efectivo.

Enfoque técnico: reutilizar patrones ya existentes en el propio código (cálculo de precio
centralizado en `catalog_engine`, impresión aislada vía `@media print` ya usada en
`table-qr-sheet.component.ts`) en vez de introducir dependencias nuevas; dos columnas nuevas
en `customer_orders` (`cash_shift_id`, `table_order_number`) para persistir la numeración
estable; un nuevo endpoint `POST /orders/{order_id}/items` para eliminar la ambigüedad de a
qué pedido anexar productos una vez que se permiten pedidos paralelos por mesa.

### Actualización 2026-09-29 — US8 a US11 (FR-014 a FR-017)

Tras implementar US1–US7 se reportaron 4 defectos más, agregados a esta misma spec
(clarificación 2026-09-29) y planificados de forma incremental (`research.md` D11–D14):

- **US8 (P2)** — en el modal del producto del Menú QR el nombre de la presentación se trunca
  hasta "…" y solo se lee la promoción: se corrige la plantilla de la fila (frontend).
- **US9 (P1)** — los adicionales no aparecen bien en el panel del pedido de la terminal:
  los ítems guardados dependen del menú vigente para resolver el nombre y los borradores usan
  otro formato. Se añaden `name`/`group_name` derivados a `OrderItemOptionResponse`
  (aditivo, sin migración) y el frontend los usa de respaldo.
- **US10 (P1)** — "TOTAL ORDEN" se desactualiza al editar un pedido manual: el desglose se
  calcula solo sobre borradores y no reacciona a los ítems guardados. Se calcula sobre el
  conjunto vigente completo, con guarda de respuesta obsoleta (frontend; sin cambio de
  contrato).
- **US11 (P2)** — nota por producto en 16px/semibold/alto contraste con un único estilo en
  el componente compartido `app-cart-item-options`.

Solo US9 toca el backend (un campo de respuesta); el resto es frontend. Ninguna migración,
ninguna venta emitida se recalcula (Principio VII).

## Technical Context

**Language/Version**: Backend: Python 3.12 (FastAPI, SQLAlchemy, Alembic). Frontend: TypeScript 5.9 / Angular 21.1.

**Primary Dependencies**: Backend: FastAPI, SQLAlchemy, Alembic, PostgreSQL driver (schema-per-tenant vía `app/scripts/tenant.for_each_tenant_schema`). Frontend: Angular (standalone components, signals — confirmado por `pos-terminal.store.ts`), sin librerías nuevas de PDF/canvas (ver `research.md` D1). No se introduce ninguna dependencia nueva en esta spec.

**Storage**: PostgreSQL 16, schema-per-tenant (`schema="tenant"` en cada modelo, una migración Alembic aplicada a todos los schemas vía decorador `@for_each_tenant_schema`).

**Testing**: Backend: `pytest` (168+ archivos `test_*.py`, incluye characterization tests en `app/characterization_tests/` protegidos por el Principio III de la constitución). Frontend: test runner nativo de Angular 21 (`@angular/build:unit-test`, `ng test`/`npm test`).

**Target Platform**: Backend: contenedor Docker (Linux), API HTTP consumida por el frontend. Frontend: SPA Angular servida en navegador (Chromium-based en producción, relevante para D1 de `research.md`).

**Project Type**: Web application — dos repositorios independientes: `../pos-backend` (API) y `../pos-heladeria` (frontend Angular), según el Alcance del Proyecto de la constitución.

**Performance Goals**: No definidos explícitamente por la spec — sin requisitos de rendimiento nuevos; se preserva el comportamiento de latencia actual de los endpoints tocados (no se introduce ningún cálculo pesado nuevo, solo se corrige lógica existente y se añaden 2 columnas indexadas).

**Constraints**: Multi-tenant schema-per-tenant (toda migración debe usar `@for_each_tenant_schema`); inmutabilidad de facturas/ventas ya emitidas (Principio VII — ver D6 en `research.md`, gate crítico); characterization tests protegidos (Principio III) no se tocan salvo autorización explícita del spec.

**Scale/Scope**: 11 user stories (P1×4, P2×5, P3×2) sobre 4 módulos (Caja, Terminal de Mesas, Menú QR, Ventas) en 2 repositorios. 2 columnas nuevas de base de datos, 1 tabla eliminada, 1 endpoint nuevo, 1 endpoint eliminado, 2 campos derivados nuevos en una respuesta, ~13 componentes/servicios de frontend afectados (ver `research.md` para el listado completo por decisión). US8–US11 (adenda): sin migraciones, sin dependencias nuevas.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluación contra los 15 principios de `.specify/memory/constitution.md` (v4.0.0):

| Principio | Estado | Nota |
|---|---|---|
| I. Nace de un spec | ✅ PASS | `specs/087-fix-caja-mesas-menu-ventas/spec.md` existe, con FRs, criterios de aceptación y clarificaciones resueltas. |
| II. Comportamiento existente protegido | ⚠️ GATE — acción requerida | Esta spec cambia comportamiento existente en al menos 4 puntos: eliminación de Arqueo Parcial (D2), numeración de pedidos reiniciada por turno (D3), contrato de "Agregar producto" ahora por `order_id` (D4), nombre de cliente ahora obligatorio (D7). **Cada uno requiere una entrada nueva en `specs/000-reconocimiento/registro-de-anomalias.md`** (formato `A-NN — [DECISIÓN DE NEGOCIO — spec 087] ...`) antes de dar la implementación por completa. Ver "Acciones pendientes" abajo. |
| III. Characterization tests protegidos | ✅ PASS (condicional) | Ningún characterization test se modifica por defecto. Si el fix de FR-011 (D6) toca código cubierto por un test `"CONGELA comportamiento actual:"` que capturó el cálculo *incorrecto*, ese test debe actualizarse explícitamente citando esta spec como autorización — a verificar en implementación. |
| IV. Nuevos specs pueden introducir comportamiento nuevo | ✅ PASS | Aplica directamente (numeración por turno, pedidos paralelos, campo obligatorio son comportamiento nuevo autorizado por este spec). |
| V. Sin refactors oportunistas | ✅ PASS (a vigilar) | `research.md` D10 marca explícitamente evaluar si vale la pena un helper compartido para "Nombre xN" solo si no excede lo que la funcionalidad exige — no extraer abstracciones más allá de los 4 puntos identificados. |
| VI. Evolución incremental | ✅ PASS (guía de secuencia) | 7 historias con prioridad P1→P3 ya declarada en la spec; se implementan en unidades separadas por historia (no en un solo cambio masivo), consistente con la convención de commits pequeños del Principio XV. |
| VII. Compatibilidad con datos históricos | 🔴 GATE CRÍTICO | El Acceptance Scenario 3 de la Historia 1 ("una venta ya finalizada... coincide con...") **no puede leerse como recálculo retroactivo de `Sale.total`** de ventas ya emitidas — ver D6 en `research.md`. El fix de FR-011 aplica solo a checkouts nuevos tras el despliegue. **Debe confirmarse con el usuario/negocio antes de implementar** si esta lectura no es la esperada. |
| VIII. Evolución del modelo de datos | ✅ PASS | `data-model.md` documenta entidades/campos nuevos, nullability, compatibilidad, migración y rollback para `customer_orders` (additive) y `cash_partial_counts` (destructiva e intencional, con nota explícita de irreversibilidad). |
| IX. Dependencias nuevas justificadas | ✅ PASS | Ninguna dependencia nueva se introduce (ver Technical Context); `research.md` D1 documenta explícitamente por qué se descarta jsPDF/html2canvas/WeasyPrint a favor de un patrón ya existente en el código. |
| X. Verificación obligatoria | ⏳ Pendiente de `tasks.md` | `quickstart.md` define escenarios de validación por historia; los tests unitarios/integración/characterization específicos se definirán en la fase de tasks. |
| XI. Decisiones de negocio vs técnicas | ✅ PASS | La ambigüedad de "a qué turno se ata la numeración" (múltiples cajas posibles) se escaló al usuario en vez de asumirse técnicamente — resuelta y documentada al inicio de `research.md`. |
| XII. Trazabilidad | ✅ PASS | Cadena Necesidad (spec) → Decisión (research.md, incluida la confirmada con el usuario) → Datos (data-model.md) → Contratos (contracts/) → Validación (quickstart.md) completa en esta fase. |
| XIII. Español de Colombia | ✅ PASS | Todos los artefactos de esta fase (spec, research, data-model, contracts, quickstart) en español; nombres de rama/commits en inglés cuando corresponda (Principio XIV/XV, fuera del alcance de `/speckit-plan`). |
| XIV. Ramas Git | ⏳ Acción en implementación | Antes de tocar código en `pos-backend`/`pos-heladeria`, crear rama `fix/087-cash-tables-menu-sales-fixes` (o una por sub-alcance si se decide dividir) — no aplica todavía a esta fase de solo documentación. |
| XV. Commits | N/A en esta fase | No se han hecho commits de código; aplica en implementación. |

### Re-evaluación para la adenda US8–US11 (2026-09-29)

| Principio | Estado | Nota |
|---|---|---|
| I. Nace de un spec | ✅ PASS | US8–US11 y FR-014–FR-017 están en `spec.md` con clarificación del 2026-09-29. |
| II. Comportamiento existente protegido | ⚠️ GATE — acción requerida | US10 cambia el total mostrado al editar un pedido manual (hoy solo borradores, sin promociones sobre ítems guardados) y US9 cambia qué se muestra en el panel. Registrar en `registro-de-anomalias.md` (siguiente número libre tras A-88, formato `[DECISIÓN DE NEGOCIO — spec 087]`) una entrada por US10 y una por US9; US8 y US11 son solo estilo, se anotan en una sola entrada agrupada o se justifican como no-comportamiento. |
| III. Characterization tests | ✅ PASS (a verificar) | Ninguno se modifica por defecto. `cart-item-options.component.spec.ts` no es characterization, pero se actualiza por FR-017; verificar en implementación que ningún `"CONGELA comportamiento actual:"` toque `OrderItemOptionResponse` ni el total del borrador. |
| V. Sin refactors oportunistas | ✅ PASS (a vigilar) | La única consolidación permitida es colapsar el bloque duplicado de la nota en `cart-item-options.component.ts`, exigida por FR-017 ("un único estilo"); no se toca nada más del componente ni del store (D14). |
| VII. Datos históricos | ✅ PASS | Nada persiste `total`; ninguna venta emitida se recalcula (`data-model.md` §6). |
| VIII. Modelo de datos | ✅ PASS | Sin cambios de esquema; campos derivados por JOIN en lectura (`data-model.md` §6). |
| IX. Dependencias | ✅ PASS | Ninguna nueva. |
| X. Verificación | ⏳ Pendiente de `tasks.md` | `quickstart.md` cubre Historias 8–11; cada causa raíz marcada **[hipótesis]** (D12) arranca con un test que la confirme. |
| XIV/XV. Ramas y commits | ⏳ En implementación | Continuar en `fix/087-cash-tables-menu-sales-fixes` (ya existe en ambos repos); commits pequeños por historia. |

Sin violaciones nuevas: no se requiere entrada en Complexity Tracking.

### Acciones pendientes antes/durante implementación (no bloquean el diseño de esta fase)

1. ~~Confirmar con el usuario la lectura de Principio VII sobre el Acceptance Scenario 3 de la
   Historia 1 (D6, gate crítico arriba).~~ Resuelto con el usuario antes de implementar US1: el fix
   aplica solo a checkouts nuevos (D6, `research.md`).
2. Redactar las entradas correspondientes en `registro-de-anomalias.md` para los 4 cambios de
   comportamiento identificados (Principio II) — hechas A-84 a A-88; **faltan** las de
   US9/US10 de la adenda (tabla anterior).
3. Verificar durante la implementación si el fix de FR-011 toca algún characterization test
   existente (Principio III).
4. Reproducir con un test la causa raíz de US9 (D12, hipótesis) antes de corregir.
5. Confirmar en US10 si el total mostrado hoy suma los combos del borrador (D13).

## Project Structure

### Documentation (this feature)

```text
specs/087-fix-caja-mesas-menu-ventas/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/
│   └── api-changes.md   # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

Repositorios reales, ubicados como hermanos de `pos-specs` (`../pos-backend`,
`../pos-heladeria`), según el Alcance del Proyecto de la constitución. Se usa la Opción 2
(web application) con rutas concretas de los archivos identificados en `research.md`:

```text
../pos-backend/
├── app/
│   ├── models/
│   │   ├── customer_order.py        # + cash_shift_id, table_order_number (data-model.md §1)
│   │   ├── cash_partial_count.py    # eliminado (data-model.md §2)
│   │   ├── cash_shift.py            # sin cambios, consultado para "turno abierto vigente"
│   │   └── sale.py                  # sin cambios de esquema (paid_amount/change_given ya existen)
│   ├── api/v1/
│   │   ├── cash/
│   │   │   ├── router.py            # retira POST .../partial-count
│   │   │   └── service.py
│   │   ├── orders/
│   │   │   ├── router.py            # + POST /orders/{order_id}/items (contracts/api-changes.md §4)
│   │   │   ├── service.py           # create_order: customer_name obligatorio, asigna table_order_number
│   │   │   ├── consolidation.py     # add_item_to_table → ajustar a pedido específico
│   │   │   └── checkout.py          # compute_checkout_preview/pay_order/etc — fix FR-011 (research.md D6)
│   │   ├── promotions/service.py    # evaluate_variant_sets — candidato a fix FR-011
│   │   ├── orders/schemas.py        # + name/group_name en OrderItemOptionResponse (US9, contracts §7)
│   │   └── sales/
│   │       ├── router.py            # GET /sales/{sale_id} — verificar SaleResponse
│   │       └── builder.py           # SaleItem.description — verificar presentación (research.md D8)
│   └── catalog_engine/core.py       # compute_line_price — candidato a fix FR-011
├── alembic/versions/
│   ├── <rev>_customer_orders_table_order_number.py   # nueva, additive (data-model.md §1)
│   └── <rev>_drop_cash_partial_counts.py              # nueva, destructiva intencional (data-model.md §2)
└── app/characterization_tests/                        # verificar impacto (Constitution Check, Principio III)

../pos-heladeria/
├── src/app/modules/
│   ├── cash-register/
│   │   ├── components/
│   │   │   ├── cash-dashboard.component.ts    # retira modal/estado de Arqueo Parcial
│   │   │   └── cash-report.component.ts       # + @media print, + document.title dinámico
│   │   ├── services/
│   │   │   ├── cash.service.ts                # retira partialCount()
│   │   │   └── cash-session.store.ts           # imprimirReporte()
│   │   └── interfaces/cash.interface.ts        # retira PartialCountResponse
│   ├── tables/
│   │   ├── components/
│   │   │   ├── pos-order-panel.component.ts    # botón "+ Agregar producto" → usa order_id
│   │   │   ├── product-select.component.ts     # lineTotal()/packageTotal() — fix FR-011; + fila de presentación sin truncar el nombre (US8, D11)
│   │   │   └── cart-item-options.component.ts  # + estilo único de nota 16px/semibold/alto contraste (US11, D14)
│   │   ├── pages/
│   │   │   └── manual-order-page.component.ts  # customer_name requerido (3 tabs); + effect del desglose depende de ítems guardados (US10, D13)
│   │   └── services/
│   │       ├── pos-terminal.store.ts           # orderTabs() → table_order_number; + opciones con respaldo de name/group_name y formatQuantifiedLabel en borradores (US9, D12); + draftPreviewPayload con ítems guardados y guarda de respuesta obsoleta (US10, D13)
│   │       └── menu-lookup.ts                  # sin cambios de firma; sigue siendo la fuente primaria de nombres de opción
│   ├── orders/pages/order-detail.component.ts  # presentación/adicionales en historial
│   └── sales/pages/sales-page.component.ts     # + línea "Cambio: $X" en modal de detalle
```

**Structure Decision**: Web application de dos repositorios ya existentes y en producción
(`pos-backend` FastAPI, `pos-heladeria` Angular), sin crear proyectos ni módulos nuevos — todo
el trabajo modifica archivos ya identificados dentro de la estructura actual. La
implementación se organiza en unidades por historia de usuario (P1 → P3, Principio VI),
respetando la separación entre las 4 áreas funcionales (Caja, Terminal de Mesas, Menú QR,
Ventas) para poder revisar y revertir cada una de forma independiente.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Ninguna violación de la constitución requiere justificación en esta tabla: las dos
banderas levantadas en el Constitution Check (Principio II — entradas pendientes en
`registro-de-anomalias.md`; Principio VII — alcance del fix de FR-011 frente a ventas
históricas) son **acciones pendientes de gobernanza**, no violaciones de un principio ni
excepciones a justificar — se resuelven cumpliendo el principio, no saltándoselo.
