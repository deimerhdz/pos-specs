# Fase 0 — Research: Catálogo de Métodos de Pago Administrado por el Super Admin

Todas las decisiones de esta fase son **técnicas** (Principio XI): el spec (`spec.md`) define el
comportamiento; este documento define cómo implementarlo sobre lo que ya existe en `pos-backend`
y `pos-heladeria`, sin contradecir el spec ni inventar requisitos nuevos.

## Decisión 1 — El catálogo vive en el esquema `shared`, no en `tenant`

**Decisión**: `PaymentMethodCatalog` es una tabla nueva con `__table_args__ = {"schema": "shared"}`
(`app/core/models.py`), junto a `Tenant`/`Role`/`User` — las únicas entidades hoy compartidas por
toda la plataforma.

**Alternativas consideradas**:
- Una tabla `tenant.payment_method_catalog` replicada en cada esquema de tenant (vía
  `@for_each_tenant_schema`, como el resto de tablas de negocio). Descartada: el catálogo es por
  definición una sola fuente de verdad administrada por el Super Admin (RF-1); replicarlo por
  tenant reintroduce exactamente el problema que el spec busca resolver (cada tenant con su propia
  copia, potencialmente divergente).
- Guardar el catálogo como filas "especiales" dentro de la propia tabla `tenant.payment_methods`
  (un tenant "de sistema"). Descartada: mezclaría el modelo de "definición" con el de
  "configuración por tenant" que el spec explícitamente separa (§1 Objetivo).

**Precedente en el código**: no existe hoy ningún catálogo compartido con "activación por tenant"
(el research confirma que `Role` es la única tabla `shared` referenciada desde otros esquemas, pero
sin una fila de "configuración por tenant" asociada) — este es un patrón nuevo en la base de
código, documentado aquí explícitamente en vez de asumido.

## Decisión 2 — Los campos de integración se modelan como JSONB, no como tabla normalizada

**Decisión**: `PaymentMethodCatalog.fields` es una columna `JSONB`, lista de objetos
`{key, label, required, format}` (`format` ∈ `"text" | "numeric" | "image"`, con `length` opcional
para formatos `numeric`/`text`). `tenant.payment_methods.payment_info` (ya existente, spec 024)
sigue siendo `JSONB` `dict[str, str]` sin cambios de tipo — lo que cambia es que ahora se valida en
el backend contra `catalog.fields` antes de guardarse.

**Alternativas consideradas**:
- Tabla normalizada `payment_method_catalog_field` (una fila por campo). Descartada: el propio
  `payment_info` de `PaymentMethod` ya usa JSONB sin esquema fijo (precedente directo, spec 024,
  `app/models/payment.py` línea 30) — mantener la misma forma de dato (JSONB) para su definición es
  consistente con cómo ya evolucionó este mismo módulo, y evita un `JOIN` extra en cada lectura del
  catálogo (que se lee completo, pocas filas, sin necesidad de filtrar por campo individual).

**Formato validado en el backend, no en la base de datos**: la validación de `payment_info` contra
`catalog.fields` (obligatoriedad + formato, FR-004/FR-009/clarificación 2026-08-24 #3) ocurre en
`app/api/v1/sales/service.py` al crear/editar un `PaymentMethod` de tenant — no como `CHECK`
constraint de Postgres (el JSONB no tiene un esquema fijo que Postgres pueda validar
declarativamente sin extensiones adicionales, y esto mantiene la misma capa de validación que ya
usa Pydantic para el resto de la API).

## Decisión 3 — `catalog_id` nullable + backfill en dos pasos, no `NOT NULL` inmediato

**Decisión**: la migración que agrega `catalog_id` a `tenant.payment_methods` lo deja **nullable**.
Un script de backfill (`app/scripts/migrate_payment_methods_catalog.py`) puebla `catalog_id`
matcheando cada fila existente contra el catálogo por nombre normalizado (minúsculas, sin tildes).
La capa de aplicación (Pydantic + `service.py`) exige `catalog_id` para **filas nuevas** desde el
día en que se despliega esta spec, sin esperar a que la columna sea `NOT NULL` a nivel de base de
datos.

**Alternativas consideradas**:
- `NOT NULL` directo en la misma migración, con backfill dentro de la propia migración de Alembic.
  Descartada: el propio repositorio ya tiene un precedente explícito para este tipo de riesgo
  (`alembic/versions/b3c4d5e6f7a8_backfill_target_pricing.py`) que separa "agregar columna" de
  "poblarla con datos existentes que pueden no matchear limpiamente", registrando las filas que no
  pudo resolver automáticamente para revisión manual en vez de fallar la migración completa o
  dejarlas en un estado incorrecto. FR-015a exige que el Super Admin revise y agregue al catálogo
  los métodos personalizados **antes** de migrar — eso solo es posible si el backfill es un paso
  separado y reejecutable, no una migración de Alembic de un solo intento.
- Agregar un `CHECK` de base de datos que exija `catalog_id NOT NULL` desde ya. Descartada por la
  misma razón: forzaría a resolver el 100% de las filas existentes antes de poder desplegar el
  cambio de esquema, bloqueando el propio proceso de revisión que FR-015a exige que ocurra primero.

**Estrategia de rollback** (Principio VIII): `op.drop_column("catalog_id")` /
`op.drop_column("is_complete")` sobre `tenant.payment_methods` (vía `@for_each_tenant_schema`);
`op.drop_table("payment_method_catalog")` sobre `shared`. Ninguna de las dos operaciones toca
`payments`/`sales`, por lo que revertir no tiene impacto en ventas ya registradas.

## Decisión 4 — `is_complete` se persiste y se recalcula al escribir, no se calcula en cada lectura

**Decisión**: `tenant.payment_methods.is_complete: bool` se recalcula en `service.py` cada vez que
se crea o edita `payment_info` (comparando contra `catalog.fields` en ese momento), y se guarda.

**Alternativas consideradas**:
- Calcularlo en cada lectura (sin columna persistida), comparando `payment_info` contra
  `catalog.fields` en el momento de servir `GET /sales/payment-methods?available=true`. Descartada:
  obligaría a resolver el `catalog.fields` vigente en cada request de checkout (más JOIN/lookup en
  el camino caliente de caja) solo para repetir un cálculo que no cambia entre escrituras; además,
  el propio edge case del spec ("editar el catálogo no invalida configuraciones ya completadas")
  exige que la completitud de un tenant se congele en el momento de guardar, no que se reevalúe
  contra una definición de campos que pudo cambiar después.

## Decisión 5 — `type`/`name` se copian del catálogo al activar, no se leen por `JOIN`

**Decisión**: al crear un `PaymentMethod` de tenant, `name` y `type` se copian del
`PaymentMethodCatalog` elegido en ese momento (denormalizados). El arqueo
(`app/api/v1/cash/service.py`, agrupa por `PaymentMethod.type`/`.name`) sigue leyendo esas dos
columnas exactamente como hoy — **sin ningún cambio en `cash/service.py`**.

**Alternativas consideradas**:
- Eliminar `name`/`type` de `tenant.payment_methods` y resolverlos siempre vía `JOIN` a
  `catalog`. Descartada: rompería el invariante `is_cash ⇔ type == 'cash'` que ya protege
  `app/models/payment.py` (línea 21) y obligaría a tocar `cash/service.py` — una refactorización no
  pedida por el spec (Principio V). Mantener las columnas denormalizadas es el cambio de menor
  superficie posible.
- Sincronizar `name`/`type` automáticamente cada vez que el Super Admin edita el catálogo.
  Descartada: contradice el edge case del spec ("editar el catálogo no invalida configuraciones ya
  completadas") — si el Super Admin renombra un método, los tenants que ya lo activaron no deberían
  ver su configuración cambiar retroactivamente sin haber vuelto a guardar nada.

## Decisión 6 — Ocultar `payment_info` al cajero: response schema separado, no un nuevo endpoint

**Decisión**: `GET /sales/payment-methods` gana un query param `available: bool = False`. Con
`available=true` (usado por el checkout, `pos-terminal.store.ts`), responde con un schema reducido
`PaymentMethodCheckoutOption {id, name}` — sin `payment_info`, `type`, `is_cash` ni `active` — y
solo incluye filas `active=true AND is_complete=true AND catalog.active=true`. Sin el query param,
se mantiene `PaymentMethodResponse` completo (uso administrativo, `payment-methods-page.component.ts`).

**Alternativas consideradas**:
- Endpoint nuevo (`GET /sales/payment-methods/checkout`). Descartada: mismo recurso, mismo filtro
  de tenant, la única diferencia real es forma de respuesta + filtro — un query param es más
  consistente con cómo ya se filtra el resto de listados de este backend (`status`, `date_from`, en
  `list_sales`, mismo router).
- Filtrar `payment_info` en el frontend (seguir devolviendo todo desde el backend y que el
  componente de checkout simplemente no lo muestre). Descartada: FR-012a exige que el cajero no
  tenga esos datos disponibles, no solo que la UI no los pinte — un query param con schema reducido
  es la única forma de que el dato no viaje hacia el cliente del cajero.

## Decisión 7 — Migración de métodos personalizados: reporte + revisión manual del Super Admin, no auto-creación

**Decisión**: `app/scripts/migrate_payment_methods_catalog.py` primero **reporta** (no crea) los
nombres de `tenant.payment_methods` que no matchean ningún `PaymentMethodCatalog.name` existente
(normalizado). El Super Admin usa ese reporte para decidir, uno por uno, si crea una entrada nueva
en el catálogo (US1, `POST /super-admin/payment-methods-catalog`) para cada método personalizado
legítimo. Solo después de esa revisión se reejecuta el backfill para esas filas.

**Alternativas consideradas**:
- Que el script cree automáticamente una entrada de catálogo por cada nombre no reconocido.
  Descartada explícitamente por la clarificación 2026-08-24 (#2): la decisión de negocio confirmada
  fue que el Super Admin revisa y agrega manualmente los métodos personalizados válidos, no que el
  sistema los incorpore sin supervisión (evita, por ejemplo, que errores de tipeo histórico —
  "Nequii", "nequi transferencia"— terminen como entradas de catálogo separadas).

## Decisión 8 — Imagen de código QR: mismo mecanismo de R2 presignado, folder nuevo

**Decisión**: se agrega `"payment-methods"` a `Literal["products", "logo"]` en
`app/api/v1/uploads/schemas.py` (`PresignRequest.folder`). El flujo es idéntico al ya usado para
imágenes de producto: el Tenant Admin pide una URL presignada (`POST /uploads/presign`,
`require_tenant_admin`), sube el archivo directo a R2, y guarda la `public_url` resultante como el
valor del campo `format: "image"` correspondiente dentro de `payment_info`.

**Alternativas consideradas**:
- Un endpoint de presign nuevo, específico de métodos de pago. Descartada: el endpoint existente ya
  soporta un tenant arbitrario vía `build_object_key(tenant_schema, folder, extension)` — namespacea
  por tenant automáticamente; solo falta el nombre de carpeta en la whitelist.
- Imagen QR a nivel de catálogo (definida por el Super Admin como un logo genérico del método, no
  por tenant). Fuera de alcance: el spec pide el QR como dato de integración **del tenant** (§1,
  ejemplo Nequi), no un ícono de marca a nivel de plataforma — el catálogo del Super Admin no sube
  ninguna imagen, solo define que el campo `qr` existe y es de tipo `image`.

## Decisión 9 — Una fila por (tenant, catalog_id) para siempre, no un índice parcial

**Decisión**: `tenant.payment_methods` gana una restricción única **normal** (no parcial) sobre
`catalog_id` (`uq_payment_method_catalog_id`): a lo sumo una fila por tenant por entrada de
catálogo, para siempre. `POST /sales/payment-methods` solo crea fila la primera vez; si ya existe
una fila para ese `catalog_id` (activa o no), el tenant la edita/reactiva vía `PATCH
/sales/payment-methods/{id}` — nunca se crea una segunda fila para el mismo método. Desactivar y
reactivar alternan `active` sobre esa misma fila, conservando su `payment_info` (FR-017; también
satisface el edge case de `spec.md`: "el tenant conserva los datos de integración que ya había
capturado... no tiene que volver a capturarlos", que solo tiene sentido si es la misma fila).

**Por qué no un índice parcial `WHERE active = true`** (diseño inicial, descartado durante la
implementación al escribir `service.py`): permitiría múltiples filas históricas inactivas por
`catalog_id`, pero todas copiarían el mismo `catalog.name` en su columna `name` — y
`payment_methods.name` ya tiene una restricción única (anterior a esta spec, protege que el arqueo
no muestre dos filas con el mismo nombre). La segunda fila (la "nueva" activación tras desactivar
la anterior) violaría esa restricción antes de llegar a cualquier guardia de aplicación. Una fila
única por `catalog_id` evita el conflicto de raíz, sin tocar la restricción única de `name` que ya
protegía otro invariante.

**Alternativas consideradas**:
- Índice único parcial (diseño inicial) + quitar la restricción única de `name`. Descartada: quitar
  esa restricción es un cambio de comportamiento más amplio de lo que esta spec necesita (Principio
  V) y debilitaría una protección ajena al catálogo (nombres duplicados en el picker del arqueo).
- Validarlo solo a nivel de aplicación, sin restricción de base de datos. Descartada: el resto de
  invariantes de esta tabla (`is_cash ⇔ type`) ya se protegen con constraints de Postgres, no solo
  en Python — mantener la misma disciplina evita una condición de carrera entre dos requests
  concurrentes.
