# Data Model: Corrección del orden de borrado de imagen en R2 (A-44)

Esta delta no agrega, elimina ni modifica ninguna tabla, columna ni relación — es un cambio de
*orden de operaciones* en tiempo de escritura sobre entidades ya existentes (spec 002). Se
documentan aquí únicamente para trazabilidad de qué parte de cada entidad participa en la
corrección.

## Product

Entidad ya definida en la spec 002. Atributo relevante a esta delta:

| Atributo | Tipo | Rol en esta corrección |
|---|---|---|
| `image_url` | `str \| None` | Valor que `update_product` reasigna (`service.py:80`) antes de decidir si hay un objeto viejo que borrar. Sin cambio de tipo, nulabilidad ni validación — mismo campo, mismo esquema (`ProductUpdate.image_url`, `ProductResponse.image_url`). |

No cambia ninguna otra columna de `Product` como parte de esta delta.

## Objeto en Cloudflare R2 (no es una tabla de base de datos)

Recurso externo, identificado por una `key` derivada de `image_url` vía `key_from_public_url`
(`app/core/storage.py`). No tiene modelo ORM — su ciclo de vida vive fuera de la transacción de
base de datos, que es precisamente el origen de A-44.

| Momento | Antes de esta corrección | Después de esta corrección |
|---|---|---|
| Commit exitoso | Objeto viejo ya borrado antes del commit (sin cambio de resultado final) | Objeto viejo se borra **después** del commit — mismo resultado final |
| Commit fallido | Objeto viejo ya borrado; `product.image_url` revierte a la URL vieja → **referencia rota** (A-44) | Objeto viejo **nunca se borra** — `product.image_url` revierte a la URL vieja, que sigue apuntando a un objeto existente |

## Transiciones de estado

No hay una máquina de estados formal — `update_product` es una operación de una sola escritura.
La única "transición" relevante a esta delta es el **orden relativo** de dos efectos secundarios
(persistencia en base de datos, borrado en almacenamiento externo) ante dos desenlaces posibles
(commit exitoso / commit fallido), documentada en la tabla anterior y en `RN1`/`RN2` de `spec.md`.

## Reglas de validación

Ninguna regla de validación de `Product`/`ProductUpdate` cambia (`RN-CAT-01` a `RN-CAT-11`, spec
002, sin cambios). Esta delta no introduce ninguna regla de validación nueva sobre el schema de
entrada — solo reordena un efecto secundario posterior a una validación ya existente.
