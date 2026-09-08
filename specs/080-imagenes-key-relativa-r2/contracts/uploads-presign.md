# Contrato: `POST /uploads/presign`

Endpoint que entrega una URL firmada para subir una imagen de **producto / logo / método de pago** directo a R2. Lo usan `product-form.component.ts`, `tenant-info.service.ts` y `payment-method.service.ts` del frontend. No cubre comprobantes del comensal (esos tienen presign propios en `cart/`).

## Request — sin cambios

```
POST /uploads/presign
Auth: admin del tenant
```

```json
{ "filename": "helado.jpg", "content_type": "image/jpeg", "folder": "products" }
```

- `folder`: whitelist `"products"` | `"logo"` | `"payment-methods"` (sin cambios, FR-016). `"comprobantes"` **no** es un valor válido aquí.
- `content_type`: debe estar en `CONTENT_TYPE_EXTENSIONS` (`image/jpeg|png|webp|gif`), si no → `422` (sin cambios).
- La key **nunca** usa `filename`; se genera con `build_object_key` (`{tenant}/{folder}/{uuid4hex}.{ext}`) (sin cambios, FR-016).

## Response — un solo cambio

```json
{
  "upload_url": "https://<r2-endpoint>/...firmada...",
  "key": "heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png",
  "public_url": "https://assets.skeilopos.com/heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png",
  "expires_in": 300
}
```

| Campo | Antes | Después |
|---|---|---|
| `upload_url` | igual | igual |
| `key` | igual (ya se devuelve hoy) | igual |
| `public_url` | `{R2_PUBLIC_BASE_URL}/{key}` (dominio `pub-…r2.dev`) | **`asset_display_url(key)` → `{ASSETS_BASE_URL}/{key}` (dominio `assets.skeilopos.com`)** |
| `expires_in` | igual | igual |

Cambio de documentación en `PresignResponse.public_url`: de *"URL pública final para guardar en image_url"* a *"URL de visualización lista para usar. **No** es lo que se persiste: en base de datos vive la `key` (spec 080). El servidor normaliza a key cualquier URL absoluta del bucket gestionado que reciba (FR-004)."*

## Qué se espera del consumidor

- **Comportamiento actual (aceptado)**: el frontend guarda `public_url` en su estado local y lo reenvía en el `PATCH`/`POST` correspondiente. El servidor lo normaliza a `key` antes de persistir (FR-004). Funciona sin cambios en el frontend.
- **Comportamiento alternativo (también válido)**: un consumidor puede guardar `key` directamente y enviarla. El servidor la persiste tal cual.

En ambos casos, en base de datos queda **solo la key** (FR-002, SC-001).

## Invariantes

- Subir a `upload_url` y luego persistir `key` (o `public_url`, que se normaliza) deja el objeto accesible en `{ASSETS_BASE_URL}/{key}` (FR-005).
- El endpoint no escribe en base de datos; solo firma.
- El proveedor, el bucket y el esquema de nombres no cambian (FR-017).
