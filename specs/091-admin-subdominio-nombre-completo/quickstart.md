# Quickstart: verificar el acceso de plataforma y el nombre completo

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Contratos**: [contracts/](./contracts/)

Guía de **validación** (no de implementación). Cada bloque indica qué historia/escenario del brief prueba.

## 0. Prerrequisitos

- `pos-backend` en la rama `feat/091-platform-host-invitation-name` y `pos-heladeria` en la suya (ambas desde `develop`). Base de datos de desarrollo migrada (`alembic upgrade head`) y servicios habituales (Postgres, Redis, API en `http://localhost:8000`).
- Un Super Admin sembrado (`app/scripts/seed_super_admin.py`) y al menos un negocio de prueba, p. ej. `acme`, con un usuario ADMIN.
- Navegador de escritorio. En Chromium/Firefox, `*.localhost` resuelve a la máquina local sin tocar `hosts`.
- Cliente HTTP para las pruebas de API (`curl`).

## 1. Verificación previa al despliegue (datos) — bloqueante

Confirmar que **ningún negocio existente** usa un host reservado (Suposición "Datos existentes"):

```sql
SELECT id, name, host FROM shared.tenants
WHERE lower(btrim(host)) IN ('www','app','admin','assets','api','docs');
```

**Esperado: 0 filas.** Si devuelve alguna, **no desplegar**: se resuelve como decisión aparte (la spec no migra ni borra negocios).

## 2. Diagnóstico del 404 (primera tarea, antes de corregir) — Historia 1

Sobre el **estado actual** (`develop` sin cambios), en `https://admin.skeilopos.com/login` con DevTools → Network:

1. Cargar `/login` con "Disable cache" y datos de sitio limpios → ¿el documento y los `.js` responden 200? (descarta HP-2/HP-3).
2. Iniciar sesión como Super Admin → anotar **qué petición** devuelve 404 y su cuerpo. Esperado según HP-1: `Tenant not found for host 'admin'` en una llamada de negocio.
3. Anotar la URL final del navegador (esperado: `/dashboard/admin`, no `/super-admin`).
4. Recargar una ruta profunda (`/super-admin/tenants`) → anotar el resultado.

Registrar todo en `implementation-notes.md`. La corrección se escribe **después** de este registro.

## 3. Pruebas automáticas

**Backend** (`pos-backend`):

```bash
python -m unittest app.characterization_tests.test_auth_login_platform_isolation -v
python -m unittest app.characterization_tests.test_reserved_hosts -v
python -m unittest app.characterization_tests.test_person_name -v
python -m unittest app.characterization_tests.test_invitation_email_greeting -v
python -m unittest app.characterization_tests.test_invitations_create app.characterization_tests.test_invitations_list app.characterization_tests.test_invitations_resend_cancel app.characterization_tests.test_auth_login_invitation_consumption -v
```

Luego la suite completa de `characterization_tests` (evidencia de Principio III: ningún `CONGELA` en rojo).

**Frontend** (`pos-heladeria`): `ng test` (suite completa) y `ng build` sin errores de tipos.

**Esperado**: todo en verde; las pruebas de reservados y del nombre recorren los vectores de [contracts/full-name-rules.md](./contracts/full-name-rules.md) y la lista literal de [contracts/tenant-creation-reserved-hosts.md](./contracts/tenant-creation-reserved-hosts.md).

## 4. Aislamiento por API (`curl`) — Historia 2, SC-002

Con el API local (`$API=http://localhost:8000/api/v1`), `SA` = credenciales del Super Admin, `UN` = las de un usuario de `acme`. Resultado esperado según [contracts/auth-login-platform-isolation.md](./contracts/auth-login-platform-isolation.md):

| Petición (`POST $API/auth/login`) | Esperado |
|---|---|
| `X-Tenant-Host: admin` + SA | **200**, `user.is_super_admin = true` |
| `X-Tenant-Host: admin` + UN | **401** `Invalid credentials` *(Admin 2)* |
| `X-Tenant-Host: acme` + SA | **401** `Invalid credentials` *(Admin 3)* |
| `X-Tenant-Host: inexistente` + SA | **401** `Invalid credentials` *(FR-005)* |
| *(sin cabecera)* + SA | **401** `Invalid credentials` *(FR-006)* |
| `X-Tenant-Host: acme` + UN | **200**, igual que antes *(FR-010)* |

Comprobar que los cuerpos de los cuatro 401 son **idénticos**.

Reservados — `POST $API/super-admin/tenants` con token de Super Admin y `host` = `admin`, `ADMIN`, ` api `, `assets`, `docs` → **422** "palabra reservada" en cada caso (y ningún negocio creado); con `admin2` y `mi-api` → pasa la validación del `host`.

## 5. Recorrido manual — Historias 1 y 2 (`pos-heladeria`, `ng serve`)

| Paso | Dirección | Esperado |
|---|---|---|
| 1 | `http://admin.localhost:4200/login` | insignia "Acceso Super Admin"; con SA → llega a `/super-admin/tenants` *(Admin 1)* |
| 2 | Recargar `/super-admin/users` | carga sin 404 y conserva la sesión |
| 3 | `http://admin.localhost:4200/login` con UN | mensaje de credenciales inválidas, sin pista *(Admin 2)* |
| 4 | `http://acme.localhost:4200/login` con SA | credenciales inválidas *(Admin 3)* |
| 5 | `http://acme.localhost:4200/login` con UN | entra como siempre |
| 6 | `http://localhost:4200/login` y `http://127.0.0.1:4200/login` | tarjeta "dirección no válida", sin formulario, sin llamadas al API (no hay "modo local" de plataforma) |
| 7 | `http://www.localhost:4200/login` y `http://otro.local.test:4200/login` | tarjeta "dirección no válida", sin formulario, sin llamadas al API |
| 8 | Con sesión SA en `admin.localhost`, abrir `acme.localhost:4200/dashboard` | redirige a login; no hay acceso |
| 9 | "Nuevo negocio" con host `admin` / `Docs` | bloqueado en pantalla con "palabra reservada" *(Admin 4)* |

## 6. Recorrido manual — Historias 3, 4 y 5 (nombre completo)

Como ADMIN de `acme` en **Usuarios → Invitar usuario**:

| # | Acción | Esperado |
|---|---|---|
| 1 | `María Pérez` + correo + rol → Enviar | invitación creada; el correo (bandeja de pruebas) dice "Hola, María Pérez:"; al primer login de la persona, la lista muestra "María Pérez" con avatar **M** *(Esc. 1)* |
| 2 | Nombre vacío / solo espacios | "El nombre es obligatorio", no se envía *(Esc. 2, 3)* |
| 3 | `A` y 101 letras | "El nombre debe tener entre 2 y 100 caracteres" *(Esc. 4)* |
| 4 | `José Ñañez O'Brien-Díaz` | se envía y se muestra idéntico *(Esc. 5)* |
| 5 | `<script>alert(1)</script>`, `Ana3`, `Ana@` | mensaje de formato; nada guardado ni enviado *(Esc. 6)* |
| 6 | Petición directa sin `name`: `curl -X POST $API/invitations -H 'X-Tenant-Host: acme' -H "Authorization: Bearer <ADMIN>" -d '{"email":"x@acme.com","role":"CASHIER"}'` | **422**, `msg` "…El nombre es obligatorio"; sin invitación ni correo *(Esc. 7)* |
| 7 | Escribir un nombre → **Cancelar** → reabrir | todos los campos y mensajes de error vacíos *(Esc. 9)* |
| 8 | Lista de pendientes | nombre junto al correo; invitaciones anteriores muestran solo el correo |
| 9 | Reenviar una invitación con nombre / una anterior sin nombre | el correo conserva el nombre / usa "Hola:" |
| 10 | Cuentas anteriores en **Usuarios** | muestran correo e inicial del correo, sin errores *(Esc. 8)* |
| 11 | Nombre de 100 caracteres en la lista | la fila no se desborda |
| 12 | Aceptar una invitación **anterior** (sin nombre) | funciona como antes; cuenta con el correo como nombre |

## 7. Recorrido manual — Historia 6 (Super Admin, decisión O-1)

En `admin.localhost:4200/super-admin/users`: abrir **Nuevo usuario** → nombre vacío o inválido muestra los mismos mensajes (6.1, 6.2); abrir **Editar** de un usuario sin nombre → guardar sin tocarlo no se bloquea por el nombre (6.4); la tabla muestra nombre/inicial de los usuarios existentes (6.3, solo con usuarios ya creados).

> **Aviso**: el guardado desde esta pantalla responde **405** porque el endpoint no existe (hallazgo H-10, anomalía A-102); no es un fallo de esta spec. Si se elige la opción B o C de [research.md](./research.md) D11, esta sección se rehace.

## 8. Despliegue y comprobación en producción (solo lectura)

Orden: **(1)** consulta del bloque 1 → **(2)** migración → **(3)** backend → **(4)** frontend, **3 y 4 seguidos** (ventana de 422 con SPA antiguo en caché, research D12).

Tras desplegar (sin credenciales reales en los comandos):

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://admin.skeilopos.com/login                 # 200
curl -s -o /dev/null -w "%{http_code}\n" https://admin.skeilopos.com/super-admin/tenants   # 200 (relleno de SPA)
curl -s -o /dev/null -w "%{http_code}\n" https://admin.skeilopos.com/super-admin/users     # 200
curl -s -o /dev/null -w "%{http_code}\n" https://admin.skeilopos.com/super-admin/plans     # 200 (ajustar a las rutas reales de app.routes.ts)
curl -s -o /dev/null -w "%{http_code}\n" https://admin.skeilopos.com/ruta-que-no-existe   # 200 (relleno de SPA)
curl -s -X POST https://api.skeilopos.com/api/v1/auth/login -H 'content-type: application/json' \
  -H 'X-Tenant-Host: zzz-no-existe' -d '{"email":"nadie@example.com","password":"x"}'      # 401 Invalid credentials
curl -s -X POST https://api.skeilopos.com/api/v1/auth/login -H 'content-type: application/json' \
  -d '{"email":"nadie@example.com","password":"x"}'                                        # 401 Invalid credentials
```

Y en navegador real: el recorrido 1–2 del bloque 5 en `https://admin.skeilopos.com` con la cuenta de plataforma, y un login normal en un negocio existente (SC-003).

## 9. Criterios de salida

- Bloque 1: 0 filas. Bloque 2: registrado en `implementation-notes.md`.
- Bloque 3: todo en verde. Bloques 4–6: todos los pasos con el resultado esperado (13 escenarios del brief cubiertos).
- Bloque 7: resultado y decisión O-1 anotados.
- Bloque 8: comprobaciones de producción OK.
