# Contrato: reordenar presentaciones de un producto

Cubre FR-001 a FR-003 y FR-010. Endpoint **nuevo** — no existe hoy.

## `PATCH /products/{product_id}/variants/reorder`

- **Request**:
  ```json
  {
    "variant_ids": [
      "<uuid presentación 1, nueva posición 1>",
      "<uuid presentación 2, nueva posición 2>",
      "<uuid presentación 3, nueva posición 3>"
    ]
  }
  ```
  `variant_ids` DEBE contener exactamente el conjunto de IDs de las presentaciones **activas** de
  `product_id`, sin duplicados y sin faltantes, en el orden deseado.

- **Response 200**:
  ```json
  { "variants": [ { "id": "...", "display_order": 1 }, { "id": "...", "display_order": 2 }, { "id": "...", "display_order": 3 } ] }
  ```
  Lista de presentaciones activas del producto con su `display_order` ya actualizado, en el mismo
  orden recibido.

- **Efecto**: en una sola transacción, asigna `display_order = 1..N` según la posición de cada ID en
  `variant_ids` (research.md Decisión 2). No modifica `name`, `sku`, `price`, `active`, ni ninguna
  receta/grupo de opciones — únicamente `display_order`.

- **Validaciones y errores**:
  - `404` si `product_id` no existe.
  - `422` si `variant_ids` no es exactamente el conjunto de IDs activos de ese producto (falta
    alguno, sobra alguno, o incluye un ID de otro producto o de una presentación desactivada) —
    detalle estructurado indicando qué IDs sobran/faltan, siguiendo el mismo estilo de error
    estructurado que ya usa `POST /products/{id}/variants` para nombres duplicados (spec 002,
    `RN-CAT-08`).
  - `422` si `variant_ids` tiene duplicados.

- **No requiere autenticación adicional** a la ya exigida por el resto de endpoints de edición de
  catálogo (mismo guard de rol administrador que `PATCH /variants/{id}`).

## Sin cambios de forma en otros endpoints

- `POST /products/{id}/variants`: sin cambio de *shape* en el request/response. Efecto nuevo:
  asigna `display_order = MAX(display_order) + 1` entre todas las presentaciones del producto
  (activas e inactivas) al crear (data-model.md, tabla de asignación).
- `PATCH /variants/{id}`, `DELETE /variants/{id}`: sin cambio de forma ni de `display_order`
  (research.md Decisión 4).
- `GET /products/{id}` (formulario) y el detalle de producto del Menú QR
  (`app/api/v1/menu/router.py`): sin cambio de forma — el campo `display_order` no se agrega a
  ningún schema de respuesta pública; el único efecto es el **orden** en que ya vienen serializadas
  las presentaciones dentro de `variants`, gracias al `order_by` de la relación (research.md
  Decisión 7). Un cliente que ignore el orden y solo lea el contenido de cada presentación no nota
  ningún cambio de forma.
