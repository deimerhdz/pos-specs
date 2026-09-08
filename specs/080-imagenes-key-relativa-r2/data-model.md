# Data Model: Almacenar imágenes como key relativa y servirlas por dominio personalizado

**Cambios de esquema: ninguno.** No hay entidades, columnas, relaciones, valores por defecto ni migraciones de Alembic nuevas (Principio VIII, FR-017). Cambia el **contenido** de tres campos que ya existen: pasan de guardar una URL pública absoluta a guardar solo la key relativa del objeto. Este documento describe (1) los tres campos afectados y sus tipos actuales, (2) las formas que puede tener un "valor de referencia de imagen", (3) las tablas de verdad de los tres helpers, y (4) el contrato de la migración de datos.

---

## 1. Campos afectados (sin cambio de tipo)

| Campo | Modelo / tabla | Schema | Tipo actual | Antes | Después |
|---|---|---|---|---|---|
| `image_url` | `Product` / `products` | por tenant | `String(500)` nullable | `https://pub-…r2.dev/{tenant}/products/{uuid}.{ext}` | `{tenant}/products/{uuid}.{ext}` (key) |
| `logo_url` | `Tenant` / `tenants` | `shared` | `String(500)` nullable | `https://pub-…r2.dev/{tenant}/logo/{uuid}.{ext}` | `{tenant}/logo/{uuid}.{ext}` (key) |
| `payment_info` (clave `format: "image"`, típ. `qr`) | `PaymentMethod` / `payment_methods` | por tenant | `JSONB` nullable (`dict[str, str]`) | valor de esa clave = `https://pub-…r2.dev/{tenant}/payment-methods/{uuid}.{ext}` | valor de esa clave = `{tenant}/payment-methods/{uuid}.{ext}` (key) |

Notas:

- La key (`{tenant}/{carpeta}/{uuid32}.{ext}`, ~55–65 caracteres) es **más corta** que la URL que reemplaza (~110+). `String(500)` sigue sobrando; no se ensancha ninguna columna.
- `payment_info` es un `dict[str, str]`. Solo las claves marcadas `format: "image"` en `catalog.fields` son referencias a un objeto. Las demás (`celular`, `cuenta`, `titular`, …) **no se tocan** ni en migración ni en escritura ni en lectura.
- **Fuera de alcance** (FR-014): `OrderPaymentAttempt.receipt_file_url` (carpeta `comprobantes`). No aparece en esta tabla porque no cambia.
- El **objeto físico** en Cloudflare R2 no se mueve, renombra ni vuelve a subir (FR-009a). Solo cambia cómo la base de datos apunta a él.

### Relación con el objeto de almacenamiento

`build_object_key(tenant_schema, folder, ext)` → `{tenant_schema}/{folder}/{uuid4().hex}.{ext}` (sin cambios, FR-016). La key almacenada tras esta spec es **exactamente** la que devolvió `build_object_key` al subir (FR-003): sirve tanto para construir la URL de visualización como para borrar el objeto.

Carpetas permitidas por `POST /uploads/presign` (whitelist, sin cambios, FR-016): `products`, `logo`, `payment-methods`. La carpeta `comprobantes` **no** está en esa whitelist (usa presign propios en `cart/service.py`).

---

## 2. Formas de un "valor de referencia de imagen"

Cualquiera de los tres campos puede contener, en un instante dado, uno de estos valores. La columna "Origen" indica cómo se llegó a él.

| # | Forma | Ejemplo | Origen |
|---|---|---|---|
| V0 | Vacío | `NULL` o `""` | producto sin foto / negocio sin logo / método sin imagen |
| V1 | **Key** (sin esquema ni dominio) | `heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png` | subida nueva tras spec 080, o fila ya migrada |
| V2 | **URL pública anterior** del bucket gestionado | `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/heladeria3/products/46f1…084.png` | fila anterior a spec 080 que aún no migró (FR-009b), o cliente que reenvía una URL vieja |
| V3 | **URL del dominio personalizado nuevo** (bucket gestionado) | `https://assets.skeilopos.com/heladeria3/products/46f1…084.png` | el frontend reenvía la URL de visualización al guardar un formulario que no tocó la imagen (FR-004) |
| V4 | **URL de otro origen** | `https://xyz.supabase.co/storage/v1/object/public/img/foto.jpg` | imagen histórica cargada antes de R2, por otra vía (FR-010) |

"Bucket gestionado" = el valor empieza por alguno de los prefijos de `_managed_bucket_prefixes()` = `{ASSETS_BASE_URL}/` **o** `{R2_PUBLIC_BASE_URL}/`. La comparación es por prefijo de dominio conocido, no por coincidencia exacta; un query string o una diferencia de mayúsculas en el host se toleran extrayendo la key desde el primer prefijo que casa (Edge Cases del spec).

**Regla de persistencia (FR-002)**: en base de datos solo pueden quedar **V0** o **V1** para valores del bucket gestionado; **V4** se conserva tal cual. **V2** y **V3** nunca deben quedar persistidos (los normaliza la escritura; los reescribe la migración).

---

## 3. Tablas de verdad de los tres helpers

Los tres viven en `app/core/storage.py`, son funciones puras y comparten el reconocimiento de prefijos. `ASSETS_BASE_URL = https://assets.skeilopos.com`, `R2_PUBLIC_BASE_URL = https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev` (ejemplos de producción).

### `normalize_asset_ref(value) -> str | None` — se aplica al **persistir** (FR-004)

| Entra | Sale | Regla |
|---|---|---|
| V0 (`None` / `""`) | `None` | vacío ⇒ sin imagen |
| V1 key | la misma key | ya es key, no se toca |
| V2 URL pública anterior | key (`value` sin el prefijo) | normaliza dominio viejo |
| V3 URL dominio nuevo | key (`value` sin el prefijo) | normaliza dominio nuevo |
| V4 URL otro origen | el mismo valor | intacto (FR-010) |

### `asset_display_url(value) -> str | None` — se aplica al **responder** (FR-005)

| Entra | Sale | Regla |
|---|---|---|
| V0 | `None` | sin imagen ⇒ sin URL |
| V1 key | `{ASSETS_BASE_URL}/{key}` | antepone dominio nuevo |
| V2 URL pública anterior | `{ASSETS_BASE_URL}/{key}` | tolerancia de lectura: reescribe al dominio nuevo (FR-009b) |
| V3 URL dominio nuevo | el mismo valor | ya está bien |
| V4 URL otro origen | el mismo valor | intacto (FR-010) |

### `object_key_for_deletion(value) -> str | None` — se aplica al **borrar el objeto anterior** (FR-011/FR-013)

| Entra | Sale | Regla |
|---|---|---|
| V0 | `None` | nada que borrar |
| V1 key | la misma key | borra ese objeto |
| V2 URL pública anterior | key | borra ese objeto |
| V3 URL dominio nuevo | key | borra ese objeto |
| V4 URL otro origen | `None` | **no se borra** nada de otro origen (FR-013) |

`object_key_for_deletion` **reemplaza** a `key_from_public_url` (hoy: reconoce solo un prefijo, devuelve `None` para todo lo demás incluida una key). Llamadores a actualizar: `products/service.py::update_product`, `tenant/router.py::update_tenant`.

### Invariantes

- `normalize_asset_ref(asset_display_url(x))` es idempotente para V0/V1/V4 y devuelve la key para V2/V3.
- Para V1: `asset_display_url(normalize_asset_ref(k)) == asset_display_url(k)`.
- `object_key_for_deletion(x) is None` ⟺ `x` es V0 o V4.
- Ninguno de los tres hace I/O de red.

---

## 4. Orden de operaciones en escritura (crítico para US3)

En `update_product` y `update_tenant`, tras esta spec:

```
1. data.image_url llega ya normalizada a key (validador AssetRefIn del esquema)   ← FR-004
2. si data.image_url is not None y data.image_url != product.image_url:           ← key vs key
3.     old_ref = product.image_url
4.     product.image_url = data.image_url
5.     old_key = object_key_for_deletion(old_ref)   # None si old_ref es V4
6. ... (resto del update, dentro de la misma transacción)
7. db.commit()                                       ← A-44: primero confirmar
8. si old_key: delete_object(old_key)                ← A-44: después borrar, best-effort
```

El punto 2 comparando **key contra key** es lo que impide que un formulario reenviado sin cambios (que manda V3) se interprete como cambio de imagen y borre en el paso 8 el objeto que sigue en uso. Este es el corazón de la Historia 3.

`create_product` / `create_payment_method`: sin lógica de borrado; solo persisten `data.*` ya normalizado.

`update_payment_method`: `method.payment_info = data.payment_info` (ya normalizado por valor en el esquema). No hay borrado del QR anterior hoy y no se añade (FR-011 cubre solo producto y logo).

---

## 5. Contrato de la migración de datos

Script: `app/scripts/migrate_image_keys_relative.py`. Detalle operativo en [`contracts/data-migration.md`](./contracts/data-migration.md).

### Conjunto objetivo

| Objetivo | Cómo se recorre |
|---|---|
| `shared.tenants.logo_url` | una vez, `with_db(None)` |
| `{schema}.products.image_url` | por cada `schema` de `SELECT schema FROM shared.tenants`, `with_db(schema)` |
| `{schema}.payment_methods.payment_info` | ídem; se inspecciona cada valor string del dict |

### Predicado y transformación (modo migración, sin flags)

Para cada valor candidato `v`:

| Condición sobre `v` | Acción |
|---|---|
| empieza por un prefijo de `_managed_bucket_prefixes()` (V2 o V3) | reescribir a `normalize_asset_ref(v)` (key) |
| ya es key (V1), vacío (V0), o URL de otro origen (V4) | **no tocar** |

`commit` por schema. Reporte por tenant/campo: `migradas / ya_key / vacias / otro_origen`.

### Idempotencia (FR-009c, SC-009)

Tras migrar, el valor es V1 → ya no cumple el predicado → una segunda corrida (o una corrida que se relanza tras interrumpirse) no toca ninguna fila ya migrada. No se necesita tabla de control. Si la app reescribe una fila a key mientras el script corre, esa fila simplemente deja de ser candidata: convergen sin lock ni ventana de mantenimiento.

### Reversión (FR-009a, SC-008) — `--revert`

Para cada valor que es key (V1) en los tres campos: reescribir a `f"{R2_PUBLIC_BASE_URL.rstrip('/')}/{key}"` (reconstruye V2, la forma exacta anterior). No toca V0 ni V4. `migrar` seguido de `--revert` deja los tres campos idénticos a su estado previo, **siempre que no se hayan subido imágenes nuevas entre medias** (que nacerían como V1 y el revert las convertiría en V2 apuntando a un objeto que sí existe — inconsistencia solo de dominio, no de rotura; se documenta como nota operativa).

### Lo que la migración NUNCA hace

- Tocar `receipt_file_url` / carpeta `comprobantes` (FR-014).
- Mover, renombrar, recrear o borrar un objeto en R2 (FR-009a).
- Alterar valores V4 (otro origen) (FR-010).
- Alterar una factura o venta emitida (Principio VII) — estos campos no son parte de su representación contable.
