---

description: "Task list template for feature implementation"
---

# Tasks: Alta de usuarios internos por invitación

**Input**: Documentos de diseño de `/specs/037-invitacion-usuarios-tenant/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Sí incluidos — `plan.md` (Testing) los pide explícitamente: `unittest` vía funciones de router directas, sin `TestClient`, un módulo de test nuevo por historia de usuario.

**Repos**: El código vive en dos repos *sibling* de este repo (`pos-specs`) — Constitución §Alcance:
- **Backend** — `../pos-backend` (Python 3.14, FastAPI). Todas las rutas de este documento que empiezan con `app/` son relativas a `pos-backend`.
- **Frontend** — `../pos-heladeria` (Angular 21). Todas las rutas que empiezan con `src/` son relativas a `pos-heladeria`.

**Organización**: Las tareas se agrupan por historia de usuario para permitir implementación y prueba independiente de cada una.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencia de una tarea sin terminar)
- **[Story]**: Historia de usuario a la que pertenece (US1, US2, US3, US4)

---

## Phase 1: Setup

**Propósito**: Confirmar la línea base antes de tocar código. Sin dependencias nuevas (plan.md, Technical Context) ni inicialización de proyecto — el feature se agrega a dos repos ya existentes.

- [X] T001 Desde `pos-backend` (venv activo), ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` y confirmar que todo pasa en verde (quickstart.md, Paso 0) — línea base antes de cualquier cambio.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: Infraestructura que TODAS las historias de usuario necesitan — modelo de datos, migración, fixtures de test y el router vacío registrado en la app.

**⚠️ CRITICAL**: Ninguna historia de usuario puede empezar hasta que esta fase esté completa.

- [X] T002 Agregar la clase `UserInvitation` en `app/core/models.py` (schema `shared`, junto a `Tenant`/`Role`/`User`/`PasswordResetToken`), según [data-model.md](./data-model.md): `UUIDPrimaryKeyMixin` + `TimestampMixin`; columnas `tenant_id` (FK `shared.tenants.id`, indexada), `email` (`String(255)`, no nulo), `role_id` (FK `shared.roles.id`), `password_hash` (`String(255)`, no nulo), `status` (`String(10)`, no nulo, `server_default='pending'`), `sent_at` (`DateTime`, no nulo, `server_default=func.now()`), `consumed_at`/`cancelled_at` (`DateTime`, nulos). `__table_args__`: índice sobre `tenant_id`; índice único parcial sobre `(tenant_id, email)` `WHERE status = 'pending'` definido con `postgresql_where=text("status = 'pending'")` **y** `sqlite_where=text("status = 'pending'")` en el mismo `Index(...)` (research.md Decisión 3, mismo patrón que `idx_pending_payment_attempt_per_order` en `app/models/order_payment_attempt.py`); `CheckConstraint("status IN ('pending', 'consumed', 'cancelled')")`; `{"schema": "shared"}`. Relaciones de solo lectura `tenant: Mapped["Tenant"]` y `role: Mapped["Role"]` (`relationship()`, sin `back_populates`).
- [X] T003 [P] Crear la migración Alembic en `alembic/versions/` para la tabla `user_invitations` (schema `shared`): generarla con `alembic revision -m "user_invitations"` desde `pos-backend` (down_revision debe encadenar con el head actual, `5a77a91b482d`) y completar `upgrade()`/`downgrade()` siguiendo el patrón de `alembic/versions/d252a23e65a1_password_reset_tokens.py` — `op.create_table("user_invitations", ..., schema="shared")` con las mismas columnas de T002, `sa.ForeignKeyConstraint` hacia `shared.tenants.id` y `shared.roles.id`, índice sobre `tenant_id`, índice único parcial sobre `(tenant_id, email)` con `postgresql_where=sa.text("status = 'pending'")` (Postgres, sin `for_each_tenant_schema` — tabla vive en `shared`, research.md sección Migraciones), y `CheckConstraint` del status. `downgrade()` = `op.drop_table("user_invitations", schema="shared")`.
- [X] T004 [P] Extender `app/characterization_tests/auth_fixtures.py`: importar `UserInvitation` desde `app.core.models`, agregarlo a `_TABLE_NAMES`, y agregar el helper `make_invitation(db, tenant, role=None, password="temporal-123", **kw)` (mismo patrón que `make_user`/`make_password_reset_token`: crea un `Role` con `make_role(db)` si no se pasa uno, y aplica defaults con `kw.setdefault(...)` para `id`, `tenant_id=tenant.id`, `role_id=role.id`, `email=f"invite-{uuid.uuid4()}@example.com"`, `password_hash=generate_passwd_hash(password)`, `status="pending"`, `sent_at=datetime.now(timezone.utc).replace(tzinfo=None)`) — necesario para que los 4 módulos de test nuevos (US1-US4) puedan sembrar invitaciones.
- [X] T005 [P] Crear el paquete `app/api/v1/invitations/`: `__init__.py` (vacío), `schemas.py` con `InvitationCreate` (`email: EmailStr`, `role: RoleName` — importar `RoleName` desde `app.api.v1.users.schemas`, sin campo de contraseña, FR-001/FR-006) e `InvitationResponse` (`id: UUID`, `email: EmailStr`, `role_name: str`, `sent_at: datetime`, `model_config = ConfigDict(from_attributes=True)` — nunca incluye `password_hash`, ver [contracts/invitations-create.md](./contracts/invitations-create.md)), y `router.py` con `router = APIRouter(prefix="/invitations", tags=["invitations"])` vacío (los cuatro endpoints se agregan en las fases de cada historia).
- [X] T006 Registrar el router en `app/main.py`: importar `from app.api.v1.invitations.router import router as invitations_router` junto a los demás imports de routers, y agregar `app.include_router(invitations_router, prefix="/api/v1")` junto a los demás `include_router` (después de `users_router`).

**Checkpoint**: Modelo, migración, fixtures y router vacío listos — las historias de usuario pueden empezar.

---

## Phase 3: User Story 1 - El ADMIN invita a alguien por correo y rol (Priority: P1) 🎯 MVP

**Goal**: Un ADMIN crea una invitación pendiente dando solo correo y rol; no se crea ningún `User`; la contraseña temporal nunca se expone; `POST /users` (creación con contraseña directa) desaparece por completo (FR-004, "misma mitad indivisible", plan.md Principio VI).

**Independent Test**: abrir el formulario "Agregar usuario", verificar que no tiene campo de contraseña, y enviarlo con un correo sin invitación previa ni cuenta → se crea la invitación pendiente, no se crea ningún usuario, y la contraseña generada no aparece en ningún lado (spec.md, Acceptance Scenarios 1-3 de US1).

### Implementación para User Story 1

- [X] T007 [P] [US1] Extender `RESOURCE_CONFIG["usuarios"]` en `app/core/plan_limits.py` (research.md Decisión 5): modificar `_count_resource` (o el dataclass `_ResourceConfig`) para que, cuando el recurso sea `"usuarios"`, el conteo sume también `SELECT count(*) FROM user_invitations WHERE tenant_id = tenant.id AND status = 'pending'` (importar `UserInvitation` desde `app.core.models`) al conteo existente de `User`. Como `count_resource_usage` reutiliza `_count_resource`, `GET /plan` (spec 033) hereda el mismo conteo combinado sin cambios adicionales (efecto colateral documentado, no oculto).
- [X] T008 [P] [US1] Agregar `invitation_email_body(tenant_name: str, login_url: str, email: str, password: str) -> str` en `app/core/mail.py`, mismo estilo HTML que `welcome_email_body` (mismos tres datos: enlace, correo, contraseña temporal — FR-005).
- [X] T009 [US1] Implementar `POST /invitations` en `app/api/v1/invitations/router.py` (depende de T007, T008): dependencias `tenant: Tenant = Depends(get_tenant)`, `admin: User = Depends(require_tenant_admin)`, `db: Session = Depends(get_db)`. Normalizar el correo (`.strip().lower()`, FR-016). Orden de validación: 1) `enforce_plan_limit(db, tenant, "usuarios")` (403 si plan vencido o límite alcanzado, FR-018/Decisión 5/6); 2) `SELECT` de `User` por `tenant_id`+email normalizado sin filtrar `active` → 409 `"Ya existe un usuario con ese correo en el tenant"` si existe (FR-015); 3) buscar `Role` por `body.role.value` → 404 si no existe; 4) generar `password = generate_random_password()` y su hash; 5) construir `login_url` igual que `_build_login_url` de `app/api/v1/admin/router.py::create_tenant` (`https://{tenant.host}.skeilopos.com/login` en prod, `http://{tenant.host}.localhost:4200/login` en el resto, según `settings.ENVIRONMENT`); 6) agregar el `UserInvitation` pendiente a la sesión y hacer `db.flush()` dentro de un `try/except IntegrityError` — si el índice único parcial rechaza (invitación `pending` duplicada, carrera de dos ADMIN) → `db.rollback()` y `409` `"Ya existe una invitación pendiente para ese correo"` (research.md Decisión 3), **sin** enviar correo; 7) si el flush pasa, llamar `send_email(create_message([email_normalizado], f"Bienvenido a {tenant.name}", invitation_email_body(...)))` de forma síncrona dentro de un `try/except RuntimeError` — si falla, `db.rollback()` y `502` `"No se pudo enviar el correo de invitación. Intenta de nuevo."` (FR-012, research.md Decisión 4); 8) si el envío tiene éxito, `db.commit()`, `db.refresh(...)` y devolver `InvitationResponse` `201`. Ver [contracts/invitations-create.md](./contracts/invitations-create.md) para el mapeo completo de códigos de respuesta.
- [X] T010 [P] [US1] Eliminar el mecanismo de creación con contraseña directa (FR-004, research.md Decisión 11): quitar `create_user()` de `app/api/v1/users/router.py` (y el import de `UserCreate`/`generate_passwd_hash` si quedan sin otro uso en ese archivo), y quitar `UserCreate` de `app/api/v1/users/schemas.py`. No dejar ningún endpoint ni flag de compatibilidad — `GET /users`, `GET /users/{user_id}`, `PATCH /users/{user_id}/role`, `PATCH /users/{user_id}/status` quedan intactos.
- [X] T011 [US1] Crear `app/characterization_tests/test_invitations_create.py` (depende de T009, T010) cubriendo quickstart.md §US1 pasos 1-7: creación exitosa sin `User` nuevo y sin contraseña en la respuesta (Acceptance Scenarios 2-3); duplicado `pending` vía carrera simulada (dos inserciones sin esperar el commit de la primera) → solo una fila sobrevive, `409`; correo ya usado por un `User` activo y por uno inactivo → mismo `409` en ambos (Clarification 1, FR-015); `CASHIER` autenticado → `403`; tenant con `plan_vence_en` vencido → `403` sin llamar a `send_email`; tenant en el límite de plan solo por invitaciones `pending` (sin ningún `User` real) → `403` (caso que distingue el conteo extendido de T007); `send_email` mockeado con `RuntimeError` → `502` y ninguna fila de `UserInvitation` persistida. Mockear `app.api.v1.invitations.router.send_email` (`unittest.mock.patch`), usar `af.make_tenant`/`af.make_role`/`af.make_user`/`af.make_invitation`.

  ```bash
  python -m unittest app.characterization_tests.test_invitations_create -v
  ```

- [X] T012 [P] [US1] Actualizar `src/app/modules/users/interfaces/user-profile.interface.ts`: quitar `password` de `TenantUserForm`/`TenantUserCreatePayload` (o eliminar esos dos tipos por completo, FR-004) y agregar `InvitationForm { email: string; role: RoleName | ''; }`, `InvitationCreatePayload { email: string; role: RoleName; }`, `PendingInvitation { id: string; email: string; role_name: string; sent_at: string; }`.
- [X] T013 [US1] Crear `src/app/modules/users/services/invitations.service.ts` (depende de T012): `@Injectable({ providedIn: 'root' })` `InvitationsService`, `baseUrl = ${environment.apiBaseUrl}/invitations`, signals `error`/`isSubmitting` (mismo patrón que `UsersService`), y `async createInvitation(form: InvitationForm): Promise<boolean>` que valida `role !== ''`, hace `POST` con `InvitationCreatePayload`, y recarga nada por su cuenta (la recarga del listado de pendientes la dispara el componente, ver US3). Dejar el archivo listo para que US3/US4 le agreguen `loadPendingInvitations`/`resendInvitation`/`cancelInvitation` sin reescribir lo de aquí.
- [X] T014 [US1] Eliminar `src/app/modules/users/components/user-form.component.ts` y crear `src/app/modules/users/components/invitation-form.component.ts` (depende de T012, T013): componente standalone con un `FormGroup` de **exactamente** dos controles — `email` (`Validators.required, Validators.email`) y `role` (`Validators.required`, select ADMIN/CASHIER) — sin campos de nombre/teléfono/contraseña (FR-001), mismo patrón de `@Output() saved`/`@Output() cancelled` que el componente eliminado, llamando a `invitationsService.createInvitation(...)`.
- [X] T015 [P] [US1] Actualizar `src/app/modules/users/services/users.service.ts`: quitar el método `createUser()` y el import de `TenantUserCreatePayload`/`TenantUserForm` si quedan sin otro uso (FR-004) — `loadUsers`/`changeRole`/`toggleActive`/`getUser` sin cambios.
- [X] T016 [US1] Actualizar `src/app/modules/users/pages/users-page.component.ts` (depende de T014, T015): reemplazar el import/uso de `UserFormComponent` por `InvitationFormComponent` (`<app-invitation-form>` en vez de `<app-user-form>`), manteniendo `showForm`/`onUserSaved` y el resto de la pantalla (paginación, modal de rol, activar/desactivar) sin cambios.

**Checkpoint**: US1 completa y probable de forma independiente — el ADMIN invita, `POST /users` ya no existe.

---

## Phase 4: User Story 2 - La persona invitada activa su cuenta en su primer ingreso (Priority: P1)

**Goal**: `POST /auth/login` crea la cuenta real (con `must_change_password=True`) y consume la invitación en una sola operación cuando las credenciales coinciden con una invitación `pending`, sin tocar el resto del flujo de login existente.

**Independent Test**: crear una invitación (US1), tomar el correo y la contraseña temporal, autenticarse con ellos, y verificar que el sistema exige el cambio de contraseña antes de cualquier otra acción (spec.md, Acceptance Scenarios de US2).

### Implementación para User Story 2

- [X] T017 [US2] Modificar `login()` en `app/api/v1/auth/routes.py` (research.md Decisión 7/8, [contracts/auth-login.md](./contracts/auth-login.md)): cuando la búsqueda normal de `User` no encuentra ninguna fila **y** `tenant is not None`, antes de lanzar el `401` de siempre, buscar una `UserInvitation` con `status='pending'`, `tenant_id=tenant.id` y el correo normalizado (`.strip().lower()`) igual al recibido, bloqueada con `.with_for_update()`. Si existe y `verify_password(body.password, invitation.password_hash)` es `True`: crear el `User` (valores exactos en [data-model.md](./data-model.md#user-shareduser--sin-cambios-de-esquema): `name=email`, `email=email`, `password_hash=invitation.password_hash`, `phone=None`, `active=True`, `must_change_password=True`, `role_id=invitation.role_id`, `tenant_id=invitation.tenant_id`), fijar `invitation.status='consumed'` y `invitation.consumed_at=now()`, hacer `db.commit()`, y construir `user_data`/tokens exactamente como en el camino existente para un usuario recién leído (mismo `200`, mismo formato de respuesta). Si no hay invitación, o `verify_password` es `False`, o la invitación ya no está `pending` tras adquirir el lock: cae al mismo `401 "Invalid credentials"` que ya existe — sin crear nada. El resto de `login()` (resolución de tenant por host, verificación `active==True`, forma del JWT, manejo de errores de conexión) no se toca.
- [X] T018 [US2] Crear `app/characterization_tests/test_auth_login_invitation_consumption.py` (depende de T017) cubriendo quickstart.md §US2 pasos 1-5: login con la contraseña temporal de una invitación `pending` → `200`, `User` creado con `must_change_password=True` y `role_id` de la invitación, invitación queda `consumed` con `consumed_at` fijado, todo en una sola llamada (Acceptance Scenario 1); repetir el mismo login → `401`, sin segundo `User` (Acceptance Scenario 4, doble consumo); verificar que el `User` recién creado sigue el flujo ya existente de `must_change_password` sin tocar nada (reusar criterio de spec 031); login con la contraseña **original** de una invitación ya reenviada (crear vía `af.make_invitation` + sobrescribir `password_hash` simulando un reenvío) → `401`; simular login concurrente (dos llamadas con la misma invitación `pending`, sin dejar completar el commit de la primera antes de la segunda) → solo un `User` creado, la segunda ve `401`.

  ```bash
  python -m unittest app.characterization_tests.test_auth_login_invitation_consumption -v
  ```

**Checkpoint**: US1+US2 completas — el flujo de alta por invitación funciona de punta a punta (invitar → primer ingreso → cuenta activa).

---

## Phase 5: User Story 3 - El ADMIN distingue pendientes de activos en el listado (Priority: P2)

**Goal**: El listado de usuarios muestra, además de las cuentas activas, las invitaciones `pending` (correo, rol, fecha de envío) en una sección visualmente diferenciada.

**Independent Test**: crear invitaciones y consumir una (US2), y verificar en el listado que pendientes y activas se muestran diferenciadas con los datos correctos (spec.md, Acceptance Scenarios de US3).

### Implementación para User Story 3

- [X] T019 [US3] Implementar `GET /invitations` en `app/api/v1/invitations/router.py`: `admin: User = Depends(require_tenant_admin)`, `db: Session = Depends(get_db)`, query params `page`/`size` idénticos a `GET /users`; `SELECT` de `UserInvitation` con `selectinload(role)` filtrado por `tenant_id == admin.tenant_id` y `status == 'pending'`, ordenado por `sent_at desc`; devolver con el helper `paginate()` existente (`app.core.pagination`) como `Page[InvitationResponse]`. Ver [contracts/invitations-list.md](./contracts/invitations-list.md) — nunca incluye invitaciones de otro tenant (FR-013) ni ya consumidas/canceladas.
- [X] T020 [US3] Crear `app/characterization_tests/test_invitations_list.py` (depende de T019) cubriendo quickstart.md §US3 pasos 1-3: tenant con 2 `User` activos y 2 `UserInvitation` pendientes → `GET /users` sin cambios, `GET /invitations` devuelve las 2 pendientes con `email`/`role_name`/`sent_at`; consumir una (login, mismo patrón de T018) → desaparece de `GET /invitations`, aparece en `GET /users`; invitación de otro tenant → nunca aparece en el listado del tenant del ADMIN.

  ```bash
  python -m unittest app.characterization_tests.test_invitations_list -v
  ```

- [X] T021 [US3] Extender `src/app/modules/users/services/invitations.service.ts` (creado en T013) con `pendingInvitations`/`pendingLoading`/`pendingTotal`/`pendingPage`/`pendingSize`/`pendingTotalPages` (signals) y `async loadPendingInvitations(page = this.pendingPage(), size = this.pendingSize()): Promise<void>` que hace `GET` a `baseUrl` con esos params y llena los signals — mismo patrón que `UsersService.loadUsers`.
- [X] T022 [US3] Crear `src/app/modules/users/components/pending-invitations-list.component.ts` (depende de T021): componente standalone que renderiza la sección "Invitaciones pendientes" (correo, badge de rol reusando `ROLE_LABELS`/`ROLE_BADGE_CLASSES` de `users-page.component.ts`, fecha de envío formateada) — visualmente distinta de la lista de usuarios activos (fondo o borde diferenciado, sin barra de "activo/inactivo"), leyendo de `InvitationsService`.
- [X] T023 [US3] Integrar `PendingInvitationsListComponent` en `src/app/modules/users/pages/users-page.component.ts` (depende de T022): agregarlo como segunda sección debajo de la lista de usuarios activos, llamando `invitationsService.loadPendingInvitations()` en `ngOnInit` junto a `usersService.loadUsers()`.

**Checkpoint**: US1+US2+US3 completas — el ADMIN ve pendientes y activos diferenciados.

---

## Phase 6: User Story 4 - El ADMIN reenvía o cancela una invitación pendiente (Priority: P2)

**Goal**: El ADMIN puede reenviar (nueva contraseña temporal, invalida la anterior de inmediato) o cancelar (invalida de inmediato) una invitación `pending`.

**Independent Test**: crear una invitación, reenviarla y verificar que la contraseña anterior deja de servir y la nueva sí funciona; por separado, crear otra, cancelarla y verificar que su contraseña deja de servir (spec.md, Acceptance Scenarios de US4).

### Implementación para User Story 4

- [X] T024 [US4] Implementar `POST /invitations/{invitation_id}/resend` en `app/api/v1/invitations/router.py` (research.md Decisión 9/10, [contracts/invitations-resend.md](./contracts/invitations-resend.md)): cargar la `UserInvitation` con `.with_for_update()` filtrada por `id` y `tenant_id == admin.tenant_id` → `404` `"Invitación no encontrada"` si no existe en ese tenant; `409` `"La invitación ya no está pendiente"` si `status != 'pending'`; si sigue `pending`, generar una contraseña nueva y su hash, y llamar `send_email(...)` de forma síncrona **antes** de tocar la fila — si falla (`RuntimeError`), `db.rollback()` (o simplemente no escribir nada) y `502`, dejando `password_hash`/`sent_at` anteriores intactos (la contraseña vieja sigue sirviendo); si el envío tiene éxito, recién ahí actualizar `password_hash`/`sent_at` de la misma fila (`id`/`status`/`email` sin cambios) y `db.commit()`, devolviendo `InvitationResponse`.
- [X] T025 [US4] Implementar `POST /invitations/{invitation_id}/cancel` en `app/api/v1/invitations/router.py` ([contracts/invitations-cancel.md](./contracts/invitations-cancel.md)): cargar la `UserInvitation` con `.with_for_update()` filtrada por `id` y `tenant_id == admin.tenant_id` → `404` si no existe en ese tenant; `409` `"La invitación ya no está pendiente"` si `status != 'pending'`; si sigue `pending`, `status='cancelled'`, `cancelled_at=now()`, `db.commit()`, devolver `InvitationResponse`. Sin envío de correo.
- [X] T026 [US4] Crear `app/characterization_tests/test_invitations_resend_cancel.py` (depende de T024, T025) cubriendo quickstart.md §US4 pasos 1-5: reenvío exitoso → `password_hash`/`sent_at` cambian, `id`/`email`/`status` no; login con la contraseña nueva → `200`; login con la contraseña anterior → `401`; `send_email` mockeado para fallar en un reenvío → `502`, login con la contraseña anterior (antes de este intento) sigue funcionando; cancelación de una `pending` → `200`, `status='cancelled'`, ya no aparece en `GET /invitations`, login con su contraseña temporal → `401`; cancelar una ya `consumed` → `409`; carrera simulada login-vs-cancel (research.md Decisión 7, "la cancelación gana" si se compromete primero, si no el cancel posterior ve `409`).

  ```bash
  python -m unittest app.characterization_tests.test_invitations_resend_cancel -v
  ```

- [X] T027 [US4] Extender `src/app/modules/users/services/invitations.service.ts` (creado en T013, extendido en T021) con `async resendInvitation(id: string): Promise<void>` y `async cancelInvitation(id: string): Promise<void>` (`POST` a `${baseUrl}/${id}/resend`/`${baseUrl}/${id}/cancel`, recargando `loadPendingInvitations()` al terminar con éxito) — mismo patrón de manejo de error que `UsersService.toggleActive`.
- [X] T028 [US4] Agregar botones "Reenviar"/"Cancelar" por fila en `src/app/modules/users/components/pending-invitations-list.component.ts` (depende de T027), llamando a `invitationsService.resendInvitation`/`cancelInvitation`, con `confirm(...)` antes de cancelar (misma convención que `onToggleActive` en `users-page.component.ts`).

**Checkpoint**: Las 4 historias de usuario completas y funcionando de forma independiente.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Propósito**: Verificación de no regresión y validación manual de frontend — sin código nuevo.

- [ ] T029 [P] Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde `pos-backend` (quickstart.md, "Verificación de no regresión") y confirmar: `POST /users` ya no existe; `GET /users`, `PATCH /users/{id}/role`, `PATCH /users/{id}/status` siguen en verde; los tests de plan (spec 033, `mesas`/`cajas`/`productos`/`métodos de pago`) siguen en verde — solo cambió la rama `"usuarios"` de `RESOURCE_CONFIG`; `test_auth_session_revocation.py`/`test_auth_forgot_password.py`/`test_auth_reset_password.py`/`test_auth_change_password.py` (spec 031) siguen en verde.
- [ ] T030 [P] Validación manual en `pos-heladeria` vía `ng serve` siguiendo quickstart.md "Frontend — validación manual" pasos 1-6: formulario "Agregar usuario" con exactamente 2 controles; invitación aparece de inmediato en "Invitaciones pendientes"; CASHIER bloqueado por el guard de ruta existente (`roleGuard([UserRole.ADMIN])`, research.md Decisión 14, sin cambios de código); primer ingreso con la contraseña temporal fuerza cambio de contraseña y el correo pasa de pendiente a activo; reenviar invalida la contraseña anterior y actualiza la fecha; cancelar la retira de pendientes e invalida su contraseña.
- [ ] T031 Verificar SC-001 a SC-006 (spec.md, Success Criteria) contra la tabla de mapeo de quickstart.md "Verificación final" — confirmar que cada criterio quedó cubierto por los tests/pasos manuales de T011/T018/T020/T026/T030, sin código adicional.

---

## Dependencies & Execution Order

### Dependencias de fase

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario.
- **User Stories (Phase 3-6)**: todas dependen de Foundational. US1 y US2 son ambas P1 pero **US2 depende de que exista una invitación creada por US1** (el login solo puede consumir lo que `POST /invitations` crea) — en la práctica se implementan en ese orden aunque ambas sean P1. US3 y US4 (P2) dependen de que exista el router `invitations` de la Foundational, y en la práctica de que US1 ya haya poblado el archivo `invitations.service.ts`/`pending-invitations-list.component.ts` (US4 modifica el componente que crea US3).
- **Polish (Phase 7)**: depende de que las 4 historias estén completas.

### Dependencias entre historias

- **US1 (P1)**: depende solo de Foundational.
- **US2 (P1)**: depende de Foundational; su test (T018) necesita una invitación `pending` — reutiliza `af.make_invitation` (Foundational), no depende del código de US1 en tiempo de ejecución, pero sí es la mitad complementaria de US1 en el flujo real.
- **US3 (P2)**: depende de Foundational; en frontend, T021/T023 extienden archivos creados en US1 (T013, T016).
- **US4 (P2)**: depende de Foundational; en frontend, T027/T028 extienden archivos creados en US1/US3 (T013/T021, T022).

### Dentro de cada historia

- Backend antes que frontend dentro de la misma historia (el frontend consume el endpoint).
- Dentro de Phase 3 (US1): T007/T008 (helpers) antes de T009 (endpoint que los usa); T009/T010 pueden ir en paralelo entre sí (archivos distintos); T011 (test) después de T009/T010.

### Oportunidades de paralelismo

- Todas las tareas [P] de la Foundational (T003, T004, T005) pueden correr en paralelo una vez completado T002.
- T007/T008/T010 (US1, backend) pueden correr en paralelo entre sí.
- T012/T015 (US1, frontend) pueden correr en paralelo entre sí y con las tareas backend de US1.
- T020 (test US3) y T021 (frontend US3) pueden correr en paralelo una vez completado T019.
- T029/T030 (Polish) pueden correr en paralelo.

---

## Parallel Example: Foundational (Phase 2)

```bash
# Tras completar T002 (modelo UserInvitation), lanzar juntas:
Task: "Crear la migración Alembic para user_invitations en alembic/versions/"
Task: "Extender auth_fixtures.py con UserInvitation y make_invitation(...)"
Task: "Crear el paquete app/api/v1/invitations/ (schemas.py + router.py vacío)"
```

## Parallel Example: User Story 1 (Phase 3, backend)

```bash
# En paralelo, antes de implementar POST /invitations:
Task: "Extender RESOURCE_CONFIG['usuarios'] en app/core/plan_limits.py"
Task: "Agregar invitation_email_body() en app/core/mail.py"
Task: "Eliminar create_user()/UserCreate (app/api/v1/users/router.py y schemas.py)"
```

---

## Implementation Strategy

### MVP primero (User Story 1 + User Story 2)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Phase 3: User Story 1 (crear invitación)
4. Completar Phase 4: User Story 2 (consumir invitación en el primer ingreso)
5. **DETENER Y VALIDAR**: con US1+US2 el flujo de alta de usuario ya funciona de punta a punta sin `POST /users` — es el reemplazo mínimo viable del mecanismo actual.

### Entrega incremental

1. Setup + Foundational → base lista.
2. US1 → el ADMIN invita (sin poder ver el listado de pendientes ni reenviar/cancelar todavía).
3. US2 → la persona invitada puede activar su cuenta — MVP funcional de punta a punta.
4. US3 → el ADMIN gana visibilidad del estado de sus invitaciones.
5. US4 → el ADMIN gana control (reenviar/cancelar) sobre invitaciones ya creadas.
6. Polish → verificación de no regresión y validación manual completa.

### Notas

- Sin `TestClient`: todos los tests llaman las funciones de router directamente (`app/api/v1/invitations/router.py`, `app/api/v1/auth/routes.py::login`), mismo criterio que spec 031 (research.md, sección Testing).
- `send_email` se mockea siempre con `unittest.mock.patch` — ningún test golpea la red real.
- No se crea ningún archivo `service.py` en `pos-backend` para `invitations` — toda la lógica vive en `router.py`, mismo patrón sin capa de servicio que ya usa `users` (Principio V, plan.md Structure Decision).
