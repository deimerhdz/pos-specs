# Contrato: `GET /cart/payment-methods` (ampliado)

Endpoint ya existente (spec 024/025), consumido por el comensal autenticado con su token de sesión de
mesa. Esta funcionalidad **no** cambia su autenticación, su método HTTP, su ruta, ni el resto de su
respuesta — solo agrega el campo `fields` a cada método de pago devuelto.

## Antes (comportamiento actual)

```json
[
  {
    "id": "uuid",
    "name": "Nequi",
    "is_cash": false,
    "payment_info": {
      "numero_celular": "3001234567",
      "codigo_qr": "https://.../qr-nequi-tenantX.png"
    }
  }
]
```

El frontend no tiene forma de saber que `codigo_qr` es una imagen y `numero_celular` es texto — hoy
renderiza ambos como texto plano (bug que corrige spec 034, User Story 3).

## Después (respuesta ampliada por esta funcionalidad)

```json
[
  {
    "id": "uuid",
    "name": "Nequi",
    "is_cash": false,
    "payment_info": {
      "numero_celular": "3001234567",
      "codigo_qr": "https://.../qr-nequi-tenantX.png"
    },
    "fields": [
      { "key": "numero_celular", "required": true, "format": "numeric" },
      { "key": "codigo_qr", "required": false, "format": "image" }
    ]
  }
]
```

- `fields` usa la misma forma que ya devuelve `GET /sales/payment-methods/catalog` para el
  tenant-admin (`PaymentMethodFieldDefinition`) — no se inventa un esquema nuevo.
- El frontend empareja cada `key` de `fields` con la clave correspondiente en `payment_info` para
  decidir el render: `format: "image"` → `<img>`; `format: "text" | "numeric"` → texto legible.
- Si un método no tiene ningún campo con `format: "image"` configurado, el frontend simplemente no
  intenta renderizar ninguna imagen para ese método (FR-013 de spec 034 — sin espacio roto).

## Compatibilidad

- Campo puramente aditivo — cualquier consumidor existente que ignore `fields` sigue funcionando sin
  cambios.
- No requiere versión de API nueva ni cambio de ruta.

## No se agregan endpoints nuevos

La creación/reintento del pedido sigue usando `POST /cart/submit` tal cual está definido en spec 025
(`SubmitCartIn{payment_method_id, receipt_file_url}`) — ver `research.md`, Decisión 2, para por qué no
se necesita un endpoint nuevo para "reconocer un comprobante ya subido tras recargar".
