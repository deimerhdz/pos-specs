# Data Model: Alta de usuarios internos por invitación

Todo lo nuevo o modificado vive en el schema `shared` (`app/core/models.py`), igual que
`Tenant`/`Role`/`User`/`PasswordResetToken` hoy — no en el schema `tenant` per-tenant. Las
decisiones de diseño detrás de cada elección están en [research.md](./research.md); este documento
se limita a columnas, restricciones y transiciones.

## UserInvitation (`user_invitations`) — NUEVA

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id` | UUID (PK) | No | `UUIDPrimaryKeyMixin`, igual que el resto de entidades de `shared`. |
| `tenant_id` | Integer (FK → `shared.tenants.id`) | No | Indexado. Tenant dueño de la invitación (FR-013). |
| `email` | `String(255)` | No | Normalizado (recorte de espacios + minúsculas) al escribir y al comparar (FR-016). Único por tenant **mientras `status='pending'`** (ver índice abajo) — no globalmente (FR-014). |
| `role_id` | UUID (FK → `shared.roles.id`) | No | Rol asignado (ADMIN o CASHIER, `RoleName`) — el mismo que hereda el `User` al consumirse (FR-007). |
| `password_hash` | `String(255)` | No | Hash bcrypt (`generate_passwd_hash`) de la contraseña temporal **vigente**. Se sobrescribe en cada reenvío (FR-010) — nunca se guarda ni se expone en texto plano (FR-006). |
| `status` | `String(10)` | No | `'pending'` \| `'consumed'` \| `'cancelled'`. `server_default='pending'`. |
| `sent_at` | `DateTime` | No | `server_default now()`. Fecha de envío mostrada en el listado (FR-009); se actualiza en cada reenvío exitoso (FR-010), no en cancelación. |
| `consumed_at` | `DateTime` | Sí | `NULL` hasta que el primer login con la contraseña temporal la consume (FR-007). |
| `cancelled_at` | `DateTime` | Sí | `NULL` hasta que el ADMIN la cancela (FR-011). |
| `created_at` / `updated_at` | `DateTime` | No / Sí | `TimestampMixin`, igual que el resto de entidades — auditoría interna, no se muestra como "fecha de envío" (ver `sent_at`). |

**Restricciones** (`__table_args__`, `{"schema": "shared"}`):
- Índice sobre `tenant_id` (listado paginado por tenant, FR-013).
- **Índice único parcial** sobre `(tenant_id, email)` `WHERE status = 'pending'` — a lo sumo una
  invitación `pending` por correo dentro de un tenant en cualquier instante (FR-015, Edge Case de
  invitación concurrente). Definido con `postgresql_where=...` **y** `sqlite_where=...` en el mismo
  `Index(...)` para que se comporte igual en Postgres (producción) y SQLite en memoria
  (characterization tests) — research.md Decisión 3.
- `CheckConstraint("status IN ('pending', 'consumed', 'cancelled')")`, mismo patrón que
  `OrderPaymentAttempt.status` (spec 024).

**Relaciones**: `tenant: Tenant` y `role: Role` (solo lectura, `selectinload` en el listado — mismo
patrón que `UserResponse.role_name`/`tenant_name` en `app/api/v1/users/router.py`).

## User (`shared.users`) — SIN CAMBIOS DE ESQUEMA

Ninguna columna nueva. Lo único nuevo es **cómo** se llega a insertar una fila: además del camino
ya existente (`tenant_create()` para el primer admin de un tenant), ahora también se crea una fila
de `User` desde `POST /auth/login` cuando las credenciales coinciden con una `UserInvitation`
`pending` (research.md Decisión 7). Valores fijados en ese momento:

| Columna | Valor al consumir una invitación |
|---|---|
| `name` | `invitation.email` (normalizado) — research.md Decisión 8, no hay otro dato disponible. |
| `email` | `invitation.email` (normalizado). |
| `password_hash` | `invitation.password_hash` (se reutiliza el mismo hash — la persona ya demostró conocer esa contraseña al autenticarse; no hace falta volver a hashear). |
| `phone` | `NULL` — no se recoge en ningún paso de esta spec. |
| `active` | `True`. |
| `must_change_password` | `True` (FR-007) — dispara el flujo ya existente de cambio obligatorio (spec 001/031, sin modificar). |
| `role_id` | `invitation.role_id`. |
| `tenant_id` | `invitation.tenant_id`. |

## Plan / conteo de "usuarios" (`app/core/plan_limits.py`) — MODIFICADA (lógica, sin cambio de esquema)

`RESOURCE_CONFIG["usuarios"]` pasa de contar solo `User` a contar `User` **+** `UserInvitation` con
`status='pending'`, ambos filtrados por `tenant_id` (research.md Decisión 5). Sin columnas nuevas —
es un cambio en `_count_resource`/`RESOURCE_CONFIG`, no en el modelo de datos.

## Transiciones de estado de `UserInvitation`

```text
                    ┌──────────┐
   (POST         ► │ pending  │
   /invitations)    └────┬─────┘
                         │
        ┌────────────────┼──────────────────┐
        │                │                  │
  login con la      el ADMIN la        el ADMIN la
  contraseña        cancela            reenvía (FR-010)
  temporal vigente  (FR-011)           — permanece "pending",
  (FR-007)               │             solo cambian
        │                │             password_hash/sent_at
        ▼                ▼
   ┌───────────┐   ┌─────────────┐
   │ consumed  │   │ cancelled   │
   └───────────┘   └─────────────┘
   (terminal)       (terminal)
```

- `pending` es el único estado desde el que se puede reenviar o cancelar, y el único que el índice
  único parcial contempla — nunca hay más de una invitación `pending` para el mismo correo dentro
  de un tenant.
- `consumed` y `cancelled` son terminales: ninguna acción vuelve una invitación a `pending`.
  Corregir un correo mal escrito es cancelar y crear una invitación **nueva** con el correo
  correcto (Assumptions de `spec.md`), no editar la fila existente.
- Reenviar **no** es una transición de estado — la fila permanece `pending`; solo cambian
  `password_hash`/`sent_at` (research.md Decisión 2).

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Exactamente correo + rol en el formulario, sin contraseña | Frontend (`InvitationFormComponent` nuevo, reemplaza `UserFormComponent`) | US1 |
| Solo ADMIN puede crear/listar/reenviar/cancelar invitaciones | `require_tenant_admin` (ya existente, reusado) en los 4 endpoints nuevos + ruta `dashboard/users` ya gateada (research.md Decisión 14) | US1, US3, US4 |
| No crear ninguna cuenta al invitar | `POST /invitations` solo hace `INSERT` en `UserInvitation`, nunca en `User` | US1 |
| Contraseña temporal nunca expuesta (pantalla, API, logs) | `InvitationResponse` no incluye `password_hash`; no se loguea en ningún punto del flujo | US1 |
| Tenant vencido/inactivo no puede invitar | `enforce_plan_limit(db, tenant, "usuarios")` → `ensure_plan_not_expired()` primero (research.md Decisión 6) | US1 |
| Límite del plan también cubre invitaciones pendientes | `RESOURCE_CONFIG["usuarios"]` extendido (research.md Decisión 5) | US1 |
| Correo ya usado (activo o inactivo) en el tenant → rechazo | `SELECT` sobre `User` por `tenant_id` + email normalizado, sin filtrar por `active` (FR-015) | US1 |
| Invitación pendiente duplicada → rechazo | Índice único parcial + `IntegrityError` → `409` (research.md Decisión 3) | US1 |
| Envío de correo síncrono; fallo = invitación no utilizable + error explícito | `send_email()` antes de `commit()`, `rollback()` si falla (research.md Decisión 4) | US1 |
| Consumo crea `User` + activa `must_change_password` + marca `consumed` en una sola operación | `POST /auth/login`, bloque `WITH FOR UPDATE` (research.md Decisión 7) | US2 |
| Cambio obligatorio de contraseña tras el primer login | Comportamiento ya existente (spec 001/031), sin modificar | US2 |
| Contraseña temporal reenviada/cancelada dejada de servir de inmediato | Reenvío sobrescribe `password_hash`; cancelación cambia `status` — ambos leídos por el login antes de `verify_password` | US2, US4 |
| Listado muestra pendientes junto a activos, diferenciados | `GET /invitations` (nuevo) + `GET /users` (existente), dos secciones en la UI (research.md Decisión 13) | US3 |
| Reenvío genera contraseña nueva, invalida la anterior, actualiza `sent_at`, envía correo | `POST /invitations/{id}/resend` | US4 |
| Reenvío que falla no pierde la contraseña anterior | `send_email()` antes de sobrescribir la fila (research.md Decisión 10) | US4 |
| Cancelación bloquea el uso inmediato de la contraseña temporal | `POST /invitations/{id}/cancel`, `status='cancelled'`, verificado por el login | US4 |
| Correo único por tenant, no global; mismo correo en dos tenants no colisiona | Todas las consultas de unicidad filtran por `tenant_id` (`UserInvitation.email` + `User.email`) | Edge Cases |
| Normalización de correo (trim + minúsculas) en toda comparación/creación | Aplicado antes de cualquier `SELECT`/`INSERT` sobre `UserInvitation.email` | FR-016 |
| Contraseña temporal sin caducidad por tiempo | Ningún campo de expiración en `UserInvitation` — solo `status` determina si sirve | FR-017 |
