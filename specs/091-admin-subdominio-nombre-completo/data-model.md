# Data Model: Acceso de Super Admin y Nombre Completo en Invitaciones

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Research**: [research.md](./research.md)

Un único cambio de esquema, aditivo y reversible. Todo lo demás de esta spec es lógica y presentación.

## Cambio de esquema

### `shared.user_invitations` — columna nueva

| Campo | Tipo | Nulo | Default | Comentario |
|---|---|---|---|---|
| `name` | `VARCHAR(100)` | **Sí** | ninguno (`NULL`) | Nombre completo capturado al invitar, ya recortado y en NFC. `NULL` en toda invitación anterior a esta spec. |

- **No** hay índice (no se busca por nombre) **ni** restricción de unicidad (FR-014: el nombre no es único).
- **Sin `CHECK`** de formato en la base: la regla vive en la aplicación (`person_name.py`) y se aplica antes de persistir. El tope de 100 lo fija el tipo de columna; el formato no se duplica en SQL para no tener dos fuentes de verdad.
- El modelo ORM añade `name: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)`.

### Migración

- Archivo nuevo en `pos-backend/alembic/versions/`, `down_revision = 'a89f0c1d2e3b'` (cabecera única actual, spec 089), nombre sugerido `<rev>_091_invitation_full_name.py`.
- `upgrade()`: `op.add_column('user_invitations', sa.Column('name', sa.String(100), nullable=True), schema='shared')`.
- `downgrade()`: `op.drop_column('user_invitations', 'name', schema='shared')`.
- **Sin backfill** y sin tocar filas existentes: no cambia el significado histórico de ningún dato (Principio VIII).
- **Compatibilidad con datos existentes**: las invitaciones `pending` anteriores quedan con `name = NULL`, se siguen aceptando igual (FR-024) y la cuenta resultante usa el correo como nombre (comportamiento actual). Las cuentas ya creadas **no se modifican**.
- **Rollback**: ejecutar `downgrade` tras revertir el código del backend. Se pierde únicamente el nombre capturado de invitaciones aún pendientes (la invitación y su contraseña temporal siguen vigentes). Con la columna ya retirada, el código anterior funciona sin cambios porque nunca la lee.
- **Tenant schemas**: la tabla vive en `shared`; no hay migración por tenant.
- **Orden de despliegue**: la migración va **antes** del backend nuevo (el modelo nuevo selecciona la columna).

## Entidades (visión funcional)

### Invitación (`UserInvitation`) — ampliada

| Atributo | Cambio | Reglas |
|---|---|---|
| `name` | **nuevo**, opcional en BD | Obligatorio **al crear** (validación de [contracts/full-name-rules.md](./contracts/full-name-rules.md)). Inmutable tras crearse; **reenviar** no lo cambia (FR-018). |
| resto (`email`, `role_id`, `password_hash`, `status`, `sent_at`, …) | sin cambio | Una sola `pending` por (tenant, correo). |

**Transiciones** (sin cambio de estados `pending → consumed | cancelled`): al `consumed`, se crea el `User` con `name = invitation.name` si existe; si es `NULL`, `name = invitation.email` (legacy).

### Cuenta (`User`) — sin cambio de esquema

`users.name` ya es `VARCHAR(150) NOT NULL`; 100 ≤ 150 por lo que un nombre válido siempre cabe. **"Sin nombre propio"** = nombre vacío o igual al correo (Suposición de la spec): se presenta con el correo, **sin migrar** nada (FR-022).

### Negocio (`Tenant`) — sin cambio de esquema

`tenants.host` conserva su forma. La reserva se aplica **al crear** (schema de entrada), no con una restricción en BD. Verificación previa al despliegue: ningún negocio existente usa un host reservado (consulta en [quickstart.md](./quickstart.md)).

## Reglas de validación (resumen; la definición completa y los vectores están en `contracts/`)

| Dato | Regla | Dónde se aplica | Error |
|---|---|---|---|
| Nombre completo | recortar + NFC → obligatorio → 2–100 caracteres → lista blanca de letras latinas, espacio, `'`, `’`, `-` con ≥ 2 letras | servidor (siempre) y pantalla | 422 / mensaje en el campo |
| `host` de negocio nuevo | `strip().lower()` no está en `{www, app, admin, assets, api, docs}` | servidor (schema) y pantalla | 422 "«x» es una palabra reservada…" |
| Ámbito del login | `x-tenant-host` = `admin` → plataforma; slug registrado → ese negocio; otro caso → rechazo | servidor | 401 `Invalid credentials` |

## Modelos de presentación (frontend, sin persistencia)

- `PendingInvitation`: `{ id, email, name: string | null, role_name, sent_at }`.
- `InvitationForm` / `InvitationCreatePayload`: añaden `name: string` (ya recortado al enviar).
- `TenantContext`: añade la variante `{ kind: 'UNRECOGNIZED', hostname }` (ver [contracts/tenant-context-resolution.md](./contracts/tenant-context-resolution.md)).
- `displayName(name, email)` e `initialOf(name, email)`: funciones puras, ver D9 de [research.md](./research.md).
