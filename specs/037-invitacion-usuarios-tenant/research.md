# Research: Alta de usuarios internos por invitación

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 2 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión 2026-08-25) y el resto de incógnitas era puramente
técnico, resuelto leyendo directamente `pos-backend`/`pos-heladeria`. Este documento registra las
decisiones de diseño y las alternativas descartadas.

## Decisión 1 — Nueva entidad `UserInvitation` en el schema `shared`, sin `@for_each_tenant_schema`

- **Decisión**: tabla nueva `user_invitations` (`app/core/models.py`, `{"schema": "shared"}`), una
  sola migración Alembic sin recorrer schemas por tenant.
- **Rationale**: mismo criterio que `password_reset_tokens` (spec 031) y `tenants.logo_url` — la
  invitación referencia `Tenant`/`Role`, que viven en `shared`, no en el schema `tenant` per-tenant.
- **Alternatives considered**: ninguna razonable — poner la tabla en el schema `tenant` obligaría a
  duplicar FKs hacia `shared.tenants`/`shared.roles` desde cada schema, algo que ninguna entidad de
  auth hace hoy.

## Decisión 2 — Una fila por invitación, sobrescrita en el reenvío (no un histórico de tokens)

- **Decisión**: a diferencia de `PasswordResetToken` (que crea una fila nueva por cada solicitud y
  usa `invalidated_at` para superseding), `UserInvitation` es **una fila por correo+tenant**
  mientras esté vigente. Reenviar (FR-010) actualiza `password_hash` y `sent_at` de la misma fila;
  no crea una fila nueva ni conserva la contraseña anterior en ningún lado.
- **Rationale**: el Key Entity del spec describe la invitación como un objeto con **un** ciclo de
  vida (pendiente → consumida/cancelada) y **una** "contraseña temporal vigente asociada", sin
  pedir histórico de reenvíos. `PasswordResetToken` necesitaba múltiples filas porque su regla de
  negocio distingue "vigente" de "invalidado por uno posterior" como estados de filas separadas
  (para poder auditar cuál enlace se abrió); esta spec no exige eso, y agregar esa complejidad sin
  que ninguna FR la pida violaría el Principio V.
- **Alternatives considered**: modelar igual que `PasswordResetToken` (tabla de tokens con
  `invalidated_at`) — descartado, complejidad no solicitada por ninguna FR de esta spec (FR-010
  solo exige que la contraseña anterior "deje de servir de inmediato", que una sobrescritura
  cumple igual de bien con menos estado).

## Decisión 3 — A lo sumo una invitación `pending` por (tenant, correo): índice único parcial

- **Decisión**: índice único parcial sobre `(tenant_id, email)` `WHERE status = 'pending'`, mismo
  patrón que `idx_pending_payment_attempt_per_order` de `OrderPaymentAttempt` (spec 024). Se define
  con `postgresql_where=...` **y** `sqlite_where=...` en el mismo `Index(...)`, para que el mismo
  modelo se comporte igual en producción (Postgres, vía Alembic) y en los characterization tests
  (SQLite en memoria, `auth_fixtures.py`) — sin esto, SQLite crearía un índice único *no* parcial y
  un test que cancela una invitación y luego invita de nuevo el mismo correo fallaría en el test
  aunque el comportamiento real en Postgres fuera correcto.
- **Rationale**: resuelve al nivel de base de datos, de forma atómica, el Edge Case "si dos ADMIN
  intentan invitar el mismo correo casi al mismo tiempo, solo una invitación queda creada" (FR-015)
  — el segundo `INSERT` falla con `IntegrityError`, que el endpoint traduce a `409`. Evita depender
  de un `SELECT` previo + `INSERT` (ventana de carrera) o de un `SELECT ... FOR UPDATE` sobre una
  fila que todavía no existe (no se puede lockear lo que no existe).
- **Alternatives considered**: `SELECT` de una invitación pendiente antes de insertar, sin índice —
  descartado, dos requests concurrentes pueden pasar ambas el `SELECT` antes de que cualquiera
  haga `commit` (la misma clase de condición de carrera que `enforce_plan_limit` ya evita con
  `FOR UPDATE` sobre `Tenant`, que aquí no aplica porque no hay una fila de invitación previa que
  lockear).

## Decisión 4 — Envío de correo de invitación: **síncrono** (`send_email()`), no `send_email_task.delay()`

- **Decisión**: crear/reenviar una invitación llama a `app/core/utils.py::send_email()` de forma
  **síncrona**, dentro de la misma transacción de base de datos, antes de hacer `commit()`. Si
  `send_email()` lanza `RuntimeError` (fallo de la API de correo), se hace `db.rollback()` y el
  endpoint responde un error explícito al ADMIN (`502` o `500`, ver
  [contracts/invitations-create.md](./contracts/invitations-create.md)); si tiene éxito, se hace
  `commit()`.
- **Rationale**: es una desviación deliberada del patrón ya establecido en el resto del backend
  (correo de bienvenida en `admin/router.py`, correo de recuperación/aviso en spec 031) — todos
  esos usan `send_email_task.delay(...)` (Celery, fire-and-forget) envuelto en `try/except` que
  solo loguea, precisamente para que un proveedor de correo caído **nunca** bloquee ni falle la
  respuesta HTTP. Esta spec exige exactamente lo contrario: **FR-012** dice textualmente que "si el
  envío... falla, la invitación no debe quedar en un estado utilizable... y el ADMIN debe ver un
  mensaje de error explícito; el sistema nunca debe confirmar un envío exitoso cuando el correo no
  salió". Cumplir eso con Celery exigiría un mecanismo de confirmación de entrega asíncrono (colas,
  callbacks, polling) que ninguna FR pide y que la propia spec descarta explícitamente
  (Assumptions: "envío fallido" se refiere solo a errores síncronos al despachar el mensaje). El
  envío síncrono ya existe en el repo (`send_email()`, con `timeout=10.0`) — no es código nuevo,
  solo un caller distinto del ya existente.
- **Alternatives considered**: `send_email_task.delay(...)` + un job/callback que marque la
  invitación como fallida si el envío asíncrono falla — descartado, agrega un mecanismo de estado
  adicional (job de reconciliación, webhook del proveedor) no pedido por ninguna FR y that spec 
  Assumptions descarta explícitamente ("no se verifica rebotes asíncronos"); bloquear la respuesta
  HTTP hasta 10s (el timeout de `send_email()`) es aceptable aquí porque, a diferencia del login o
  el checkout, crear una invitación es una acción administrativa de baja frecuencia sin
  requisito de latencia en el spec.

## Decisión 5 — `enforce_plan_limit("usuarios")` se extiende para contar invitaciones `pending`

- **Decisión**: `RESOURCE_CONFIG["usuarios"]` (`app/core/plan_limits.py`) gana un conteo adicional:
  el cupo de "usuarios" de un tenant pasa a ser `count(User where tenant_id=...) +
  count(UserInvitation where tenant_id=... AND status='pending')`. La creación de una invitación
  llama `enforce_plan_limit(db, tenant, "usuarios")` **dentro de la misma transacción** que el
  `INSERT` de `UserInvitation`, con el mismo `SELECT ... FOR UPDATE` sobre `Tenant` que ya usa
  `create_user` hoy (spec 033, FR-015) — mismo patrón, ahora también reservando el cupo antes de
  que la invitación se consuma.
- **Rationale**: sin este conteo, esta spec **abriría una vía para eludir en silencio** el límite
  de usuarios del plan vigente (spec 033, FR-005/FR-006/FR-007, comportamiento ya protegido por el
  Principio II) — un tenant podría crear invitaciones ilimitadas (no son filas de `User`, así que
  no cuentan hoy) y solo se enteraría del límite, de forma confusa, cuando la persona invitada
  intentara autenticarse por primera vez (un momento no autenticado, sin nadie con permiso para
  actuar). Bloquear en el momento de invitar —igual que ya se bloquea al crear un usuario
  directamente— preserva la garantía existente sin que ninguna FR de esta spec necesite mencionarlo
  explícitamente: es una consecuencia técnica necesaria de que la creación de cuentas ahora pasa
  por dos pasos en vez de uno.
- **Efecto colateral aceptado (documentado, no oculto)**: `GET /plan` (spec 033, FR-013) también ve
  el nuevo conteo combinado para el recurso "usuarios", vía `count_resource_usage`, que reutiliza el
  mismo `_count_resource`. Esto hace que el número mostrado al ADMIN sea más preciso (refleja
  compromisos reales, no solo cuentas ya creadas) en vez de un efecto secundario oculto.
- **Alternatives considered**: no contar invitaciones pendientes contra el límite — descartado por
  la razón de arriba (bypass silencioso de una regla de negocio ya protegida); verificar el límite
  recién al consumir la invitación (en `login()`) — descartado, dejaría sin cupo a una persona que
  ya recibió su invitación y su contraseña temporal, un rechazo confuso en un flujo no autenticado
  donde nadie puede "arreglarlo" en el momento (peor experiencia que rechazar al ADMIN de entrada).

## Decisión 6 — FR-018 (tenant inactivo/suspendido) se resuelve reusando `ensure_plan_not_expired`

- **Decisión**: no se agrega ningún campo nuevo de "tenant activo/inactivo" — se confirmó que
  `Tenant` (`app/core/models.py`) no tiene ninguna columna de estado booleano hoy. El único
  mecanismo existente que puede dejar a un tenant sin poder operar es el vencimiento de plan
  (`tenant.plan_vence_en`, spec 033), ya encapsulado en `ensure_plan_not_expired()`
  (`app/core/plan_limits.py`), que además ya se invoca automáticamente como primer paso de
  `enforce_plan_limit()` (spec 033, research.md Decisión 14: "ningún router de recurso necesita
  invocarla por su cuenta"). Como la Decisión 5 de este documento ya hace que la creación de
  invitaciones llame `enforce_plan_limit(db, tenant, "usuarios")`, FR-018 queda cubierto **sin**
  código adicional dedicado.
- **Rationale**: "inactivo o suspendido" en el lenguaje del spec no distingue matices (Assumptions
  lo dice explícitamente) — corresponde 1:1 al único estado de bloqueo operativo que el sistema ya
  modela. Introducir un campo booleano nuevo sería duplicar un concepto que ya existe.
- **Alternatives considered**: agregar `Tenant.active: bool` nuevo — descartado, no hay ninguna otra
  funcionalidad hoy que necesite distinguir "activo" de "con plan vencido", y esta spec tampoco lo
  requiere; sería una columna sin otro consumidor, en contra del Principio V.

## Decisión 7 — Consumo de la invitación en `POST /auth/login`: bloqueo pesimista sobre la fila

- **Decisión**: dentro de `login()` (`app/api/v1/auth/routes.py`), cuando la búsqueda normal de
  `User` no encuentra nada **y** hay un `tenant` resuelto por `x-tenant-host`, se busca una
  `UserInvitation` con `status='pending'`, `tenant_id=tenant.id` y el correo normalizado, con
  `WITH FOR UPDATE` sobre esa fila — mismo patrón que ya usa `_resolve_reset_token(..., lock=True)`
  (spec 031, Decisión 5) para el doble consumo de un enlace de reset. Si `verify_password(...)`
  contra `invitation.password_hash` es `True`: se crea el `User` (ver Decisión 8), se fija
  `invitation.status='consumed'` + `consumed_at=now()`, se hace `commit()`, y el login continúa
  exactamente como con cualquier otro usuario (mismo `user_data`, mismos tokens). Si es `False`, o
  si la invitación ya no está `pending` tras adquirir el lock (cancelada o consumida por una
  petición concurrente), el login cae al mismo `401 "Invalid credentials"` de siempre.
- **Rationale**: sin el lock, dos intentos de login casi simultáneos con la misma contraseña
  temporal (doble clic, reintento de red) podrían pasar ambos la verificación antes de que
  cualquiera hiciera `commit`, creando dos cuentas para el mismo correo (violaría la unicidad de
  `email` por tenant) o dejando el estado de la invitación inconsistente. El mismo lock resuelve el
  Edge Case "si se cancela una invitación mientras la persona está en medio de su primer ingreso, la
  cancelación gana": si el `UPDATE ... SET status='cancelled'` de `POST /invitations/{id}/cancel`
  alcanza a comprometerse primero, el `SELECT ... FOR UPDATE ... WHERE status='pending'` del login
  ya no encuentra la fila y el intento falla; si el login adquiere el lock primero, el cancel espera
  y luego actúa sobre una invitación que ya quedó `consumed` (ver Decisión 9).
- **Alternatives considered**: verificación optimista (`UPDATE ... WHERE status='pending' AND
  email=... RETURNING ...`) — funcionalmente equivalente en Postgres; se documenta como
  implementación aceptable equivalente, la spec no exige una u otra, solo el resultado.

## Decisión 8 — `User.name` de una cuenta creada por invitación se fija igual al correo

- **Decisión**: al consumir una invitación, el `User` nuevo se crea con `name = invitation.email`
  (el correo ya normalizado). No se agrega ningún campo de nombre al formulario de invitación ni a
  ningún paso del primer ingreso.
- **Rationale**: `User.name` es `NOT NULL` (`app/core/models.py`) y el formulario "Agregar usuario"
  de esta spec tiene **exactamente dos controles** (FR-001: correo y rol) — ninguna FR ni pantalla
  de esta spec captura un nombre en ningún momento del flujo. No es una decisión de negocio (ningún
  criterio de aceptación depende de qué valor tenga `name` tras la invitación) sino la única forma
  de satisfacer una restricción de columna ya existente con los datos que el spec efectivamente
  recoge. Editar el nombre después de la primera vez queda fuera de alcance de esta spec (no existe
  hoy ninguna pantalla de "editar mi perfil/nombre").
- **Alternatives considered**: pedir el nombre en el primer ingreso (una pantalla adicional tras
  autenticarse con la contraseña temporal) — descartado, ninguna Acceptance Scenario de User
  Story 2 lo menciona y agregaría una pantalla no pedida (Principio V); hacer `User.name` nullable
  — descartado, cambiaría el contrato de una columna consumida por otras funcionalidades
  (`UserResponse.name`, avatares con inicial en `users-page.component.ts`) sin que ninguna FR lo
  requiera.

## Decisión 9 — Reenvío/cancelación re-verifican `status='pending'` antes de actuar

- **Decisión**: `POST /invitations/{id}/resend` y `POST /invitations/{id}/cancel` cargan la fila con
  `WITH FOR UPDATE` y verifican `status == 'pending'` antes de escribir; si ya no lo está (consumida
  o cancelada por otra petición mientras tanto), responden `409` con un mensaje explícito en vez de
  sobrescribir un estado terminal.
- **Rationale**: cierra la otra mitad de la condición de carrera de la Decisión 7 — si el login
  alcanza a consumir la invitación primero, un `cancel` o `resend` concurrente no debe "revivirla"
  ni pisar silenciosamente una cuenta que ya existe.
- **Alternatives considered**: ninguna — es la contraparte obligatoria de la Decisión 7, no una
  elección de diseño independiente.

## Decisión 10 — Reenvío que falla al enviar: no se pierde la contraseña anterior

- **Decisión**: en `POST /invitations/{id}/resend`, la contraseña nueva se genera y el correo se
  intenta enviar **antes** de escribir `password_hash`/`sent_at` nuevos en la fila (o, equivalente,
  dentro de la misma transacción con `rollback()` si el envío falla). Si `send_email()` falla, la
  fila conserva su `password_hash`/`sent_at` anteriores intactos y el ADMIN ve el error — la
  invitación sigue siendo utilizable con la contraseña que ya tenía, en vez de quedar con una
  contraseña nueva que nadie recibió.
- **Rationale**: es la misma garantía de FR-012 ("nunca... un estado utilizable... si el correo no
  salió") aplicada al caso de reenvío, donde "no utilizable" sería peor que en la creación: dejaría
  una invitación que ya tenía dueño sin ninguna contraseña funcional conocida por nadie.
- **Alternatives considered**: invalidar la contraseña anterior de inmediato y solo entonces
  intentar enviar la nueva — descartado, es exactamente el escenario que FR-012 prohíbe.

## Decisión 11 — Eliminación de `POST /users` (FR-004): sin tests protegidos que actualizar

- **Decisión**: se elimina por completo `create_user()` y el schema `UserCreate` de
  `app/api/v1/users/router.py`/`schemas.py`. No se deja ningún endpoint alterno, flag, ni modo de
  compatibilidad.
- **Rationale**: se confirmó que no existe ningún characterization test (`"CONGELA comportamiento
  actual:"`) que cubra `app/api/v1/users` — no hay ningún test protegido en rojo que este cambio
  deba justificar ante el Principio III. La propia Assumption de `spec.md` ("no queda ninguna vía
  alterna, ni siquiera interna o de soporte") ya autoriza la eliminación total como decisión de
  negocio confirmada.
- **Alternatives considered**: dejar el endpoint respondiendo `410 Gone` por compatibilidad —
  descartado, ningún consumidor externo depende de él (es interno al mismo frontend que se modifica
  en el mismo cambio) y la spec pide remoción completa, no un aviso de baja.

## Decisión 12 — Reutilización total de `generate_random_password`/`generate_passwd_hash`

- **Decisión**: sin cambios en `app/core/utils.py`. `generate_random_password(length=12)` ya genera
  exactamente el alfabeto que pide FR-005 (mayúsculas, minúsculas, dígitos, `!@#$%*?`) — es el mismo
  mecanismo que `tenant_create()` ya usa para la contraseña del primer admin de un tenant (la
  referencia explícita del spec: "recorra el mismo primer ingreso que ya recorre el usuario inicial
  de un tenant"). `generate_passwd_hash` (bcrypt) hashea esa contraseña exactamente igual que
  cualquier otra.
- **Rationale**: ninguna alternativa razonable — son los valores y funciones que la propia spec
  referencia por comparación con un mecanismo ya existente y verificado.

## Decisión 13 — Listado de pendientes: endpoint nuevo separado, no una unión con `GET /users`

- **Decisión**: `GET /invitations` (paginado, mismo patrón `Page[T]`/`paginate()` que `GET /users`)
  devuelve solo invitaciones `status='pending'` del tenant. El frontend renderiza dos secciones en
  la misma pantalla ("Usuarios" y "Invitaciones pendientes"), cada una alimentada por su propio
  endpoint — no se fusionan ambos tipos en una sola respuesta ni en un único modelo de fila.
- **Rationale**: `UserResponse`/`Page[UserResponse]` no se tocan (Principio V) y la Acceptance
  Scenario de US3 ("ve ambos grupos claramente diferenciados") se satisface igual de bien con dos
  secciones que con una lista fusionada — fusionar exigiría un tipo de fila heterogéneo
  (usuario-o-invitación) que ninguna FR pide y que complica la paginación (dos totales distintos
  bajo una sola página).
- **Alternatives considered**: extender `UserResponse` con un campo `is_pending_invitation` y
  devolver ambos tipos desde `GET /users` — descartado, mezclaría dos entidades con ciclos de vida y
  acciones distintas (reenviar/cancelar solo aplican a invitaciones) bajo un único contrato,
  complicando el frontend sin necesidad.

## Decisión 14 — Frontend: el botón "Agregar usuario" ya está gateado a ADMIN a nivel de ruta

- **Decisión**: no se agrega ningún chequeo de rol nuevo en `UsersPageComponent` — la ruta
  `dashboard/users` (`pos-heladeria/src/app/modules/dashboard/routes.ts`) ya tiene
  `canActivate: [roleGuard([UserRole.ADMIN])]`. Un CASHIER autenticado no puede ni cargar la
  pantalla completa hoy.
- **Rationale**: la Acceptance Scenario 4 de User Story 1 ("un CASHIER no ve el botón 'Agregar
  usuario'") ya se cumple con el guard existente — agregar un segundo chequeo a nivel de componente
  sería redundante (Principio V).

## Testing — extensión del patrón de `auth_fixtures.py`/`fixtures.py`

- `auth_fixtures.py` (spec 031) gana: `UserInvitation` en `_TABLE_NAMES` y `Base.metadata`, y un
  helper `make_invitation(db, tenant, role, password="temporal-123", **kw)` con los mismos defaults
  razonables que `make_user`/`make_password_reset_token`.
- Los tests llaman las funciones de router directamente (`app/api/v1/invitations/router.py`,
  `app/api/v1/auth/routes.py::login`), sin `TestClient` — mismo criterio que spec 031, Decisión 10,
  sin precedente de `TestClient` en el repo.
- `send_email()` (síncrono, Decisión 4) se mockea con `unittest.mock.patch("app.core.utils.send_email")`
  para simular tanto el éxito como el `RuntimeError` de fallo, sin llamadas de red reales.

## Migraciones — estrategia de rollback (Principio VIII)

Todo lo nuevo vive en el schema `shared` (mismo patrón que `password_reset_tokens`, spec 031): una
sola migración, sin `for_each_tenant_schema`.

- `user_invitations` (tabla nueva, schema `shared`, FKs a `shared.tenants.id` y `shared.roles.id`):
  rollback = `op.drop_table`. Sin datos preexistentes que preservar — es una entidad enteramente
  nueva, no hay invitaciones antes de esta spec.
- `plan_limits.py::RESOURCE_CONFIG["usuarios"]` cambia su fórmula de conteo (Decisión 5), pero no
  toca ninguna columna ni tabla — no requiere migración, solo código de aplicación; el rollback es
  revertir el commit.
- `app/api/v1/users/router.py`/`schemas.py` pierden `create_user`/`UserCreate` (Decisión 11) — sin
  columnas afectadas, sin migración; el rollback es revertir el commit (no hay datos que perder,
  ya que ningún usuario existente se creó "mediante" el endpoint como un estado persistente
  distinguible de cualquier otro usuario).
