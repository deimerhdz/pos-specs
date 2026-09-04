# Contrato: Navegación y rutas del frontend para el rol Mesero

**Spec**: [../spec.md](../spec.md) | **Research**: [../research.md](../research.md) (D4)

Lista autoritativa de qué recibe `UserRole.MESERO` en los tres puntos del
frontend que hoy controlan acceso por rol. Ningún otro punto del frontend
necesita cambios — el resto de rutas ya excluye a Cajero hoy y debe seguir
excluyendo a Mesero de la misma forma (sin acción adicional: si una ruta no
lista `UserRole.MESERO` en su `roleGuard`, ya queda bloqueada).

## `NAV_ITEMS` (`src/app/core/config/navigation.config.ts`)

| Ítem | Roles hoy | Roles después |
|---|---|---|
| Terminal de mesas | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| Órdenes | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| (todos los demás) | sin cambio | sin cambio |

## `dashboardRoutes` — `roleGuard([...])` (`src/app/modules/dashboard/routes.ts`)

| Ruta | Roles hoy | Roles después |
|---|---|---|
| `mesas-sesiones` | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| `mesas-sesiones/:tableId/orden-manual` | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| `orders` | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| `orders/:id` | `[ADMIN, CASHIER]` | `[ADMIN, CASHIER, MESERO]` |
| `mi-cuenta`, `mi-plan` | sin `roleGuard` (cualquier autenticado) | sin cambio |
| (todas las demás) | sin cambio | sin cambio |

## `ROLE_DEFAULT_ROUTES` (`src/app/core/guards/role.guard.ts`)

Ruta a la que se redirige un usuario cuando `roleGuard` le deniega el acceso
a la ruta solicitada (y, en la práctica, también su destino tras iniciar
sesión):

| Rol | Ruta por defecto |
|---|---|
| `SUPER_ADMIN` | `/dashboard/admin` (sin cambio) |
| `ADMIN` | `/dashboard/admin` (sin cambio) |
| `CASHIER` | `/dashboard/caja` (sin cambio) |
| `MESERO` (nuevo) | `/dashboard/mesas-sesiones` |

## Gestión de usuarios — opción de rol seleccionable

| Punto | Cambio |
|---|---|
| `RoleName` (`src/app/modules/users/interfaces/user-profile.interface.ts`) | `'ADMIN' \| 'CASHIER'` → `'ADMIN' \| 'CASHIER' \| 'MESERO'` |
| `invitation-form.component.ts` (invitar usuario) | agrega `<option value="MESERO">Mesero</option>` |
| `user-role-modal.component.ts` (cambiar rol) | agrega `<option value="MESERO">Mesero</option>` y reconoce `'MESERO'` en el guard de `ngOnChanges()` |
| `users-page.component.ts` — `ROLE_LABELS` | agrega `MESERO: 'Mesero'` |
| `users-page.component.ts` — clases de badge por rol | agrega una clase de color propia para Mesero (distinta de Admin/Cajero) |

## Mapeos exhaustivos de `UserRole` fuera de gestión de usuarios/navegación

`UserRole` es un enum de TypeScript, y varios puntos del frontend (ajenos al
alcance por pantalla) lo usan como clave de un `Record<UserRole, string>`
**exhaustivo** — el compilador de Angular (`ng build`) exige una entrada por
cada valor del enum, así que agregar `MESERO` al enum obliga a completar
estos también, aunque no formaban parte del research original de esta spec
(surgieron al correr `ng build --configuration production` durante la
implementación, no durante el diseño):

| Archivo | Constante | Valor agregado para Mesero |
|---|---|---|
| `src/app/core/guards/password-change.guard.ts` | `ROLE_HOME` | `/dashboard/mesas-sesiones` |
| `src/app/modules/auth/pages/change-password.component.ts` | `ROLE_HOME` | `/dashboard/mesas-sesiones` |
| `src/app/modules/auth/pages/login.component.ts` | `ROLE_HOME` | `/dashboard/mesas-sesiones` (destino real tras iniciar sesión) |
| `src/app/modules/account/pages/account-settings.component.ts` | `ROLE_LABEL` | `'Mesero'` |
| `src/app/modules/dashboard/layout/header.component.ts` | `ROLE_LABELS` | `'Mesero'` |
| `src/app/modules/dashboard/layout/header.component.ts` | `ROLE_BADGE_CLASSES` | clase de color propia |
| `src/app/modules/settings/pages/tenant-info.component.ts` | `ROLE_LABEL` | `'Mesero'` |
| `src/app/modules/super-admin/pages/super-admin-users-page.component.ts` | `ROLE_LABELS` | `'Mesero'` |

Sin estos, `ng build` falla con `TS2741` y, más importante, el login de un
usuario Mesero rompería en tiempo de ejecución al no encontrar su entrada en
`ROLE_HOME` de `login.component.ts` — es el mapa que realmente decide a dónde
aterriza un usuario justo después de autenticarse.
