# Contradicción 03 — "¿Este insumo está bajo mínimo?" se calcula en tres sitios distintos del backend, y solo dos de los tres coinciden

**Fecha**: 2026-08-15
**Alcance**: `pos-backend` (`app/api/v1/inventory/service.py`,
`app/api/v1/inventory/router.py`, `app/api/v1/reports/service.py`) y `pos-heladeria`
(`src/app/modules/inventory/services/inventory.service.ts`,
`src/app/modules/reports/services/reports.service.ts`,
`src/app/modules/inventory/pages/inventory-page.component.ts`).
**Método**: lectura directa de las tres consultas SQL y de los tres puntos del frontend que
las consumen. Corresponde y amplía `RN-INV-15 [DISCREPANCIA]` de `reglas-de-negocio.md:501`.

---

## 1. Las implementaciones implicadas

**(A) Filtro `low_stock` del listado general** — `app/api/v1/inventory/service.py:24-43`
(`list_items_query`), expuesto en `GET /inventory/items?low_stock=true` vía
`app/api/v1/inventory/router.py:31-48`.

```python
if low_stock:
    stmt = stmt.where(InventoryItem.current_stock <= InventoryItem.min_stock)
```

El filtro `active` es un parámetro **independiente y opcional** de la misma consulta
(`router.py:37`, `active: bool | None = Query(None)`) — si no se pasa explícitamente,
`list_items_query` no restringe por `active` en absoluto (línea 39-40:
`if active is not None: ...`).

**(B) Endpoint dedicado `/items/low-stock`** — `app/api/v1/inventory/router.py:51-61`.

```python
return db.execute(
    select(InventoryItem).where(
        InventoryItem.active.is_(True),
        InventoryItem.current_stock <= InventoryItem.min_stock,
    ).order_by(InventoryItem.name)
).scalars().all()
```

Consulta inline en el router, sin pasar por `service.py`. Exige `active=True` siempre, sin
parámetro para desactivarlo.

**(C) `below_min` del reporte de inventario** — `app/api/v1/reports/service.py:114-128`
(`inventory_report`), expuesto en `GET /reports/inventory`.

```python
rows = db.execute(
    select(InventoryItem).where(InventoryItem.active.is_(True)).order_by(InventoryItem.name)
).scalars().all()
...
"below_min": stock <= Decimal(it.min_stock),
```

También exige `active=True` (línea 116), calculado en Python tras traer las filas, con la
misma comparación `<=`.

**Consumo en el frontend, cada uno por su lado**:

- (A) se dispara desde el checkbox "bajo mínimo" de la tabla de Insumos:
  `inventory.service.ts:96` (`itemsLowStock` signal) y `:169` (`if (lowStock) params =
  params.set('low_stock', 'true')`); el filtro `active` de esa misma pantalla
  (`itemsActive`, línea 95) por defecto vale `''` (sin filtrar) — línea 167-168 solo añade
  `active=true/false` si el usuario lo elige explícitamente en un selector aparte.
- (B) alimenta la tarjeta de estadística de la página de Inventario:
  `inventory.service.ts:187-199` (`loadLowStock`, GET a `/items/low-stock`) →
  `lowStockItems` signal, consumido en `inventory-page.component.ts:106`
  (`{{ service.lowStockItems().length }}`).
- (C) alimenta el reporte de "Insumos por reponer":
  `reports.service.ts:112-115` (`inventoryQuery`, GET a `/reports/inventory`) y
  `:177-188` (`lowStockIngredients`, filtra `.filter(i => i.below_min)`), consumido en
  `reports-page.component.ts:269-307,420-427` (conteo y listado en la página de Reportes).

## 2. ¿Usan la misma convención o algoritmo?

La comparación numérica es idéntica en las tres (`current_stock <= min_stock`, sin
tolerancia ni redondeo). La divergencia es exclusivamente sobre **si se filtran los
insumos inactivos**:

| Implementación | Filtra `active=True` | Uso |
|---|---|---|
| (A) `/items?low_stock=true` | No (a menos que el usuario active el filtro aparte) | Checkbox en la tabla de Insumos |
| (B) `/items/low-stock` | Sí, siempre | Tarjeta "bajo mínimo" del dashboard de Inventario |
| (C) `/reports/inventory` → `below_min` | Sí, siempre | Reporte "Insumos por reponer" |

(B) y (C) coinciden entre sí. (A) es la que diverge, y es la única con alcance ajustable
por el usuario — pero cuyo valor por defecto (sin filtro de `active`) no coincide con el de
las otras dos.

## 3. Ejemplo concreto con resultado distinto

Insumo "Leche en Polvo — línea descontinuada": `active=False` (el negocio dejó de comprarla
a ese proveedor y la desactivó para que no aparezca en nuevas recetas, pero conserva
existencias residuales), `current_stock=2`, `min_stock=10` — claramente bajo mínimo por
cualquier criterio numérico.

- **Tarjeta del dashboard de Inventario** (fuente B): no la cuenta. El número que ve el
  dueño al entrar a "Inventario" ("X insumos bajo mínimo") no incluye este insumo.
- **Reporte "Insumos por reponer"** (fuente C): tampoco la cuenta, por el mismo motivo —
  coincide con B.
- **Tabla de Insumos con el checkbox "bajo mínimo" activado, sin tocar el selector de
  Activo/Inactivo** (fuente A, comportamiento por defecto de la pantalla): **sí** la
  incluye en los resultados, porque `active` no se pasa como parámetro salvo que el
  usuario lo seleccione explícitamente.

El mismo insumo, en el mismo instante, aparece en una pantalla del sistema y no en las
otras dos — sin que ninguna de las tres se declare "la lista completa" ni "la lista
filtrada": cada una simplemente responde la pregunta con un criterio distinto, sin
avisarlo.

## 4. Cuándo se manifiesta y cuándo coinciden

Coinciden siempre que no haya insumos inactivos por debajo de su mínimo — es decir,
siempre que la práctica del negocio sea reactivar o ajustar `min_stock` a 0 al retirar un
insumo del catálogo. Divergen en el momento en que un insumo se desactiva sin que nadie
ajuste su `min_stock` o su `current_stock` — un caso plausible porque desactivar (`active`)
y "vaciar" el insumo (poner su stock o su mínimo a cero) son dos acciones administrativas
distintas, y nada en el sistema obliga a hacer la segunda al hacer la primera. Esto
explica por qué la divergencia puede pasar desapercibida durante mucho tiempo: el dueño
del negocio confía en el número de la tarjeta del dashboard (fuente B, la más visible) y
en el reporte (fuente C) — ambos consistentes entre sí — y probablemente nunca activa el
checkbox de la tabla de Insumos con el filtro de Activo/Inactivo en blanco a la vez, que es
la única combinación donde la tercera fuente se aparta de las otras dos.

## 5. Historia probable

Las tres implementaciones viven en módulos distintos con historias de desarrollo
independientes: `inventory/router.py` es el CRUD original de insumos; `/items/low-stock`
tiene toda la apariencia de haberse añadido después, como atajo directo para una tarjeta de
UI puntual (está escrita inline en el router en vez de reutilizar `list_items_query`, señal
de que se resolvió rápido sin pasar por la capa de servicio ya existente); y
`reports/service.py:inventory_report` pertenece al módulo de Reportes, añadido en otra
iteración del proyecto con su propia consulta desde cero. Que (B) y (C) coincidan
exactamente en su criterio (`active=True` siempre) sugiere que quien escribió el reporte
copió, consciente o inconscientemente, el mismo criterio de "bajo mínimo" que ya existía en
el endpoint dedicado — pero nadie hizo el mismo ejercicio al añadir el parámetro
`low_stock` al listado general (A), que se trató como "un filtro más" de una consulta ya
genérica y por tanto heredó el comportamiento por defecto de esa consulta (no restringir
por `active` salvo que se pida). No hay comentario en ninguno de los tres sitios que
reconozca la existencia de las otras dos implementaciones.

---

**Pregunta abierta al negocio**: cuando un insumo se marca inactivo con existencias por
debajo de su mínimo, ¿el negocio espera seguir viéndolo en las alertas de "bajo mínimo", o
considera que "inactivo" ya implica "fuera de las alertas de reposición"? La respuesta
determina cuál de las tres implementaciones refleja la intención real.
