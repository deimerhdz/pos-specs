# Tasks: Recuperación y Cambio de Contraseña (Personal)

**Input**: Design documents from `/specs/031-recuperacion-cambio-contrasena/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Incluidos — `plan.md` (Technical Context §Testing) y `quickstart.md` especifican
explícitamente los ficheros de test nuevos por historia de usuario (`unittest`, sin `pytest`/
`TestClient`, llamando las funciones de `routes.py` directamente).

**Organización**: Tareas agrupadas por historia de usuario para permitir implementación y prueba
independientes de cada una. Rutas relativas a la raíz de `pos-specs` (`../pos-backend`,
`../pos-heladeria`, repos sibling — Constitución §Alcance).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: Historia de usuario a la que pertenece (US1, US2, US3)

---

## Phase 1: Setup (infraestructura compartida)

**Propósito**: configuración y fixtures de test compartidas por todas las historias, sin tocar
comportamiento de negocio.

- [ ] T001 [P] Agregar `PASSWORD_RESET_MAX_REQUESTS` (default `3`), `PASSWORD_RESET_WINDOW_SECONDS`
  (default `900`) y `PASSWORD_RESET_TOKEN_EXPIRY_MINUTES` (default `30`) en
  `../pos-backend/app/core/config.py`, mismo patrón que las variables `RATE_LIMIT_*` ya existentes
  (research.md Decisión 3).
- [ ] T002 [P] Crear `../pos-backend/app/characterization_tests/auth_fixtures.py`: extiende el
  truco de `schema_translate_map` de `fixtures.py` (líneas 63-84) para colapsar **también** el
  schema `shared` (además de `tenant`) a `None` sobre SQLite en memoria, y crea las tablas
  `Tenant`/`Role`/`User`/`PasswordResetToken` (research.md Decisión 10). No depende de T003/T004
  para escribirse, pero sus `create_all` deben incluir las columnas/tabla nuevas una vez existan.

**Checkpoint**: infraestructura de configuración y fixtures lista para que Foundational y las
historias de usuario la usen.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: mecanismo de cierre de sesiones y demás infraestructura que **ambas** historias
US1 y US2 necesitan para cumplir FR-009/FR-017. No cambia ningún comportamiento existente por sí
sola (nadie fija `tokens_valid_after` todavía).

**⚠️ CRÍTICO**: ninguna historia de usuario puede darse por completa sin esta fase.

- [ ] T003 Agregar el modelo `PasswordResetToken` nuevo en `../pos-backend/app/core/models.py`
  (schema `shared`, `UUIDPrimaryKeyMixin`, `user_id` FK a `shared.users.id` indexado, `token_hash`
  `String(64)` único e indexado, `email_snapshot` `String(255)`, `issued_at`/`expires_at`
  `DateTime`, `used_at`/`invalidated_at` `DateTime` nulables) — ver
  [data-model.md](./data-model.md#passwordresettoken-password_reset_tokens--nueva).
- [ ] T004 Agregar la columna `tokens_valid_after: DateTime | None` (nulable, sin default) al
  modelo `User` en `../pos-backend/app/core/models.py` — ver
  [data-model.md](./data-model.md#user-sharedusers--modificada).
- [ ] T005 Crear la migración Alembic nueva en
  `../pos-backend/alembic/versions/{rev}_password_reset_tokens.py`: columna `users.tokens_valid_after`
  y tabla `password_reset_tokens`, ambas en schema `shared`, **sin** `@for_each_tenant_schema`
  (mismo patrón que `alembic/versions/c2d3e4f5a6b7_tenant_logo_url.py`). Rollback:
  `op.drop_column`/`op.drop_table` (research.md, sección final "Migraciones"). Depende de T003, T004.
- [ ] T006 [P] Agregar `enforce_sliding_window(key, limit, window_seconds)` nueva en
  `../pos-backend/app/core/rate_limit.py`: `ZADD`+`ZREMRANGEBYSCORE`+`ZCARD`+`EXPIRE` sobre el
  cliente Redis ya importado (`token_blocklist`), fail-open si Redis no responde (mismo criterio
  que `enforce()`) — research.md Decisión 3. No modifica `enforce()` existente.
- [ ] T007 [P] Agregar `password_changed_email_body(when, email)` nueva en
  `../pos-backend/app/core/mail.py`, mismo estilo HTML inline que `welcome_email_body` (aviso de
  cambio de contraseña con fecha/hora, FR-022) — research.md Decisión 8. Usada por US1 y US2.
- [ ] T008 [P] Agregar el chequeo `token_data["iat"] < user.tokens_valid_after` (→ `401` con detalle
  distinguible, p. ej. `"Session revoked due to password change"`) en `get_current_user` y
  `get_authenticated_user`, `../pos-backend/app/core/dependencies.py`, **después** de la relectura
  por id y el chequeo `active==True` ya existentes (`RN-AUTH-07`, protegido A-23 — no se modifica
  ese chequeo, solo se agrega el nuevo a continuación). Depende de T004.
- [ ] T009 [P] Agregar el mismo chequeo en el handler de `GET /auth/refresh-token`,
  `../pos-backend/app/api/v1/auth/routes.py` (líneas 111-150 aprox.), después de su relectura por
  id + `active==True` ya existentes. Depende de T004.
- [ ] T010 Crear `../pos-backend/app/characterization_tests/test_auth_session_revocation.py` con
  los pasos 1-3 de quickstart.md §"US1 + US2 — Cierre de sesiones": (1) **no-regresión A-23**:
  cuenta `active=False`, `tokens_valid_after=None` → `refresh-token` sigue devolviendo
  `401 "User not found or inactive"` exactamente igual que hoy — este caso debe quedar en verde
  antes de continuar con el resto de la fase; (2) login → fijar manualmente
  `user.tokens_valid_after = now() + 1s` → `get_current_user`/`get_authenticated_user` con el
  access emitido → `401` con el detalle nuevo (distinto de "revoked"/"inactive"); (3) mismo
  escenario sobre `refresh-token`. Depende de T003, T004, T008, T009.

**Checkpoint**: mecanismo de cierre de sesiones verificado e inerte — listo para que US1 y US2 lo
activen fijando `tokens_valid_after` tras un cambio exitoso.

---

## Phase 3: User Story 1 - Recuperar el acceso cuando se olvida la contraseña (Priority: P1) 🎯 MVP

**Goal**: un usuario que olvidó su contraseña pide un enlace desde el login, lo abre, define una
contraseña nueva, y entra con ella — sin depender de US2.

**Independent Test**: pedir un enlace con un correo registrado, abrirlo, definir contraseña nueva,
confirmar que el login funciona con la nueva y falla con la anterior (spec.md, US1).

### Backend

- [ ] T011 [P] [US1] Agregar `ForgotPasswordRequest` (`email: EmailStr`) y `ResetPasswordRequest`
  (`token: str`, `new_password: Field(min_length=8, max_length=12)`) en
  `../pos-backend/app/api/v1/auth/schemas.py` — [contracts/forgot-password.md](./contracts/forgot-password.md),
  [contracts/reset-password.md](./contracts/reset-password.md).
- [ ] T012 [P] [US1] Agregar `password_reset_email_body(reset_url, expiry_minutes)` nueva en
  `../pos-backend/app/core/mail.py` (research.md Decisión 8). URL construida igual que `login_url`
  en `app/api/v1/admin/router.py:31-34` (research.md Decisión 7):
  `https://{tenant.host}.skeilopos.com/reset-password?token=...` en producción,
  `http://{tenant.host}.localhost:4200/reset-password?token=...` en desarrollo.
- [ ] T013 [US1] Implementar `POST /auth/forgot-password` en
  `../pos-backend/app/api/v1/auth/routes.py`: resolver tenant por `x-tenant-host` (`400` si falta);
  `enforce_sliding_window` sobre `rl:pwreset:{tenant_id}:{email_normalizado}` (`429` sin tocar BD
  ni enviar correo si excede 3/15min); si no bloqueado, buscar `User` activo en ese tenant —
  si existe: invalidar cualquier `PasswordResetToken` vigente de la cuenta, crear la fila nueva
  (`token_hash` = SHA-256 de `secrets.token_urlsafe(32)`, `email_snapshot`, `issued_at`,
  `expires_at = issued_at + PASSWORD_RESET_TOKEN_EXPIRY_MINUTES`), despachar
  `send_email_task.delay(...)` con `password_reset_email_body`; si no existe/inactiva/otro tenant:
  no crear fila ni enviar correo; responder siempre el mismo `200` con el mensaje genérico
  ([contracts/forgot-password.md](./contracts/forgot-password.md)). Depende de T005, T006, T011, T012.
- [ ] T014 [US1] Implementar `GET /auth/reset-password/validate` en
  `../pos-backend/app/api/v1/auth/routes.py`: sin efecto secundario (no consume el token);
  `200 {"valid": true}` si vigente; `400/404 {"valid": false, "reason": "expired"|"used"|"invalid"}`
  según el estado derivado de [data-model.md](./data-model.md#estado-derivado-no-persistido-como-columna--se-deriva-al-leer-igual-que-pendiente-de-pago-en-spec-024)
  ([contracts/reset-password.md](./contracts/reset-password.md)). Depende de T005. Mismo fichero
  que T013 — secuencial, no paralelo.
- [ ] T015 [US1] Implementar `POST /auth/reset-password` en
  `../pos-backend/app/api/v1/auth/routes.py`: `SELECT ... WHERE token_hash=:hash WITH FOR UPDATE`
  (research.md Decisión 5); re-validar con las mismas reglas de `.../validate` (`400`/`404` con
  `reason` si no vigente, sin aplicar cambios); `new_password` distinta de la actual
  (`verify_password(new_password, user.password_hash)` debe ser `False` → `400` si es igual,
  FR-021); si todo pasa: `password_hash = generate_passwd_hash(new_password)`,
  `must_change_password = False` (FR-009, `RN-AUTH-02`), `tokens_valid_after = now()` (FR-009),
  marcar `used_at = now()` en el token, `db.commit()`; despachar
  `send_email_task.delay(...)` con `password_changed_email_body` envuelto en `try/except`
  (FR-022/FR-028); responder `200` ([contracts/reset-password.md](./contracts/reset-password.md)).
  Depende de T007, T014 (mismo fichero, secuencial).
- [ ] T016 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_auth_forgot_password.py`
  cubriendo quickstart.md §US1 pasos 1-4 y 8: correo registrado → `200` + mock de `send_email_task`
  llamado con el link; correo inexistente → mismo `200`, mock sin llamadas (SC-003); cuenta
  `active=False` → mismo trato; cuenta en otro tenant → mismo trato; 3 solicitudes en ventana +
  4ª bloqueada (`429`, sin correo) + 5ª tras salir de ventana → `200` normal (ventana deslizante
  genuina, no fija — research.md Decisión 3). Depende de T013.
- [ ] T017 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_auth_reset_password.py`
  cubriendo quickstart.md §US1 pasos 5-7: `.../validate` vigente/caducado por reloj mockeado;
  `POST` exitoso deja `tokens_valid_after` fijado, `must_change_password=False`, login con
  contraseña anterior falla y con la nueva funciona; segunda confirmación del mismo enlace (doble
  clic) → `400 reason="used"` sin segundo cambio; enlace superseded por uno más nuevo →
  `reason="invalid"` en el anterior; `new_password` con espacios al inicio/final y con
  tildes/eñes → login solo funciona con la contraseña exacta, sin recortar ni alterar caracteres
  (FR-025, FR-026); mutar `user.email` después de emitir un token vigente → `.../validate`
  devuelve `reason="invalid"` (FR-012). Depende de T014, T015.

### Frontend

- [ ] T018 [P] [US1] Agregar `ForgotPasswordRequest`, `ResetPasswordRequest`,
  `ValidateResetTokenResponse` en `../pos-heladeria/src/app/core/auth/auth.models.ts`.
- [ ] T019 [US1] Agregar `forgotPassword()`, `validateResetToken()`, `resetPassword()` en
  `../pos-heladeria/src/app/core/auth/auth-api.service.ts`. Depende de T018.
- [ ] T020 [US1] Agregar `forgotPassword()`/`resetPassword()` en
  `../pos-heladeria/src/app/core/services/auth.service.ts`. Depende de T019.
- [ ] T021 [P] [US1] Crear `../pos-heladeria/src/app/modules/auth/pages/forgot-password.component.ts`
  (campo "Correo electrónico" + botón "Enviar enlace", mensaje genérico de FR-003, funcional en
  móvil FR-024). Depende de T020.
- [ ] T022 [P] [US1] Crear `../pos-heladeria/src/app/modules/auth/pages/reset-password.component.ts`
  (valida el token al cargar vía `GET .../validate` sin consumirlo, limpia cualquier sesión
  existente antes de mostrar el formulario FR-006, campos "Nueva contraseña"/"Confirmar nueva
  contraseña" reusando el patrón de validador cruzado de `change-password.component.ts`, muestra
  error con botón "Pedir un enlace nuevo" si inválido/caducado/usado; funcional en móvil FR-024).
  Depende de T020.
- [ ] T023 [P] [US1] Agregar el enlace "Restablecer contraseña" bajo el campo de contraseña en
  `../pos-heladeria/src/app/modules/auth/pages/login.component.ts`, visible sin scroll a 375px de
  ancho (FR-001).
- [ ] T024 [US1] Agregar las rutas `forgot-password` (guard `redirectIfAuthGuard`, igual que
  `login`) y `reset-password` (pública, sin guard) en `../pos-heladeria/src/app/app.routes.ts`.
  Depende de T021, T022, T023.

**Checkpoint**: User Story 1 completa y verificable de forma independiente (MVP).

---

## Phase 4: User Story 2 - Cambiar la contraseña voluntariamente desde Ajustes de cuenta (Priority: P2)

**Goal**: un usuario con sesión activa cambia su contraseña desde Ajustes de cuenta sin afectar el
resto de su perfil, sin depender de US1.

**Independent Test**: con sesión iniciada, cambiar la contraseña actual por una nueva válida desde
Ajustes de cuenta, confirmar mensaje de éxito y que el login posterior funciona con la nueva
(spec.md, US2).

### Backend

- [ ] T025 [US2] Cambiar `ChangePasswordRequest.new_password` de
  `Field(min_length=6, max_length=128)` a `Field(min_length=8, max_length=12)` en
  `../pos-backend/app/api/v1/auth/schemas.py` (FR-019, "Cambio de comportamiento explícito #1").
- [ ] T026 [US2] Modificar `POST /auth/change-password` en
  `../pos-backend/app/api/v1/auth/routes.py:92-108`: mantener sin cambios la verificación de
  `current_password` (`RN-AUTH-01`) y la limpieza de `must_change_password` (`RN-AUTH-02`);
  agregar la verificación de `new_password` distinta de la actual (`400` si igual, FR-021);
  agregar `user.tokens_valid_after = now()` (FR-017); agregar el despacho de
  `send_email_task.delay(...)` con `password_changed_email_body`, envuelto en `try/except`
  (FR-022/FR-028) — [contracts/change-password.md](./contracts/change-password.md). Depende de
  T007, T025.
- [ ] T027 [P] [US2] Crear `../pos-backend/app/characterization_tests/test_auth_change_password.py`
  cubriendo quickstart.md §US2: éxito con `tokens_valid_after` fijado; `current_password`
  incorrecta → `400`, hash sin cambios (`RN-AUTH-01` intacto); `new_password` igual a la actual →
  `400` (FR-021); `new_password` de 13 caracteres → `422` (antes válida, FR-019); el endpoint no
  modifica `name`/`email`/`phone`/`role_id` (SC-007). Depende de T026.
- [ ] T028 [US2] Completar `../pos-backend/app/characterization_tests/test_auth_session_revocation.py`
  (mismo fichero de T010) con el paso 4 de quickstart.md: login → `POST /auth/change-password`
  exitoso → re-login con la contraseña nueva (mismo patrón que `AuthService.changePassword()`) →
  el token del segundo login pasa `get_current_user` sin problema; el token del primer login
  (previo al cambio) ya no sirve, verificable con `GET /auth/refresh-token` (SC-005). Depende de
  T010, T026.

### Frontend

- [ ] T029 [P] [US2] Cambiar `Validators.minLength(8)`/`maxLength(12)` (antes 6/128) en
  `../pos-heladeria/src/app/modules/auth/pages/change-password.component.ts` (FR-019). Solo el
  rango cambia.
- [ ] T030 [P] [US2] Crear
  `../pos-heladeria/src/app/modules/account/pages/account-settings.component.ts` ("Ajustes de
  cuenta", sección "Cambiar contraseña" con los campos "Contraseña actual"/"Nueva contraseña"/
  "Confirmar nueva contraseña" y botón "Actualizar contraseña", FR-013/FR-014; solo actualiza la
  contraseña sin tocar otros campos del perfil aunque estén editados sin guardar, FR-015;
  accesible a cajero y admin, **sin** `roleGuard([UserRole.ADMIN])` — research.md Decisión 6;
  funcional en móvil FR-024).
- [ ] T031 [P] [US2] Agregar la opción "Cambiar contraseña" en el dropdown de usuario de
  `../pos-heladeria/src/app/modules/dashboard/layout/header.component.ts`, junto a "Cerrar
  sesión".
- [ ] T032 [US2] Agregar la ruta `dashboard/mi-cuenta` (sin `roleGuard`) en
  `../pos-heladeria/src/app/modules/dashboard/routes.ts`. Depende de T030.

**Checkpoint**: User Story 1 y 2 funcionan de forma independiente.

---

## Phase 5: User Story 3 - Recibir aviso de seguridad tras un cambio de contraseña (Priority: P3)

**Goal**: tras cualquier cambio exitoso por cualquiera de los dos flujos, el titular recibe un
correo de aviso.

**Independent Test**: disparar un cambio de contraseña exitoso por cualquiera de los dos flujos y
confirmar que llega el correo de aviso (spec.md, US3). El envío ya está implementado dentro de los
flujos de US1 (T015) y US2 (T026) según sus contratos — esta fase agrega la verificación explícita
por FR/Acceptance Scenario que `plan.md` asigna a US3, sin pantalla ni endpoint propios.

- [ ] T033 [P] [US3] Agregar a
  `../pos-backend/app/characterization_tests/test_auth_reset_password.py` (mismo fichero de T017)
  la verificación de que un `POST /auth/reset-password` exitoso despacha **dos** llamadas a
  `send_email_task` (correo del enlace al pedirlo + correo de aviso al guardar, con asuntos
  distintos, FR-022, Acceptance Scenario 1 de US3) y que si `send_email_task.delay` lanza una
  excepción el endpoint sigue respondiendo `200` normal (FR-028). Depende de T015, T017.
- [ ] T034 [P] [US3] Agregar a
  `../pos-backend/app/characterization_tests/test_auth_change_password.py` (mismo fichero de T027)
  la verificación de que un `POST /auth/change-password` exitoso despacha **una** llamada a
  `send_email_task` con el correo de aviso (FR-022, Acceptance Scenario 2 de US3) y que un fallo
  del envío no bloquea la respuesta `200` (FR-028). Depende de T026, T027.

**Checkpoint**: las tres historias de usuario quedan verificadas de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Propósito**: cobertura de Vitest en el frontend y verificación final end-to-end de todo `spec.md`.

- [ ] T035 [P] Extender `../pos-heladeria/src/app/core/auth/auth-api.service.spec.ts` con specs
  para `forgotPassword()`, `validateResetToken()`, `resetPassword()`.
- [ ] T036 [P] Extender `../pos-heladeria/src/app/core/services/auth.service.spec.ts` con specs
  para `forgotPassword()`/`resetPassword()`.
- [ ] T037 Ejecutar la suite completa de backend
  (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde
  `../pos-backend`) y confirmar que `POST /auth/login` (`RN-AUTH-03/04/05`), `GET /auth/logout`
  (`RN-AUTH-08`) y `RN-AUTH-07`/A-23 siguen comportándose igual que antes de esta spec
  (quickstart.md §"Verificación de no regresión"). Depende de T001-T034 completas.
- [ ] T038 Validación manual de frontend según quickstart.md §"Frontend — validación manual":
  login en 375px sin scroll (FR-001); Flujo A completo en navegador (pedir enlace → abrir → definir
  contraseña → login); sesión existente limpiada al abrir un enlace de reset en la misma pestaña
  (FR-006, verificar `localStorage`); Flujo B completo con la sesión de origen intacta mientras
  otra pestaña pierde acceso al recargar (SC-005). Depende de T024, T032.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — bloquea todas las historias de usuario.
- **User Story 1 (Phase 3)**: depende de Foundational. Sin dependencia de US2/US3.
- **User Story 2 (Phase 4)**: depende de Foundational. Sin dependencia de US1 (usa
  `password_changed_email_body` de T007, ya en Foundational).
- **User Story 3 (Phase 5)**: depende de que **al menos** T015 (US1) o T026 (US2) exista, porque su
  propio "Independent Test" (spec.md) requiere disparar un cambio exitoso por alguno de los dos
  flujos. En la práctica, sus dos tareas son independientes entre sí (T033 solo depende de US1,
  T034 solo depende de US2).
- **Polish (Phase 6)**: depende de las historias que se quieran cubrir (T037/T038 asumen las tres
  completas).

### Dentro de cada fase

- Foundational: T003→T004→T005 secuencial (mismo fichero `models.py` y su migración); T006, T007,
  T008, T009 en paralelo entre sí; T010 al final (depende de T003/T004/T008/T009).
- US1: T011/T012 en paralelo → T013→T014→T015 secuencial (mismo fichero `routes.py`) → T016/T017
  en paralelo. Frontend: T018→T019→T020 secuencial → T021/T022/T023 en paralelo → T024 al final.
- US2: T025→T026 secuencial (mismo fichero) → T027/T028 después. Frontend: T029/T030/T031 en
  paralelo → T032 al final.
- US3: T033/T034 en paralelo entre sí.

### Parallel Opportunities

- Setup: T001, T002.
- Foundational: T006, T007, T008, T009.
- US1 backend: T011, T012 — luego T016, T017.
- US1 frontend: T021, T022, T023.
- US2 backend: T027 (tras T026).
- US2 frontend: T029, T030, T031.
- US3: T033, T034.
- Polish: T035, T036.
- Una vez completa Foundational, US1 y US2 pueden trabajarse en paralelo por desarrolladores
  distintos (comparten solo `routes.py`/`schemas.py` como ficheros de posible conflicto — T013-T015
  y T026 tocan secciones distintas del mismo `routes.py`, coordinar el orden de merge).

---

## Parallel Example: User Story 1 (backend)

```bash
# Tras completar Foundational:
Task: "ForgotPasswordRequest/ResetPasswordRequest en app/api/v1/auth/schemas.py"
Task: "password_reset_email_body() en app/core/mail.py"

# Tras completar routes.py (T013-T015):
Task: "test_auth_forgot_password.py"
Task: "test_auth_reset_password.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup.
2. Completar Phase 2: Foundational (crítico — bloquea todo).
3. Completar Phase 3: User Story 1.
4. **Detener y validar**: correr `test_auth_forgot_password.py`/`test_auth_reset_password.py`,
   validación manual del Flujo A en navegador.
5. Desplegar/demo si está listo — ya resuelve el problema central del spec (cuenta bloqueada sin
   recuperación).

### Entrega incremental

1. Setup + Foundational → base lista.
2. + User Story 1 → probar independientemente → demo (MVP).
3. + User Story 2 → probar independientemente → demo.
4. + User Story 3 → probar independientemente → demo.
5. Polish (Vitest + verificación final SC-001 a SC-008).

---

## Notes

- `[P]` = ficheros distintos, sin dependencias pendientes entre sí.
- `[Story]` mapea cada tarea a su historia de usuario para trazabilidad (Constitución Principio
  XII).
- El chequeo `RN-AUTH-07`/A-23 (protegido) no se modifica en ningún momento — T008/T009 agregan el
  chequeo de `tokens_valid_after` **después**, nunca en su lugar; T010 lo verifica explícitamente
  antes de que ninguna historia empiece a fijar `tokens_valid_after`.
- Ningún endpoint nuevo introduce una dependencia nueva (Principio IX) — todo usa
  `bcrypt`/`PyJWT`/`redis.asyncio`/Celery ya presentes.
- Commitear tras cada tarea o grupo lógico; detenerse en cada checkpoint para validar la historia
  de forma independiente.
