# Data Model: Corrección — falla al crear un tenant con usuario por migraciones rotas

Este hotfix no introduce, modifica ni elimina ninguna entidad, tabla, columna ni relación
(Constitution Check § Principio VIII: N/A). No hay modelo de datos nuevo que documentar.

Lo único que cambia es **dónde**, dentro de dos archivos de Python ya existentes, se calculan
cinco nombres de restricción de base de datos — de nivel de módulo a nivel de función. A modo de
inventario del cambio (research.md § 1 y § 3):

## Archivo 1: `alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py`

| Constante | Valor (`op.f(...)`) | Usada en |
|---|---|---|
| `_CK_TYPE` | `"ck__promotions__ck_promotion_type"` | `upgrade()`, `downgrade()` |

## Archivo 2: `alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py`

| Constante | Valor (`op.f(...)`) | Usada en |
|---|---|---|
| `_CK_TYPE` | `"ck__promotions__ck_promotion_type"` | `upgrade()`, `downgrade()` |
| `_CK_VALUE` | `"ck__promotions__ck_promotion_value_positive"` | `upgrade()`, `downgrade()` |
| `_CK_MIN_QTY` | `"ck__promotions__ck_promotion_min_qty"` | `upgrade()`, `downgrade()` |
| `_CK_PERCENT` | `"ck__promotions__ck_promotion_percent_range"` | `upgrade()`, `downgrade()` |

En ambos archivos, cada constante pasa de ser una asignación de nivel de módulo a una variable
local, calculada con la misma llamada `op.f(...)` dentro de cada función que la usa — el valor
de cadena resultante no cambia (research.md § 4), solo el momento en que Python la evalúa.
