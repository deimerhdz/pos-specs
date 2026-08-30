# Contrato: anuncio de promociones de presentación en el menú QR público

Cubre el Incremento D (research.md D12, FR-021, US5). Toca `GET /menu` (público, comensal anónimo) y
su render en `../pos-heladeria`.

---

## 1. Cómo se expone el anuncio

**Decisión (cerrada): `_build_menu` NO cambia de firma.** Sigue devolviendo
`list[MenuCategoryResponse]` (`../pos-backend/app/api/v1/menu/router.py:80`). Cambiar su tipo de
retorno rompería el test `"CONGELA comportamiento corregido:"` de `test_menu_router.py` (que hace
`menu[0].products[0]`), la fixture compartida `cart_fixtures.py:379` y el endpoint QR
(`menu/router.py:210-214`), lo que exigiría una decisión de negocio (Principio III). No procede.

Los anuncios se exponen **aparte**, con una función nueva `_build_menu_promotions(db, now) ->
list[MenuPromotionAnnouncement]` (hermana de `_build_menu`, mismo módulo), en dos superficies:

1. **Endpoint hermano nuevo** `GET /menu/promotions` (público, mismo `x-tenant-host` que
   `GET /menu`) → `list[MenuPromotionAnnouncement]`.
2. **Flujo QR con token** (`menu/router.py:210-214`, ya devuelve un `dict` sin `response_model`):
   gana una clave `"promotions": list[MenuPromotionAnnouncement]` junto a `"menu"`, `"table"`,
   `"business"`.

```text
MenuPromotionAnnouncement {
  promotion_id: UUID
  promotion_name: string
  rules: [
    {
      presentation_name: string
      min_qty: int
      pack_price: Decimal
      text: string            # legible, construido en backend:
                              # "Llevando {min_qty} de cualquier sabor en presentación
                              #  {presentation_name} por {pack_price}"
    }, ...
  ]
}
```

`MenuCategoryResponse` / `MenuProductResponse` / `MenuVariantResponse` y el `response_model` de
`public_menu` (`list[MenuCategoryResponse]`) **no cambian** (incluido `discounted_price` /
`discount_kind`, que siguen sin reflejar esta modalidad — ver `cobro-y-preview.md` §4).

## 2. Regla de vigencia del anuncio (FR-021, aclaración 2026-08-26, SC-006)

`_build_menu_promotions(db, now)` (hermana de `_build_menu`, en el mismo módulo) devuelve **solo**
las promociones `type == "qty_price_presentation"` que cumplan **todas**:

- `status == "active"`, y
- `_valid_now(promo, now)` verdadero — es decir, **vigente en este instante** según su ventana de
  día y de hora, evaluada en la **zona horaria del tenant**.

Fuera de la ventana de día/hora **no se anuncia**, aunque la promoción no esté pausada. `now` se
construye como `datetime.now(timezone.utc)` (aware, `menu/router.py:82`) y `local_now`
(`service.py:62-73`) lo convierte — este camino no arrastra el bug A-08 (que afecta a construcciones
`naive`); se verifica en el test.

Una promoción activa pero fuera de ventana: el catálogo (`GET /menu`) se calcula igual que hoy y la
lista de `_build_menu_promotions` sale vacía para ella.

## 3. Frontend (`../pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`)

- Banner/sección visible **sin agregar nada al carrito** (CA-11), con el `text` de cada regla.
- Si la lista de anuncios viene vacía, no se renderiza nada (CA-11 escenario 2: fuera de ventana no
  aparece).
- `menu.service.ts`: método/tipo nuevo para `GET /menu/promotions` (o la clave `promotions` del
  contexto QR con token). El tipo de retorno de `getMenu()` **no cambia**.

## 4. Fuera de este contrato

- El menú del **staff** (POS) — su previsualización de vigencia es el dominio de A-09; esta
  modalidad no cambia ese trato (research.md D13).
- El precio con descuento por variante en el menú — no aplica a esta modalidad (§1, nota).
