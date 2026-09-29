# Contrato: comprobantes del comensal

Aplica a `POST /cart/submit`, `POST /cart/payment-attempts/{attempt_id}/receipt` y a las tres respuestas que exponen `receipt_file_url`. Los presign (`POST /cart/payment-receipt/presign`, `POST /cart/payment-attempts/{id}/receipt/presign`) **no cambian**: siguen devolviendo `key` y `public_url` (dominio público anterior).

## Entrada

| Endpoint | Campo | Antes | Después |
|---|---|---|---|
| `POST /cart/submit` | `receipt_file_url` (opcional, máx. 500) | texto libre | key **o** URL absoluta del bucket gestionado; se valida y se persiste **la key** |
| `POST /cart/payment-attempts/{id}/receipt` | `file_url` (requerido, máx. 500) | texto libre | ídem |

Los nombres de campo **no cambian** (compatibilidad con el frontend actual, que reenvía `public_url`). Formato aceptado (FR-008, ambos durante la transición):

| Valor recibido | Resultado |
|---|---|
| `acme/comprobantes/{n}` (existe) | se guarda tal cual |
| `https://pub-….r2.dev/acme/comprobantes/{n}` (dominio público anterior, existe) | se extrae la key y se guarda la key |
| `https://assets.skeilopos.com/acme/comprobantes/{n}` (dominio de assets, existe) | ídem |
| key inventada (no existe) | **422** "La imagen no se encontró en el almacenamiento. Sube el archivo de nuevo." |
| key de otro negocio, de otra carpeta o mal formada (`..`, `//`, `\`, mayúsculas distintas) | **422** "El comprobante no pertenece a este negocio o no es válido." |
| URL de otro origen | **422** "El comprobante debe ser un archivo subido desde la aplicación." |
| R2 no responde | **503** reintentable, sin crear orden ni intento |

Orden de evaluación (respeta los códigos actuales):

- `submit_cart`: (1) errores actuales de carrito/orden activa/método → sin cambios; (2) efectivo con comprobante → 422 actual; (3) no efectivo sin comprobante → 422 actual; (4) **nuevo**: `resolve_receipt_key` del valor recibido; (5) `check_availability` y creación de la orden con la **key**. Un 422/503 en (4) no crea orden, ni intento, ni borra el carrito.
- `attach_receipt`: (1) intento propio y pendiente (404/409 actuales); (2) efectivo → 409 actual; (3) **`409` "El intento ya tiene un comprobante adjunto" se mantiene primero** (US5-5); (4) **nuevo**: `resolve_receipt_key`; (5) se guarda la key.

En base de datos **solo la key** (FR-008). El `receipt_hash` de los eventos de auditoría de estas dos operaciones se calcula sobre la **key** persistida (research D11).

Sin garantía de autoría: solo prefijo del negocio + carpeta `comprobantes` + existencia (FR-008, Assumptions).

## Salida

`receipt_file_url` pasa a serializarse con `AssetUrl` en:

| Esquema | Superficie |
|---|---|
| `PaymentAttemptResponse` (`orders/schemas.py`) | `GET /orders/{id}/payment-attempts` — **"Pagos por confirmar" del cajero** |
| `CurrentPaymentAttemptSummary` (`orders/schemas.py`) | `current_payment_attempt` dentro de `OrderResponse` (cajero y comensal) |
| `DinerPaymentAttempt` (`cart/schemas.py`) | respuestas del comensal (`attach_receipt`, `create_payment_attempt`) |

| Valor almacenado | Valor devuelto |
|---|---|
| key | `{ASSETS_BASE_URL}/{key}` |
| URL absoluta histórica del dominio público anterior o de assets (fila sin migrar) | `{ASSETS_BASE_URL}/{key}` (tolerancia de lectura) |
| URL de otro origen (fila histórica) | intacta |
| `null` | `null` |

El contrato hacia el frontend se conserva: sigue recibiendo **una URL lista para renderizar** (SC-006, SC-008). Solo puede cambiar el dominio (`assets.skeilopos.com` en vez de `pub-…r2.dev`) para los comprobantes.

## No cambia

`409` por comprobante ya adjunto, `422` de efectivo con comprobante y de método que exige comprobante, límite de 500 caracteres, presign (forma y dominio de `public_url`), y ningún cálculo de montos (Superficies de cobro: ninguna cambia su total; "Pagos por confirmar" sigue mostrando el comprobante).

## Compatibilidad con clientes

| Cliente | Efecto |
|---|---|
| Frontend actual (envía `public_url`) → backend nuevo | Funciona: se extrae la key del dominio público anterior. |
| Frontend que envía `key` → backend **anterior** | **No usar**: el backend anterior guardaría la key cruda y el cajero vería una imagen rota. Por eso el envío de `key` desde el frontend es el último paso opcional (research D10). |
