---

description: "Task list template for feature implementation"
---

# Tasks: Corrección — falla al crear un tenant por una referencia entre schemas no resuelta

**Input**: Documentos de diseño de `/specs/070-fix-metadata-tenant-fk/`
**Repositorio de implementación**: `../pos-backend` (todas las rutas de archivo de abajo son
relativas a ese repositorio hermano, no a `pos-specs`)
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, quickstart.md — y spec 069 ya
aplicada (el `NameError` de Alembic que bloqueaba este mismo flujo un paso antes).

**Tests**: No hay characterization tests sobre `get_tenant_specific_metadata()` ni sobre
`tenant_create()` (`research.md` § 5) y el defecto solo se reproduce contra PostgreSQL real — la
verificación es funcional/de integración, no unitaria (igual que spec 069). La corrección **ya
se validó de punta a punta contra Postgres real durante `/speckit-plan`** (research.md § 3); las
tareas de esta fase repiten esa misma verificación de forma formal, sobre el cambio ya aplicado
al archivo real (no en un script aparte).

**Organization**: Una sola historia de usuario (US1, P1).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Corrección puntual sobre una función ya existente en `../pos-backend/app/core/db.py`. Sin Setup
ni Foundational — se pasa directo a la única historia de usuario.

---

## Phase 1: User Story 1 - Crear un tenant nuevo con su usuario administrador, hasta el final (Priority: P1) 🎯 MVP

**Goal**: Que `get_tenant_specific_metadata()` resuelva correctamente cualquier referencia de una
tabla de tenant hacia una tabla de otro schema, para que `tenant_create()` complete la creación
del tenant de principio a fin.

**Independent Test**: Contra PostgreSQL real, crear un tenant de punta a punta vía la API (200,
con todas las tablas de su schema creadas, incluida `payment_methods`) y confirmar que ese tenant
puede usar el catálogo compartido de métodos de pago sin error — descrito en `quickstart.md`.

### Implementation for User Story 1

- [X] T001 [US1] En `../pos-backend/app/core/db.py`, extender `get_tenant_specific_metadata()`
  para que, después de copiar las tablas de `schema == "tenant"` a la metadata aislada, recorra
  las `foreign_keys` de esas mismas tablas y copie también (`Table.to_metadata(meta)`) cualquier
  tabla cuyo `schema != "tenant"` que alguna de ellas referencie y que todavía no esté en `meta`
  — generalizado (FR-003), no hardcodeado a `payment_method_catalog`. Código exacto ya validado
  en `research.md` § 3. Depende de: nada (spec 069 ya aplicada).
- [X] T002 [US1] Crear un tenant de punta a punta vía `POST /api/v1/super-admin/tenants`
  (`quickstart.md` § 1): confirmar `200` con `{"status": "ok", "tenant": "..."}`, y que
  `payment_methods` (entre otras) existe en el schema del tenant nuevo. Depende de: T001.
- [X] T003 [US1] Con el tenant creado en T002, activar un método de pago del catálogo compartido
  de la plataforma (`quickstart.md` § 2) y confirmar que funciona sin error — verifica que la FK
  quedó bien formada, no solo que la creación "no falló". Depende de: T002.
- [X] T004 [US1] Confirmar que la tabla compartida (`shared.payment_method_catalog`) no se
  duplicó ni se recreó — mismo conteo de filas antes y después (`quickstart.md` § 3). Depende de:
  T002.

**Checkpoint**: User Story 1 es funcional y verificada de forma independiente — el hotfix
completo (spec 069 + spec 070) permite crear un tenant de principio a fin.

---

## Phase 2: Polish & Cross-Cutting Concerns

**Purpose**: Confirmar que el fix no introdujo ninguna regresión, y cerrar la verificación en
producción.

- [X] T005 [P] Ejecutar
  `python -m unittest discover -s app/characterization_tests` en `../pos-backend` (suite
  completa) y confirmar que ningún characterization test quedó en rojo por este cambio
  (`quickstart.md` § 4).
- [ ] T006 Repetir el paso 1 de `quickstart.md` (crear un tenant de punta a punta) contra el
  entorno de **producción**, una vez confirmado en local — recién ahí se considera cerrado el
  incidente completo (spec 069 + spec 070), coincidiendo con el criterio de cierre ya usado en
  spec 069. Depende de: T002, T003, T004. **Pendiente**: requiere desplegar este cambio a
  producción primero (no tengo acceso a ese entorno desde este sandbox) — queda para que el
  usuario lo confirme tras el despliegue.

---

## Dependencies & Execution Order

### Phase Dependencies

- No hay Setup ni Foundational — se pasa directo a la Fase 1 (User Story 1), que es el hotfix
  completo.
- **Polish (Fase 2)**: depende de que la Fase 1 esté completa.

### Within User Story 1

- T001 es la única tarea de implementación; T002-T004 son verificación secuencial sobre el mismo
  tenant creado en T002 (T003 y T004 pueden ejecutarse en cualquier orden entre sí una vez T002
  está hecho, pero ambas dependen de T002).

### Parallel Opportunities

- T005 puede correr en paralelo con T002-T004 (verificación de regresión independiente del
  entorno con PostgreSQL real usado para crear el tenant de prueba).

---

## Implementation Strategy

### MVP (única historia)

1. Completar T001 (el cambio ya está validado en `research.md` § 3 — aplicarlo al archivo real
   es reproducir exactamente ese mismo código).
2. Completar T002 → T003 → T004 (verificación de punta a punta en local).
3. **Detenerse y validar**: confirmar que los tres pasos anteriores pasan antes de tocar
   producción.
4. Completar Fase 2 (T005 en paralelo con lo anterior; T006 al final, contra producción) —
   recién ahí el incidente completo (spec 069 + spec 070) se considera cerrado.

---

## Notes

- T001 no cambia ninguna tabla, columna ni FK del modelo de datos — solo qué tablas incluye, en
  memoria, la copia de metadata que se usa para crear el schema de un tenant nuevo (`plan.md` §
  Constraints, `research.md` § 4).
- No hay tareas de tests unitarios nuevos: el defecto solo se manifiesta contra PostgreSQL real
  (igual que spec 069).
- Cometer (`git commit`) después de T001, como de costumbre en este proyecto.
