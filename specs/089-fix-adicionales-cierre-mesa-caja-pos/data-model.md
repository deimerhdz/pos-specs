# Modelo de datos — Spec 089

**Fecha**: 2026-09-30 · **Spec**: [spec.md](./spec.md) · **Decisiones**: [research.md](./research.md) D1–D7

Solo la Historia 1 (y por arrastre la 3) toca el modelo de datos. Las Historias 2, 4 y 5 **no** cambian el esquema.

## 1. Cambios de esquema (una migración de Alembic por tenant)

Revisión nueva, `down_revision = 'deb68185a606'` (cabeza actual: `087_drop_cash_partial_counts`). Usa `@for_each_tenant_schema` como las demás migraciones por tenant.

| Tabla (`tenant.*`) | Columna nueva | Tipo | Nulo | Default de servidor | Uso |
|---|---|---|---|---|---|
| `cart_items` | `addons_total` | `NUMERIC(12,2)` | NO | `0` | Σ(precio adicional × cantidad elegida), una vez por línea |
| `order_items` | `addons_total` | `NUMERIC(12,2)` | NO | `0` | Ídem, copiado del carrito al confirmar |
| `cart_item_options` | `per_line` | `BOOLEAN` | NO | `false` | La opción es un adicional de la regla nueva |
| `order_item_options` | `per_line` | `BOOLEAN` | NO | `false` | Ídem |

Restricción: `CHECK (addons_total >= 0)` en ambas tablas de ítems (`ck_cart_item_addons_total_nonneg`, `ck_order_item_addons_total_nonneg`).

`sale_items`: **sin columna nueva**. `line_total` (ya guardado) es autoritativo. El snapshot JSONB `options` gana la clave opcional `"per_line": true` en líneas nuevas (las antiguas no la tienen ⇒ se lee como `false`).

Sin índices nuevos (ninguna consulta filtra por estas columnas). Sin tocar `invoices`.

## 2. Semántica por tipo de línea

| Tipo de línea | `unit_price` | `addons_total` | Opciones | `line_total` |
|---|---|---|---|---|
| **Histórica** (antes del despliegue) | presentación + extras por unidad | `0` | `per_line = false` | `unit_price × qty` (igual que hoy) |
| **Terminal POS / pedido manual / mostrador** | ídem histórica | `0` | `per_line = false` | ídem |
| **Menú QR nueva o editada** | solo precio de la presentación | Σ(extra × cantidad de opción) de los grupos **con recargo** | recargo → `true`; incluido → `false` | `unit_price × qty + addons_total` |

Invariantes (se prueban en los tests de servicio):
1. `addons_total = Σ(option.extra_price × option_row.quantity)` de las filas con `per_line = true`, calculado **una vez** al crear/editar la línea y nunca releído del precio vigente.
2. Una línea con `addons_total > 0` tiene al menos una opción `per_line = true`. Una línea con opciones `per_line = true` de precio $0 tiene `addons_total = 0` (adicional gratuito: solo cambia su consumo).
3. `per_line = true` **solo** se escribe desde `cart/service.py` (`add_item` y `update_item`) y se **copia** (nunca se recalcula) en `submit_cart`, `consolidate_table` y `set_assignments`.

## 3. Compatibilidad con datos existentes

- Todas las filas actuales quedan con `addons_total = 0` y `per_line = false` por el `DEFAULT` de la propia migración ⇒ **cero** cambio de total, comanda o descuento de inventario (FR-008, SC-003). No hay `UPDATE` de backfill.
- Agregar columnas con `DEFAULT` constante es solo metadatos en PostgreSQL 16 (sin reescritura de tabla) ⇒ la migración es instantánea incluso con muchos negocios.
- Carritos abiertos al desplegar: conservan su regla (por unidad) hasta que el comensal edite o agregue una línea; una línea editada pasa a la regla nueva (spec: "líneas nuevas o editadas").
- El código nuevo funciona con datos sin migrar **no** (necesita las columnas): la migración corre antes de arrancar la versión nueva del backend (misma disciplina que las demás migraciones del repositorio).

## 4. Migración

`upgrade()` (por tenant, idempotente con `IF NOT EXISTS` vía `inspect`):
```
ALTER TABLE tenant.cart_items          ADD COLUMN addons_total NUMERIC(12,2) NOT NULL DEFAULT 0;
ALTER TABLE tenant.order_items         ADD COLUMN addons_total NUMERIC(12,2) NOT NULL DEFAULT 0;
ALTER TABLE tenant.cart_item_options   ADD COLUMN per_line BOOLEAN NOT NULL DEFAULT false;
ALTER TABLE tenant.order_item_options  ADD COLUMN per_line BOOLEAN NOT NULL DEFAULT false;
+ CHECK (addons_total >= 0) en cart_items y order_items
```

## 5. Reversión (Principio VIII)

Tres niveles, de menos a más invasivo:

1. **Apagar el comportamiento** — `QR_ADDONS_PER_LINE=false` y reiniciar el backend. Las líneas nuevas vuelven a la regla por unidad; las ya creadas con la regla nueva siguen valiendo (se leen por sus marcas). No hay pérdida de datos ni descuadre.
2. **Retirar el código sin borrar columnas** — solo seguro si **no existen** líneas con `addons_total > 0` (`SELECT count(*) FROM tenant.order_items WHERE addons_total > 0` = 0 en todos los esquemas). Si existen, un binario anterior las **subcobraría** (ignora la columna): no revertir el binario; usar el nivel 1.
3. **Emergencia con datos ya creados** — `python -m app.scripts.fold_line_addons` (nuevo, simulación por defecto, `--apply`): para **`cart_items`** (carritos aún no enviados) pliega `addons_total` dentro de `unit_price` cuando `addons_total` es divisible entre `quantity` a 2 decimales (`unit_price += addons_total/quantity`, `addons_total = 0`, `per_line = false`); lista las que no son exactas para resolución manual. **No toca `order_items`** ni ventas: su inventario ya se descontó con la regla nueva y una reversa posterior con la regla vieja descuadraría el kardex. `downgrade()` de Alembic elimina las columnas y **exige** que no exista ninguna línea con `addons_total > 0` en `cart_items` ni en `order_items`; si existe, aborta con un mensaje que apunta **solo** a la bandera `QR_ADDONS_PER_LINE=false`. Como el script no toca `order_items`, **el downgrade es de un solo sentido** en cuanto exista el primer pedido confirmado con adicionales: desde entonces la única reversa soportada es el nivel 1 (bandera). El script solo sirve para limpiar carritos abiertos.

## 6. Otros datos

- **Sin migración** para las Historias 2, 4 y 5.
- **Configuración**: `QR_ADDONS_PER_LINE: bool = True` en `app/core/config.py` (Settings). No requiere variable de entorno para el funcionamiento normal.
- **Evento SSE `session.closed`**: el campo `reason` admite ahora `empty` además de `paid | swept | released` (campo libre para el cliente, que lo ignora; ver [contracts/session-closed.md](./contracts/session-closed.md)).

## 7. Entidades y relaciones (resumen)

```
cart_items (1) ──< cart_item_options      (per_line)      ─┐ copia al enviar
   unit_price, addons_total                                 ├─► order_items (1) ──< order_item_options (per_line)
                                                            │      unit_price, addons_total, discounted_*
order_items ──(cobro)──► sale_items { unit_price, line_total, options[JSONB{per_line?}] }
```
