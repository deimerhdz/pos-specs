# Contrato: Exclusividad de producto entre promociones vigentes (bug 3, nuevo)

Cubre spec.md FR-021 a FR-025. Cita anomalía **A-78** (research.md D7) — debe existir antes de
mergear. Complementa, sin reemplazar, la guarda existente `_guard_variant_overlap`
(`app/api/v1/promotions/service.py:579-632`, spec 063 FR-014).

## Backend — nueva guarda de servicio

**Función nueva**: `_guard_product_overlap` (nombre de trabajo), junto a `_guard_variant_overlap`
en `app/api/v1/promotions/service.py`. Invocada desde los mismos puntos que esa guarda:
`create()` (~línea 742) y `update_shape()` (~línea 781), **después** de `_guard_variant_overlap`.

**Criterio** (distinto al de `_guard_variant_overlap`, ver research.md D3):

```
Para la promoción que se está creando/editando (promo_actual):
  product_ids_nuevos = { product_id de cada variante en las reglas de promo_actual }
  Para cada otra promoción con status == "active" (excluyendo promo_actual.id):
    product_ids_cubiertos = { product_id de cada variante en sus reglas }
    intersección = product_ids_nuevos ∩ product_ids_cubiertos
    Si intersección no está vacía → 409, listando qué producto(s) y qué promoción(es)
    activa(s) ya lo(s) cubre(n)
```

Sin evaluación de vigencia horaria (fechas/días/horas) — a diferencia de `_guard_variant_overlap`,
el criterio es únicamente `status == "active"` (spec.md, Clarifications: "por ahora, un producto no
puede incluirse en otra promoción si ya hace parte de una promoción vigente" → se interpretó como
`status=active`, no `draft`/`paused`).

**Mensaje de error** (409): `"El producto '<nombre>' ya hace parte de la promoción activa
'<nombre_promoción>'."` — spec.md FR-022 exige que el administrador sepa **por qué**, no solo que
la operación falló.

**No retroactivo** (FR-025): esta guarda solo se evalúa al crear o guardar/activar una promoción
(mismos puntos que hoy usa `_guard_variant_overlap`); promociones `active` ya guardadas hoy que
compartan un producto entre sí (posible bajo el criterio anterior, por variante) no se re-validan
retroactivamente — solo se re-evalúan si alguna de ellas se edita o se intenta activar de nuevo.

**Dentro de la misma promoción** (FR-024): la comparación excluye siempre `promo_actual.id` de la
lista de "otras promociones" — seleccionar dos variantes del mismo producto dentro de la misma
promoción (p. ej. "Pequeña" y "Grande") no se ve afectado.

## Backend — endpoint de apoyo para el Paso 1 del frontend

Para que el buscador de productos del Paso 1 pueda excluir/deshabilitar productos ya cubiertos
**antes** de que el administrador intente guardar (spec.md FR-022, no solo rechazar al guardar),
`GET /products` (o el endpoint que alimenta el buscador del Paso 1) gana un parámetro opcional de
consulta, p. ej. `exclude_active_promotions=true`, que anota cada producto candidato con
`blocked_by_promotion: { id, name } | null` cuando alguna de sus variantes ya está cubierta por una
promoción `active` distinta a la que se está editando (si se está editando una, su id se excluye
vía `?exclude_promotion_id=<id>`). Alternativa equivalente: resolverlo enteramente en frontend
llamando a `GET /promotions?status=active` una vez al entrar al Paso 1 y cruzando localmente contra
los productos candidatos — decisión de detalle a definir en tasks.md según el volumen típico de
promociones activas por tenant (research.md no encontró un límite documentado que obligue a una
opción sobre la otra).

## Frontend — Paso 1 (`promotions-page.component.ts`)

- Un producto marcado como bloqueado no aparece como seleccionable (o aparece deshabilitado con
  tooltip/texto: "Ya hace parte de la promoción activa '<nombre>'").
- Al guardar/activar, si el backend igual rechaza por una condición de carrera (dos promociones
  guardándose casi al mismo tiempo), el mensaje 409 se muestra tal cual al administrador — no se
  reintenta automáticamente ni se oculta el error (spec.md, Edge Cases y User Story 5 escenario 3).

## Casos de aceptación cubiertos

- Producto con una variante ya cubierta por una promoción `active` → no seleccionable en el Paso 1
  de una promoción distinta, con explicación.
- Mismo producto, dos variantes distintas, ambas dentro de la misma promoción en edición → permitido
  sin cambio.
- Dos promociones guardándose casi al mismo tiempo compartiendo un producto → la segunda en
  guardarse/activarse es rechazada por el backend con el mismo mensaje.
- Promociones ya `active` hoy que comparten un producto (bajo el criterio anterior, por variante) →
  sin cambio hasta que alguna se edite o reactive.
