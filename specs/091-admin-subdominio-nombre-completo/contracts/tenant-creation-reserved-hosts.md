# Contrato: alta de negocio — subdominios reservados

**Spec**: FR-008, FR-009 | **Decisión**: [research.md](../research.md) D3 | Anomalía: A-101

## Lista reservada (única, coherente en servidor y pantalla)

`www`, `app`, `admin`, `assets`, `api`, `docs`

- Fuente en el servidor: `RESERVED_SUBDOMAINS` en `app/core/reserved_hosts.py`.
- Fuente en el frontend: `environment.reservedSlugs` (ambos entornos, misma lista) + `environment.platformSlug = 'admin'`.
- Una prueba por repositorio **fija la lista literal**; cambiar uno sin el otro rompe un test.

## `POST /api/v1/super-admin/tenants`

Cuerpo sin cambios (`TenantCreateWithUser`). Validación nueva sobre `host`:

- Se compara `host.strip().lower()` contra la lista. Coincidencia ⇒ **422**, `detail[0].loc = ["body","host"]`, `detail[0].msg` = `Value error, «admin» es una palabra reservada y no puede usarse como subdominio` (con el valor ya normalizado entre «»).
- `Admin`, `ADMIN`, ` admin `, `Api`, ` DOCS ` ⇒ rechazados. `admin2`, `admin-prueba`, `mi-api`, `docs1` ⇒ **aceptados** (la reserva es por igualdad, no por prefijo).
- Se rechaza **antes** de crear el schema, el negocio o el usuario administrador (el validador corre en el modelo de entrada; `tenant_create()` ni se invoca). Es el único llamador de `tenant_create` (research H-7), por lo que no hace falta una segunda guarda.
- El valor **guardado** no se transforma (se conserva el comportamiento actual de `host`); la normalización solo se usa para comparar.
- No cambia nada para hosts no reservados (FR-010).

## Pantalla "Nuevo negocio" (frontend)

- El campo **Host** valida contra `environment.reservedSlugs` (misma normalización: recorte + minúsculas) y muestra: `«admin» es una palabra reservada y no puede usarse como subdominio`; bloquea el envío mientras sea inválido.
- La sugerencia automática del host desde el nombre (`toSlug`) puede producir un reservado (negocio "Admin"); el validador lo marca igualmente.
- El mensaje del servidor (422 con arreglo) se muestra si la petición llegara a pasar la pantalla.

## Datos existentes

Fuera de este contrato: **no** se migra ni se borra ningún negocio. Antes de desplegar se ejecuta la consulta de [quickstart.md](../quickstart.md); si devolviera filas, se resuelve como decisión aparte (Edge Case de la spec).
