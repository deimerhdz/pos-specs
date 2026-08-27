# Implementation Plan: Guardado Unificado de Producto (Crear y Actualizar)

**Branch**: `043-guardado-unificado-producto` | **Date**: 2026-08-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/043-guardado-unificado-producto/spec.md`

## Summary

Se extiende, de forma aditiva, el schema de `POST /products` y `PATCH`/`PUT /products/{id}`
(`ProductCreate`/`ProductUpdate`, `app/api/v1/products/schemas.py`) con un campo opcional
`variants: list[VariantSaveIn]` que trae, por cada presentación, su receta (`recipe`) y sus grupos
de opciones (`option_groups`) — el mismo árbol que `ProductDraft`/`VariantDraft` ya arman hoy en
memoria en `pos-heladeria` (`product.interface.ts:249-292`), pero que hoy se envía en hasta
`1 + 1 + 3·N + M` llamadas HTTP secuenciales orquestadas por `ProductService.saveProduct`/
`saveNewProduct`/`saveExistingProduct` (`product.service.ts:439-536`; el propio código lo
documenta: *"El backend no admite creación anidada... on create the product may remain partially
saved"*). El backend resuelve todo el árbol dentro de una única transacción SQLAlchemy (un solo
`db.commit()` al final; cualquier fallo hace `rollback()` sin persistir nada, FR-004), reutilizando
el patrón de dos pasadas de `display_order` que ya usa `reorder_variants` (spec 042) para no violar
`UNIQUE(product_id, display_order)`, y el patrón de reemplazo total que ya usan
`set_recipe`/`set_variant_option_groups` para receta y grupos. En `pos-heladeria`, `saveProduct`
colapsa a una sola llamada `POST`/`PATCH` que envía `draft` casi tal cual, en vez del bucle
`for` + 3 llamadas por presentación + `reorderVariants` final. Una vez migrado ese formulario, se
retiran los cinco endpoints que ese bucle usaba (`POST /products/{id}/variants`, `PATCH
/variants/{id}`, `PUT /variants/{id}/recipe`, `PUT /variants/{id}/option-groups`, `PATCH
/products/{id}/variants/reorder`), documentado como decisión de negocio en
[`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md). No se modifica
ninguna tabla ni se agrega ninguna migración — solo el contrato de los dos endpoints existentes y
la orquestación del frontend.

## Technical Context

**Language/Version**: Backend — Python 3.14 (`pos-backend`). Frontend — TypeScript 5.9.2 (Angular
21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync), Pydantic 2.13.4, Alembic 1.18.4 (sin
  migración nueva en esta spec). Ninguna dependencia nueva.
- Frontend: Angular 21.1, `@tanstack/angular-query-experimental`, RxJS ~7.8. Ninguna dependencia
  nueva (Principio IX no aplica).

**Storage**: PostgreSQL 16, schema-per-tenant (`schema_translate_map`, `app/core/db.py`). **Sin
migración** — `ProductVariant`, `RecipeItem` y `VariantOptionGroup` ya tienen todas las columnas
que el árbol consolidado necesita (data-model.md).

**Testing**: Backend — `unittest` vía `python -m unittest` (sin `pytest` en el repo, confirmado:
no hay `pytest.ini`/`pyproject.toml` con esa config); characterization tests en
`app/characterization_tests/`. Se extiende `test_products_service.py` con los casos del árbol
anidado y todo-o-nada; `test_product_variant_reorder.py` y `test_catalog_service_sku.py` corren sin
modificar como red de no-regresión (reutilizan lógica que no cambia de forma). Frontend — Vitest vía
`@angular/build:unit-test`; se extiende `product-form.component.spec.ts`.

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`) — usado por
administradores de catálogo desde el formulario de productos.

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de `pos-specs`).

**Performance Goals**: SC-002 — el guardado consolidado completo (crear o actualizar) para un
producto típico (hasta 10 presentaciones) responde en menos de 2 segundos, con una reducción de al
menos 50% frente al tiempo actual medido para el mismo caso (research.md Decisión 6: se logra
agrupando el transporte HTTP existente en una transacción, sin optimización de consultas SQL
adicional).

**Constraints**:
- Todo el guardado (producto + presentaciones + receta + grupos + orden) DEBE ejecutarse en una
  sola transacción de base de datos — cualquier fallo de validación en cualquier parte del árbol
  NO DEBE persistir nada (FR-004, research.md Decisión 2).
- La extensión de `ProductCreate`/`ProductUpdate` DEBE ser aditiva (`variants` opcional) — ningún
  llamador existente que no lo envíe puede cambiar de comportamiento (Principio III, research.md
  Decisión 1).
- El retiro de los cinco endpoints (FR-007) está condicionado a verificar primero que ningún otro
  consumidor además del formulario de productos los usa (quickstart.md Historia 4).

**Scale/Scope**: 0 tablas/columnas/migraciones nuevas. 2 endpoints existentes con schema extendido
(`POST /products`, `PATCH`/`PUT /products/{id}`), 5 endpoints retirados
(`app/api/v1/catalog/router.py`), 1 servicio backend modificado (`ProductService`, reutiliza
funciones ya existentes de `catalog/service.py`). En `pos-heladeria`: 1 servicio modificado
(`ProductService` — colapsa `saveNewProduct`/`saveExistingProduct` a una llamada cada uno, elimina
`createVariant`/`updateVariant`/`deleteVariant`/`reorderVariants`/`setRecipe`/
`setVariantOptionGroups` como llamadas de red), 1 componente modificado (`product-form.component.ts`
— `restoreVariant` deja de ser una llamada de red), 0 componentes nuevos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas y 8 FRs, sin `[NEEDS CLARIFICATION]` pendientes (sesiones `/speckit-specify` y `/speckit-clarify`, 2026-08-27), antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Los cuatro comportamientos que cambian (contrato aditivo de `POST`/`PATCH /products`, todo-o-nada transaccional, retiro de 5 endpoints, reactivación deja de ser instantánea) están documentados explícitamente en `spec.md` ("Naturaleza de esta spec", Clarifications) y registrados como decisión de negocio con quién/cuándo/qué/por qué/afectados en [`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md). | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_products_service.py` (`TestUpdateProductA44`) y `test_catalog_service_sku.py`/`test_product_variant_reorder.py` verifican lógica de negocio que **no cambia de forma** (orden R2/commit, generación de SKU, asignación de `display_order`) — siguen en verde sin modificar como red de no-regresión (quickstart.md). Ningún test `CONGELA` se reescribe sin autorización; los que sí cubren directamente las 5 rutas retiradas se reescriben citando esta spec (A-55), no se eliminan en silencio. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El árbol anidado, el todo-o-nada y el retiro de endpoints son comportamiento nuevo autorizado explícitamente por el spec (Clarifications); las reglas de negocio subyacentes (precio, SKU, nombre, receta, grupos) se preservan sin exigir equivalencia con el paso a paso anterior. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se reutilizan los patrones ya existentes (reemplazo total de `set_recipe`/`set_variant_option_groups`, dos pasadas de `reorder_variants`) en vez de introducir uno nuevo. No se toca la UI de arrastre de spec 042 ni ninguna regla de precio/SKU/nombre — solo la orquestación de red. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas 4 historias del spec (crear consolidado → editar consolidado → atomicidad → retiro de endpoints), cada una verificable por separado (quickstart.md). El retiro de endpoints (Historia 4) es la única unidad que depende de una verificación previa (ningún otro consumidor) antes de ejecutarse. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca ninguna venta, factura, pago ni movimiento de inventario ya generado. | PASS (no aplica directamente) |
| **VIII. Evolución del Modelo de Datos** | Sin migración — data-model.md confirma que las tres entidades persistidas ya tienen todas las columnas necesarias; solo cambia el contrato HTTP de escritura. | PASS (no aplica) |
| **IX. Dependencias Nuevas Permitidas con Justificación** | Ninguna dependencia nueva en ningún repositorio. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest`/Vitest ejecutables, más la suite completa de characterization tests como red de no-regresión antes de ejecutar el retiro de endpoints (Historia 4). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Qué se consolida, que sea todo-o-nada, y que los endpoints viejos se retiren por completo son las decisiones de negocio (spec.md, Clarifications). Cómo modelar el payload anidado (`VariantSaveIn`), dónde poner la transacción, el patrón de dos pasadas para `display_order`, y cómo resolver la reactivación sin llamada de red son las decisiones técnicas correspondientes, documentadas en research.md sin mezclarse con las de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión vía Clarifications) → `registro-de-anomalias.md` A-55 (Decisión de negocio registrada) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica, verificada contra el código real de ambos repos) → `tasks.md` (Fase 2, no generada por este comando) → implementación → characterization tests actualizados sin regresión → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/043-guardado-unificado-producto/
├── plan.md                              # Este fichero (/speckit-plan)
├── research.md                          # Fase 0 — decisiones técnicas y hechos verificados
├── data-model.md                        # Fase 1 — DTOs nuevos, sin migración
├── quickstart.md                        # Fase 1 — validación ejecutable por historia de usuario
├── contracts/                           # Fase 1 — contrato de los dos endpoints extendidos
│   └── product-save-endpoints.md
└── tasks.md                             # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend — sin migración (data-model.md)

app/api/v1/products/
├── schemas.py                          # MODIFICADO — ProductCreate gana `variants:
                                          list[VariantSaveIn] = []`; ProductUpdate gana `variants:
                                          list[VariantSaveIn] | None = None`; nuevo
                                          ProductSaveResponse(ProductResponse) con
                                          `variants: list[VariantSaveOut]` (data-model.md)
└── service.py                          # MODIFICADO — create_product/update_product resuelven el
                                          árbol completo dentro de una sola transacción, reutilizando
                                          ensure_default_variant, variante_duplicada, _unique_sku,
                                          _slug, _next_display_order y el patrón de dos pasadas de
                                          reorder_variants (importados de catalog.service, mismo
                                          cruce que ya existe hoy en la línea 19)

app/api/v1/catalog/
├── schemas.py                          # MODIFICADO — nuevo VariantSaveIn/VariantSaveOut
                                          (data-model.md); RecipeItemIn/VariantOptionGroupIn se
                                          reutilizan sin cambio de forma
└── router.py                           # MODIFICADO (FR-007, tras verificación de consumidores) —
                                          se retiran: POST /products/{id}/variants,
                                          PATCH /variants/{id}, PUT /variants/{id}/recipe,
                                          PUT /variants/{id}/option-groups,
                                          PATCH /products/{id}/variants/reorder.
                                          GET /products/{id}/variants, GET /variants/{id}/recipe,
                                          GET /variants/{id}/option-groups NO se tocan (lectura,
                                          fuera de alcance)

app/characterization_tests/
├── test_products_service.py            # MODIFICADO — nuevos casos: árbol anidado en creación y
                                          edición, todo-o-nada ante fallo en cualquier parte,
                                          back-compat sin `variants`. TestUpdateProductA44 (orden
                                          R2/commit) sin cambios
├── test_product_variant_reorder.py     # SIN CAMBIOS DE FORMA — sigue verificando
                                          reorder_variants/_next_display_order, ahora invocados
                                          desde el guardado consolidado en vez del endpoint retirado
└── test_catalog_service_sku.py         # SIN CAMBIOS DE FORMA — sigue verificando _slug/
                                          _unique_sku/variante_duplicada, reutilizados tal cual

# pos-heladeria

src/app/modules/products/interfaces/
└── product.interface.ts                # MODIFICADO — ProductCreatePayload/ProductUpdatePayload
                                          ganan `variants?: VariantSavePayload[]`; nuevo
                                          VariantSavePayload (id?, name, price, sku?, active?,
                                          recipe, option_groups) mapeado directo desde VariantDraft
                                          (líneas 249-258, ya tiene esa forma)

src/app/modules/products/services/
└── product.service.ts                  # MODIFICADO — saveNewProduct (líneas 469-490) y
                                          saveExistingProduct (492-536) colapsan a una sola llamada
                                          POST/PATCH que envía el draft mapeado; se eliminan como
                                          llamadas de red createVariant/updateVariant/deleteVariant/
                                          reorderVariants/setRecipe/setVariantOptionGroups (research.md
                                          Decisión 1/3); restoreVariant (línea 297-299) deja de
                                          llamar red — pasa a mutar el draft en memoria (research.md
                                          Decisión 4)

src/app/modules/products/pages/
├── product-form.component.ts           # MODIFICADO — restoreVariant (líneas 923-941) deja de ser
                                          `await this.service.restoreVariant(dv.id)` con espera de
                                          red; el resto de la UI (botón "Guardar" + isSubmitting, ya
                                          FR-008) no cambia
└── product-form.component.spec.ts      # MODIFICADO (si existe) — nuevos casos: una sola llamada de
                                          red por guardado; restaurar no dispara red; todo-o-nada
                                          ante error del backend
```

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
