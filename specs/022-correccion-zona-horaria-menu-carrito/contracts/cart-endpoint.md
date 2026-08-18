# Contrato: endpoints de `cart/router.py` que devuelven `CartResponse`

Endpoints existentes (`app/api/v1/cart/router.py`: `GET /cart`, `POST /cart/items`,
`PATCH /cart/items/{item_id}`, `DELETE /cart/items/{item_id}`, y el alta de combo), todos
terminan en `service.get_cart` → `service.serialize_cart` (`cart/service.py:266-271`,
research.md Decisión 4), **sin cambios de forma** en esta spec — mismo esquema `CartResponse`,
mismos códigos de estado. Lo único que cambia es **cuándo**, en el límite horario de una
promoción, `serialize_cart` la considera vigente para calcular `discounted_total`.

## Request

```
GET    /cart
POST   /cart/items
PATCH  /cart/items/{item_id}
DELETE /cart/items/{item_id}
Authorization: token de sesión de comensal (Depends(get_session_context)) — sin cambios
```

## Response — `200 OK` (`CartResponse`)

Esquema **sin cambios**. Cambia únicamente `discounted_total`/`discounted_unit_price`/
`discounted_line_total` de cada línea cuando una promoción con ventana horaria está cerca de su
límite: antes de esta corrección, `serialize_cart` podía aplicar (o no aplicar) el descuento hasta
5 horas fuera de la ventana real; después, coincide exactamente con la hora local real y con
`RN2` de `spec.md` (paridad con el cobro real, ya corregido por A-07).

## Response — `404` / `422`

Sin cambios: participante o ítem inexistentes, datos de entrada inválidos.

## Consumidor (`pos-heladeria`)

El carrito del comensal ya consume estos endpoints. Ningún cambio de contrato de tipos ni de
código de estado — el único efecto visible es que `discounted_total` deja de divergir del monto
que efectivamente se cobrará al confirmar el pedido.
