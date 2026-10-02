# Data Model: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Research**: [research.md](./research.md)

## Alcance de este documento

**No hay cambios de persistencia.** Esta spec es exclusivamente de presentación: cero entidades nuevas, cero campos nuevos en base de datos, cero cambios de relaciones, cero migraciones y, por lo tanto, cero estrategia de rollback de datos. El **Principio VIII no aplica** y está declarado así en el Constitution Check de [plan.md](./plan.md).

Lo que sí hay que modelar es el **modelo de vista**: qué datos consume el resumen, de dónde salen exactamente, y qué dos derivaciones nuevas aparecen en el cliente. Eso es lo que documenta este archivo.

## Origen del dato: una sola cadena, sin eslabones nuevos

```text
PostgreSQL (carts, cart_items, promociones vigentes)
        ↓  cálculo oficial del backend — NO se toca
GET /cart  →  CartResponse { total, discounted_total, items[] }
        ↓  checkoutHydrationGuard → cart.load()  (ya existe, una vez por entrada al checkout)
DiningCartService  { lines, total, count, grossTotal*, savings* }
        ↓  inputs
app-checkout-order-summary   ←  consumido por review-step (paso 1) y transfer-details-step (paso 3)
```

`*` = los dos únicos miembros nuevos de toda la funcionalidad.

Ningún eslabón se añade: no hay petición HTTP nueva (FR-010), no hay campo nuevo en `CartResponse` (spec → *Decisiones de Compatibilidad*) y no hay cálculo nuevo en el cliente (FR-014 / RN-001). El guard `checkoutHydrationGuard` ya recarga el carrito en cada entrada al checkout y devuelve al comensal al menú si está vacío — por eso el resumen **no tiene estado vacío que modelar** (spec → *Edge Cases*).

## Entidades del modelo de vista

### 1. Línea del pedido — `CartLine` (existente, sin cambios)

Definida en `src/app/modules/tables/services/dining-cart.service.ts`. El resumen consume estos campos y **ningún otro**:

| Campo | Tipo | Qué pinta el resumen | FR |
|-------|------|----------------------|-----|
| `id` | `string` | Clave de `@for` (`track line.id`) | — |
| `quantity` | `number` | `"2×"` al inicio del renglón | FR-007 |
| `productName` | `string` | Nombre del producto | FR-007 |
| `variantName` | `string` | Presentación, tras un `·` | FR-007 |
| `optionNames` | `string[]` | Adicionales elegidos, unidos con `", "`; el renglón no se pinta si está vacío | FR-007 |
| `notes` | `string \| null` | Nota del comensal, entre comillas y en cursiva; no se pinta si es `null` o vacío | FR-007 |
| `lineTotal` | `number` | Valor total de la línea, a la derecha | FR-007, FR-015 |

**Reglas de validación / invariantes** (todas ya garantizadas aguas arriba, el resumen no las revalida):

- `lineTotal` ya es `unit_price × quantity + addons_total`, con el descuento de línea aplicado si lo hay. Los adicionales se cobran **una sola vez por línea** y no escalan con `quantity` (spec 089, A-94). El resumen **muestra** ese número; no lo recompone (FR-015).
- `optionNames` llega ya formateado con la cantidad de cada adicional (`formatQuantifiedLabel`). El resumen no la vuelve a formatear.
- `lines` nunca está vacío en estas dos pantallas: el guard redirige al menú antes (FR — sin estado vacío).

### 2. Total del pedido — `DiningCartService.total` (existente, sin cambios de semántica)

`Signal<number>`. Es `effectivePrice(cart.total, cart.discounted_total)`: el total **vigente**, ya con el descuento de promoción aplicado cuando existe (FR-004). Lo consumen hoy `review-step`, `cart.component` y `public-menu`; su significado **no cambia** con esta spec, para no arrastrar pantallas fuera de alcance (Principio V).

### 3. Total de lista — `DiningCartService.grossTotal` ⭐ NUEVO

| Propiedad | Valor |
|-----------|-------|
| Tipo | `Signal<number>` (`signal`, escrito en `apply()`) |
| Origen | `Number(CartResponse.total)` — el total **sin** descuento |
| Hoy | Se **descarta**: `apply()` sólo conserva el resultado de `effectivePrice(...)` |
| Por qué se necesita | Es el minuendo del "Ahorro" (FR-012). Sin él no hay forma de conocer el descuento sin inventar un cálculo propio, prohibido por FR-014 |
| Valor por defecto | `0`, igual que `total`, antes de la primera carga |

### 4. Ahorro por promoción — `DiningCartService.savings` ⭐ NUEVO

| Propiedad | Valor |
|-----------|-------|
| Tipo | `Signal<number>` (`computed`) |
| Fórmula | `Math.max(0, grossTotal() - total())` |
| Semántica | `0` significa "no hay promoción aplicada" → la fila **no se pinta** (FR-012: ni en cero ni vacía) |
| Contrato aguas arriba | Fijado por characterization tests de `pos-backend` (`test_cart_service.py`): `discounted_total` es `None` sin promoción (no cero) y **estrictamente menor** que `total` con promoción |
| Por qué `max(0, …)` | Si alguna vez llegara un `discounted_total ≥ total`, la fila simplemente no aparece, en vez de mostrar un "Ahorro" negativo |

No es un campo persistido ni un campo de `CartResponse`: es un **derivado de dos números que el backend ya resolvió**, exactamente como lo describe la spec ("no es un campo permanente del pedido").

### 5. Conteo de productos — `DiningCartService.count` (existente, sin cambios)

`Signal<number>` = `lines().reduce((n, l) => n + l.quantity, 0)`. Son **unidades**, no líneas (FR-003).

> **Invariante que no se puede perder de vista**: el conteo que se le muestra al comensal se mide en **unidades** (`count()`), mientras que el umbral de colapso se mide en **líneas** (`lines().length`). Son dos medidas con propósitos distintos y sólo la primera se muestra (FR-003, FR-006 vs. FR-008). Un pedido de 2 líneas con 4 unidades cada una dice "8 productos" y aparece **expandido**.

## Transiciones de estado

El modelo de datos no tiene transiciones nuevas. La única máquina de estados que aparece es la del elemento `<details>`, y es del navegador, no de la aplicación:

```text
                      lines().length ≤ 3
       ┌──────────────────────────────────────┐
       │                                      ▼
   [entrada al paso 3]                   ABIERTO ◄──┐
       │                                      │     │ click/Enter/Espacio
       │          lines().length > 3          ▼     │ en el <summary>
       └──────────────────────────────► CERRADO ────┘
```

- El estado inicial se **escribe una sola vez** y nunca se reafirma (research.md D2): el componente lo congela en `ngOnInit`, así que ninguna detección de cambios puede cerrar de golpe el panel que el comensal abrió (FR-009).
- Las transiciones posteriores las ejecuta el navegador. **No hay estado de aplicación asociado**, no se persiste, no sobrevive a una navegación, y **ninguna transición dispara efecto alguno** — ni mutación del carrito, ni petición, ni envío (FR-010). Esa garantía es estructural: no hay handler donde colgar un efecto.
- En el paso 1 esta máquina **no existe**: el resumen se renderiza plano, sin `<details>` y sin control de colapso (FR-022).

## Lo que deliberadamente NO está en el modelo

| Ausencia | Por qué |
|----------|---------|
| **Impuestos** (a nivel de línea, pedido o método de pago) | El sistema no los calcula ni los almacena en ningún nivel del pedido del comensal. Modelarlos está **fuera de alcance** (FR-013, RN-003). Una fila derivada o en cero sería información falsa frente al comensal. |
| **Subtotal** | Misma razón: no existe en `CartResponse` y el front no inventa cifras intermedias (FR-013). |
| **Estado "resumen vacío"** | Inalcanzable: `checkoutHydrationGuard` devuelve al comensal al menú cuando `cart.isEmpty()`, incluso si recarga el navegador directamente en el paso 3. |
| **Estado de apertura persistido** | Nadie lo pidió y persistirlo requeriría estado propio, que es justo lo que research.md D1/D2 eliminan. |
| **Recarga en vivo del pedido en el paso 3** | Fuera de alcance por la propia spec (*Assumptions* y *Fuera de Alcance*). El resumen refleja el pedido tal como se cargó al entrar al checkout. |

## Compatibilidad y rollback

| Dimensión | Situación |
|-----------|-----------|
| **Datos existentes** | Intactos. No se escribe nada en ninguna tabla. |
| **Facturas emitidas** | Sin tocar, ni en importe ni en representación (Principio VII). El resumen opera sobre el carrito borrador, que todavía no es pedido ni factura. |
| **Contrato de API** | Sin cambios. `grossTotal` se alimenta de `CartResponse.total`, un campo que ya existe y ya viaja en la respuesta. |
| **Backends antiguos** | Compatible: si `discounted_total` viene ausente o `null`, `grossTotal() === total()` y `savings()` es `0`, así que la fila "Ahorro" no aparece — que es exactamente el comportamiento correcto para un pedido sin promoción. |
| **Rollback** | Revertir los commits del frontend. No queda ningún rastro en datos, ningún esquema que deshacer y ninguna sesión de comensal en estado incompatible (un comensal en el paso 3 ve la pantalla anterior al recargar, con su comprobante rehidratado y su pedido intacto). |
