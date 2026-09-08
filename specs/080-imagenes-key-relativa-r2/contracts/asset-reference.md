# Contrato: referencia de imagen (key ↔ URL)

Define el comportamiento observable de los tres helpers de `app/core/storage.py` que gobiernan cómo se guarda, se muestra y se borra una imagen de producto / logo / método de pago. Es el único lugar donde vive la regla; migración, escritura, lectura y borrado la consumen (FR-004, FR-005, FR-009, FR-010, FR-011, FR-013).

## Configuración

| Setting | Ejemplo producción | Uso |
|---|---|---|
| `ASSETS_BASE_URL` | `https://assets.skeilopos.com` | **nuevo**. Base para armar la URL de visualización. Obligatorio (`Field(...)`). Cambiarlo = cambiar el dominio de todas las imágenes sin tocar datos (SC-005). |
| `R2_PUBLIC_BASE_URL` | `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev` | **sin cambios**. Lo siguen usando los presign de comprobantes (`public_url_for`) y la reversión de la migración. También se reconoce como prefijo de "bucket gestionado" (tolerancia de lectura, FR-009b). |

`_managed_bucket_prefixes() -> tuple[str, ...]`: `(f"{ASSETS_BASE_URL.rstrip('/')}/", f"{R2_PUBLIC_BASE_URL.rstrip('/')}/")`, de-duplicado.

## Firmas

```python
def normalize_asset_ref(value: str | None) -> str | None
def asset_display_url(value: str | None) -> str | None
def object_key_for_deletion(value: str | None) -> str | None
```

Puras (sin I/O). Ver la tabla de verdad completa en [`../data-model.md`](../data-model.md) §3. Contrato por caso:

### `normalize_asset_ref` — al persistir

| Entrada | Salida |
|---|---|
| `None`, `""`, `"   "` | `None` |
| `"heladeria3/products/x.png"` (key) | `"heladeria3/products/x.png"` |
| `"https://pub-…r2.dev/heladeria3/products/x.png"` | `"heladeria3/products/x.png"` |
| `"https://assets.skeilopos.com/heladeria3/products/x.png"` | `"heladeria3/products/x.png"` |
| `"https://assets.skeilopos.com/heladeria3/products/x.png?v=2"` | `"heladeria3/products/x.png?v=2"` (se extrae desde el prefijo; el consumidor no añade query strings hoy, se documenta el comportamiento) |
| `"https://otro.com/foto.jpg"` (otro origen) | `"https://otro.com/foto.jpg"` |

### `asset_display_url` — al responder

| Entrada | Salida |
|---|---|
| `None`, `""` | `None` |
| `"heladeria3/products/x.png"` | `"https://assets.skeilopos.com/heladeria3/products/x.png"` |
| `"https://pub-…r2.dev/heladeria3/products/x.png"` | `"https://assets.skeilopos.com/heladeria3/products/x.png"` |
| `"https://assets.skeilopos.com/heladeria3/products/x.png"` | `"https://assets.skeilopos.com/heladeria3/products/x.png"` |
| `"https://otro.com/foto.jpg"` | `"https://otro.com/foto.jpg"` |

### `object_key_for_deletion` — al borrar el objeto anterior

| Entrada | Salida | Efecto |
|---|---|---|
| `None`, `""` | `None` | no se borra nada |
| `"heladeria3/products/x.png"` | `"heladeria3/products/x.png"` | `delete_object` de esa key |
| `"https://pub-…r2.dev/heladeria3/products/x.png"` | `"heladeria3/products/x.png"` | idem |
| `"https://assets.skeilopos.com/heladeria3/products/x.png"` | `"heladeria3/products/x.png"` | idem |
| `"https://otro.com/foto.jpg"` | `None` | **no se borra** (FR-013) |

## Aplicación en los bordes

### Entrada (normalización, FR-004) — tipo `Annotated` reutilizable

```python
AssetRefIn = Annotated[str | None, AfterValidator(normalize_asset_ref)]
```

| Esquema.campo | Endpoint |
|---|---|
| `ProductCreate.image_url`, `ProductUpdate.image_url` | `POST /products`, `PATCH /products/{id}`, guardado consolidado |
| `TenantUpdate.logo_url` | `PATCH /tenant` |
| `PaymentMethodCreate.payment_info`, `PaymentMethodUpdate.payment_info` | `POST /sales/payment-methods`, `PATCH /sales/payment-methods/{id}` — validador de campo que aplica `normalize_asset_ref` a **cada valor** del dict (los valores no-gestionados quedan idénticos) |

Debe normalizarse **antes** de la comparación "¿cambió la imagen?" en `update_product` / `update_tenant` (ver [`../data-model.md`](../data-model.md) §4).

### Salida (ensamblado, FR-005/FR-006/FR-007) — tipo `Annotated` reutilizable

```python
AssetUrl = Annotated[str | None, PlainSerializer(asset_display_url, return_type=str | None)]
```

| Esquema.campo | Endpoint(s) |
|---|---|
| `ProductResponse.image_url` (y subclases `ProductListResponse` / `ProductDetailResponse` / `ProductSaveResponse`) | `GET /products`, `GET /products/{id}`, respuestas de guardado |
| `TenantInfoResponse.logo_url` | `GET /tenant` |
| `MenuProductResponse.image_url`, `MenuBusinessResponse.logo_url` | `GET /menu` |
| `DinerPaymentMethod.payment_info[clave con format="image"]` | `GET /cart/payment-methods` — `@model_validator(mode="after")` usando `self.fields` |
| `PaymentMethodResponse.payment_info[clave con format="image"]` | `GET /sales/payment-methods` (sin `available`), `POST`, `PATCH` — helper `payment_method_response(method)` usando `method.fields` |

El valor guardado en base de datos **no cambia** al serializar (FR-006): la columna sigue con la key; la URL absoluta solo existe en el cuerpo de la respuesta.

### Borrado (FR-011/FR-012/FR-013)

| Llamador | Cambio |
|---|---|
| `products/service.py::update_product` | `key_from_public_url` → `object_key_for_deletion`; comparación key vs key; orden `commit` → `delete_object` best-effort **intacto** (A-44) |
| `tenant/router.py::update_tenant` | idem para `logo_url` |
| métodos de pago | sin cambios — no se borra el QR anterior hoy y no se añade |

## Fuera de este contrato

- **Comprobantes del comensal** (`receipt_file_url`, carpeta `comprobantes`): ninguno de los tres helpers se aplica. `cart/service.py::presign_payment_receipt` / `presign_receipt` / `attach_receipt` siguen usando `public_url_for(key)` → `{R2_PUBLIC_BASE_URL}/{key}`. Sin cambios (FR-014).
- `build_object_key`, `generate_presigned_put_url`, `public_url_for`, `delete_object`: sin cambios de firma ni de comportamiento (FR-016, FR-017).
