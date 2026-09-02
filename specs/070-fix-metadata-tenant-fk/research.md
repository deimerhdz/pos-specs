# Research: Corrección — falla al crear un tenant por una referencia entre schemas no resuelta

Auditoría y validación realizadas directamente sobre `../pos-backend`, contra PostgreSQL real,
antes de escribir este plan.

## 1. Causa raíz, confirmada línea por línea

`../pos-backend/app/core/db.py::get_tenant_specific_metadata()` (línea ~161):

```python
def get_tenant_specific_metadata():
    meta = MetaData(schema="tenant")
    for table in Base.metadata.tables.values():
        if table.schema == "tenant":
            table.to_metadata(meta)
    return meta
```

Construye una `MetaData` **nueva y aislada**, copiando (`Table.to_metadata()`) únicamente las
tablas cuyo `schema == "tenant"` desde `Base.metadata` (la metadata original, donde todas las
tablas del proyecto — tenant y compartidas — están correctamente vinculadas entre sí).

`app/models/payment.py::PaymentMethod.catalog_id` (tabla `payment_methods`, `schema="tenant"`)
tiene: `ForeignKey("shared.payment_method_catalog.id")` — una referencia real hacia
`payment_method_catalog` (`schema="shared"`, catálogo de la plataforma, spec 032). Al copiar
`payment_methods` hacia la metadata aislada, `to_metadata()` intenta resolver esa FK **dentro de
la nueva metadata** — y como `payment_method_catalog` nunca se copió ahí (no es `schema ==
"tenant"`), la resolución falla con `NoReferencedTableError`, antes de que `create_all()` cree
ninguna tabla del tenant nuevo.

`app/core/db.py::tenant_create()` (línea ~100) es el único punto de llamada:
`get_tenant_specific_metadata().create_all(bind=db.connection())` — de ahí que el síntoma sea
"crear un tenant falla", aunque la causa esté en cómo se prepara la metadata, no en la operación
de creación en sí.

## 2. Alcance de la búsqueda: ¿hay más referencias de este tipo?

```bash
grep -rn 'ForeignKey("shared\.' app/models/*.py
```

Único resultado: `app/models/payment.py:45` (`payment_methods.catalog_id`). Es, **hoy**, la
única referencia de una tabla de tenant hacia una tabla compartida en todo el modelo de datos.
Por eso `spec.md` FR-003 exige que la corrección cubra el caso general (cualquier FK de tenant
hacia otro schema), no solo este caso puntual — para no dejar la misma trampa esperando a la
próxima tabla que dependa de algo compartido.

## 3. Corrección diseñada y validada de punta a punta, antes de este plan

**Decisión**: extender `get_tenant_specific_metadata()` para que, después de copiar las tablas
de tenant, también copie a la misma metadata aislada cualquier tabla de otro schema que alguna
de esas tablas de tenant referencie por FK:

```python
def get_tenant_specific_metadata():
    meta = MetaData(schema="tenant")

    tenant_tables = [t for t in Base.metadata.tables.values() if t.schema == "tenant"]
    for table in tenant_tables:
        table.to_metadata(meta)

    for table in tenant_tables:
        for fk in table.foreign_keys:
            target = fk.column.table
            if target.schema != "tenant" and target.key not in meta.tables:
                target.to_metadata(meta)

    return meta
```

**Por qué es seguro que `create_all()` no intente recrear/duplicar la tabla compartida ya
existente**: `MetaData.create_all()` usa `checkfirst=True` por defecto — antes de emitir un
`CREATE TABLE`, consulta si la tabla ya existe en el schema de destino. Como la tabla compartida
(`shared.payment_method_catalog`) ya existe (creada una única vez por
`initialize_database()`/`get_shared_metadata()` cuando la base de datos se inicializó por primera
vez, código preexistente no tocado por este spec), `create_all()` la detecta y no intenta
recrearla — solo usa su definición para poder resolver y emitir correctamente el `FOREIGN KEY`
de `payment_methods`.

**Validado empíricamente, no solo razonado**, en tres pasos, contra el Postgres real del usuario:

1. Compilación de DDL en aislado (sin tocar la base de datos): con la corrección aplicada, el
   `CREATE TABLE` de `payment_methods` compila correctamente y contiene la referencia a
   `payment_method_catalog` — 1 tabla compartida copiada por el fix (coincide exactamente con el
   único caso encontrado en la sección 2).
2. `create_all()` de punta a punta contra un schema de prueba real (`zzz_dryrun_070`, creado y
   eliminado en la misma verificación): las 41 tablas del tenant se crean sin ningún error.
3. Confirmado que la tabla compartida real (`shared.payment_method_catalog`) sigue teniendo
   exactamente los mismos registros de catálogo que antes — no se duplicó ni se recreó.

**Alternativa considerada y descartada**: dejar de usar una metadata aislada por completo y
llamar directamente `Base.metadata.create_all(bind=conn, tables=[tablas de tenant])` sobre la
metadata original (que ya tiene todo correctamente vinculado). Técnicamente también resuelve el
defecto, pero cambia el punto de llamada en `tenant_create()` y el comportamiento general de la
función (deja de construir una metadata "solo de tenant" reutilizable) — un cambio de mayor
superficie que el necesario para este hotfix. Se prefirió la corrección mínima: arreglar la
función existente donde está el defecto, sin cambiar su contrato (sigue devolviendo una
`MetaData`, sigue llamándose desde el mismo sitio de la misma forma).

## 4. Impacto en datos/tenants existentes — verificado, no asumido

La corrección no cambia ninguna tabla, columna ni FK del modelo de datos — solo agrega, a una
metadata que ya se descarta después de cada uso (se reconstruye desde cero en cada llamada a
`get_tenant_specific_metadata()`), una copia adicional de la tabla compartida que ya hacía falta
para resolver correctamente la FK. No hay ningún tenant existente cuyo esquema dependa de esta
función — solo se usa al **crear** un tenant nuevo.

## 5. Verificación

No existe ningún characterization test sobre `get_tenant_specific_metadata()` ni sobre
`tenant_create()` en general (spec 069, research.md § 5, ya documentó por qué: `tenant_create()`
exige Postgres real desde su primera línea, no es unit-testeable contra SQLite). La verificación
de este hotfix es, igual que en spec 069, funcional y de integración real — ya ejecutada una vez
como parte de esta investigación (sección 3), y formalizada como pasos repetibles en
`quickstart.md`.
