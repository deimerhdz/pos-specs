# Contrato: contexto de host del SPA → cabecera hacia el API

**Spec**: FR-001, FR-006, FR-007, FR-009 | **Decisión**: [research.md](../research.md) D4

Aplica a `pos-heladeria`: `core/tenant/tenant-resolver.ts`, `tenant-context.service.ts`, `auth-token.interceptor.ts` y los guards.

## Configuración (`environment`)

| Campo | Producción | Desarrollo | Cambio |
|---|---|---|---|
| `rootDomain` | `skeilopos.com` | `localhost` | sin cambio |
| `devRootHosts` | — | — | **eliminado** (ya no hay "modo local" de plataforma; el usuario lo descartó el 2026-10-02) |
| `platformSlug` | `admin` | `admin` | **nuevo** (interfaz `AppEnvironment`) |
| `reservedSlugs` | `www, app, admin, assets, api, docs` | igual | **ampliada** (lista única, FR-009) |

## Resolución (`resolveTenantContext(hostname)`)

Se normaliza a minúsculas y sin espacios; `location.hostname` ya no trae puerto.

| Host | Contexto | `X-Tenant-Host` hacia el API |
|---|---|---|
| `admin.skeilopos.com` | **SUPER_ADMIN** | `admin` |
| `admin.localhost` | **SUPER_ADMIN** | `admin` |
| `localhost`, `127.0.0.1` | **UNRECOGNIZED** | *(no se manda)* |
| `acme.skeilopos.com`, `acme.localhost` | **TENANT** (`slug = acme`) | `acme` |
| `skeilopos.com` (raíz) | **UNRECOGNIZED** | *(no se manda)* |
| `www.`, `app.`, `assets.`, `api.`, `docs.` + `skeilopos.com` / `.localhost` | **UNRECOGNIZED** | *(no se manda)* |
| cualquier otro host (IP, `*.pages.dev`, dominio ajeno) | **UNRECOGNIZED** | *(no se manda)* |
| multinivel (`x.admin.skeilopos.com`) | se toma la **primera** etiqueta como hoy (`x`) → TENANT `x` | `x` |

**Cambios de comportamiento respecto de hoy** (todos amparados por A-99): el dominio raíz y los hosts desconocidos **dejan de ser** SUPER_ADMIN; `www`/`app` **dejan de ser** SUPER_ADMIN; `admin` **pasa de** TENANT a SUPER_ADMIN.

## Estado UNRECOGNIZED

- `/login` muestra una tarjeta "Esta dirección no corresponde a ningún acceso" **sin formulario** y no hace llamadas al API.
- Cualquier ruta protegida (`/super-admin`, `/dashboard`) redirige a `/login`.
- `menu/t/:token` (comensal) **sigue funcionando** en cualquier host: el token firmado lleva el negocio y esas rutas no usan `X-Tenant-Host`.

## Servicio y guards

- `TenantContextService` añade `isTenant` (kind = TENANT) y `tenantHostHeader` (`slug` | `platformSlug` | `null`). `isSuperAdmin` y `tenantSlug` conservan su significado.
- `tenantDomainGuard`: pasa **solo** si `isTenant()` (antes pasaba con cualquier no-SuperAdmin).
- `superAdminDomainGuard`: sin cambio (`isSuperAdmin()`).
- **FR-007 en pantalla**: el almacenamiento de sesión es por origen (cada subdominio, el suyo); una sesión de plataforma no existe en un subdominio de negocio y viceversa. El guard correspondiente redirige a `/login` si alguien fuerza la URL.

## Interceptor (`decorate`)

Usa `tenantHostHeader()` en vez de `tenantSlug()`: manda `X-Tenant-Host: <valor>` solo si no es `null`. El resto (Bearer, refresco, rutas del comensal) no cambia. Sin cabecera, el login responde 401 (contrato del login, fila 8).

## Pruebas requeridas

- `tenant-resolver.spec.ts`: la tabla completa de arriba (los casos que hoy afirman SuperAdmin para raíz/`www`/desconocido se **actualizan citando A-99**) y el caso multinivel.
- `auth-token.interceptor.spec.ts`: cabecera `admin` en plataforma, `acme` en negocio, ausente en no reconocido.
- `login.component.spec.ts`: tarjeta de dirección no válida (sin formulario) y badge "Acceso Super Admin" en plataforma; redirección al panel tras el login.
- Guards: `tenantDomainGuard` rechaza UNRECOGNIZED y SUPER_ADMIN.
- Prueba de paridad: `reservedSlugs` de ambos `environment` igual a la lista literal de [tenant-creation-reserved-hosts.md](./tenant-creation-reserved-hosts.md).
