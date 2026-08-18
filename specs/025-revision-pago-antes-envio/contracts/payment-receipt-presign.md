# Contrato: `POST /cart/payment-receipt/presign` (US3, US4)

Endpoint nuevo. Auth: `Depends(get_session_context)` — mismo patrón que el resto de `cart/router.py`
(el tenant sale del token de sesión, invariante A-24). A diferencia del presign ya existente de
spec 024 (`POST /cart/payment-attempts/{attempt_id}/receipt/presign`), **no recibe ningún
identificador de recurso** — en este punto del flujo todavía no existe ninguna orden ni ningún
intento de pago al que asociar el archivo (research.md, Decisión 2).

```text
Request:
  { content_type: str }   # mismo whitelist de CONTENT_TYPE_EXTENSIONS ya usado en todo presign

Response: 200
  { upload_url: str, key: str, public_url: str, expires_in: int }   # mismo shape que siempre
```

**Reglas**:
- `422` si `content_type` no está en la whitelist — único caso de error posible; no hay `404`/`409`
  porque no se valida contra ningún recurso.
- El comensal sube el archivo directo a R2 con `PUT {upload_url}` (fuera de la API, igual que
  siempre) y conserva `public_url` en memoria hasta llamar a `POST /cart/submit` con él
  (`receipt_file_url`).
- Si el comensal nunca completa el envío (vuelve atrás, cambia de método, cierra la pestaña), el
  archivo subido a R2 queda sin ningún registro que lo referencie — costo aceptado, no una falla
  (research.md, Decisión 6).
- Si `POST /cart/submit` falla después de una subida exitosa, el mismo `public_url` sirve para
  reintentar sin volver a llamar a este endpoint ni a subir nada de nuevo (FR-012, research.md
  Decisión 5).

## Lo que NO cambia

`POST /cart/payment-attempts/{attempt_id}/receipt/presign` (spec 024) sigue existiendo tal cual,
para el único caso donde sí hay un intento de pago ya creado al que asociar el archivo: el
reintento tras un rechazo (spec 024, Historia 5). Los dos presign coexisten porque resuelven
necesidades distintas — uno antes de que la orden exista, el otro después.
