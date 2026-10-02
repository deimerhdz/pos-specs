# Notas de implementación — spec 091

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Tareas**: [tasks.md](./tasks.md)

Ramas de código: `feat/091-platform-host-invitation-name` en `pos-backend` y `pos-heladeria` (desde `develop`).

## Diagnóstico del 404

**Estado**: parcial. Lo que se pudo comprobar sin credenciales reales:

- **API local (HP-1, mitad del backend)**: `GET /api/v1/users` con `X-Tenant-Host: admin` responde
  `404 {"detail":"Tenant not found for host 'admin'"}`. Es exactamente el 404 que describe research H-4:
  `get_tenant` no halla un negocio llamado `admin` y toda llamada de negocio falla.
- **Lectura de código**: el resolver del frontend trata `admin` como slug de negocio (no está en
  `reservedSlugs`), el interceptor manda `X-Tenant-Host: admin` y el login del backend, al no hallar ese
  negocio, busca entre los Super Admin (H-1). Tras entrar, el SPA se cree en contexto negocio y navega a
  `/dashboard/admin`.
- **Sondas de producción (research H-5, 2026-10-02)**: `admin.skeilopos.com/login` y rutas profundas
  responden 200 (relleno de SPA activo), lo que descarta HP-2 en ese momento.
- **No comprobado**: el recorrido completo en navegador sobre `https://admin.skeilopos.com/login` con
  DevTools (T005) requiere credenciales reales de Super Admin de producción, que no se usan en comandos
  ni notas. **Pendiente de que el dueño de la spec lo ejecute** según `quickstart.md` §2 y anote aquí qué
  petición devuelve 404 y su cuerpo. Hipótesis más probable: HP-1.
- El login local del Super Admin con las credenciales del `.env` devolvió 401 en el backend en marcha
  (puerto 8000), así que tampoco se pudo reproducir el recorrido completo en `admin.localhost:4200`.

T005b: no aplica mientras no se confirme HP-2/HP-3 (la evidencia disponible apunta a HP-1).

## Decisiones

- Historia 6 sigue la **opción A** de research D11 por defecto (solo frontend; el guardado del formulario
  del Super Admin seguirá respondiendo 405 hasta que el dueño decida B o C). **O-1 abierta.**
- Anomalías A-99…A-102 registradas en `specs/000-reconocimiento/registro-de-anomalias.md` antes de tocar
  código (numeración libre confirmada: la última era A-98).
- **T006 (verificación previa de datos)**, BD de desarrollo, 2026-10-02:
  `SELECT id, name, host FROM shared.tenants WHERE lower(btrim(host)) IN ('www','app','admin','assets','api','docs');`
  → **0 filas** (la BD tiene 1 negocio). **Pendiente ejecutar la misma consulta en producción antes de
  desplegar.**

### Corrección posterior (2026-10-02): se elimina el "modo local" de plataforma

Al probar la implementación, el usuario abrió `http://localhost:4200` y llegó al login de Super Admin. Era
el "modo local" (`devRootHosts`: `localhost`/`127.0.0.1` → SUPER_ADMIN) que el plan había supuesto y que
contradecía lo aclarado: al login de plataforma solo se llega con la URL con `admin`. Se retiró de la spec
(Clarifications, edge case de `localhost`), `research.md` (D4), el contrato `tenant-context-resolution.md`,
`quickstart.md` y `tasks.md` (T021), y del código (`devRootHosts` eliminado del resolver, la interfaz, ambos
entornos y el inicializador). Ahora `localhost` y `127.0.0.1` son "host no reconocido": aviso sin formulario ni
llamadas al API. Verificado en navegador (`localhost:4200/` y `/login`) y con pruebas del resolver
(incluido el entorno de desarrollo con `rootDomain: 'localhost'`). En desarrollo la plataforma se prueba en
`admin.localhost:4200`.

### Hallazgos durante la implementación

- **El handler de errores 422 de `/super-admin` reventaba como 500** cuando un validador lanzaba
  `ValueError` (`ctx.error` no es serializable). Se descubrió al probar el rechazo de hosts reservados por
  HTTP (los tests unitarios del schema no pasan por ese handler). Corrección mínima en
  `app/core/error_response.py` (`jsonable_encoder(..., custom_encoder={BaseException: str})`) con prueba en
  `test_reserved_hosts.py`. Las invitaciones (fuera de ese prefijo) ya respondían 422 bien. El frontend
  lee ahora `error.details.errors[0].msg` en `TenantService`.
- **Bucle de redirección evitado**: con el estado nuevo UNRECOGNIZED, `redirectIfAuthGuard` enviaría a un
  usuario con sesión a `/dashboard`, y `tenantDomainGuard` lo devolvería a `/login` (bucle). Se corrigió en
  `auth.guard.ts` (en UNRECOGNIZED `/login` se muestra tal cual).
- **Preexistente, no corregido (fuera de alcance)**: el layout compartido del dashboard dispara en el
  contexto de plataforma llamadas de negocio (`/notifications`, `/tenant`, `/plan`, `/realtime/ticket`)
  que responden 404 (`Tenant not found for host 'admin'`). El panel renderiza bien; son llamadas de fondo.
  Conviene una spec aparte (no mostrarlas/llamarlas en contexto de plataforma). Puede ser parte de lo que el
  usuario percibía como "404" tras entrar por `admin`.
- Fallos preexistentes del frontend (19 en 6 archivos: `app.spec`, `auth.service.spec` —sin proveedor
  `SwPush`—, `menu.service.spec`, `tenant.service.spec` —URL `/admin/tenants` vieja—,
  `pos-checkout-panel` y `pos-order-panel`): idénticos con y sin los cambios de esta spec (verificado con
  `git stash`).

## Verificación automática

- Backend (`python -m unittest discover -s app/characterization_tests -t .`): **1260 pruebas, OK** (antes
  de la spec: 1259 sin las nuevas; nuevas: `test_reserved_hosts`, `test_person_name`,
  `test_auth_login_platform_isolation`, `test_invitation_email_greeting` + ampliaciones de invitaciones y
  consumo en login). Ningún test existente se debilitó; las llamadas de creación de invitación ahora envían
  `name`.
- Migración `c91a4e7b2d58`: `alembic heads` con cabecera única; `upgrade head` → `downgrade -1` →
  `upgrade head` verificados en la BD de desarrollo.
- Frontend (`ng test`): **1213 pasan, 19 fallan (preexistentes, ver arriba), 1 omitida** (antes: 1098
  pasan, 19 fallan). `ng build` OK (advertencias de presupuesto y `qrcode` preexistentes).

## Verificación manual

Hecha el 2026-10-02 contra el backend local (puertos 8000 y 8001) y `ng serve`, con dos usuarios
temporales `091-test-*@pruebas091.com` creados y **ya borrados** de la BD de desarrollo.

- **API (T043)**: SA@`admin` → 200; SA@`ADMIN:4200` → 200; negocio@`admin` → 401; SA@`heladeria` → 401;
  SA@host inexistente → 401; SA sin cabecera → 401; negocio@`heladeria` → 200; los cuatro 401 tienen el
  mismo cuerpo (mismo hash). Alta de negocio con `admin`, `ADMIN`, ` api `, `assets`, `docs` → 422 con
  «…es una palabra reservada y no puede usarse como subdominio», sin crear negocio (tras la corrección del
  handler). Invitación sin `name` → 422 `Value error, El nombre es obligatorio`.
- **Navegador (T043)**: `admin.localhost:4200/login` muestra "Acceso Super Admin"; el login lleva a
  `/super-admin/tenants`; recargar `/super-admin/users` conserva la sesión y lista usuarios;
  `www.localhost:4200/login` muestra "Esta dirección no corresponde a ningún acceso" sin formulario;
  `heladeria.localhost:4200/login` muestra la insignia del negocio; un SA en el login del negocio recibe
  "Credenciales incorrectas"; un usuario del negocio entra a `/dashboard/admin`.
- **Usuarios del negocio (parte de T069)**: el formulario "Invitar usuario" tiene "Nombre completo" primero;
  vacío, `A`, `Ana3` y `<script>…` muestran el mensaje correcto y **no** envían la petición; una cuenta
  anterior (nombre = correo) se ve con su correo e inicial.
- **No ejecutado (pendiente del dueño)**: envío real de invitación y aceptación con correo (no hay servicio
  de correo local), reenvío con/sin nombre en pantalla, recorrido completo de `quickstart.md` §6 y §7
  (formulario del Super Admin; su guardado sigue dando 405, H-10/A-102) (`localhost:4200` ya no es un modo local: ver corrección posterior).

## Verificación en producción

Pendiente: T080 solo tras desplegar, a petición del usuario.

## Plan de despliegue (T079, `research.md` D12)

1. Consulta del T006 en **producción** → debe dar 0 filas.
2. Migración `c91a4e7b2d58` (`alembic upgrade head`).
3. Backend.
4. Frontend (3 y 4 seguidos: un SPA antiguo en caché manda invitaciones sin nombre → 422 hasta recargar).

Rollback: revertir frontend y backend; luego `alembic downgrade -1` (borra solo `user_invitations.name`).

## Limitaciones

- `X-Tenant-Host` lo fija el cliente: el aislamiento es política del producto, no una frontera
  criptográfica; la protección real sigue siendo credenciales + rol + `tenant_id` del token.
- Ventana de 422 con SPA antiguo en caché entre desplegar backend y frontend.
- O-1 abierta: el formulario del Super Admin valida el nombre pero guardar da 405 hasta decidir A/B/C.
- Cuentas del Super Admin abiertas desde el dominio raíz u otros hosts: el SPA ya no las trata como
  plataforma; hay que entrar de nuevo por `admin.`.
