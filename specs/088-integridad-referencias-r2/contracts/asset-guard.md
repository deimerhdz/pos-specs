# Contrato: primitivas de integridad de referencias (`app/core/storage.py` y `app/core/asset_refs.py`)

Contrato interno de las funciones que aplican FR-002 … FR-008. Son la **única fuente de la regla**; ningún servicio la reimplementa. `normalize_asset_ref`, `asset_display_url` y `object_key_for_deletion` (spec 080) **no cambian**.

## Constantes

```python
FOLDER_PRODUCTS = "products"
FOLDER_LOGO = "logo"
FOLDER_PAYMENT_METHODS = "payment-methods"
FOLDER_RECEIPTS = "comprobantes"
```

## `app/core/storage.py`

### `validate_asset_key(value: str, tenant_schema: str, folder: str) -> str`

Pura, sin I/O. Devuelve `value` sin modificarlo si es una key **nueva** válida; si no, lanza `AssetKeyError` (los llamadores la convierten en 422).

Válida ⇔ `re.fullmatch(f"{re.escape(tenant_schema)}/{folder}/[A-Za-z0-9][A-Za-z0-9._-]{{0,199}}", value)` **y** `".." not in value`.

| Entrada (`tenant_schema="acme"`, `folder="products"`) | Resultado |
|---|---|
| `acme/products/46f1a4d1c4aa4c68ba7b32642334d084.png` | ✔ |
| `acme/products/foto.v2.jpg` | ✔ |
| `globex/products/x.jpg` (otro negocio) | ✘ |
| `acme/logo/x.png` (otra carpeta) | ✘ |
| `ACME/products/x.jpg` (mayúsculas distintas) | ✘ |
| `acme/products/../logo/x.png` | ✘ |
| `acme//products/x.jpg` · `acme/products//x.jpg` | ✘ |
| `acme\products\x.jpg` | ✘ |
| `acme/products/x.jpg\n` · `acme/products/x\x00.jpg` | ✘ |
| `acme/products/%2e%2e/x.jpg` · `acme/products/a b.jpg` | ✘ |
| `acme/products/` (sin nombre) · `acme/products/.hidden` | ✘ |
| `acme/products/a/b.jpg` (subcarpeta) | ✘ |
| `""` | ✘ (los vacíos nunca llegan: son `KEEP`) |

No normaliza, no recorta y no "corrige" nada. **No se llama** para URLs de otro origen ni para un valor igual al vigente (research D2/D6).

### `object_exists(key: str) -> bool`

`head_object` con timeouts 3 s / 5 s y 2 intentos (research D3).

| Respuesta de R2 | Resultado |
|---|---|
| 200 | `True` |
| `ClientError` `404` / `NoSuchKey` / `NotFound` | `False` |
| cualquier otro `ClientError` (403, 5xx…), `EndpointConnectionError`, `ConnectTimeoutError`, `ReadTimeoutError`, `BotoCoreError` | lanza `StorageUnavailable` |

Nunca guarda ni borra nada. El llamador convierte `StorageUnavailable` en **503**.

### `deletable_key(value: str | None, tenant_schema: str, folder: str) -> str | None`

Key que **se puede** borrar al reemplazar `value`, o `None` (no borrar). Compone `object_key_for_deletion(value)` (spec 080) con la convención (FR-004: "también al decidir qué archivo borrar").

| `value` | Resultado |
|---|---|
| `None` / vacío | `None` |
| URL de otro origen | `None` (FR-005) |
| key/URL gestionada en convención del propio negocio y carpeta | la key |
| key/URL gestionada **fuera de convención** (histórica, o de otro negocio/carpeta) | `None` (queda huérfana, es seguro) |

### `get_r2_client()`

Mismo cliente `lru_cache`, con `BotoConfig(signature_version="s3v4", region_name="auto", connect_timeout=3, read_timeout=5, retries={"max_attempts": 2, "mode": "standard"})`.

## `app/core/asset_refs.py`

### `decide_image_change(*, tenant_schema, folder, sent, base_provided, base, current, is_creation) -> ImageDecision`

Pura. Todos los valores de imagen entran ya normalizados (`normalize_asset_ref`). Devuelve `KEEP`, `IGNORE` o `APPLY`, o lanza `AssetKeyError`. Orden fijo (research D4):

1. `sent is None` → `KEEP`
2. `sent == current` → `KEEP`
3. `sent` gestionado y `validate_asset_key` falla → `AssetKeyError` (**siempre**, aunque no haya base o esté desactualizada)
4. `is_creation` → `APPLY`
5. `not base_provided` o `base != current` → `IGNORE`
6. `sent` de otro origen → `APPLY` (sin verificación)
7. `sent` gestionado → `APPLY` (el llamador verifica existencia)

"Gestionado" = no contiene `://` tras normalizar. `base` y `current` se comparan como valores normalizados (una fila histórica con URL absoluta equivale a su key).

### `ensure_image_exists(key: str) -> None`

Llamada por el servicio cuando la decisión fue `APPLY` y la key es gestionada (o en una creación).

- existe → retorna
- no existe → `HTTPException(422, "La imagen no se encontró en el almacenamiento. Sube el archivo de nuevo.")`
- `StorageUnavailable` → `HTTPException(503, "El almacenamiento de archivos no responde. Intenta de nuevo en unos segundos.")`

### `is_key_referenced(db: Session, key: str) -> bool`

`True` si **alguna** de estas cuatro fuentes referencia la key (o una URL absoluta gestionada equivalente): `products.image_url`, valores de `payment_methods.payment_info`, `order_payment_attempts.receipt_file_url` (esquema del negocio de `db`) y `shared.tenants.logo_url` (global, todos los negocios). Consulta de solo lectura. Detalle en research D9.

### `asset_key_lock(db: Session, key: str) -> None`

`SELECT pg_advisory_xact_lock(hashtextextended(:key, 0))` si el dialecto es PostgreSQL; **no-op** en cualquier otro (los tests usan SQLite). Se libera solo al terminar la transacción (research D13).

### `resolve_receipt_key(value: str, tenant_schema: str) -> str`

Para comprobantes nuevos (FR-008). Devuelve la key lista para persistir o lanza `HTTPException`.

| Entrada | Resultado |
|---|---|
| `acme/comprobantes/{n}` existente | key |
| URL del dominio público anterior o del de assets con esa key | la key extraída |
| key con forma inválida, de otro negocio o de otra carpeta | 422 "El comprobante no pertenece a este negocio o no es válido." |
| URL de otro origen | 422 "El comprobante debe ser un archivo subido desde la aplicación." |
| key válida cuyo archivo no existe | 422 (mismo texto que `ensure_image_exists`) |
| R2 no responde | 503 |

## Mensajes (español de Colombia, Principio XIII)

| Situación | Código | Mensaje |
|---|---|---|
| key inválida / ajena / otra carpeta (imagen de producto, logo, QR) | 422 | "La imagen no es válida para este negocio." |
| archivo no existe | 422 | "La imagen no se encontró en el almacenamiento. Sube el archivo de nuevo." |
| almacenamiento no responde | 503 | "El almacenamiento de archivos no responde. Intenta de nuevo en unos segundos." |
| comprobante ajeno/mal formado | 422 | "El comprobante no pertenece a este negocio o no es válido." |
| comprobante de otro origen | 422 | "El comprobante debe ser un archivo subido desde la aplicación." |

El detalle de la key rechazada **no** se incluye en el mensaje (evita reflejar entradas manipuladas); se registra en el log del servidor.

## Invariantes

- Ninguna función de esta tabla modifica la base de datos ni borra objetos de R2, salvo `delete_object` (spec 021), que sigue siendo el único borrado.
- Un 422/503 nunca deja el registro modificado ni un archivo borrado (SC-002).
- `IGNORE` nunca produce error ni aviso hacia el cliente (FR-002).
