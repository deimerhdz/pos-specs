# Data Model: Corrección — falla al crear un tenant por una referencia entre schemas no resuelta

Este hotfix no introduce, modifica ni elimina ninguna entidad, tabla, columna ni relación
(Constitution Check § Principio VIII: N/A). No hay modelo de datos nuevo que documentar.

Lo único que cambia es **qué tablas incluye**, en memoria, la copia de metadata que
`get_tenant_specific_metadata()` (`app/core/db.py`) arma para crear el schema de un tenant nuevo.
A modo de inventario del cambio (research.md § 1 y § 3):

## Relación afectada

| Tabla (schema) | Columna | Referencia (FK) | Tabla referenciada (schema) |
|---|---|---|---|
| `payment_methods` (`tenant`) | `catalog_id` | `ForeignKey("shared.payment_method_catalog.id")` | `payment_method_catalog` (`shared`) |

Es, hoy, la **única** relación de este tipo (de una tabla de tenant hacia una tabla compartida)
en todo el modelo de datos (research.md § 2) — la corrección se generaliza para cubrir cualquier
otra que exista en el futuro (spec.md FR-003), sin necesitar nombrarla explícitamente en el
código.

## Función corregida

| Función | Archivo | Cambio |
|---|---|---|
| `get_tenant_specific_metadata()` | `app/core/db.py` | Después de copiar las tablas de `schema == "tenant"` a la metadata aislada, copia también cualquier tabla de otro schema que alguna de ellas referencie por FK — hoy, solo `payment_method_catalog`. |

Ninguna otra función cambia (`get_shared_metadata()`, `tenant_create()`, y el resto de
`app/core/db.py` quedan intactos).
