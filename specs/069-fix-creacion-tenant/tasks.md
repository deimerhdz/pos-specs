---

description: "Task list template for feature implementation"
---

# Tasks: Corrección — falla al crear un tenant con usuario por migraciones rotas

**Input**: Documentos de diseño de `/specs/069-fix-creacion-tenant/`
**Repositorio de implementación**: `../pos-backend` (todas las rutas de archivo de abajo son
relativas a ese repositorio hermano, no a `pos-specs`)
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, quickstart.md

**Tests**: No hay characterization tests sobre estos dos archivos de migración
(`research.md` § 5) y el defecto no es reproducible contra la infraestructura de tests unitarios
del repo (solo se manifiesta contra PostgreSQL real, cargando el catálogo completo de
migraciones) — la verificación de este hotfix es funcional/de integración, no unitaria. Ver
Fase 3 y `quickstart.md`.

**Organization**: Una sola historia de usuario (US1, P1) — el hotfix completo es una sola unidad
verificable.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Corrección puntual sobre 2 archivos ya existentes en `../pos-backend/alembic/versions/`. Sin
Setup ni Foundational: no hay infraestructura compartida que construir antes de aplicar el fix
(plan.md § Project Structure) — se pasa directo a la única historia de usuario.

---

## Phase 1: User Story 1 - Crear un tenant nuevo con su usuario administrador (Priority: P1) 🎯 MVP

**Goal**: Que `POST /api/v1/super-admin/tenants` (y cualquier operación que dependa de cargar el
catálogo completo de migraciones) deje de fallar por el `NameError` de Alembic reportado, sin
cambiar el DDL que ninguna migración aplica.

**Independent Test**: Contra PostgreSQL real, correr `alembic history` (carga sin error) y crear
un tenant de punta a punta vía la API (200, con el tenant y su schema realmente creados) — ambos
descritos en `quickstart.md`.

### Implementation for User Story 1

- [X] T001 [P] [US1] En
  `../pos-backend/alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py`, eliminar
  la asignación de nivel de módulo `_CK_TYPE = op.f("ck__promotions__ck_promotion_type")`
  (línea ~41) y, en su lugar, calcular `_CK_TYPE = op.f("ck__promotions__ck_promotion_type")`
  como variable local al inicio del cuerpo de `upgrade()` y, por separado, al inicio del cuerpo
  de `downgrade()` — mismo valor de cadena, mismo nombre de variable, solo cambia dónde se
  evalúa (`data-model.md` § Archivo 1, `research.md` § 1 y § 3).
- [X] T002 [P] [US1] En
  `../pos-backend/alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py`,
  eliminar las cuatro asignaciones de nivel de módulo (`_CK_TYPE`, `_CK_VALUE`, `_CK_MIN_QTY`,
  `_CK_PERCENT`, líneas ~52-55) y, en su lugar, calcular las cuatro como variables locales al
  inicio del cuerpo de `upgrade()` y, por separado, al inicio del cuerpo de `downgrade()` —
  mismos valores de cadena, mismos nombres de variable (`data-model.md` § Archivo 2,
  `research.md` § 1 y § 3).
- [X] T003 [US1] Ejecutar `alembic history` en `../pos-backend` contra una base de datos
  PostgreSQL real (local) y confirmar que el catálogo completo de revisiones carga sin
  `NameError` (`quickstart.md` § 1). Depende de: T001, T002. **Hecho** — `alembic history`
  y `alembic heads` cargan las ~40 revisiones sin ningún error.
- [X] T004 [US1] Ejecutar `alembic upgrade head` en `../pos-backend` contra esa misma base de
  datos y confirmar que no reporta ningún cambio de esquema pendiente inesperado — confirma que
  el DDL de ambas migraciones sigue siendo exactamente el mismo que antes del fix
  (`quickstart.md` § 2, `spec.md` FR-004). Depende de: T003. **Hecho** — `alembic current` ya
  mostraba `94144eaa60b5 (head)` antes de correr `upgrade head`; el comando no aplicó ningún DDL
  nuevo (confirma FR-004: cero cambio de efecto).
- [ ] T005 [US1] Crear un tenant de punta a punta vía `POST /api/v1/super-admin/tenants`
  (`quickstart.md` § 3): confirmar respuesta `200` con `{"status": "ok", "tenant": "..."}`, una
  fila nueva en `shared.tenants`, y que el schema del tenant existe en PostgreSQL. Depende de:
  T003. **Bloqueada por un hallazgo nuevo, fuera del alcance de este spec** — ver nota debajo de
  la tabla de tareas. El `NameError` de Alembic que motivó este spec ya no ocurre (confirmado:
  la ejecución avanza mucho más allá del punto donde antes fallaba), pero `tenant_create()`
  ahora falla más adelante, en un paso distinto, por una causa distinta y preexistente.

**Checkpoint**: La causa raíz de este spec (el `NameError` de Alembic) está corregida y
verificada — T001-T004 completas. La Historia de Usuario 1 **no** queda cerrada de punta a
punta todavía: crear un tenant sigue fallando, pero ahora por un defecto distinto y ya
preexistente (ver hallazgo abajo), no por el que motivó este spec.

---

## Phase 2: Polish & Cross-Cutting Concerns

**Purpose**: Confirmar que el fix no introdujo ninguna regresión en el resto del backend, y
cerrar la verificación en producción.

- [X] T006 [P] Ejecutar
  `python -m unittest discover -s app/characterization_tests` en `../pos-backend` (suite
  completa) y confirmar que ningún characterization test quedó en rojo por este cambio — se
  espera que ninguno lo esté, dado que ninguno cubre estos dos archivos de migración
  (`research.md` § 5), pero se corre igual como red de seguridad. **Hecho** — 619 tests, 619 en
  verde.
- [ ] T007 Repetir los pasos 1 y 3 de `quickstart.md` (`alembic history` y crear un tenant de
  punta a punta) contra el entorno de **producción**, una vez confirmado en local — recién ahí
  se considera cerrado el incidente reportado por el usuario (`spec.md` SC-001/SC-002). Depende
  de: T003, T004, T005. **Bloqueada**: no aplica repetir el paso 3 contra producción hasta
  resolver el hallazgo nuevo que bloquea T005 (abajo) — el paso 1 (`alembic history`) sí se
  puede confirmar en producción de forma independiente cuando se despliegue este fix.

## Hallazgo nuevo, fuera del alcance de este spec: `tenant_create()` falla después, por otra causa

Al ejecutar T005 (crear un tenant de punta a punta contra Postgres real, en local), la ejecución
avanzó correctamente más allá del punto donde antes fallaba (`context.get_current_revision() !=
SCRIPT.get_current_head()`, la causa raíz de este spec) — eso confirma que el fix de T001/T002
es correcto y suficiente para su propio alcance. Pero la creación del tenant sigue sin
completarse: ahora falla más adelante, dentro de la misma función
(`../pos-backend/app/core/db.py::tenant_create()`), en
`get_tenant_specific_metadata().create_all(bind=db.connection())`, con:

```text
sqlalchemy.exc.NoReferencedTableError: Foreign key associated with column
'payment_methods.catalog_id' could not find table 'shared.payment_method_catalog'
with which to generate a foreign key to target column 'id'
```

**Causa, ya identificada**: `get_tenant_specific_metadata()` (`app/core/db.py`, código
preexistente) construye un `MetaData` **aislado** copiando únicamente las tablas cuyo
`schema == "tenant"`. `payment_methods` (tabla de tenant) tiene una FK real hacia
`shared.payment_method_catalog` (tabla compartida, spec 032) — al copiar `payment_methods` a esa
metadata aislada sin también incluir la tabla compartida que referencia, SQLAlchemy no puede
resolver esa FK al generar el DDL, y `create_all()` falla antes de crear ninguna tabla.

**Por qué está fuera del alcance de este spec**: es una causa completamente distinta (una FK
cruzada entre schema de tenant y schema compartido, no calculada correctamente al construir la
metadata aislada del tenant), en una función distinta, sin relación con `op.f()` ni con el
catálogo de migraciones de Alembic — spec.md de este feature (069) delimita expresamente el
alcance al defecto de los dos archivos de migración (FR-005: "la corrección NO DEBE cambiar
ninguna regla de negocio... el alcance de este hotfix es exclusivamente que la operación deje de
fallar por esta causa técnica" — la causa técnica de *este* spec).

**Verificado, no solo reproducido una vez**: la prueba se hizo llamando `tenant_create()`
directamente contra Postgres real (sin pasar por HTTP, ver nota abajo); el `rollback` de la
transacción funcionó correctamente — no quedó ningún schema ni fila de tenant huérfana en la
base de datos tras el fallo.

**No se investigó ni se tocó ningún archivo para este hallazgo** (Principio V/instrucción del
spec: documentar, no modificar salvo que sea estrictamente necesario) — se deja documentado para
que el usuario decida si abre un spec/hotfix nuevo para esto. Dado que sigue bloqueando
completamente la creación de tenants en producción (el objetivo original del usuario), muy
probablemente sí lo amerita.

---

## Dependencies & Execution Order

### Phase Dependencies

- No hay Setup ni Foundational — se pasa directo a la Fase 1 (User Story 1), que es el hotfix
  completo.
- **Polish (Fase 2)**: depende de que la Fase 1 esté completa.

### Within User Story 1

- T001 y T002 son independientes entre sí (archivos distintos) — ambas deben completarse antes
  de T003, porque `alembic history` falla si **cualquiera** de los dos archivos sigue roto
  (`research.md` § 2: Alembic carga todos los archivos de `alembic/versions/` para construir el
  mapa de revisiones).
- T004 y T005 dependen de T003 (confirmar primero que el catálogo carga, antes de aplicar
  migraciones o crear un tenant).

### Parallel Opportunities

- T001 y T002 son totalmente paralelas entre sí (archivos distintos, mismo tipo de cambio).
- T006 puede correr en paralelo con T004/T005 (verificación de regresión independiente del
  entorno con PostgreSQL real).

---

## Parallel Example: User Story 1

```bash
# Lanzar juntas las dos correcciones (archivos distintos):
Task: "Mover _CK_TYPE a variable local en ba4b6bd573a6_063b_promociones_retiro_estructura_.py (T001)"
Task: "Mover las 4 constantes a variables locales en 94b7e35f5e5e_063d_promociones_reglas_destructivo.py (T002)"
```

---

## Implementation Strategy

### MVP (única historia)

1. Completar T001 y T002 (en paralelo).
2. Completar T003 → T004 → T005 (verificación de punta a punta en local).
3. **Detenerse y validar**: confirmar que los tres pasos anteriores pasan antes de tocar
   producción.
4. Completar Fase 2 (T006 en paralelo con lo anterior; T007 al final, contra producción) —
   recién ahí el hotfix se considera cerrado.

### Incremental Delivery

Al ser un hotfix de un solo incidente, no aplica una entrega incremental por historias — el
criterio de "listo" es la Fase 1 completa y verificada en local (T001-T005), seguida de la
confirmación en producción (T007).

---

## Notes

- Ninguna tarea de este documento modifica el DDL que aplica cada migración (mismas columnas,
  mismas restricciones, mismo orden) — solo el momento en que Python evalúa 5 expresiones
  `op.f(...)` (`plan.md` § Constraints, `research.md` § 4).
- No hay tareas de tests unitarios nuevos: el defecto solo se manifiesta contra PostgreSQL real
  cargando el catálogo completo de migraciones, algo que la infraestructura de tests del repo
  (SQLite en memoria, funciones invocadas en proceso) no reproduce — de ahí que la verificación
  sea funcional (T003-T005, T007), no unitaria.
- Cometer (`git commit`) después de T001+T002 juntas (un solo cambio lógico: el hotfix), como de
  costumbre en este proyecto.
