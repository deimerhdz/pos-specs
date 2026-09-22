# Implementation Plan: Exportación de Inventario a Excel

**Branch**: `086-exportacion-excel-inventario` | **Date**: 2026-09-22 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/086-exportacion-excel-inventario/spec.md`

## Summary

Agregar un botón "Exportar Inventario" en la pantalla de Inventario (`pos-heladeria`) que,
al presionarse, descarga automáticamente un archivo `.xlsx` con el respaldo completo del
inventario del tenant — todos los insumos, activos e inactivos, sin importar filtros,
búsqueda u orden aplicados en pantalla — con las columnas `Nombre, Tipo, Unidad, Stock,
Mínimo, Costo, Estado` en ese orden, valores numéricos nativos (no texto) en las columnas
Stock/Mínimo/Costo, y codificación correcta de caracteres del español.

Enfoque técnico: un endpoint nuevo en `pos-backend`
(`GET /api/v1/inventory/items/export`, protegido con el mismo rol `ADMIN` que hoy guarda el
acceso a la pantalla de Inventario) consulta *todos* los insumos del tenant sin paginar,
construye el workbook `.xlsx` completo en memoria con `openpyxl` (dependencia nueva,
justificada en `research.md`) y lo retorna con `Content-Disposition: attachment`. En
`pos-heladeria`, `InventoryService` gana un método `exportItems()` que pide ese endpoint
con `HttpClient` (`responseType: 'blob'`, reutilizando el interceptor de autenticación
existente), y `inventory-page.component.ts` dispara la descarga vía blob URL + `<a
download>` sintético, con retroalimentación de error mediante el `ToastService` ya usado en
ese mismo componente. Ningún dato ni comportamiento existente se modifica: el endpoint y el
botón son aditivos; la columna "Estado" exportada replica exactamente la fórmula que la
pantalla ya calcula hoy (`activo AND stock_actual <= stock_mínimo`), sin introducir un
tercer estado nuevo (ver `research.md` §4 para el detalle de por qué esto es una decisión
deliberada, no una omisión).

## Technical Context

**Language/Version**:
- Backend (`pos-backend`): Python 3.12 (`Dockerfile: FROM python:3.12-slim`), FastAPI
  `0.136.3`, SQLAlchemy `2.0.50`, Pydantic `2.13.4`.
- Frontend (`pos-heladeria`): TypeScript `~5.9.2` sobre Angular `^21.1.0` (standalone
  components).

**Primary Dependencies**:
- Backend: `openpyxl` — **dependencia nueva**, justificada en `research.md` §1 (Principio
  IX). El resto reutiliza infraestructura ya existente: `fastapi.Response` (patrón nuevo de
  uso, sin dependencia nueva, `research.md` §2), `Depends(require_tenant_admin)` y
  `Depends(require_module_access("inventario"))` ya existentes
  (`app/core/dependencies.py`, `app/core/plan_limits.py`).
- Frontend: `HttpClient` de Angular (`responseType: 'blob'`, uso nuevo de una API ya
  existente — sin dependencia nueva), `ToastService` ya existente
  (`shared/feedback/toast.service.ts`). No se introduce ninguna librería nueva en el
  frontend.

**Storage**: PostgreSQL 16, schema-per-tenant (sin cambios de esquema — solo lectura de
`InventoryItem` y `UnitMeasure`, ya existentes; ver `data-model.md`). El archivo `.xlsx`
generado no se persiste en ningún lado (se entrega directo como descarga, spec Key
Entities).

**Testing**:
- Backend: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  (no usa `pytest`; suite basada en `unittest` de la librería estándar sobre SQLite en
  memoria, ver `pos-backend/README.md:189-211`). El endpoint nuevo requiere tests propios
  de la funcionalidad (no hay comportamiento previo que "congelar" porque el endpoint no
  existe hoy — Principio III no aplica a código nuevo).
- Frontend: `ng test` (`@angular/build:unit-test` + Vitest `^4.0.8` + `jsdom`, no
  Karma/Jasmine). Suite existente relevante: `inventory-page.component.spec.ts`
  (`pos-heladeria/src/app/modules/inventory/pages/`).

**Target Platform**: Web — API HTTP (Linux server, contenedor Docker) consumida por SPA
Angular en navegador de escritorio (mismo target que el resto del módulo de Inventario).

**Project Type**: Aplicación web — cambio en ambos repos (`pos-backend` + `pos-heladeria`).

**Performance Goals**: N/A explícito — el spec asume (Assumptions) que "el volumen actual
de insumos del negocio no representa un riesgo de rendimiento relevante para una
generación síncrona del archivo en el momento del clic". No se define un SLA de latencia;
la generación es síncrona y bloqueante por diseño (research.md §2).

**Constraints**:
- La exportación siempre trae el total absoluto de insumos (activos + inactivos), nunca
  filtrado por lo que el usuario tenga aplicado en pantalla (FR-003, Clarification 1).
- Ninguna columna nueva que distinga activo/inactivo — se mantienen exactamente las 7
  columnas de FR-004 (Clarification 2).
- Las columnas numéricas deben ser numéricas nativas en el archivo, no texto (FR-006).
- Ante un error a mitad de la generación, no debe entregarse un archivo descargado
  corrupto o incompleto (Edge Case) — resuelto construyendo el workbook completo en memoria
  antes de responder (research.md §2).
- Autorización: solo el rol que hoy tiene acceso a la pantalla de Inventario (`ADMIN`)
  puede generar el archivo, verificado también del lado del servidor y no solo ocultando el
  botón (FR-009, research.md §3).
- No se introduce un rol nuevo "Gestor de Inventario" (Assumptions del spec).
- No se guarda historial de exportaciones ni auditoría de quién exportó y cuándo
  (Assumptions del spec) — fuera de alcance de esta feature.

**Scale/Scope**: 2 repos, cambio acotado:
- `pos-backend`: 1 endpoint nuevo + 1 dependencia nueva (`openpyxl`) + 1 helper de
  construcción del `.xlsx` + 1 función de query sin paginar en el service existente del
  módulo de inventario.
- `pos-heladeria`: 1 método nuevo en `InventoryService` + 1 botón y su handler en
  `inventory-page.component.ts` (el mismo archivo ya tocado por las specs 079/085 en este
  módulo).
- Sin nuevas entidades de datos, sin migraciones, sin nuevos módulos/rutas de navegación.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principio | Evaluación |
|---|-----------|------------|
| I | Nacen de un spec | **PASS** — `specs/086-exportacion-excel-inventario/spec.md` aprobado (con sesión de clarificación 2026-09-22), define problema, alcance, reglas de negocio, criterios de aceptación. |
| II | Comportamiento existente protegido | **PASS** — la feature es aditiva: nuevo endpoint, nuevo botón. La columna "Estado" exportada replica exactamente la fórmula ya existente en pantalla (`active AND current_stock <= min_stock`), sin introducir un tercer estado "Agotado" que no existe hoy (research.md §4) — evita un cambio de comportamiento no autorizado. `GET /inventory/items` (endpoint de listado existente) no se modifica. No hay decisión de negocio que registrar en `registro-de-anomalias.md` porque no se cambia ningún comportamiento existente. |
| III | Characterization tests | **N/A** — no existen tests `"CONGELA comportamiento actual:"` sobre el endpoint de listado de inventario ni sobre `inventory-page.component.ts` en relación a exportación (funcionalidad nueva, sin comportamiento previo que proteger en este alcance). |
| IV | Nuevos specs pueden introducir nuevo comportamiento | **PASS** — el spec introduce comportamiento nuevo (exportación) de forma explícita y acotada; no se exige equivalencia con nada anterior porque no existía. |
| V | Nuevas funcionalidades antes que refactors oportunistas | **PASS** — el cambio se limita al endpoint nuevo, su service/helper de soporte, el método nuevo del `InventoryService` del frontend y el botón/handler en `inventory-page.component.ts`; no se toca ninguna otra pestaña (Compras, Movimientos), modal, ni se aprovecha para refactorizar código existente no relacionado. |
| VI | Evolución incremental | **PASS** — unidad única y verificable: exportación bajo demanda, sin migración de datos, sin cambio de arquitectura. La spec ya fue dividida en dos historias independientes (P1 descarga, P2 consistencia de columnas), consistente con el flujo de `speckit-tasks` posterior. |
| VII | Compatibilidad con datos históricos | **N/A** — no hay facturas ni datos históricos involucrados; se leen datos actuales de inventario, no se recalcula ni reemite nada histórico. |
| VIII | Evolución del modelo de datos | **N/A** — sin cambios de modelo de datos, sin migraciones (ver `data-model.md`). |
| IX | Dependencias nuevas | **PASS con justificación** — `openpyxl` es la única dependencia nueva, justificada en `research.md` §1 (problema que resuelve, por qué no basta la librería estándar, alternativas consideradas, impacto en mantenimiento/despliegue). Frontend: cero dependencias nuevas. |
| X | Verificación obligatoria | **PASS** — se verificará con: tests nuevos del endpoint de exportación (backend, `unittest`), tests nuevos del componente/servicio de exportación (frontend, `ng test`), y la guía manual `quickstart.md` contra los 4 criterios de aceptación de ambas historias y los edge cases del spec. Pendiente de ejecución en la fase de implementación (`tasks.md`). |
| XI | Decisiones de negocio vs técnicas | **PASS** — el spec (con sus Clarifications) es la decisión de negocio (qué exportar, con qué alcance); este plan documenta únicamente decisiones técnicas de cómo implementarlo (research.md), sin redefinir ninguna regla de negocio no cubierta por el spec. |
| XII | Trazabilidad | **PASS** — cadena Necesidad (Contexto de negocio del spec: auditorías/contabilidad externa) → Spec 086 (con Clarifications) → este plan/research/data-model/contracts → Implementación (tasks.md, pendiente) → Tests → Verificación (quickstart.md), íntegra y documentada. |
| XIII | Español de Colombia | **PASS** — todos los artefactos de esta spec (spec.md, plan.md, research.md, data-model.md, contracts/, quickstart.md) en español de Colombia; el código, los mensajes de commit y los nombres de rama se escriben en inglés conforme a los Principios XIV/XV. |
| XIV | Estrategia y convención de ramas Git | **Pendiente en fase de implementación** — antes de tocar código en `pos-backend` o `pos-heladeria` debe crearse una rama nueva por repo con estructura `<tipo>/<numero_spec>-<nombre_spec>` (p. ej. `feat/086-inventory-excel-export`), partiendo de la rama actual de cada repo. No aplica a esta fase de planeación (solo toca `pos-specs`). |
| XV | Política de commits Git | **N/A en esta fase** — esta fase no genera commits de código; cuando se implemente, los commits se harán solo bajo pedido explícito del usuario, en unidades pequeñas y lógicas, en inglés, sin marcas de autoría de IA. |

**Resultado**: sin violaciones bloqueantes. La única dependencia nueva (`openpyxl`) está
justificada conforme al Principio IX. No aplica Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/086-exportacion-excel-inventario/
├── plan.md                            # Este archivo (/speckit-plan)
├── research.md                        # Fase 0 (/speckit-plan) — 8 decisiones técnicas
├── data-model.md                      # Fase 1 (/speckit-plan) — proyección de InventoryItem/UnitMeasure, sin cambios de esquema
├── contracts/
│   └── inventory-export-api.md        # Fase 1 (/speckit-plan) — contrato del endpoint GET /inventory/items/export
├── quickstart.md                      # Fase 1 (/speckit-plan) — guía de validación manual (4 escenarios)
├── checklists/
│   └── requirements.md                # Checklist de calidad del spec
└── tasks.md                           # Fase 2 (/speckit-tasks) — aún no generado
```

### Source Code (repository root)

```text
pos-backend/                                        # API — cambios nuevos, aditivos
└── app/
    ├── api/v1/inventory/
    │   ├── router.py                                # + GET /items/export (nuevo endpoint)
    │   ├── service.py                                # + función de query sin paginar (todos los insumos, activos e inactivos, order_by name)
    │   ├── export.py                                 # NUEVO — helper build_inventory_excel() con openpyxl: encabezados, mapeo de Tipo/Estado, celdas numéricas
    │   └── schemas.py                                # sin cambios (el endpoint no usa un schema Pydantic de salida — retorna un Response binario)
    ├── core/
    │   └── dependencies.py                           # sin cambios — se reutiliza require_tenant_admin ya existente
    └── characterization_tests/ (o carpeta de tests equivalente)
        └── test_inventory_export.py                  # NUEVO — tests del endpoint (200 con datos, 200 vacío, 403 sin rol, columnas/tipos de celda)
requirements.txt                                       # + openpyxl

pos-heladeria/                                       # Frontend — cambios nuevos, aditivos
└── src/app/modules/inventory/
    ├── services/
    │   ├── inventory.service.ts                      # + método exportItems() (HttpClient, responseType: 'blob', observe: 'response')
    │   └── inventory.service.spec.ts                 # + tests del método nuevo
    └── pages/
        ├── inventory-page.component.ts                # + botón "Exportar Inventario" junto al bloque de filtros/tabla + handler de descarga (blob URL + <a download>) + manejo de error vía ToastService ya inyectado
        └── inventory-page.component.spec.ts            # + tests del botón/handler nuevo
```

**Structure Decision**: aplicación web con cambios coordinados en los dos repos hermanos
(`pos-backend` para el endpoint y la generación del archivo, `pos-heladeria` para el botón
y la descarga), siguiendo el mismo patrón por módulo (`router.py` + `service.py` +
`schemas.py` en backend; `services/` + `pages/` en frontend) que ya usan el resto de
endpoints y componentes de Inventario. No se crean módulos, rutas de navegación ni
entidades nuevas — todo el cambio vive dentro del módulo `inventory`/`inventario` ya
existente en ambos repos.

## Complexity Tracking

*No aplica — el Constitution Check no registró violaciones bloqueantes. La única
dependencia nueva (`openpyxl`) está justificada en `research.md` §1 conforme al Principio
IX, no como una excepción a registrar aquí.*
