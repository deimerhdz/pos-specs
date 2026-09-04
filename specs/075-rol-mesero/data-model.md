# Data Model: Rol Mesero con Acceso Restringido

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

No se crea ninguna entidad nueva ni se modifica el esquema de ninguna tabla.
Esta funcionalidad agrega un **valor de dato** (una fila) a una entidad ya
existente y usa la relación ya existente entre `Usuario` y `Rol` para
determinar el alcance de acceso de cada usuario.

## Rol (`shared.roles`, modelo `Role`)

Catálogo de roles asignables a un usuario. Sin cambios de esquema.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID | sin cambio |
| `name` | `String(150)` | gana un tercer valor asignable desde el panel de tenant: `"MESERO"` (junto a `"ADMIN"` y `"CASHIER"`; `"SUPER_ADMIN"` sigue sin ser asignable desde ese panel) |
| `active` | `bool` | sin cambio; la fila nueva se crea con `active=true` |

**Compatibilidad con datos existentes**: agregar la fila `MESERO` no toca
ninguna fila existente de `shared.roles`, ni ningún `User`/`UserInvitation`
que ya referencie `ADMIN`/`CASHIER`/`SUPER_ADMIN` por `role_id`. Es
estrictamente aditivo.

**Estrategia de migración**: un `INSERT` idempotente (verifica por `name`
antes de insertar, mismo criterio que `_seed_shared_data()`) contra
`shared.roles`, en una migración de Alembic nueva encadenada después del
`head` actual (`f3a9c1b7e2d4`). No requiere `ALTER TABLE` porque `name` ya es
`String(150)` sin restricción `CHECK`/enum a nivel de base de datos — ver
research.md, D1.

**Estrategia de rollback**: la migración de bajada elimina la fila `MESERO`
únicamente si ningún `User` ni `UserInvitation` la referencia todavía (FK
entrante); si alguno la referencia, la bajada debe fallar en vez de dejar un
`role_id` huérfano.

## Usuario (`User`)

Sin cambios de esquema. `User.role_id` ya es una FK a `Role` — un usuario
puede tener `role_id` apuntando a la fila `MESERO` exactamente igual que hoy
puede apuntar a `ADMIN` o `CASHIER`; ninguna validación adicional a nivel de
modelo distingue un valor de otro.

**Transición de estado**: un cambio de rol (`PATCH /users/{id}/role`, o la
aceptación de una invitación con `role=MESERO`) es una actualización simple
de `role_id` — no dispara ninguna migración de datos ni afecta ninguna otra
entidad relacionada con el usuario (turnos de caja abiertos, órdenes
tomadas, etc. no se reasignan ni se ven afectadas por un cambio de rol).

## Regla de autorización por rol (no es una entidad de base de datos)

El "alcance" de Mesero —qué puede y qué no puede alcanzar— **no se modela
como datos** (no hay una tabla de permisos por rol): es una lista estática,
definida en código, de pares (método HTTP, plantilla de ruta) permitidos
para el rol `MESERO`, evaluada en cada solicitud dentro de
`get_current_user()` (research.md, D2). La lista completa vive en
[contracts/backend-endpoint-access.md](./contracts/backend-endpoint-access.md).

Se documenta aquí por completitud del modelo de datos de la funcionalidad,
no porque introduzca una entidad nueva: es la única pieza de "conocimiento
de negocio" que esta funcionalidad agrega además de la fila `Rol`.
