# Implementation Plan: Corrección — falla al crear un tenant por una referencia entre schemas no resuelta

**Branch**: `070-fix-metadata-tenant-fk` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/070-fix-metadata-tenant-fk/spec.md`

**Repositorio de implementación**: `../pos-backend`.

## Summary

`app/core/db.py::get_tenant_specific_metadata()` construye una `MetaData` de SQLAlchemy aislada,
copiando (`Table.to_metadata()`) únicamente las tablas cuyo `schema == "tenant"`. La tabla
`payment_methods` (tenant) tiene una FK real hacia `payment_method_catalog` (`schema ==
"shared"`, spec 032) que esa copia no incluye — `to_metadata()` no puede resolver la FK dentro de
la metadata aislada, y `create_all()` falla con `NoReferencedTableError` antes de crear ninguna
tabla del tenant nuevo. La corrección extiende esa misma función para que, además de las tablas
de tenant, copie también cualquier tabla de otro schema que alguna tabla de tenant referencie por
FK — generalizado (no hardcodeado a `payment_method_catalog`), para cubrir FR-003 sin depender de
que siga siendo el único caso. Ya se validó de punta a punta contra PostgreSQL real, antes de
escribir este plan (`research.md` § 3): `create_all()` sobre la metadata corregida crea las 41
tablas del tenant sin error, y detecta correctamente que la tabla compartida ya existe
(`checkfirst=True`, no la duplica).

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend/Dockerfile`)

**Primary Dependencies**: SQLAlchemy 2.0.50 (ya en `requirements.txt`, sin cambios) — la
corrección no agrega ninguna dependencia nueva; usa únicamente API pública ya usada en el mismo
archivo (`Table.to_metadata`, `Table.foreign_keys`, `MetaData`).

**Storage**: PostgreSQL 16, schema-per-tenant. La corrección no cambia ninguna tabla, columna ni
FK del modelo de datos — solo cómo Python arma, en memoria, la metadata usada para generar el
DDL de un tenant nuevo (Constitution Check § Principio VIII: N/A).

**Testing**: `unittest` de la librería estándar, mismo patrón que el resto del repositorio;
verificación adicional de punta a punta contra Postgres real (igual que spec 069, research.md §
5 de ese spec: este defecto tampoco es reproducible contra SQLite en memoria).

**Target Platform**: Servidor Linux (Docker); afecta tanto al entorno local como al de
producción.

**Project Type**: Corrección puntual dentro de un servicio web existente (backend FastAPI) — sin
superficie nueva, ni de API ni de UI.

**Performance Goals**: N/A — el costo adicional (recorrer `foreign_keys` de ~41 tablas de tenant
una vez, por creación de tenant) es despreciable frente al resto de `tenant_create()` (crear un
schema completo con sus tablas).

**Constraints**: Cambio confinado a `get_tenant_specific_metadata()` en
`../pos-backend/app/core/db.py` (una sola función, el único punto de llamada de
`tenant_create()`, spec.md § Assumptions); no se toca `get_shared_metadata()` (usada también por
`initialize_database()`) ni ninguna otra función de ese archivo.

**Scale/Scope**: 1 archivo, 1 función, ~10 líneas nuevas.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra los trece principios de `.specify/memory/constitution.md` (v3.0.0):

- **I. Nace de un spec** — ✅ Cumple: este plan deriva de `spec.md` (feature 070), aprobado antes
  de tocar código.
- **II. Comportamiento existente protegido** — ✅ Cumple por diseño: el comportamiento correcto
  (qué tablas tiene un tenant, qué datos comparte con el catálogo de la plataforma) no cambia —
  se corrige un defecto que impedía que ese comportamiento se ejecutara. No es un cambio de
  comportamiento de negocio.
- **III. Characterization tests protegen lo heredado** — N/A: no existe ningún characterization
  test sobre `get_tenant_specific_metadata()` ni sobre la creación completa de un tenant contra
  Postgres real (spec 069 ya documentó por qué: `tenant_create()` no es unit-testeable contra
  SQLite — `test_tenant_plan_assignment.py`).
- **IV. Nuevos specs pueden introducir comportamiento nuevo** — N/A: no se introduce
  comportamiento nuevo, se corrige uno existente y roto.
- **V. Nuevas funcionalidades antes que refactor oportunista** — ✅ Cumple: el cambio se limita a
  una función; no se toca `get_shared_metadata()`, ni `tenant_create()`, ni el modelo de datos.
- **VI. Evolución incremental** — ✅ Cumple: un solo tipo de cambio (corrección de un defecto de
  construcción de metadata), sin mezclarlo con ninguna funcionalidad nueva ni con el hotfix de
  spec 069 (ya cerrado, causa distinta).
- **VII. Compatibilidad con datos históricos** — ✅ Cumple: no se toca ningún dato ni esquema de
  ningún tenant ya existente; el catálogo compartido tampoco se modifica (research.md § 3:
  verificado que `checkfirst=True` no lo duplica ni lo recrea).
- **VIII. Evolución del modelo de datos** — N/A: no se agrega, quita ni modifica ninguna
  columna/tabla/FK — se corrige únicamente cómo Python arma una copia de metadata en memoria.
- **IX. Dependencias nuevas justificadas** — N/A: no se agrega ninguna dependencia.
- **X. Verificación obligatoria** — ✅ Ya verificado en Fase 0 (`research.md` § 3) contra Postgres
  real, antes de este plan; se repite formalmente en `quickstart.md`.
- **XI. Decisiones de negocio vs. técnicas** — ✅ Decisión puramente técnica; sin conflicto con
  ninguna regla de negocio.
- **XII. Trazabilidad** — ✅ Necesidad (hallazgo documentado en spec 069 § tasks.md) → Spec (070)
  → Plan (este documento) → Implementación (tasks.md, pendiente) → Verificación (`quickstart.md`).
- **XIII. Todo en español de Colombia** — ✅ Este plan y sus artefactos se escriben en español.

**Resultado**: PASS. No hay violaciones que requieran `Complexity Tracking`.

## Project Structure

### Documentation (this feature)

```text
specs/070-fix-metadata-tenant-fk/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command) — N/A, ver nota abajo
├── quickstart.md        # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

No se genera `contracts/`: es una corrección interna que no expone ni modifica ninguna interfaz
externa — la única superficie observable (que crear un tenant se complete de principio a fin) ya
está descrita como criterio de éxito en `spec.md`.

`data-model.md` documenta, a modo de inventario, la única función y relación afectadas — no hay
ningún cambio de esquema (Constitution Check § Principio VIII).

### Source Code (repository `../pos-backend`)

```text
pos-backend/
└── app/core/db.py     # MODIFICADO: get_tenant_specific_metadata() — ahora también copia,
                        # a la metadata aislada del tenant, cualquier tabla de otro schema
                        # que una tabla de tenant referencie por FK (generalizado, FR-003)
```

**Structure Decision**: Corrección puntual sobre una función ya existente en
`../pos-backend/app/core/db.py`. No se crea ningún archivo nuevo de producción; no aplica
ninguna de las estructuras de proyecto del template.

## Complexity Tracking

*Sin violaciones de la constitución que requieran justificación — tabla omitida.*
