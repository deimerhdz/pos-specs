# Phase 1 — Data Model

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-01

## Resumen: cero cambios de esquema

**Esta feature no crea, modifica ni elimina ninguna tabla, columna, restricción, índice ni
relación. No lleva migración de Alembic. No tiene estrategia de rollback de datos porque no toca
datos.** Lo exige FR-019 y el diseño lo cumple sin excepciones.

Todo lo que aquí se describe es **información derivada**: se calcula al construir la respuesta,
viaja al cliente y se descarta. Ninguna de estas entidades se persiste, y por lo tanto el
Principio VIII (evolución del modelo de datos) no se activa — no hay valores por defecto que
definir, ni compatibilidad hacia atrás de datos existentes que negociar.

El modelo persistido sigue siendo exactamente el de la spec 063:

```text
Promotion (vigencia + estado)
└── PromotionRule (type, value, min_qty)          ← se LEE, no se modifica
    └── PromotionVariant (product_variant_id)     ← se LEE, no se modifica
        └── ProductVariant (name, price) ── Product (name)   ← se LEE para nombrar
```

Lo único que cambia respecto de producción es **de dónde sale el nombre**: hasta hoy
`variant_set_condition_text` no consultaba `ProductVariant.name` ni `Product.name` en absoluto.

---

## Entidades derivadas

### 1. Descriptor del conjunto

Representación legible del conjunto elegible de una regla. Existe solo durante la construcción del
texto de condición.

| Campo | Tipo | Origen |
|---|---|---|
| `names` | `list[str]` | Nombres **distintos** y no vacíos de las variantes del conjunto, deduplicados y ordenados (FR-002, FR-003). |
| `text` | `str` | Los nombres unidos según el tamaño de `names` (1 / 2-3 / >3). |
| `multiple` | `bool` | `len(names) > 1`. Gobierna el conector `"entre "` de FR-004. |
| `fallback` | `bool` | `True` cuando ninguna variante aportó nombre → se usa el descriptor por conteo (FR-006). |

**Reglas de validación**
- El nombre utilizable de una variante es `variant.name.strip()`; si queda vacío,
  `product.name.strip()`; si también queda vacío, la variante **no aporta nombre**.
- La deduplicación es por el nombre **tal como se muestra**, no por variante: ocho variantes
  `"Pequeño 8oz"` de ocho productos distintos producen un solo elemento (FR-003).
- El orden es alfabético sobre la clave normalizada (sin tildes, sin distinguir mayúsculas); el
  texto conserva el nombre original, no la clave.
- `names` nunca menciona la cantidad de variantes del conjunto (FR-003).

**No se persiste**: se calcula con los nombres vigentes del catálogo cada vez que se muestra. Si
el administrador renombra una presentación, el descriptor cambia en la siguiente lectura, sin
migración ni recálculo.

**Transiciones de estado**: ninguna. Es un valor, no una entidad con ciclo de vida.

Especificación normativa completa (algoritmo, tabla de casos, formatos):
[contracts/texto-condicion.md](./contracts/texto-condicion.md).

---

### 2. Información de promoción por variante (`MenuVariantPromotion`)

El paquete de datos que el menú necesita para pintar una presentación cubierta por una regla
vigente. Lo calcula el backend (FR-007) y viaja dentro de `MenuVariantResponse.promotion`.

| Campo | Tipo | Nulo | Significado |
|---|---|---|---|
| `condition_text` | `str` | no | Condición completa de FR-004, **idéntica** a la del cartel y a la de administración. Es lo que hace verificable SC-005 y lo que FR-016 muestra en la terminal. |
| `short_condition` | `str` | no | Condición corta de FR-008: `"2 x $12.000"` (paquete) o `"3 x -15%"` (porcentaje). |
| `unit_equivalent` | `Decimal` | no | Equivalente por unidad, ya redondeado al peso (FR-009). |
| `unit_equivalent_approx` | `bool` | no | `True` cuando el importe exacto no era entero en pesos (FR-009). |
| `unit_equivalent_text` | `str` | no | `"$6.000 c/u"` o `"≈ $4.333 c/u"`, ya formateado. |
| `display_text` | `str` | no | La línea completa que pinta el modal: `"2 x $12.000 · $6.000 c/u"`. |
| `type` | `"percent" \| "package_price"` | no | Tipo de la regla que cubre la variante. |
| `min_qty` | `int` | no | Cantidad mínima de la regla. |
| `value` | `Decimal` | no | Valor de la regla (precio de paquete o porcentaje). |

**Reglas de validación e invariantes**
- Se puebla **solo** si la variante pertenece al conjunto de una regla cuya promoción está vigente
  en ese instante (estado, fechas, días y franja horaria en la zona del tenant). En cualquier otro
  caso, `promotion is None` (FR-007).
- Hay **a lo sumo una** regla vigente por variante: la spec 063 FR-014 bloquea el solapamiento
  real, así que no existe criterio de desempate (FR-012). La implementación toma la primera
  coincidencia sin ordenar.
- `unit_equivalent` es **informativo**: no entra en el carrito, ni en el subtotal, ni en el cobro
  (FR-011).
- Para `package_price`, `unit_equivalent` es el **mismo para todas** las variantes del conjunto
  (`value / min_qty`); para `percent` se calcula sobre el precio normal **de esa variante**
  (FR-008).
- Con `min_qty == 1` el bloque se sigue emitiendo (`short_condition` describe la regla) **y**
  además se puebla `discounted_price` — ver abajo.

**No se persiste.** Es un DTO de respuesta.

Especificación normativa completa (cálculo, redondeo, formatos):
[contracts/menu-info-promocion.md](./contracts/menu-info-promocion.md).

---

### 3. Campos existentes cuyo significado se extiende

Estos **no son nuevos**; ya viajan en `MenuVariantResponse` desde la spec 063. Cambia qué los
puebla.

| Campo | Antes | Ahora |
|---|---|---|
| `discounted_price` | Solo `percent` con `min_qty == 1`. | `percent` con `min_qty == 1` (sin cambio) **y** `package_price` con `min_qty == 1`, donde vale el **valor de la regla** (FR-010, A-68). |
| `discount_kind` | Siempre `"percent"` cuando había descuento. | El `type` real de la regla que lo pobló: `"percent"` o `"package_price"`. |

**Invariante nuevo de FR-010**: para `package_price` con `min_qty == 1`, `discounted_price` es el
valor de la regla **siempre**, incluso si resulta mayor o igual que `price`. El menú muestra
entonces ese valor sin tachado y sin señal de descuento (FR-015); nunca se presenta un aumento
como ahorro. La consecuencia buscada es que **el importe mostrado coincida con el cobrado**
(SC-003), que es el defecto que A-68 autoriza a corregir.

Ese caso —valor de regla ≥ precio normal— **es alcanzable en producción** aunque
`_guard_package_is_discount` lo rechace al crear, actualizar y activar: la guarda no corre cuando
el catálogo cambia un precio. Ver [research.md D-6](./research.md).

---

### 4. Insignia de promoción (tarjeta del menú QR)

Señal booleana por producto, **derivada en el frontend**: `product.variants.some(v => v.promotion
!= null)` (FR-013). No es un campo de la API ni una entidad — se documenta aquí porque la spec la
lista como entidad clave.

No se añade `has_promotion` a `MenuProductResponse`: sería superficie de contrato duplicada y
desincronizable, y FR-013 pide explícitamente derivarla de la información por variante. Ver
[research.md D-11](./research.md).

---

## Lo que esta feature NO toca (FR-019, verificado)

- `Promotion`, `PromotionRule`, `PromotionVariant`: ni columnas ni restricciones ni relaciones.
- `evaluate_variant_sets`, `_greedy_units`, `_distribute_group_discount`: el motor de cálculo del
  cobro, intacto. SC-006 se verifica sin modificar ningún aserto de importe de la spec 063.
- `_valid_now`, `local_now`, `_in_time_window`: el criterio de vigencia, intacto (A-57 incluido).
- `_guard_variant_overlap`, `_guard_package_is_discount`, `PROMOTION_TRANSITIONS`: bloqueo de
  solape, guarda de descuento y máquina de estados, intactos.
- `sales.applied_promotions`, `invoices.applied_promotions`, `customer_orders.discount`: la
  persistencia del descuento en la venta, intacta. Ninguna venta, factura ni pedido ya emitido se
  altera en importe ni en representación (FR-020, Principio VII).
