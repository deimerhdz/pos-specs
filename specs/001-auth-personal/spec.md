# Feature Specification: Identidad y acceso del personal (cajero/admin)

**Feature Branch**: `001-auth-personal`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/auth/routes.py`, para que sirva de contrato formal de cara a la
modernización (Principio III de la [Constitución](../../.specify/memory/constitution.md)).
Donde el resto de las specs de este proyecto describen lo que el sistema **debe** hacer, esta
describe lo que el sistema **efectivamente hace** — incluidas sus anomalías, con su tratamiento
acordado citado de `registro-de-anomalias.md`.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de identidad y acceso del personal del local (cajero/admin) en el sistema POS Heladería, tomado
de `reglas-de-negocio.md` §1 (RN-AUTH-01 a RN-AUTH-10) y de `registro-de-anomalias.md`
(A-18, A-21, A-22, A-23), para que sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `auth/routes.py`, no uno deseado.
  Las anomalías conocidas se marcan inline con su tratamiento acordado (registro-de-anomalias.md).
-->

### User Story 1 - Inicio de sesión del personal (Priority: P1)

Un cajero o administrador entra su email y contraseña en la pantalla de login del local. El
sistema debe reconocerlo solo si pertenece al tenant correcto (o, si no hay tenant identificado
por el host, solo entre súper-administradores globales) y si su cuenta sigue activa.

**Why this priority**: sin login no hay acceso a caja, inventario ni ninguna otra función —
es la puerta de entrada de todo el sistema para el personal.

**Independent Test**: se puede probar completamente enviando `POST /auth/login` contra un
`pos-backend` en ejecución con distintas combinaciones de credenciales, header `x-tenant-host`
y estado `active`, sin depender de ningún otro módulo.

**Acceptance Scenarios**:

1. **Given** un usuario con contraseña correcta, `active=True`, dentro del tenant que resuelve
   el header `x-tenant-host`, **When** `POST /auth/login`, **Then** responde `200` con
   `access_token`, `refresh_token` (dos JWT distintos) y los datos del usuario
   (`RN-AUTH-04`, `RN-AUTH-05`).
2. **Given** una contraseña incorrecta, **When** se reintenta `POST /auth/login` cualquier
   número de veces con el mismo email, **Then** cada intento responde `401 "Invalid
   credentials"` idéntico al primero, sin bloqueo de cuenta, captcha ni retraso creciente
   (`RN-AUTH-03`). **Anomalía A-22 (ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de
   ratificación real)**: se corregirá en modernización con el mismo mecanismo de rate-limit ya
   usado en `menu` (`app/core/rate_limit.py`).
3. **Given** una cuenta con `active=False` y contraseña correcta, **When** `POST /auth/login`,
   **Then** responde `403 "User account is inactive"` — código distinto del `401` de
   credenciales inválidas (`RN-AUTH-04`).
4. **Given** un header `x-tenant-host` que resuelve a un tenant registrado, **When** existe un
   usuario con el mismo email en otro tenant, o un súper-admin global con ese email, **Then**
   el login falla con `401` para ese header aunque la contraseña sea correcta para esa otra
   cuenta — el usuario se busca únicamente `WHERE email=... AND tenant_id=<tenant resuelto>`
   (`RN-AUTH-05`).
5. **Given** ausencia del header `x-tenant-host`, o un host que no coincide con ningún tenant,
   **When** `POST /auth/login`, **Then** el usuario se busca únicamente entre cuentas con
   `tenant_id IS NULL` (súper-administradores globales) (`RN-AUTH-05`).

---

### User Story 2 - Renovar la sesión sin volver a loguearse (Priority: P1) — [PROTEGIDA A-23]

Mientras trabaja, el access token del cajero expira antes que su sesión completa. El cliente
usa el refresh token para obtener un nuevo access sin pedirle credenciales de nuevo — pero el
sistema revalida contra base de datos que la cuenta sigue activa en cada renovación.

**Why this priority**: sostiene toda una jornada de turno sin recargas de login constantes, y
es el único punto donde una desactivación de cuenta a mitad de turno tiene efecto antes de que
expire el refresh (hasta 7 días).

**Independent Test**: se puede probar logueándose, esperando (o forzando) la expiración del
access, y llamando `GET /auth/refresh-token` con el refresh vigente, con y sin desactivar la
cuenta entre medio.

**Acceptance Scenarios**:

1. **Given** un login exitoso, **When** se inspeccionan los dos tokens emitidos, **Then** el
   access expira a los `ACCESS_TOKEN_EXPIRY` minutos (default 1.440 min = 24 h) y el refresh a
   los `REFRESH_TOKEN_EXPIRY_MINUTES` minutos (default 10.080 min = 7 días) — vidas distintas e
   independientes (`RN-AUTH-06`).
2. **Given** un refresh token vigente, **When** `GET /auth/refresh-token`, **Then** el sistema
   vuelve a consultar el usuario en base de datos por su id (no reutiliza los claims del
   refresh) y exige `active==True`; si la cuenta sigue activa, responde `200` con un
   `access_token` nuevo (`RN-AUTH-07`).
3. **Given** el mismo refresh token vigente, **When** la cuenta fue desactivada (`active=False`)
   después de emitirlo, **Then** `GET /auth/refresh-token` responde `401 "User not found or
   inactive"` aunque el JWT del refresh en sí siga siendo válido y no haya expirado
   (`RN-AUTH-07`). **Regla [PROTEGIDA] A-23** — citada en `memoria-historica.md` #4
   (2026-07-28, commit `5c1db9ed`, Deimer Hernandez): especificar tal cual, **no modificar**
   este comportamiento en la modernización.
4. **Given** un access token expirado, **When** se usa para llamar cualquier endpoint
   protegido, **Then** responde `401` por expiración de JWT, sin que esto afecte la validez del
   refresh token asociado (`RN-AUTH-06`).

---

### User Story 3 - Cambio de contraseña autenticado (Priority: P2)

Un cajero o admin ya logueado cambia su propia contraseña, ya sea por rutina o porque la cuenta
se creó con una contraseña temporal generada por el sistema.

**Why this priority**: es el único mecanismo de cambio de contraseña que existe hoy (no hay
"reset" por correo dentro de este módulo); habilita el ciclo completo de una credencial
temporal.

**Independent Test**: se puede probar completamente con un usuario ya autenticado llamando
`POST /auth/change-password` con distintas combinaciones de `current_password`/`new_password`.

**Acceptance Scenarios**:

1. **Given** un usuario autenticado que conoce su contraseña vigente, **When**
   `POST /auth/change-password` con `current_password` correcta y una `new_password` válida
   (6-128 caracteres), **Then** responde `200`, el hash se actualiza y se persiste con
   `db.commit()` (`RN-AUTH-01`).
2. **Given** el mismo request, **When** el cambio se completa con éxito, **Then** el flag
   `must_change_password` de la cuenta pasa a `False` en la misma operación — sin este cambio,
   una cuenta creada con contraseña temporal seguiría marcada como pendiente de cambio para
   siempre (`RN-AUTH-02`).
3. **Given** un usuario autenticado, **When** envía `current_password` incorrecta, **Then**
   responde `400 "Current password is incorrect"` y la contraseña no cambia — no existe forma
   de cambiarla sin conocer la vigente por esta vía (`RN-AUTH-01`).
4. **Given** una `new_password` cuyos primeros 72 bytes UTF-8 coinciden con otra contraseña más
   larga, **When** se hashea (aquí o en cualquier login posterior), **Then** ambas autentican de
   forma idéntica — el hash y la verificación truncan a 72 bytes (límite de bcrypt), y el
   schema de la petición no valida ese límite (solo exige 6-128 **caracteres**, no bytes)
   (`RN-AUTH-09`). **Anomalía A-22 (ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de
   ratificación real)**: se corregirá en modernización validando la longitud máxima acorde a 72
   bytes, incluyendo una validación equivalente en el frontend (hoy inexistente).

---

### User Story 4 - Cerrar sesión (logout) (Priority: P3)

Un cajero o admin termina su turno y cierra sesión desde el cliente.

**Why this priority**: es el camino menos crítico de los cuatro (no bloquea el trabajo si
falla), pero documenta un comportamiento con implicación de seguridad real: qué queda vivo tras
un logout.

**Independent Test**: se puede probar logueándose, llamando `GET /auth/logout` con el access
token, y verificando por separado qué sigue funcionando (el refresh) y qué no (ese access).

**Acceptance Scenarios**:

1. **Given** un access token válido (jti=X), **When** `GET /auth/logout`, **Then** responde
   `200 "Logged Out Successfully"` y el `jti` X se agrega a una blocklist con un TTL igual a su
   `exp` original — no una revocación de toda la sesión, solo de ese token puntual
   (`RN-AUTH-08`).
2. **Given** ese mismo access token ya revocado, **When** se vuelve a usar para llamar un
   endpoint protegido, **Then** responde `401 "Token has been revoked"`.
3. **Given** el refresh token emitido en el mismo login que el access recién cerrado, **When**
   se usa para `GET /auth/refresh-token`, **Then** sigue funcionando y emite un access nuevo con
   un `jti` distinto, no bloqueado — el logout del access **no** revoca el refresh
   (`RN-AUTH-08`). **Anomalía A-22 (ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de
   ratificación real)**: se corregirá en modernización revocando ambos tokens (access y
   refresh) en el mismo logout.

---

### User Story 5 - Contraseñas temporales emitidas por el sistema (Priority: P4)

Cuando el sistema crea una credencial nueva (por ejemplo, al dar de alta un usuario), genera una
contraseña temporal en vez de dejarla en blanco.

**Why this priority**: es un detalle de bajo impacto operativo diario, pero con implicación de
seguridad concreta sobre la fortaleza de las credenciales iniciales.

**Independent Test**: se puede probar invocando el generador de forma aislada y verificando la
longitud y el alfabeto del resultado, sin depender de ningún endpoint HTTP.

**Acceptance Scenarios**:

1. **Given** el sistema necesita emitir una contraseña temporal, **When** se genera, **Then**
   produce exactamente 12 caracteres elegidos con un generador criptográficamente seguro
   (`secrets.choice`, no `random`) de un alfabeto de letras mayúsculas, minúsculas, dígitos y
   los símbolos `!@#$%*?` (`RN-AUTH-10`).

---

### User Story 6 - Rol legado `STAFF` remapeado a `CASHIER` (Priority: P4) — cerrada sin corrección pendiente

Una sesión JWT viva que trae el rol legado `staff` (el backend nunca emite ese valor; solo
`ADMIN`/`CASHIER`) se degrada en el frontend al rol `CASHIER`, no a `ADMIN`, en vez de invalidar
la sesión.

**Why this priority**: es un remapeo puramente defensivo en el cliente, sin regla `RN-*` propia
de backend, y confirmado sin impacto real.

**Independent Test**: se puede probar de forma aislada llamando `mapBackendRole("staff")` en el
frontend y verificando que devuelve `CASHIER`.

**Acceptance Scenarios**:

1. **Given** un JWT vivo cuyo claim de rol es `"staff"`, **When** el frontend lo normaliza con
   `mapBackendRole`, **Then** el resultado es `UserRole.CASHIER`, nunca `UserRole.ADMIN` ni una
   sesión inválida (`pos-heladeria/src/app/core/interfaces/user.interface.ts:22-30`, anomalía
   **A-18**).

**Nota de cierre**: documentar tal cual, **sin corrección pendiente** — cerrado sin impacto en
la entrevista de negocio (P13: nunca hubo cuentas con rol `STAFF` en producción, acta
`entrevista-negocio.md` P13). No es una regla `RN-AUTH-*` (vive en el frontend, no en
`auth/routes.py`), se incluye aquí porque es la única evidencia de código sobre administración
de roles que entra en el alcance de esta spec (ver "Out of Scope").

---

### User Story 7 - Dónde vive la sesión del personal en el cliente (Priority: P4) — diseño confirmado, no anomalía abierta

El par de tokens (access y refresh) que recibe el personal al loguearse se guarda en el
navegador para sobrevivir recargas y pestañas.

**Why this priority**: es una decisión de arquitectura ya cerrada por el negocio, documentada
aquí para que la modernización no la reabra por error.

**Independent Test**: se puede probar logueándose e inspeccionando el `localStorage` del
navegador.

**Acceptance Scenarios**:

1. **Given** un login exitoso del personal, **When** se inspecciona el `localStorage` del
   navegador, **Then** las claves `pos.access_token` y `pos.refresh_token` contienen los JWT de
   access y refresh emitidos (`pos-heladeria/src/app/core/auth/token-storage.service.ts:12-24`,
   anomalía **A-21**).

**Nota de cierre**: confirmado por decisión de negocio (P15, `entrevista-negocio.md`) como
**diseño definitivo** — no está en el plan mover esto a una cookie `httpOnly`. Se documenta tal
cual. Nota independiente, sin relación con esta decisión de almacenamiento: la actualización de
`@angular/core` (6 vulnerabilidades XSS "high" reportadas por `npm audit`, registro de riesgos
R3) sigue siendo una corrección inmediata, no condicionada a si el token vive en `localStorage`
o en una cookie.

---

### Edge Cases

- **Host con puerto en el header**: `x-tenant-host: heladeria-a.pos.com:8443` — el login corta
  en el primer `:` antes de resolver el tenant, así que el puerto no afecta la resolución
  (`auth/routes.py:25`).
- **`uid` inválido dentro del refresh token**: si el claim `uid` no es un UUID bien formado,
  `GET /auth/refresh-token` responde `401 "Invalid token payload"` antes de tocar la base de
  datos (`auth/routes.py:118-124`).
- **Usar un access token donde se espera un refresh (o viceversa)**: cada endpoint exige un tipo
  de token específico (`AccessTokenBearer`/`RefreshTokenBearer`); presentar el tipo equivocado
  se rechaza antes de llegar a la lógica de negocio.
- **El chequeo de `active=True` no se limita a login/refresh**: cualquier endpoint protegido en
  el resto del backend (no solo `auth`) vuelve a exigir `active==True` en cada llamada a través
  de las dependencias `get_current_user`/`get_authenticated_user`
  (`app/core/dependencies.py:104-121,133-152`) — una cuenta desactivada pierde acceso de
  inmediato en la siguiente petición a cualquier endpoint, no solo al intentar renovar el
  access.
- **Reutilizar un access ya revocado por logout**: responde `401 "Token has been revoked"` vía
  blocklist (Redis), distinto del `401` genérico de token inválido/expirado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Al cambiar la contraseña, el sistema DEBE exigir la contraseña actual correcta,
  verificada contra el hash almacenado; si no coincide, DEBE rechazar con `400` sin modificar la
  contraseña ni el flag de "debe cambiar contraseña" (`RN-AUTH-01`).
- **FR-002**: Al completar un cambio de contraseña exitoso, el sistema DEBE apagar el flag
  `must_change_password` de la cuenta en la misma operación (`RN-AUTH-02`).
- **FR-003**: El sistema NO bloquea hoy ninguna cuenta ni limita la tasa de intentos de
  `POST /auth/login` tras credenciales incorrectas; cada intento fallido responde `401`
  idéntico al anterior, sin captcha ni retraso creciente (`RN-AUTH-03`). **[Anomalía A-22 —
  ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de ratificación real]**: corregir en
  modernización con el mismo mecanismo de rate-limit ya usado en `menu`.
- **FR-004**: El login DEBE permitirse únicamente a usuarios con `active=True`; una cuenta
  desactivada con contraseña correcta DEBE recibir `403`, código distinto del `401` de
  credenciales inválidas (`RN-AUTH-04`).
- **FR-005**: El login DEBE resolver el usuario dentro del tenant identificado por el header
  `x-tenant-host` cuando ese host coincide con un tenant registrado; si no hay header o no
  coincide con ningún tenant, DEBE buscar exclusivamente entre usuarios sin tenant
  (súper-administradores globales) (`RN-AUTH-05`).
- **FR-006**: El access token y el refresh token emitidos en un mismo login DEBEN tener tiempos
  de expiración distintos e independientes, configurables por variable de entorno (por defecto
  1.440 minutos / 24 h para el access, 10.080 minutos / 7 días para el refresh) (`RN-AUTH-06`).
- **FR-007 [PROTEGIDA — A-23]**: `GET /auth/refresh-token` DEBE volver a consultar el usuario en
  base de datos por id (no reutilizar los claims codificados en el refresh) y DEBE exigir
  `active==True`; si el usuario ya no existe o fue desactivado, DEBE rechazar con `401` aunque
  el JWT del refresh en sí siga siendo válido (`RN-AUTH-07`). **Este comportamiento no debe
  modificarse en la modernización.**
- **FR-008**: `GET /auth/logout` DEBE revocar el `jti` del access token presentado agregándolo a
  una blocklist hasta su expiración natural (`exp` original); hoy NO revoca el refresh token
  asociado, que sigue siendo válido para emitir nuevos access tokens (`RN-AUTH-08`). **[Anomalía
  A-22 — ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de ratificación real]**: corregir
  en modernización revocando ambos tokens (access y refresh) en el mismo logout.
- **FR-009**: El sistema DEBE truncar la contraseña a sus primeros 72 bytes UTF-8 antes de
  generar o verificar su hash (límite de bcrypt); dos contraseñas que solo difieran después del
  byte 72 DEBEN autenticar de forma idéntica. El schema de cambio de contraseña hoy no valida
  ese límite (permite hasta 128 caracteres, no bytes) (`RN-AUTH-09`). **[Anomalía A-22 —
  ACCIDENTAL, confirmada en ronda 3 simulada, pendiente de ratificación real]**: corregir en
  modernización validando la longitud máxima acorde a 72 bytes, incluida una validación
  equivalente en el frontend (hoy inexistente).
- **FR-010**: Cuando el sistema genera una contraseña temporal, DEBE producir exactamente 12
  caracteres elegidos con un generador criptográficamente seguro (`secrets.choice`) de un
  alfabeto de letras mayúsculas, minúsculas, dígitos y los símbolos `!@#$%*?` (`RN-AUTH-10`).
- **FR-011**: El frontend DEBE remapear cualquier sesión JWT con el rol legado `staff` al rol
  `CASHIER` (nunca a `ADMIN`), preservando la sesión en vez de invalidarla. Documentado tal
  cual; **cerrado sin corrección pendiente** (`A-18`; P13, `entrevista-negocio.md`: nunca hubo
  cuentas `STAFF` en producción).
- **FR-012**: El personal (cajero/admin) DEBE persistir su par de tokens (access y refresh) en
  `localStorage` del navegador (claves `pos.access_token`/`pos.refresh_token`). Confirmado como
  **diseño definitivo** por decisión de negocio (`A-21`; P15, `entrevista-negocio.md`); no está
  planificado moverlo a una cookie `httpOnly`. Nota independiente y no condicionada a esta
  decisión: la actualización de `@angular/core` (6 vulnerabilidades XSS "high", R3) sigue siendo
  una corrección inmediata.

### Key Entities *(include if feature involves data)*

- **User (personal)**: cuenta de cajero o administrador. Atributos relevantes a esta spec:
  `email`, `password_hash`, `active`, `tenant_id` (`NULL` para súper-admin global), `role`,
  `must_change_password`.
- **Tenant**: local/heladería identificado por un `host`; determina en qué universo de usuarios
  busca el login cuando el header `x-tenant-host` resuelve a un tenant registrado.
- **Access Token / Refresh Token (JWT)**: par emitido en cada login, cada uno con su propio
  `jti`, `exp` y claim `refresh` (booleano) que los distingue. El access autoriza operaciones;
  el refresh solo sirve para pedir un access nuevo.
- **Blocklist de `jti`**: registro (Redis) de tokens de access revocados por logout, vivo hasta
  la expiración natural (`exp`) del token que revoca.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-AUTH-01` a `RN-AUTH-10` puede verificarse ejecutando los
  pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar leer
  `auth/routes.py` para entender el comportamiento esperado.
- **SC-002**: Un cajero o admin con contraseña correcta y cuenta activa obtiene un par de tokens
  funcional en un único request de login (`POST /auth/login`), sin pasos adicionales.
- **SC-003**: Una cuenta desactivada pierde la capacidad de renovar su sesión en, como máximo, el
  tiempo que tarda su siguiente intento de `GET /auth/refresh-token` — no tiene que esperar a
  que expire el refresh completo (hasta 7 días).
- **SC-004**: El equipo de modernización puede usar esta spec como criterio de aceptación de un
  characterization test automatizado para el módulo `auth`, hoy inexistente (ningún script de
  `app/scripts/test_*.py` ni spec de `pos-heladeria` cubre `auth` — zona oscura 13 de
  `mapa-sistema.md`).
- **SC-005**: Cada una de las tres reglas agrupadas bajo la anomalía A-22 (`RN-AUTH-03`,
  `RN-AUTH-08`, `RN-AUTH-09`) y la regla protegida A-23 (`RN-AUTH-07`) queda trazable 1:1 a un
  requisito funcional de esta spec, sin ambigüedad sobre cuál debe cambiar en la modernización y
  cuál debe preservarse tal cual.

## Out of Scope

- **Sesión del comensal por QR** — cubierta por la spec 007, incluye su propio manejo de token y
  almacenamiento.
- **Administración de roles vía panel** — aparece en esta spec solo como evidencia de código de
  la anomalía A-18 (remapeo `STAFF`→`CASHIER`); no tiene reglas `RN-*` propias extraídas en
  `reglas-de-negocio.md`, así que no se especifica más allá de esa mención.
- **Integridad del token QR firmado** — regla `RN-CART-26`, anomalía **A-24 [PROTEGIDA]**,
  cubierta por la spec 007. No se repite aquí porque pertenece al flujo del comensal, no al del
  personal.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP y nombres de campo **son** el contrato observable que se está documentando — se
  citan explícitamente porque los criterios de aceptación deben ser verificables directamente
  contra `pos-backend` en ejecución (no hay un characterization test existente que citar en su
  lugar; ver SC-004).
- **Valores por defecto configurables**: los tiempos de vida de los tokens (`RN-AUTH-06`) y los
  límites de longitud de contraseña (`RN-AUTH-09`) son configurables por variable de entorno;
  los valores citados aquí son los defaults de `app/core/config.py` y
  `app/api/v1/auth/schemas.py` al momento de esta extracción (2026-08-16). Si el entorno real
  los sobreescribe, deben re-verificarse antes de usar esta spec como characterization test.
- **Las correcciones de A-22 dependen de ratificación real**: las tres reglas agrupadas en A-22
  se confirmaron `ACCIDENTAL` en una tercera ronda de entrevista **simulada** (a petición
  explícita del usuario de este repositorio, asumiendo el rol de negocio, no una conversación
  real) — ver aviso de método en `registro-de-anomalias.md`. Antes de tratar la corrección de
  A-22 como definitiva para producción, debe ratificarse con el negocio real.
- **A-23 es intocable**: [PROTEGIDA], con dos testigos (CÓDIGO + NEGOCIO histórico,
  `memoria-historica.md` #4). Cualquier plan de modernización que toque `/auth/refresh-token`
  debe preservar exactamente esta relectura de BD y el chequeo `active==True`.
- **A-18 y A-21 están cerradas, no son riesgo abierto**: A-18 sin impacto confirmado (nunca hubo
  cuentas `STAFF`); A-21 es diseño definitivo confirmado por negocio, no una tarea pendiente de
  "arreglar hacia cookie httpOnly".
