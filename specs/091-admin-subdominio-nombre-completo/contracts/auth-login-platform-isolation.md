# Contrato: `POST /api/v1/auth/login` — aislamiento plataforma / negocio

**Spec**: FR-001…FR-007, FR-010 | **Decisión**: [research.md](../research.md) D2

## Qué cambia y qué no

- **No cambia**: ruta, cuerpo (`{email, password}`), forma de la respuesta 200 (`message`, `access_token`, `refresh_token`, `user`), claims del token, `403 User account is inactive`, `503`/`500` de base de datos, consumo de invitación pendiente (spec 037).
- **Cambia**: cómo se decide **en qué ámbito se busca la cuenta**. Antes: sin cabecera o con host no registrado ⇒ buscar entre Super Admin. Ahora: solo el host explícito de plataforma abre ese ámbito.

## Entrada que decide el ámbito

Cabecera `x-tenant-host` (el puerto, si viene, se descarta como hoy: `host.split(":", 1)[0]`). Para reconocer la plataforma se compara `host.strip().lower() == "admin"` (`PLATFORM_HOST`, definido en `app/core/reserved_hosts.py`). La búsqueda de un negocio sigue siendo **por igualdad exacta** con `Tenant.host`, como hoy (FR-010).

## Tabla de decisión (única fuente de verdad)

Cuentas: **SA** = Super Admin (`tenant_id IS NULL`); **UN** = usuario de un negocio registrado `acme`.

| # | `x-tenant-host` | Cuenta que envía credenciales correctas | Resultado |
|---|---|---|---|
| 1 | `admin` | SA activa | **200**, token con `is_super_admin: true` *(Escenario Admin 1)* |
| 2 | `admin` | SA inactiva | **403** `User account is inactive` *(Historia 1.4, igual que hoy)* |
| 3 | `admin` | UN de `acme` | **401** `Invalid credentials` — no se busca en ningún negocio *(Admin 2)* |
| 4 | `acme` | UN activo de `acme` | **200**, igual que hoy *(FR-010)* |
| 5 | `acme` | SA | **401** `Invalid credentials` *(Admin 3)* |
| 6 | `no-existe` | SA | **401** `Invalid credentials` *(FR-005)* |
| 7 | `no-existe` | UN | **401** `Invalid credentials` |
| 8 | ausente / vacío | SA o UN | **401** `Invalid credentials` *(FR-005/FR-006: el dominio raíz y los clientes sin cabecera no tienen acceso)* |
| 9 | cualquiera | credenciales incorrectas | **401** `Invalid credentials` (sin cambio) |
| 10 | `acme` | invitación pendiente con contraseña correcta | **200** y se crea la cuenta (sin cambio; spec 037) |
| 11 | `admin` | correo con invitación pendiente en algún negocio | **401** — nunca se busca invitación sin negocio resuelto *(ya hoy)* |

## Invariantes

- **I-1 Indistinguibilidad**: los casos 3, 5, 6, 7, 8, 9 devuelven **exactamente** el mismo status y cuerpo `{"detail":"Invalid credentials"}`. El cuerpo no varía por el motivo.
- **I-2 Motivo solo en el log**: el servidor registra (nivel `info`) un `reason` interno (`platform`, `tenant`, `no_host`, `unknown_host`) y el correo ya registrado hoy; **nunca** la contraseña.
- **I-3 Sin efecto lateral en rechazo**: un rechazo por ámbito no crea `User`, no consume invitación ni toca la base fuera de la lectura.
- **I-4 Un solo ámbito por petición**: nunca se consulta `tenant_id IS NULL` y un negocio en la misma petición.
- **I-5 FR-007 sin cambios**: el token de plataforma no pasa `get_current_user` (exige `User.tenant_id == tenant.id`) y el de negocio no pasa `get_current_super_admin` (exige `is_super_admin`). Las pruebas de aislamiento lo **verifican**, no lo reimplementan.

## Pruebas requeridas (`test_auth_login_platform_isolation.py`)

Una prueba por fila 1–8 y 10–11, reutilizando el patrón de `test_auth_login_invitation_consumption.py` (parchea `with_db` con la sesión SQLite de `auth_fixtures`). Más: (a) los cuatro cuerpos de rechazo son idénticos byte a byte; (b) `ADMIN`/` admin ` (mayúsculas/espacios) cuentan como plataforma; (c) `admin:4200` (con puerto) cuenta como plataforma; (d) un negocio cuyo `host` es `admin-prueba` **no** se confunde con la plataforma.

## Sin cambios en otros endpoints

`get_tenant` (cabecera → negocio) no cambia: con `x-tenant-host: admin` devuelve 404 como para cualquier host sin negocio, que es lo correcto (la plataforma no tiene endpoints de negocio). `/auth/refresh-token`, `/auth/logout` y `/auth/change-password` no dependen del host. `/auth/forgot-password` exige un negocio (`get_tenant`): para el Super Admin **ya hoy** no aplica y queda **fuera de alcance**.
