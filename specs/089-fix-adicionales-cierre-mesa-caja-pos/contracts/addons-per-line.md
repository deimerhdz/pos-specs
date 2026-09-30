# Contrato — Adicionales cobrados una vez por línea (Menú QR)

**Historias**: 1 y 3 · **FR**: 001–013 · **Decisiones**: research D1–D7 · **Modelo**: [../data-model.md](../data-model.md)

## 1. Regla de cálculo (única fuente)

Nuevo módulo puro `app/catalog_engine/core.py`:

```python
class ChosenOption(NamedTuple):
    option: "Option"
    quantity: int
    per_line: bool = False          # NUEVO — default = comportamiento histórico

def compute_unit_price(variant, options) -> Decimal
    # precio de la presentación + Σ(extra × quantity) de las opciones per_line=False
def compute_addons_total(options) -> Decimal
    # Σ(extra × quantity) de las opciones per_line=True
def compute_line_price(variant, options) -> Decimal
    # SIN CAMBIO de firma ni de resultado para llamadores existentes:
    # = compute_unit_price + compute_addons_total  (con per_line=False ⇒ igual que hoy)
def line_total(unit_price, quantity, addons_total=0) -> Decimal
    # unit_price × quantity + addons_total    ← la única fórmula de total de línea
```

`cart/service.py` marca `per_line=True` en las opciones de grupos `pricing_type == 'con_recargo'` cuando `settings.QR_ADDONS_PER_LINE` es verdadero, y guarda `unit_price = compute_unit_price(...)`, `addons_total = compute_addons_total(...)`. Cualquier otro llamador sigue pasando `per_line=False` (=> `unit_price = compute_line_price`, `addons_total = 0`).

### Ejemplos normativos

| Caso | unit_price | addons_total | Cantidad | line_total |
|---|---|---|---|---|
| 2 hamburguesas $15.000 + 1 tocino $3.000 (QR) | 15.000 | 3.000 | 2 | **33.000** |
| Subir a 3 hamburguesas, tocino sigue x1 | 15.000 | 3.000 | 3 | **48.000** |
| 1 hamburguesa + 2 unidades del mismo adicional ($3.000) | 15.000 | 6.000 | 1 | 21.000 |
| Promoción 20 % sobre hamburguesa + tocino x1, cant. 2 | 15.000 | 3.000 | 2 | 33.000 − 6.000 = **27.000** (el 20 % solo sobre 30.000) |
| Línea histórica (2 × $18.000 incl. tocino por unidad) | 18.000 | 0 | 2 | 36.000 (**sin cambio**) |
| Grupo incluido (sabores) cantidad 2 | según hoy | 0 | 2 | según hoy |

## 2. Consumo de inventario y comanda

`plan_line_consumption` (`catalog_engine/consumption.py`), por opción con insumo:

```
per_line == True :  per_unit × chosen.quantity                 # NO × quantity de la línea
per_line == False:  per_unit × quantity_de_la_línea × chosen.quantity   # igual que hoy
```
Receta fija de la variante: `× quantity` (igual). Un helper `chosen_from_rows(db, item)` construye `ChosenOption` **leyendo `per_line` de la fila**; lo usan descuento, reversa, `_cart_consumption`, consolidación y anulación/reemplazo. Descuento y reversa **siempre** usan la misma marca de fila.

## 3. Cambios de respuesta HTTP (aditivos, retrocompatibles)

| Endpoint / esquema | Campo nuevo | Notas |
|---|---|---|
| `CartItemResponse` (`GET/POST/PATCH /cart…`) | `addons_total: Decimal = 0` | `line_total` ya existía y ahora = `unit_price×qty + addons_total` |
| `CartItemOptionResponse` | `per_line: bool = false` | |
| `OrderItemResponse` (`/orders…`, `/cart/orders`) | `addons_total: Decimal = 0`, `line_total: Decimal` | `line_total` **nuevo** (antes el cliente lo calculaba con `unit_price×qty`) |
| `OrderItemOptionResponse` | `per_line: bool = false` | |
| `BillItemLine` (`/orders/tables/{id}/bill`) | — | `line_total` ya existe, ahora incluye adicionales |
| `SessionBillItem` (`/table-sessions/{id}/bill`) | — | ídem |
| `SaleItemResponse` / `InvoiceItem` | — | `line_total` ya guardado; `unit_price` sigue siendo "por unidad de producto" |

Ningún campo existente cambia de tipo ni se elimina. Un cliente que ignore los campos nuevos y siga usando `unit_price × quantity` **subcobraría en pantalla** líneas nuevas: por eso el frontend de esta spec migra todas sus lecturas a `line_total` (ver §5).

## 4. Endpoints que ya soportan la Historia 3 (sin cambio de firma)

- `PATCH /cart/items/{item_id}` — body `{quantity?, options?, notes?}`. Con `options` reemplaza la selección, revalida (`load_valid_options(..., variant)`), recalcula `unit_price`/`addons_total`, marca `per_line`. Errores: 404 línea inexistente/ajena/ya enviada; 422 selección inválida (mensaje del grupo); 409 stock.
- Aislamiento por comensal y "solo sin enviar": el carrito abierto se resuelve por `participant_id` del token; al enviar el carrito se elimina físicamente (spec 038).

## 5. Sitios que reemplazan `unit_price × quantity` por `line_total`

**Backend**: `cart/service.py` (`serialize_cart`, `submit_cart`), `cart/router.py` (total del evento `order.created`), `orders/checkout.py` (`compute_bill`, `order_sale_lines` → `SaleLine(addons_total=…)`), `table_sessions/service.py` (`compute_bill` vía `SaleLine`), `orders/tables_advanced.py` (vía `SaleLine`), `sales/builder.py` (`SaleLine.line_total`, `base_unit_price`), `orders/schemas.py::OrderItemResponse`.

**Frontend** (`pos-heladeria`): `dining-cart.service.ts`, `cart.component.ts` (etiqueta "c/u" y total), `public-menu.component.ts` (`itemLineTotal`, historial), `pos-terminal.store.ts` (`itemUnitPrice`/subtotales ~l.967-990, 1725, 2380-2394), `pos-order-panel.component.ts`, `payment-attempt-review-panel.component.ts` (~l.371), `dining.interface.ts`/`diner.interface.ts` (tipos).

### Presentación en el carrito del Menú QR

- Cada línea muestra el precio de la presentación ("$15.000 c/u"), los adicionales con su multiplicador exacto ("Tocino x1", spec 087 FR-013) y el total de la línea. El "c/u" se calcula con `unit_price` (no incluye adicionales).
- Acción **"Editar adicionales"** por línea del carrito (icono + texto, objetivo táctil ≥ 44 px): abre `ProductSelectComponent` con `initialSelection` (variante, cantidad y notas de la línea; opciones y notas preseleccionadas) y `addonsPerLine = true`. **Guardar cambios** → `PATCH /cart/items/{id}` con `{options, notes}` (la `quantity` no cambia). **Cancelar** no llama al backend.
- Selección inválida (mínimo/máximo/tope por opción): el botón ya se bloquea con `blockingLabel()`; si aun así el backend responde 422, se muestra su mensaje sin cerrar el selector.
- Línea cuyo producto/presentación se desactivó: sin acción "Editar", solo quitar.

## 6. Compatibilidad y pruebas

- Total de toda línea/pedido/venta/factura anterior: **idéntico** (defaults `0`/`false`).
- `"CONGELA comportamiento actual:"` afectados (se actualizan citando **A-94** en el mismo commit y con evidencia de que el resto sigue verde): `test_catalog_line_pricing`, `test_catalog_consumption_plan`, `test_cart_service`, `test_orders_consolidation`, `test_orders_service`.
