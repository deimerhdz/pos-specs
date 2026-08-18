# Implementation Plan: Corrección del orden de borrado de imagen en R2 (A-44)

**Branch**: `021-correccion-orden-borrado-imagen-r2` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/021-correccion-orden-borrado-imagen-r2/spec.md`

## Summary

`update_product` (`app/api/v1/products/service.py:67-91`) hoy llama `delete_object(old_key)`
(línea 84) **antes** de `db.commit()` (línea 89) al reemplazar la imagen de un producto. Si el
commit falla después por cualquier razón no relacionada con la imagen, el `rollback()` implícito
revierte `product.image_url` a la URL vieja, pero el objeto que esa URL señala ya fue borrado en R2
un paso antes — el producto queda con una referencia de imagen rota, sin ningún registro (A-44,
`registro-de-anomalias.md`, `registro-riesgos.md` R23).

Esta spec invierte el orden: mueve la llamada a `delete_object(old_key)` a después de
`db.commit()` (FR-001), preservando exactamente el resultado del camino feliz (FR-003) y el
carácter best-effort del borrado (FR-004). No toca `create_product`, `soft_delete`,
`key_from_public_url` ni ningún otro caller de `app/core/storage.py`. Este orden ya estaba
documentado como "tratamiento acordado" en la propia spec 002 (`FR-012`, User Story 7) — spec 002
lo dejó explícitamente sin aplicar ("documentar el orden actual tal cual, sin implementar la
corrección acordada"); esta es esa corrección.

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: FastAPI + SQLAlchemy (ya en uso); `boto3` vía `app/core/storage.py`
(`delete_object`, `key_from_public_url`, ya existentes) — ningún import nuevo (Constitución,
Principio IV: no aplica justificación porque no se añade nada)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambio de esquema — el fix es de
orden de operaciones en tiempo de escritura, no de datos); Cloudflare R2 (API S3-compatible vía
`boto3`) para el objeto de imagen; SQLite en memoria vía
`app/characterization_tests/fixtures.py` para los tests de base de datos, con `delete_object`
mockeado (`unittest.mock.patch`) para no requerir R2 real en los tests — mismo patrón ya usado en
`cart_fixtures.py`/`table_sessions_fixtures.py` para mockear dependencias externas

**Testing**: `unittest` vía `python -m unittest` (mismo patrón que el resto de
`app/characterization_tests/`, sin `pytest.ini` en el repo). No existe hoy ningún
`test_products_*.py` en `app/characterization_tests/` — esta spec crea el primero
(`test_products_service.py`), acotado a `update_product` y la anomalía A-44, sin caracterizar el
resto del servicio de productos (fuera de alcance de esta delta)

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: corrección puntual dentro de un servicio backend único (no hay frontend
involucrado — el panel de administración ya consume `PATCH /products/{id}` / `PUT /products/{id}`;
el único cambio observable es *cuándo* se borra la imagen vieja en R2, nunca el contrato de
request/response)

**Performance Goals**: sin objetivo nuevo — la corrección reordena una llamada ya existente
(`delete_object`), sin agregar ninguna consulta ni operación de red adicional

**Constraints**: no se toca `create_product` (`service.py:41-56`, no maneja reemplazo de imagen) ni
`soft_delete` (`service.py:93-96`) — Principio III, un módulo a la vez; no se toca
`app/core/storage.py` (`delete_object`, `key_from_public_url`) en sí — su contrato (best-effort,
`None` si la URL no pertenece al bucket) queda intacto (FR-004); no se recalcula ni repara ningún
`Product` cuya imagen ya haya quedado con referencia rota antes de esta corrección (FR-005)

**Scale/Scope**: reordenar 2 líneas dentro de 1 función de 1 fichero de producción
(`app/api/v1/products/service.py:78-89`, `update_product`); 1 fichero de test nuevo
(`test_products_service.py`) con los casos de FR-001 a FR-006; ningún fichero de `pos-heladeria`
ni migración de base de datos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | El cambio de comportamiento está autorizado por escrito en `registro-de-anomalias.md`, entrada A-44 (ACCIDENTAL, `registro-riesgos.md` R23, "tratamiento acordado: corregir en fase de modernización"), y ya estaba anticipado como corrección pendiente en la propia spec 002 (`FR-012`). Citado en `spec.md` §Autorización de negocio. Ningún otro comportamiento cambia por criterio técnico. | PASS |
| **II. Los Characterization Tests son el Árbitro** | No existe hoy ningún test `"CONGELA comportamiento actual:"` sobre `update_product` que congele el orden defectuoso (no hay `test_products_*.py` en `app/characterization_tests/`) — no hay test que modificar citando la decisión; se crea `test_products_service.py` directamente con el comportamiento corregido, más un caso que demuestra el contraste antes/después (FR-006). Ningún test `CONGELA` existente de otro módulo se toca. | PASS |
| **III. Estrangulamiento antes que Reescritura** | Un solo módulo en juego: `products.service.update_product`, dos líneas reordenadas dentro de una función ya existente. `app/core/storage.py`, `create_product` y `soft_delete` quedan explícitamente sin tocar (Out of Scope de `spec.md`). No hay otra extracción/reescritura en curso que se solape. | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia ni ningún import — `delete_object`/`key_from_public_url` ya están importados en `service.py:16` y en el ámbito de `update_product`. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | FR-005 lo exige explícitamente; `update_product` no tiene mecanismo de recálculo retroactivo — el cambio solo afecta actualizaciones de imagen ejecutadas **después** del despliegue. Ningún `Product` ya actualizado se toca, incluyendo los que ya quedaron con referencia rota por el orden antiguo. | PASS |
| **VI. Todo en Español de Colombia** | Esta spec, este plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que la spec de origen (002) y el resto de `pos-specs`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/021-correccion-orden-borrado-imagen-r2/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — por qué reordenar basta, sin cola async
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md          # Fase 1 (/speckit-plan)
├── contracts/
│   └── update-product-endpoint.md  # Fase 1 (/speckit-plan) — contrato HTTP (sin cambio de forma)
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en el repositorio sibling
`pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance). Rutas
listadas relativas a la raíz de `pos-backend`.

```text
app/
├── api/v1/products/
│   ├── service.py          # ÚNICO fichero de producción que cambia — update_product:78-89
│   │                          # reordena delete_object(old_key) para que ocurra después de
│   │                          # db.commit() (FR-001/FR-002); el resto de la función (demás
│   │                          # campos, create_product, soft_delete) SIN CAMBIOS
│   ├── router.py             # SIN CAMBIOS — mismo endpoint (PATCH y PUT /products/{id}), mismo
│   │                           # response_model; ningún cambio de contrato observable
│   └── schemas.py             # SIN CAMBIOS — ProductUpdate/ProductResponse sin cambios de forma
│
├── core/storage.py             # SIN CAMBIOS — delete_object/key_from_public_url conservan su
│                                  # contrato best-effort (FR-004); esta delta solo cambia cuándo
│                                  # update_product llama a delete_object, no su implementación
│
└── characterization_tests/
    ├── test_products_service.py   # NUEVO — primer characterization test de update_product;
    │                                # cubre FR-001 a FR-006, citando A-44
    └── fixtures.py                  # SIN CAMBIOS — ya crea Product/Category vía SQLite en
                                       # memoria (`fixtures.py`); el test nuevo mockea
                                       # `app.api.v1.products.service.delete_object` con
                                       # `unittest.mock.patch`, mismo patrón que
                                       # `cart_fixtures.py`/`table_sessions_fixtures.py`
```

**Structure Decision**: reordenar dos líneas dentro de una función de producción ya existente
(`app/api/v1/products/service.py:78-89`), sin paquete ni módulo nuevo — no es una extracción
(Principio III no exige "estrangulamiento" para invertir el orden de dos llamadas dentro de un
módulo ya caracterizado por la spec 002). El único artefacto nuevo es el fichero de test
`test_products_service.py`, porque hoy no existe ninguna caracterización de `products.service` —
verificado listando `app/characterization_tests/` (no aparece ningún `test_products_*.py`).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
