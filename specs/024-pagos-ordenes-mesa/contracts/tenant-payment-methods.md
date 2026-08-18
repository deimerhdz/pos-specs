# Contrato: configuración de métodos de pago del tenant (US1)

Router: `app/api/v1/sales/router.py`. Auth: `Depends(require_tenant_admin)` para escritura,
`Depends(get_current_user)` para lectura (sin cambio — ya así hoy).

## `GET /sales/payment-methods` — SIN CAMBIO DE CONTRATO

Ya existe (`router.py:25`). Gana el campo `payment_info` en la respuesta (ver abajo) — mismo
endpoint, mismo status code, mismo caso de éxito.

## `POST /sales/payment-methods` — MODIFICADO (campo nuevo opcional)

Ya existe (`router.py:30`). `PaymentMethodCreate` gana:

```text
PaymentMethodCreate:
  name: str (requerido, único por tenant — sin cambio)
  type: "cash" | "card" | "transfer" | "other" (requerido — sin cambio)
  payment_info: dict[str, str] | null = null   # NUEVO — solo tiene sentido si type != "cash"
```

Respuesta (`PaymentMethodResponse`) gana `payment_info: dict[str, str] | null`.

Sin cambio de status codes (`201` éxito, `409` si `name` ya existe — `ensure_unique`, sin cambio).

## `PATCH /sales/payment-methods/{id}` — NUEVO

No existe hoy (verificado: el router solo tiene `GET`/`POST` para `payment-methods`). Es el
endpoint que faltaba para editar datos, activar y desactivar (US1, Acceptance Scenarios 1-4).

```text
Request:
  PaymentMethodUpdate:
    name: str | undefined          # si se envía, se re-valida unicidad (ensure_unique)
    payment_info: dict[str,str] | null | undefined
    active: bool | undefined

Response: PaymentMethodResponse (200)
```

**Reglas**:
- Si `active` pasa de `true` a `false`: el servicio cuenta métodos del tenant con `active = true`
  excluyendo este; si el resultado sería 0, `409 Conflict` con mensaje explicando que debe quedar al
  menos un método activo (FR-003, Acceptance Scenario 3 de US1).
- Editar `payment_info`/`name` **no** modifica ningún `OrderPaymentAttempt` ya creado con este
  método — el intento solo guarda `payment_method_id`, no una copia de los datos (Acceptance
  Scenario 4 de US1).
- `404` si el `id` no pertenece al tenant.

**Responses**:
| Código | Caso |
|---|---|
| `200` | Actualizado. |
| `404` | Método no existe en este tenant. |
| `409` | `name` duplicado, o desactivación dejaría 0 métodos activos. |
| `401`/`403` | Sin autenticar / no es admin del tenant (sin cambio del patrón existente). |
