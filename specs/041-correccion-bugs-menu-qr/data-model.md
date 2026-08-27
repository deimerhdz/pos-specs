# Data Model — Corrección de bugs y mejoras — Menú QR

Ningún bug de esta spec modifica el esquema de base de datos de `pos-backend` (Principio VIII: no
aplica — sin migración, sin campo nuevo, sin tabla nueva). Este documento existe por trazabilidad
(Principio XII): describe las entidades existentes que cada corrección lee/usa sin modificar, y la
única entidad nueva, que vive exclusivamente en el navegador (Bug 1).

## Entidades existentes (sin cambios de esquema)

### SessionParticipant (backend, spec 007)
- Atributos relevantes para esta spec: `status` (`open`/`closed`).
- Uso en esta spec: Bug 1 depende de que `close_participant` (`pos-backend/app/core/qr_context.py:72-93`)
  siga marcando `status = "closed"` al cerrar sesión — comportamiento **protegido**, no modificado
  (FR-001).

### TableSession (backend, spec 007)
- Uso en esta spec: Edge Cases de Bug 1 aclaran que un cambio de `table_session` activa de la mesa
  (mesa liberada y una sesión nueva abierta) **no** levanta automáticamente la marca de acceso
  cerrado (Clarifications, sesión 2026-08-27) — sin cambios en esta entidad.

### DiningTable (backend, `pos-backend/app/models/dining_table.py:15-17`)
- Atributos usados: `number` (identificador único, siempre presente), `name` (opcional).
- Uso en esta spec: Bug 2 los lee para componer el identificador visible en el PNG del QR
  (`Mesa {number}`, con `name` como dato secundario) — sin cambios en el modelo.

### Product (backend, `pos-backend/app/models/product.py`)
- Atributos usados: `image_url` (Bug 3 — determina si se muestra imagen real o placeholder),
  `tracks_inventory` (Bug 4 — determina si el botón "Copiar insumos..." se muestra).
- Sin cambios en el modelo; ambos atributos ya existen y ya se leen en el frontend
  (`draft().tracks_inventory`, `product.image_url`).

### ProductVariant / Insumos (receta fija, grupos de opciones) (backend, spec 003/027)
- Uso en esta spec: Bug 4 solo condiciona la **visibilidad** del botón que copia esta configuración
  entre tamaños; la estructura y persistencia de la receta/insumos no cambia.

## Entidad nueva — exclusivamente de navegador (Bug 1)

### Marca de acceso cerrado (`ExitedAccessMarker`, nombre conceptual)

No es una entidad de base de datos: vive en `sessionStorage` del navegador del comensal, scoped a
la pestaña actual (ver `research.md`, Decisión 1).

| Campo | Tipo | Descripción |
|---|---|---|
| Clave de almacenamiento | `string` | Constante fija, p. ej. `pos.diner.exited_token` (mismo archivo que `STORAGE_KEY` de `diner-token.store.ts`, para mantener el patrón de nombres `pos.diner.*` ya usado). |
| Valor almacenado | `string` (token QR firmado) | El token de la mesa (`:token` de la ruta `/menu/t/:token`) cuyo acceso quedó cerrado en esta pestaña. |

**Ciclo de vida**:
- **Se crea/actualiza**: dentro de `exit()` (`public-menu.component.ts:1013-1032`), en el mismo
  momento en que se invalida el `session_token` — antes de fijar `view.set('error')`.
- **Se consulta**: al inicio de `ngOnInit` (`public-menu.component.ts:651-689`), antes de decidir
  entre `view.set('name')` (primer acceso legítimo) y el nuevo estado de "acceso finalizado"; y en
  `expireSession()` (`:1164-1174`), ver `research.md` Decisión 2.
- **Se destruye**: nunca de forma explícita por esta spec (sin temporizador, sin limpieza por
  cambio de `table_session` — decisión de negocio, Clarifications). Deja de aplicar naturalmente
  cuando el navegador abre un contexto de navegación distinto (pestaña nueva), porque
  `sessionStorage` no se comparte entre pestañas — no hay una operación de "borrado" que
  implementar para ese caso, es una propiedad del propio mecanismo de almacenamiento elegido.
- **Alcance**: por pestaña (`sessionStorage`), no por mesa ni por comensal en base de datos — dos
  pestañas distintas del mismo navegador, aunque compartan `localStorage`, no comparten esta marca.

**Validación**: ninguna validación de formato adicional — se compara por igualdad de string contra
el `:token` de la ruta actual antes de decidir el `view`.

**Relación con entidades existentes**: no tiene clave foránea ni tabla asociada; su único vínculo es
lógico, contra el valor del token QR (`Token QR`, spec 007) que la URL de esa pestaña sigue
llevando.
