# Contradicción 06 — "Cuánto se le debe cobrar a esta mesa" tiene tres implementaciones independientes, y dos de ellas ignoran las promociones y las órdenes ya pagadas

**Fecha**: 2026-08-15
**Alcance**: `pos-backend`, `app/api/v1/table_sessions/service.py`,
`app/api/v1/orders/checkout.py`, `app/api/v1/orders/tables_advanced.py`. Incluye
verificación de uso real en `pos-heladeria`.
**Método**: lectura directa de las tres funciones, sus consultas de origen (qué órdenes
incluyen) y su tratamiento de promociones; `grep` de sus llamadores en backend y frontend
para establecer cuáles están vivas. Amplía `RN-ORD-02/RN-ORD-03` de
`reglas-de-negocio.md:951,1107` y `RN-ORD-64` (`reglas-de-negocio.md:1454`), y añade una
tercera implementación (`group_bill`) no contrastada en esos hallazgos.

---

## 1. Las implementaciones implicadas

**(A) `table_sessions/service.py:compute_bill(db, table_session_id)`** — líneas 139-181.
Docstring propio: "mismo cálculo por comensal que usa `_close_split` al cobrar, para que
este preview coincida con lo que realmente se cobrará."

- Fuente de órdenes: `_billable_orders` (líneas 126-136) — excluye explícitamente
  `('cancelada', 'pagada')` (línea 133).
- Aplica descuentos: sí — `promotions.evaluate(...)` (línea 165) y
  `promotions.combo_discount_for_lines(...)` (línea 166), por comensal.
- Expuesto en `GET /table-sessions/{id}/bill` (`table_sessions/router.py:117`).
- **Uso real confirmado en frontend**: `table-session.service.ts:53` →
  `GET /table-sessions/{id}/bill`. Es la vía activa para la cuenta de una mesa normal
  (sin fusionar con otras).

**(B) `orders/checkout.py:compute_bill(db, table_id)`** — líneas 127-180.

- Fuente de órdenes: consulta inline, líneas 130-138 — excluye únicamente
  `CustomerOrder.status != "cancelada"` (línea 135). **No excluye `"pagada"`.**
- Aplica descuentos: **no** — `line_total = Decimal(it.unit_price) * it.quantity` (línea
  155), suma directa sin llamar a `promotions.evaluate` ni a
  `combo_discount_for_lines`.
- Expuesto en `GET /orders/tables/{table_id}/bill` (`orders/router.py:307-312`).
- **Uso real confirmado en frontend**: ninguno — `grep` sobre `pos-heladeria/src/app` no
  encuentra ninguna llamada a esta ruta. Coherente con el hallazgo H2 de
  `flujo-pedido-qr.md:573-579`: pertenece al ciclo de cobro legado por pedido individual
  (`pay_order`/`block_order`), reemplazado por el cierre de sesión unificado.

**(C) `orders/tables_advanced.py:group_bill(db, group_id)`** — líneas 92-114.

- Fuente de órdenes: `CustomerOrder.merged_group_id == group_id` (línea 96) — **sin
  ningún filtro de `status`**; solo excluye ítems individuales con
  `estado_cocina == "anulado"` (línea 106).
- Aplica descuentos: **no** — misma suma directa `unit_price * quantity` que (B) (líneas
  104-108).
- Expuesto en `GET /orders/group/{group_id}/bill` (`orders/router.py:125-131`).
- **Uso real confirmado en frontend**: `table.service.ts:133` →
  `GET /orders/group/${groupId}/bill`. Es la vía activa para la cuenta consolidada de
  **mesas fusionadas** (feature de "mesas avanzadas": mover, fusionar y agrupar mesas).

## 2. ¿Usan la misma convención o algoritmo?

No. Las tres calculan "la suma de lo que hay que cobrar", pero con dos ejes de diferencia:

| | (A) sesión, activo | (B) mesa, sin caller | (C) grupo, activo |
|---|---|---|---|
| Excluye `pagada` | Sí | **No** | **No** |
| Excluye `cancelada` | Sí | Sí | Sí (implícito: no filtra status, pero anulación es por ítem, no por orden) |
| Resta promociones/combos | Sí (`promotions.evaluate` + `combo_discount_for_lines`) | No | No |

(A) es la única que replica exactamente lo que se cobrará de verdad. (B) y (C) comparten el
mismo patrón — suma cruda sin descuentos ni exclusión de pagadas — pero (B) está inactiva
mientras que (C) es la vía real y vigente para mesas fusionadas.

## 3. Ejemplo concreto con resultado distinto

Mesa 5, dos rondas de pedido en la misma sesión/mesa:

- **Orden A** (ronda 1): subtotal $20.000, ya cerrada y pagada individualmente
  (`status='pagada'`) — un escenario que el propio sistema permite a través del ciclo
  legado pedido-a-pedido (H2), aunque hoy sin caller de UI conocido, o mediante cualquier
  vía futura que reactive ese flujo.
- **Orden B** (ronda 2): subtotal bruto $15.000, con una promoción `percent` del 10%
  vigente sobre esa categoría → subtotal real tras descuento $13.500. Sigue `abierta`.

**Vía (A), `table_sessions.compute_bill`** (la que efectivamente ve el cajero en el panel
de cuenta hoy): `_billable_orders` descarta la orden A (`pagada`) → solo procesa la orden
B → aplica el 10% → **total: $13.500**.

**Vía (B), `orders.checkout.compute_bill`** (sin caller hoy, pero funcional si se
invocara directamente o se reactivara el flujo legado): no descarta la orden A (solo
excluye canceladas) → suma A + B **sin** aplicar el 10% de B → **total: $20.000 + $15.000
= $35.000**.

Diferencia: **$21.500** sobre una cuenta de $13.500 reales — un 159% de sobreestimación,
por dos motivos acumulados (orden ya pagada que vuelve a sumarse, y descuento vigente que
no se resta).

**Vía (C), `group_bill`** (la vía real y activa para mesas fusionadas): si la mesa 5
estuviera fusionada con otra dentro de un `merged_group_id`, y la orden A estuviera
`pagada` mientras la B sigue abierta, `group_bill` reproduciría el mismo error que (B):
sumaría los $20.000 de la orden ya pagada y no restaría el descuento de B, mostrando
$35.000 en la cuenta del grupo de mesas en vez de los $13.500 reales — con la diferencia de
que esta vía **sí está en uso hoy** para cualquier negocio que use la función de mesas
fusionadas.

## 4. Cuándo se manifiesta y cuándo coinciden

Las tres coinciden cuando ninguna orden de la mesa/sesión/grupo alcanzó el estado `pagada`
mientras otras siguen abiertas (es decir, mientras el cobro sea siempre "todo junto, al
final") y cuando ninguna promoción automática está vigente sobre las líneas en juego. Este
es, con alta probabilidad, el caso normal en una heladería que cobra la cuenta completa de
la mesa al final del servicio, lo que explica por qué la divergencia de (B) no se ha
notado (no tiene caller) y por qué la de (C) tampoco, pese a estar en uso: solo se activa
en la combinación específica de "mesas fusionadas" **y** ("una orden del grupo ya se pagó
por separado" **o** "hay una promoción automática vigente sobre alguna línea del grupo").
Ambas condiciones son plausibles pero no triviales de que coincidan a la vez en el uso
diario, y ninguna de las dos por sí sola requiere que el negocio use mesas fusionadas con
frecuencia para que el riesgo sea real.

## 5. Historia probable

El comentario de la propia `table_sessions.compute_bill` ("mismo cálculo... para que este
preview coincida con lo que realmente se cobrará") y el commit `ee94f30`
("feat(promotions): add promotions and combos module", visto también en la Contradicción
05) describen explícitamente el objetivo de esa migración: "Wires the same discount
evaluation into all three checkout paths (counter sale, legacy `pay_order`, and table
session close) **plus the session bill preview**, which previously summed raw line totals
and disagreed with what the cashier was actually charged." Es decir, el propio autor del
motor de promociones identificó y corrigió este problema — pero solo para el preview de
**sesión** (A), que es el que menciona explícitamente el mensaje del commit. `group_bill`
(C), que vive en `tables_advanced.py` (un módulo de "mesas avanzadas": mover, fusionar,
agrupar — una funcionalidad añadida en el commit `f7e5582`, "feat(orders): advanced table
management (states, move, merge, group bill)", anterior al módulo de promociones), quedó
fuera del alcance de esa migración porque conceptualmente es una vista distinta (la cuenta
de un *grupo de mesas*, no de una *sesión*) y nadie volvió a tocarla al añadir promociones.
`orders/checkout.py:compute_bill` (B) es, con toda probabilidad, el remanente del ciclo de
cobro por pedido individual anterior al rediseño de sesiones de mesa unificadas (coherente
con H2 de `flujo-pedido-qr.md`), dejado en el código sin desmontar tras la migración.

---

**Pregunta abierta al negocio**: ¿se usa la función de fusionar/agrupar mesas en el
día a día del local? Si es así, ¿alguna vez se ha cobrado una orden de una mesa fusionada
por separado del resto del grupo antes de cerrar todo el grupo? Eso determinaría si el
escenario de la sección 3 ya ocurrió en la práctica.
