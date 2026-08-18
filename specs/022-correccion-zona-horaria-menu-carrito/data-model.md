# Data Model: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

Esta delta no agrega, elimina ni modifica ninguna tabla, columna ni relación — es un cambio de
*qué valor de tiempo* se pasa a una función de evaluación ya existente (spec 012). Se documentan
aquí los tipos y roles relevantes para trazabilidad.

## Promotion (spec 012, sin cambios)

| Atributo | Tipo | Rol en esta corrección |
|---|---|---|
| `start_time` / `end_time` | `time` | Ventana horaria contra la que se compara la hora local calculada — sin cambio de tipo ni de semántica. |
| `status` | `str` | Filtro de estado (`active`), sin cambio. |

## El `datetime` que se compara (no es una entidad de base de datos)

| Punto | Antes de esta corrección | Después de esta corrección |
|---|---|---|
| `menu/router.py:82` (`_build_menu`) | `datetime.now(timezone.utc).replace(tzinfo=None)` — naive, tratado como local por error | `datetime.now(timezone.utc)` — aware, `local_now()` lo convierte correctamente |
| `cart/service.py:205` (`serialize_cart`) | `_now()` → `datetime.now(timezone.utc).replace(tzinfo=None)` — mismo defecto | `datetime.now(timezone.utc)` — aware, calculado localmente sin pasar por `_now()` |
| `cart/service.py:107` (`open_session`, `expires_at`) | `_now()` → naive | **sin cambio** — sigue naive, `_now()` no se toca (FR-004) |

## Transiciones de estado

No hay una máquina de estados formal — `_build_menu` y `serialize_cart` son funciones de lectura
que recalculan la vigencia en cada invocación, sin persistir su resultado. La única "transición"
relevante a esta delta es qué tipo de `datetime` (naive vs. aware) recibe `local_now()` en cada uno
de los tres puntos de la tabla anterior.

## Reglas de validación

Ninguna regla de `Promotion`, `Product`, `ProductVariant` ni `Cart` cambia. Esta delta no introduce
ninguna regla de validación nueva — solo corrige qué instante se compara contra una ventana horaria
ya definida.
