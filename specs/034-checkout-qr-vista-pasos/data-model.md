# Data Model: Vista de Pasos para Revisión y Pago del Menú QR

## Progreso de Revisión y Pago (nuevo — solo cliente, sin persistencia en base de datos)

Registro guardado en `localStorage` del navegador del comensal, con una clave por sesión/participante
(ej. `pos.diner.checkout_progress.<table_session_id>.<participant_id>`), junto a la clave que ya usa
el token de sesión (`pos.diner.session_token`, spec 007).

| Campo | Tipo | Descripción |
|---|---|---|
| `step` | `'method' \| 'transfer'` | Paso alcanzado en la vista (mismos dos pasos que ya modela `reviewStep` hoy). |
| `payment_method_id` | `string \| null` | Método de pago elegido, si ya se eligió uno. |
| `receipt_file_url` | `string \| null` | Referencia (`public_url`/`key`) devuelta por `POST /cart/payment-receipt/presign` una vez el comprobante se subió con éxito; `null` mientras no se haya subido nada. |
| `saved_at` | `string` (ISO datetime) | Momento en que se guardó el registro, usado solo para descartar registros fuera de la ventana de sesión vigente. |

**Reglas de validez** (FR-005 a FR-010 de spec 034):
- Se descarta (se trata como inexistente) si `saved_at` cae fuera de la ventana de sesión del
  comensal ya vigente (spec 007, ventana deslizante / tope absoluto) — no introduce un TTL propio.
- Se descarta si `payment_method_id` ya no corresponde a un método activo del tenant al momento de
  hidratar la vista (FR-010) — en ese caso se conserva el resto del progreso (resumen del pedido) y
  se le pide al comensal elegir de nuevo.
- Se elimina en cuanto el pedido se crea con éxito (`POST /cart/submit` responde 2xx) — a partir de
  ahí rigen **Orden**, **Intento de Pago** y **Comprobante** (spec 024/025), no este registro.
- Es local al dispositivo/navegador donde se guardó (Clarification Session 2026-08-24) — no se
  sincroniza ni se recupera desde otro dispositivo.

**No es una entidad de base de datos**: no requiere tabla, migración, ni estrategia de rollback
(Principio VIII no aplica) — vive enteramente en el navegador.

## DinerPaymentMethod (respuesta existente, ampliada)

`GET /cart/payment-methods` (`pos-backend/app/api/v1/cart/schemas.py:119`), sin cambios de base de
datos — agrega un campo derivado de datos que ya existen:

| Campo | Tipo | Estado |
|---|---|---|
| `payment_info` | `dict[str, str] \| null` | Ya existente — valores de integración capturados por el tenant (spec 032). |
| `fields` | `list[PaymentMethodFieldDefinition]` | **Nuevo** — misma definición que ya usa `GET /sales/payment-methods/catalog` (`sales/schemas.py:57`, `super_admin/schemas.py:18-26`): por cada campo, su clave, si es obligatorio, y su `format` (`text \| numeric \| image`). Permite al frontend saber qué clave de `payment_info` renderizar como `<img>`. |

**Origen del dato agregado**: `PaymentMethodCatalog.fields` (`models/payment_method_catalog.py:23-26`,
spec 032) — ya persistido; `list_payment_methods` (`cart/service.py:648-655`) solo necesita incluirlo
en la respuesta que ya arma hoy. Sin migración.

## Entidades reutilizadas sin cambio

- **Orden / CustomerOrder**, **Intento de Pago / OrderPaymentAttempt**, **Comprobante**: definidas en
  specs 024/025; esta funcionalidad no les agrega campos ni cambia su ciclo de vida.
- **Configuración de Método de Pago por Tenant** (spec 032): sin cambios; esta funcionalidad solo
  agrega cómo se **presenta** al comensal (vía el campo `fields` de arriba), no cómo se captura ni se
  administra.
