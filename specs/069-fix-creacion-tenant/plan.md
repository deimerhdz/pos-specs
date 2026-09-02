# Implementation Plan: Corrección — falla al crear un tenant con usuario por migraciones rotas

**Branch**: `069-fix-creacion-tenant` | **Date**: 2026-09-02 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/069-fix-creacion-tenant/spec.md`

**Repositorio de implementación**: `../pos-backend` (este repositorio, `pos-specs`, solo contiene
la especificación; el código y las migraciones viven en el repositorio hermano `pos-backend`).

## Summary

Corregir dos archivos de migración de Alembic
(`alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py` y
`alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py`) que calculan
`op.f("ck__promotions__ck_promotion_...")` como constantes de **módulo** (evaluadas al
importarse el archivo), en vez de calcularlas dentro de `upgrade()`/`downgrade()` — el único
momento en que el proxy de operaciones de Alembic existe. Como Alembic importa **todos** los
archivos de `alembic/versions/` para construir el mapa de revisiones, cualquiera de los dos
archivos rotos hace fallar esa carga completa — y `tenant_create()`
(`app/core/db.py`) la dispara en cada creación de tenant para verificar que la base de datos
está al día (spec 033), antes de escribir cualquier dato. La corrección es mover esas cuatro
asignaciones de nivel de módulo a variables locales dentro de cada función que las usa,
replicando exactamente el patrón que el resto de esos mismos archivos ya usa correctamente
(`op.f(...)` llamado inline, dentro de `upgrade()`/`downgrade()`).

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend/Dockerfile`)

**Primary Dependencies**: Alembic 1.18.4, SQLAlchemy 2.0.50 (ya en `requirements.txt`, sin
cambios) — la corrección no agrega ninguna dependencia nueva.

**Storage**: PostgreSQL 16, schema-per-tenant. Ambas migraciones son *schema-per-tenant*
(decoradas con `@for_each_tenant_schema`, `app/scripts/tenant.py`) — la corrección no cambia el
DDL que aplican, solo cómo Python calcula dos nombres de restricción antes de ejecutarlo (ver
`research.md` § 1-2).

**Testing**: `unittest` de la librería estándar, mismo patrón que el resto del repositorio.

**Target Platform**: Servidor Linux (Docker); afecta tanto al entorno local como al de
producción, según el reporte del usuario.

**Project Type**: Corrección de infraestructura de migraciones dentro de un servicio web
existente (backend FastAPI) — sin superficie nueva, ni de API ni de UI.

**Performance Goals**: N/A — no hay objetivo de rendimiento nuevo; el costo de calcular
`op.f(...)` es una operación de cadena de texto trivial, sin diferencia medible entre calcularla
una vez a nivel de módulo o repetidamente dentro de cada invocación de `upgrade()`/`downgrade()`
por tenant.

**Constraints**: Cambios confinados a los dos archivos de migración ya identificados; **cero**
cambios al DDL que cada migración aplica (mismos nombres de restricción, mismas columnas, mismo
orden de operaciones) — el defecto está en *cuándo* Python evalúa `op.f(...)`, no en *qué*
aplica la migración (spec.md FR-004).

**Scale/Scope**: 2 archivos, ~5 líneas cada uno.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra los trece principios de `.specify/memory/constitution.md` (v3.0.0):

- **I. Nace de un spec** — ✅ Cumple: este plan deriva de `spec.md` (feature 069), aprobado antes
  de tocar código.
- **II. Comportamiento existente protegido** — ✅ Cumple por diseño: el comportamiento *correcto*
  de ambas migraciones (qué DDL aplican, sobre qué schemas, con qué nombres de restricción) no
  cambia en absoluto — se corrige únicamente un defecto que impedía que ese comportamiento se
  ejecutara. No es un cambio de comportamiento de negocio; no aplica registrar nada en
  `registro-de-anomalias.md`.
- **III. Characterization tests protegen lo heredado** — N/A: no existe ningún characterization
  test sobre el contenido de estas dos migraciones (verificado en `research.md` § 3); nada que
  proteger de una modificación no autorizada.
- **IV. Nuevos specs pueden introducir comportamiento nuevo** — N/A: este spec no introduce
  comportamiento nuevo, corrige uno existente y roto.
- **V. Nuevas funcionalidades antes que refactor oportunista** — ✅ Cumple: el cambio se limita
  estrictamente a mover 4 asignaciones de `op.f(...)` de nivel de módulo a nivel de función; no
  se toca ningún otro archivo, DDL, ni lógica de negocio de creación de tenants.
- **VI. Evolución incremental** — ✅ Cumple: un solo tipo de cambio (corrección de un defecto de
  infraestructura de migraciones), sin mezclarlo con ninguna funcionalidad nueva.
- **VII. Compatibilidad con datos históricos** — ✅ Cumple: la corrección no cambia el DDL
  aplicado por ninguna migración; cualquier base de datos donde ya se ejecutaron exitosamente
  (antes de que este defecto se introdujera, o en un entorno donde se aplicaron sorteando el
  error) queda exactamente igual. No hay `Sale`/`Invoice`/`CustomerOrder` involucradas.
- **VIII. Evolución del modelo de datos** — N/A en la práctica: no se agrega, quita ni modifica
  ninguna columna/tabla/restricción respecto a lo que las migraciones ya definían — se corrige
  únicamente cómo Python calcula el nombre de 4 restricciones ya existentes en el diseño
  original de esas migraciones.
- **IX. Dependencias nuevas justificadas** — N/A: no se agrega ninguna dependencia.
- **X. Verificación obligatoria** — ✅ Se cubre en `quickstart.md`: cargar el catálogo completo
  de migraciones sin error, y crear un tenant de punta a punta contra una base de datos real.
- **XI. Decisiones de negocio vs. técnicas** — ✅ Esta es una decisión puramente técnica (corregir
  un defecto de sintaxis/orden de evaluación); no hay conflicto con ninguna regla de negocio.
- **XII. Trazabilidad** — ✅ Necesidad (reporte del usuario, con traza exacta) → Spec (069) → Plan
  (este documento, con research.md como evidencia) → Implementación (tasks.md, pendiente) →
  Verificación (`quickstart.md`).
- **XIII. Todo en español de Colombia** — ✅ Este plan y sus artefactos se escriben en español.

**Resultado**: PASS. No hay violaciones que requieran `Complexity Tracking`.

## Project Structure

### Documentation (this feature)

```text
specs/069-fix-creacion-tenant/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command) — N/A, ver nota abajo
├── quickstart.md        # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

No se genera `contracts/`: esta corrección no expone ni modifica ninguna interfaz externa (API,
CLI, esquema) — es una corrección interna de cómo Python evalúa dos archivos de migración. La
única superficie observable (que `POST /api/v1/super-admin/tenants` deje de fallar) ya está
descrita como criterio de éxito en `spec.md`; no hay un contrato nuevo que documentar.

`data-model.md` se genera igual, pero con contenido mínimo: no hay entidades nuevas ni cambios de
esquema (Constitution Check § Principio VIII) — documenta, en su lugar, los dos archivos y las
cuatro constantes afectadas, a modo de inventario del cambio.

### Source Code (repository `../pos-backend`)

```text
pos-backend/
└── alembic/versions/
    ├── ba4b6bd573a6_063b_promociones_retiro_estructura_.py       # MODIFICADO: 1 constante
    │                                                              # (_CK_TYPE) movida de nivel
    │                                                              # de módulo a nivel de función
    └── 94b7e35f5e5e_063d_promociones_reglas_destructivo.py       # MODIFICADO: 4 constantes
                                                                    # (_CK_TYPE/_CK_VALUE/
                                                                    # _CK_MIN_QTY/_CK_PERCENT)
                                                                    # movidas igual
```

**Structure Decision**: Corrección puntual sobre 2 archivos ya existentes en
`../pos-backend/alembic/versions/`. No se crea ningún archivo nuevo de producción; no aplica
ninguna de las estructuras de proyecto del template (no es un proyecto nuevo, ni una app web con
frontend propio) — es un fix acotado dentro del backend FastAPI ya existente.

## Complexity Tracking

*Sin violaciones de la constitución que requieran justificación — tabla omitida.*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
