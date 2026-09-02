# Research: Corrección — falla al crear un tenant con usuario por migraciones rotas

Auditoría realizada directamente sobre `../pos-backend` antes de diseñar la corrección.

## 1. Causa raíz, confirmada línea por línea

`../pos-backend/alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py:52-55`:

```python
_CK_TYPE = op.f("ck__promotions__ck_promotion_type")
_CK_VALUE = op.f("ck__promotions__ck_promotion_value_positive")
_CK_MIN_QTY = op.f("ck__promotions__ck_promotion_min_qty")
_CK_PERCENT = op.f("ck__promotions__ck_promotion_percent_range")
```

Estas cuatro líneas están al **nivel de módulo** (fuera de `upgrade()`/`downgrade()`) — se
ejecutan en el instante en que Python importa el archivo, no cuando Alembic corre una migración.
`op.f(...)` necesita el *proxy* de la clase `Operations` de Alembic, que solo existe **dentro**
de una migración en ejecución — de ahí el error exacto que reportó el usuario: *"Can't invoke
function 'f', as the proxy object has not yet been established for the Alembic 'Operations'
class"*.

`../pos-backend/alembic/versions/ba4b6bd573a6_063b_promociones_retiro_estructura_.py:41` tiene
exactamente el mismo defecto, con una sola constante:

```python
_CK_TYPE = op.f("ck__promotions__ck_promotion_type")
```

**Búsqueda exhaustiva confirmada**: se recorrió el AST de cada archivo en `alembic/versions/`
buscando asignaciones de nivel de módulo cuyo valor contenga `op.f(...)` — estos dos archivos son
los **únicos** con el defecto; ningún otro migration file lo tiene.

## 2. Por qué esto rompe la creación de un tenant específicamente

`../pos-backend/app/core/db.py::tenant_create()` (línea ~70, código preexistente, no tocado por
este spec) hace, antes de escribir cualquier dato:

```python
context = MigrationContext.configure(db.connection())
SCRIPT = script.ScriptDirectory.from_config(alembic_config)
if context.get_current_revision() != SCRIPT.get_current_head():
    raise RuntimeError(...)
```

`SCRIPT.get_current_head()` obliga a Alembic a **importar todos** los archivos de
`alembic/versions/` para construir el mapa de revisiones (confirmado con la traza completa que
compartió el usuario: `revision.py::_revision_map` → `base.py::_load_revisions` →
`Script._from_path` → `exec_module` → la línea 52 del archivo roto). Con cualquiera de los dos
archivos fallando al importarse, esa construcción entera falla — y como es un paso obligatorio
antes de crear el tenant, la operación completa falla, **antes de escribir ningún dato** (el
`INSERT` de `Tenant` ocurre líneas después, nunca se alcanza).

**Alcance real del defecto, más allá de "crear un tenant"**: cualquier comando o código que
necesite el mapa completo de revisiones (`alembic upgrade`, `alembic history`, y este mismo
chequeo de `tenant_create()`) falla igual. `alembic current`/`alembic show <rev>` puntual sobre
una sola revisión concreta pueden no necesitar cargar el árbol completo dependiendo de la
implementación interna, pero no se asume nada al respecto — el fix cubre la causa raíz
(el error de sintaxis al importar), no un síntoma puntual.

## 3. Por qué el fix es seguro: el patrón correcto ya existe en el mismo archivo

En ambos archivos, **la mayoría** de las llamadas a `op.f(...)` ya están hechas correctamente,
inline, dentro de `upgrade()`/`downgrade()` — por ejemplo,
`94b7e35f5e5e_...py:184` (`op.f("uq__promotion_variants__promotion_id__product_variant_id")`,
dentro de `upgrade()`) o `ba4b6bd573a6_...py:61`
(`op.f("ix__product_variants__presentation_id")`, también dentro de `upgrade()`). Solo las
constantes de nivel de módulo (`_CK_TYPE`/`_CK_VALUE`/`_CK_MIN_QTY`/`_CK_PERCENT`) rompen ese
patrón — el fix simplemente las alinea con el resto del archivo.

**Decisión**: mover cada asignación al cuerpo de cada función que la usa, como variable local,
calculada con `op.f(...)` en el mismo punto donde antes se leía la constante de módulo. Ambas
funciones (`upgrade`/`downgrade`) usan las mismas cuatro constantes en
`94b7e35f5e5e_...py`, y la misma única constante en `ba4b6bd573a6_...py` — se duplica la
asignación en cada función (2-4 líneas cada vez), sin introducir ningún helper nuevo ni cambiar
la firma de `upgrade`/`downgrade` (ambas ya reciben `schema: str`, decoradas con
`@for_each_tenant_schema` — `op.f(...)` no depende del schema, así que no hace falta pasarle
nada adicional).

**Alternativa considerada y descartada**: envolver las 4 (o 1) asignaciones en una función
`_check_names()` compartida, llamada al inicio de `upgrade`/`downgrade`. Se descartó por ser una
abstracción innecesaria para 1-4 líneas que solo se usan dentro del mismo archivo — consistente
con evitar jerarquías/indirecciones innecesarias (Constitución, espíritu del Principio V).

## 4. Impacto en datos existentes — verificado, no asumido

Ninguna de las dos migraciones cambia de contenido (mismo DDL, mismos nombres de restricción,
mismo orden de operaciones) — el fix solo cambia el momento en que Python evalúa 5 expresiones
`op.f(...)`, cuyo resultado (el nombre de cadena de la restricción) es **idéntico** antes y
después del fix, porque `op.f()` no depende de ningún estado que cambie entre el momento en que
se evaluaba antes (import) y el momento en que se evaluará después (dentro de `upgrade`/
`downgrade`) — solo depende de la convención de nombres fija del proyecto
(`app/core/models.py::convention`). Por eso el `checksum`/efecto de cada migración sobre
cualquier base de datos donde ya se ejecutó exitosamente (fuera de este defecto, p. ej. en un
entorno donde se corrió antes de que se introdujera, o vía otro mecanismo) no cambia.

**Verificado**: ningún otro archivo del backend importa estos dos módulos de migración ni sus
constantes (`grep` sin resultados) — el cambio de nivel de módulo a nivel de función no tiene
ningún consumidor externo que romper.

## 5. Verificación

No existe ningún characterization test sobre estas dos migraciones ni sobre
`ck_promotion_type`/constraints relacionadas (`grep` sin resultados en
`app/characterization_tests/`) — no hay nada que este fix pueda poner en rojo por protección de
comportamiento heredado. La verificación de este hotfix es, en cambio, funcional y de
integración real: que el catálogo de migraciones cargue sin error, y que crear un tenant de
punta a punta contra Postgres real tenga éxito — ambas cubiertas en `quickstart.md`.
