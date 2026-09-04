---

description: "Task list template for feature implementation"
---

# Tasks: Rol Mesero con Acceso Restringido a Terminal de Mesas y Órdenes

**Input**: Design documents from `/specs/075-rol-mesero/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: La spec no pide explícitamente TDD. Se incluyen tareas de test puntuales solo donde el propio research.md señala una pieza crítica de seguridad (la verificación de alcance por rol en el backend, User Story 3) — consistente con la convención existente del repo de tener characterization tests para lógica de autorización.

**Organization**: Las tareas están agrupadas por historia de usuario (spec.md) para poder implementarlas y probarlas de forma independiente.

**Repos**: Todas las rutas de archivo son relativas a los repos hermanos de este spec-kit: `../pos-backend/` (FastAPI) y `../pos-heladeria/` (Angular). Este repo (`pos-specs`) no tiene código propio que tocar.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

---

## Phase 1: Setup

**Purpose**: Confirmar el punto de partida real antes de crear la migración de datos.

- [X] T001 Confirmar la revisión `head` actual de Alembic en `../pos-backend` (`alembic heads`) para usarla como `down_revision` de la migración nueva — al momento de escribir este plan es `f3a9c1b7e2d4`, pero puede haber avanzado si se fusionaron otras specs primero.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: El valor de rol "MESERO" debe existir en el sistema de tipos de ambos repos y en la base de datos antes de que cualquier historia de usuario tenga sentido.

**⚠️ CRITICAL**: Ninguna historia de usuario puede empezar hasta que esta fase esté completa.

- [X] T002 [P] Agregar `MESERO = "MESERO"` al enum `RoleName` en `../pos-backend/app/api/v1/users/schemas.py`
- [X] T003 [P] Agregar `"MESERO"` a la tupla `ROLE_NAMES` en `../pos-backend/app/core/db.py` (para que quede sembrado en cualquier base de datos nueva que se inicialice a partir de ahora)
- [X] T004 Crear la migración de Alembic `../pos-backend/alembic/versions/<rev>_seed_mesero_role.py` (`down_revision` = el head confirmado en T001) que inserta la fila `MESERO` en `shared.roles` de forma idempotente (verifica por `name` antes de insertar, mismo criterio que `_seed_shared_data()` en `app/core/db.py`), con `downgrade()` que la elimina solo si ningún `User`/`UserInvitation` la referencia todavía — ver data-model.md y research.md D1. Depende de: T001, T002.
- [X] T005 [P] Agregar `MESERO = 'mesero'` al enum `UserRole` en `../pos-heladeria/src/app/core/interfaces/user.interface.ts`
- [X] T006 [P] Agregar `'MESERO'` al tipo `RoleName` (`'ADMIN' | 'CASHIER' | 'MESERO'`) en `../pos-heladeria/src/app/modules/users/interfaces/user-profile.interface.ts`

**Checkpoint**: El rol "MESERO" existe de punta a punta (backend, base de datos, tipos del frontend) — todavía no es asignable desde ninguna pantalla ni tiene ningún efecto de acceso.

---

## Phase 3: User Story 1 - Un Admin asigna el rol Mesero a un miembro de su equipo (Priority: P1) 🎯 MVP parcial

**Goal**: Un Admin puede invitar a un usuario nuevo con rol Mesero, o cambiar el rol de uno existente a Mesero, desde el mismo panel donde ya asigna Admin/Cajero, y verlo etiquetado correctamente.

**Independent Test**: Como Admin, invitar/editar un usuario seleccionando "Mesero" y verificar que queda registrado y etiquetado como "Mesero" en el listado de usuarios.

### Implementation for User Story 1

- [X] T007 [P] [US1] Agregar `<option value="MESERO">Mesero</option>` al `<select>` de rol en `../pos-heladeria/src/app/modules/users/components/invitation-form.component.ts`
- [X] T008 [P] [US1] Agregar `<option value="MESERO">Mesero</option>` al `<select>` de rol en `../pos-heladeria/src/app/modules/users/components/user-role-modal.component.ts`, y actualizar el guard de `ngOnChanges()` (línea con `current === 'ADMIN' || current === 'CASHIER'`) para que también reconozca `'MESERO'` — si no, el modal se abre en blanco al editar un usuario que ya es Mesero.
- [X] T009 [US1] Agregar `MESERO: 'Mesero'` a `ROLE_LABELS` y una clase de color de badge propia (distinta de Admin/Cajero) en `../pos-heladeria/src/app/modules/users/pages/users-page.component.ts`

**Checkpoint**: User Story 1 funciona y es verificable de forma independiente (aunque, hasta que las Historias 2 y 3 estén listas, un usuario Mesero recién asignado todavía tendría el mismo acceso amplio que un Cajero — ese es justamente el problema que cierran las siguientes dos historias).

---

## Phase 4: User Story 2 - Un Mesero solo ve y usa Terminal de Mesas y Órdenes (Priority: P1)

**Goal**: Al iniciar sesión, un usuario Mesero solo ve "Terminal de mesas" y "Órdenes" en la navegación, aterriza en Terminal de Mesas, y cualquier intento de entrar por URL directa a otra pantalla lo redirige ahí.

**Independent Test**: Con un usuario de rol Mesero (asignado vía Historia 1, o directamente por API/DB), iniciar sesión y verificar el menú, el destino por defecto, y la redirección al intentar entrar a una pantalla fuera de alcance.

### Implementation for User Story 2

- [X] T010 [P] [US2] Agregar `UserRole.MESERO` al arreglo `roles` de los ítems "Terminal de mesas" y "Órdenes" en `../pos-heladeria/src/app/core/config/navigation.config.ts` (ver contracts/frontend-route-access.md)
- [X] T011 [P] [US2] Agregar `UserRole.MESERO` al `roleGuard([...])` de las rutas `mesas-sesiones`, `mesas-sesiones/:tableId/orden-manual`, `orders` y `orders/:id` en `../pos-heladeria/src/app/modules/dashboard/routes.ts`
- [X] T012 [P] [US2] Agregar `[UserRole.MESERO]: '/dashboard/mesas-sesiones'` a `ROLE_DEFAULT_ROUTES` en `../pos-heladeria/src/app/core/guards/role.guard.ts`
- [X] T013 [US2] Revisar `../pos-heladeria/src/app/modules/orders/pages/orders-page.component.ts` y `order-detail.component.ts`: ocultar/deshabilitar para el rol Mesero cualquier acción que modifique el estado de una orden (si existe alguna en esas pantallas), dejando solo consulta — FR-005. **Verificado sin cambios**: ninguna de las dos pantallas expone hoy ninguna acción que modifique el estado de una orden para ningún rol (`orders-page.component.ts` solo filtra/recarga; `order-detail.component.ts` es una vista de solo lectura) — esas acciones viven en Terminal de Mesas, que Mesero ya tiene permitida. FR-005 queda satisfecho sin tocar código.
- [X] T013a [US2] **Tarea emergente, detectada por `ng build --configuration production`** (no estaba en el research original — `UserRole` es un enum con varios `Record<UserRole, string>` exhaustivos fuera de `navigation.config.ts`/`routes.ts`/`role.guard.ts`, y el compilador de Angular exige cubrir todos sus valores). Agregar la entrada de Mesero a cada uno: `ROLE_HOME` en `../pos-heladeria/src/app/core/guards/password-change.guard.ts`, `../pos-heladeria/src/app/modules/auth/pages/change-password.component.ts` y `../pos-heladeria/src/app/modules/auth/pages/login.component.ts` (los tres → `/dashboard/mesas-sesiones`, mismo criterio que `ROLE_DEFAULT_ROUTES`); `ROLE_LABEL`/`ROLE_LABELS` en `../pos-heladeria/src/app/modules/account/pages/account-settings.component.ts`, `../pos-heladeria/src/app/modules/dashboard/layout/header.component.ts`, `../pos-heladeria/src/app/modules/settings/pages/tenant-info.component.ts` y `../pos-heladeria/src/app/modules/super-admin/pages/super-admin-users-page.component.ts` (→ `'Mesero'`); `ROLE_BADGE_CLASSES` en `header.component.ts` (→ clase de color propia). Sin esto, `ng build` falla (`TS2741`) y el login de un usuario Mesero rompería en tiempo de ejecución al no encontrar su entrada en `ROLE_HOME`.

**Checkpoint**: User Story 2 funciona y es verificable de forma independiente — la navegación y el enrutamiento ya reflejan el alcance reducido, aunque hasta la Historia 3 el backend todavía respondería a una llamada directa fuera de esa navegación (igual que hoy le pasa a Cajero con varios endpoints).

---

## Phase 5: User Story 3 - El bloqueo de acceso del Mesero también se aplica del lado del servidor (Priority: P1)

**Goal**: El servidor deniega con 403 cualquier solicitud de un usuario Mesero fuera de Terminal de Mesas/Órdenes-consulta, sin importar por qué vía llegue, sin afectar en nada a Admin ni a Cajero.

**Independent Test**: Con el token de un usuario Mesero, llamar directamente a un endpoint fuera de alcance (p. ej. inventario, reportes, caja) y verificar 403; llamar a uno dentro de alcance y verificar que funciona con normalidad; repetir con Admin/Cajero y verificar cero cambios.

### Implementation for User Story 3

- [X] T014 [US3] Implementar en `get_current_user()` (`../pos-backend/app/core/dependencies.py`) la verificación centralizada de alcance por rol descrita en research.md D2: cuando `user.role.name == "MESERO"`, comparar `(request.method, plantilla de ruta ya resuelta)` contra la lista de permitidos de `specs/075-rol-mesero/contracts/backend-endpoint-access.md` (sección "Permitido para Mesero") y responder `403 Forbidden` si no está en esa lista; para Admin/Cajero, el comportamiento de `get_current_user()` no cambia en nada. Depende de: T002, T003, T004. **Hecho**: `_MESERO_ALLOWED_ROUTES` (42 entradas) + `_enforce_mesero_scope()` agregadas a `dependencies.py`, invocada al final de `get_current_user()`.
- [X] T015 [US3] Agregar `../pos-backend/app/characterization_tests/test_mesero_role_scope.py` cubriendo, con un usuario Mesero de prueba: (a) un endpoint de Terminal de Mesas y uno de Órdenes-consulta responden con normalidad; (b) un endpoint bloqueado de cada categoría de `contracts/backend-endpoint-access.md` (Caja-escritura, Ventas-mostrador, Inventario, Reportes, Usuarios) responde 403; (c) los mismos endpoints bloqueados, llamados con un usuario Admin y uno Cajero, siguen respondiendo exactamente igual que antes de esta funcionalidad (SC-004). Depende de: T014. **Hecho**: 12 tests, todos en verde. Además se corrió la suite completa de characterization tests (`python -m unittest discover -s app/characterization_tests`, 721 tests) — sin regresiones.

**Checkpoint**: Las tres historias de usuario funcionan juntas — el rol Mesero cumple de punta a punta la promesa de "solo tendrá acceso a Terminal de Mesas y consulta de Órdenes", con bloqueo real en ambas capas.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Cierre de la funcionalidad — documentación y verificación de extremo a extremo.

- [X] T016 [P] Actualizar `../pos-backend/README.md` (línea ~75: "siembra los roles (`SUPER_ADMIN`, `ADMIN`, `CASHIER`)") para incluir `MESERO` en la lista de roles sembrados en el primer arranque.
- [X] T017 [P] Ejecutar de punta a punta la validación de [quickstart.md](./quickstart.md) (las 4 secciones: asignación, navegación, bloqueo de servidor, edge cases) contra un ambiente local con la migración aplicada. **Parcial**: Docker no está disponible en este entorno de ejecución (`docker` sin permiso sobre el socket), así que no se pudo levantar Postgres/Redis/el backend real ni ejercer el flujo de navegador de punta a punta. En su lugar se corrió la validación automatizada más fuerte disponible: 721 characterization tests del backend (incluye los 12 de T015) en verde, `npx tsc --noEmit` y `ng build --configuration production` del frontend sin errores, y `ng test --watch=false` (687/694, mismas 7 fallas preexistentes que sin esta funcionalidad — ver T018). **Pendiente para quien continúe con acceso a Docker**: correr las 4 secciones de quickstart.md contra un ambiente real.
- [X] T018 Revisión de regresión: con un usuario Admin y uno Cajero existentes, repetir los pasos 2 y 3 de quickstart.md sobre pantallas/endpoints que **no** cambian con esta funcionalidad, y confirmar cero diferencias de comportamiento (SC-004, Principio II de la constitución). **Hecho** (a nivel automatizado, mismas limitaciones de entorno que T017): los 12 tests de `test_mesero_role_scope.py` incluyen 3 casos explícitos de no-regresión para Admin/Cajero (T015), y la suite completa de 721 characterization tests —que cubre extensamente flujos de Admin y Cajero ya existentes— sigue en verde. En el frontend, se confirmó con `git stash` que las 7 fallas de `ng test` son idénticas (mismos 7 tests, mismos archivos) con y sin los cambios de esta funcionalidad — ninguna regresión introducida.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: depende de Setup (T001 alimenta T004) — bloquea las tres historias de usuario.
- **User Stories (Phase 3-5)**: las tres dependen de que Foundational esté completo. Entre sí, US1/US2/US3 no dependen unas de otras a nivel de archivo (tocan archivos distintos) y pueden implementarse en paralelo si hay varias personas — pero **la funcionalidad no cumple su propósito real hasta que las tres estén completas**, porque las tres son P1 y cada una cierra una pieza distinta de la misma promesa ("Mesero solo tiene acceso a...").
- **Polish (Phase 6)**: depende de que las tres historias estén completas.

### User Story Dependencies

- **User Story 1 (P1)**: puede empezar después de Foundational. No depende de US2 ni US3.
- **User Story 2 (P1)**: puede empezar después de Foundational. No depende de US1 (puede probarse asignando el rol directamente por API/DB en vez de por la UI de Historia 1) ni de US3.
- **User Story 3 (P1)**: puede empezar después de Foundational. No depende de US1 ni de US2 (puede probarse con un usuario Mesero creado directamente en la base de datos).

### Parallel Opportunities

- T002, T003, T005, T006 (Foundational) se pueden hacer en paralelo — archivos distintos, sin dependencia entre sí.
- T007, T008 (US1) se pueden hacer en paralelo — componentes distintos.
- T010, T011, T012 (US2) se pueden hacer en paralelo — archivos distintos.
- Una vez completada la Fase 2, un equipo de tres personas podría tomar US1, US2 y US3 en paralelo (no comparten archivos entre sí).
- T016, T017 (Polish) se pueden hacer en paralelo.

---

## Parallel Example: Foundational

```bash
# Backend (en paralelo entre sí, antes de T004 que sí depende de T001+T002):
Task: "Agregar MESERO al enum RoleName en app/api/v1/users/schemas.py"
Task: "Agregar 'MESERO' a ROLE_NAMES en app/core/db.py"

# Frontend (en paralelo, sin relación con lo anterior):
Task: "Agregar MESERO al enum UserRole en user.interface.ts"
Task: "Agregar 'MESERO' al tipo RoleName en user-profile.interface.ts"
```

---

## Implementation Strategy

### Alcance mínimo con valor real

Como las tres historias son P1 (no hay P2/P3 en esta spec), el "MVP" de esta
funcionalidad **son las tres historias juntas**: User Story 1 sin 2 y 3 solo
crea un rol que se ve distinto pero no restringe nada; User Story 2 sin 3
oculta la navegación pero deja el backend abierto a quien llame la API
directamente — ninguna de las dos por separado cumple la promesa central del
spec ("Mesero solo tendrá acceso a..."). Aun así, cada historia es
independientemente implementable y verificable (ver Independent Test de
cada una), lo cual sirve para dividir el trabajo o para detectar regresiones
temprano.

### Orden recomendado

1. Setup + Foundational (T001-T006) — el rol existe en todo el sistema.
2. User Story 1 (T007-T009) — se puede asignar y ver el rol.
3. User Story 2 (T010-T013) — la navegación ya refleja el alcance reducido.
4. User Story 3 (T014-T015) — el servidor cierra el círculo con bloqueo real.
5. Polish (T016-T018) — documentación y verificación de regresión.

### Estrategia con varias personas

Tras completar Foundational, tres desarrolladores pueden tomar US1
(frontend, gestión de usuarios), US2 (frontend, navegación/rutas) y US3
(backend, autorización) en paralelo — no comparten ningún archivo entre sí.

---

## Notes

- [P] = archivos distintos, sin dependencias pendientes entre esas tareas.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII de la constitución).
- La lista exacta de endpoints/rutas permitidos y bloqueados no se repite aquí: vive en `contracts/backend-endpoint-access.md` y `contracts/frontend-route-access.md`, y T014/T010-T012 deben implementarse exactamente contra esas listas.
- Commitear después de cada tarea o grupo lógico.
- Validar cada historia de forma independiente en su checkpoint antes de continuar con la siguiente.
