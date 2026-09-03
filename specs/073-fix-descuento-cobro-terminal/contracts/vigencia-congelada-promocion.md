# Contrato: Instante congelado de vigencia de promociones

**Cubre**: FR-008 a FR-012a — User Story 4 (spec.md). **Anomalía**: `A-70` (pendiente de
registrar en `registro-de-anomalias.md` antes de implementar — Principio II).
**Research**: [research.md](../research.md) D1, D2, D3, D7, D8.

Este contrato no es un endpoint REST — es el contrato **interno** de cómo el instante de vigencia
fluye desde que se crea un pedido hasta que queda registrado en la venta. Vive aquí porque, a
diferencia de un detalle de implementación, es observable desde afuera: decide qué descuento
recibe un pedido y qué queda escrito en la venta emitida (FR-011a), así que cualquier
implementación de esta spec debe respetarlo exactamente.

## El flujo

```
1. Crear pedido (orders/service.py::create_order o cart/service.py)
   └─ CustomerOrder.promotion_evaluated_at = datetime.now(timezone.utc)   [FR-008]
      (aware UTC — la columna es DateTime(timezone=True), NO naive; ver data-model.md.
       Una sola vez, nunca se actualiza — ni al agregar/anular ítems, ni al mover de mesa)

2. Ítems cambian después de creado (agregar / anular — FR-010)
   └─ CustomerOrder.promotion_evaluated_at NO cambia
      El descuento SÍ se recalcula en el siguiente preview/cobro, porque auto_discount()
      siempre lee las líneas VIVAS de la orden — nunca cachea un monto.

3. Cobrar (preview o cobro real — cualquier call site de research.md D3)
   └─ instant = promotion_evaluation_instant(orders, now=hora_del_cobro)   [FR-009, FR-012, FR-012a]
        • 1 pedido:      instant = pedido.promotion_evaluated_at  (o `now` si es NULL → FR-012)
        • N pedidos       instant = MIN(promotion_evaluated_at de los que lo tengan)
        (rondas, mesa):   (o `now` si NINGUNO lo tiene → FR-012)                    [FR-012a]
        • mesas fusionadas (group_bill): el helper se llama POR PEDIDO,
          promotion_evaluation_instant([o], now=now) dentro del bucle — NO el MIN
          del grupo, porque cada pedido se cobra individualmente                    [FR-018a]
   └─ discount = auto_discount(db, lines, instant)
        • La vigencia TEMPORAL (fechas/día/franja) se evalúa contra `instant`.
        • El ESTADO de la promoción (activa/pausada/borrada) se lee SIEMPRE
          contra el momento actual de la base de datos, nunca congelado.        [FR-009a]

4. Emitir venta (build_sale)
   └─ Sale.promotion_evaluated_at = instant   (el mismo valor usado en el paso 3)  [FR-011a]
      Sale.discount / Sale.applied_promotions = lo que ya calculaba antes (sin cambio de forma)
      Una vez creada, la venta es inmutable — nada de este flujo la vuelve a tocar.  [FR-011]

5. Leer la venta (SaleResponse / detalle de venta en el frontend)              [FR-011a, SC-009]
   └─ SaleResponse expone promotion_evaluated_at (UtcDatetime | None); GET /sales/{id}
      lo devuelve. El detalle de venta pinta ese instante cuando hubo descuento, para
      distinguir una promoción hoy vencida de una falla. No se expone applied_promotions
      (fuera de alcance — spec 063).
```

## Invariantes que cualquier implementación debe preservar

1. **`promotion_evaluated_at` de un `CustomerOrder` se escribe exactamente una vez, en su
   creación, y nunca se lee para nada más que decidir el `instant` del paso 3.** No es un
   timestamp de auditoría general — moverlo, tocarlo al editar el pedido, o reutilizarlo con otro
   propósito rompe FR-008.
2. **El estado de la promoción nunca se congela.** Ningún cambio de esta spec debe empezar a
   cachear `Promotion.status` en ningún punto — `active_variant_set_rules` sigue consultando la
   tabla `promotions` en vivo en cada llamada (FR-009a, ya así hoy, research.md D8).
3. **Pedidos sin `promotion_evaluated_at` (anteriores a esta spec) se comportan exactamente igual
   que hoy** — `instant` cae a la hora del cobro, sin ninguna rama especial que los distinga
   "peor" o "mejor" (FR-012). No hay ninguna migración retroactiva que les asigne un valor.
4. **En una cuenta con varias rondas, todas las líneas de un mismo comensal (de todas las rondas)
   se evalúan juntas contra el MISMO `instant`** (FR-012a) — la agrupación de líneas para el
   umbral de cantidad (`evaluate_variant_sets`) no cambia; solo cambia contra qué instante se
   valida la vigencia temporal de la regla que las cubre.
4b. **En la cuenta consolidada de mesas fusionadas cada pedido usa su propio instante** (FR-018a),
   no el `MIN` del grupo — porque esos pedidos se cobran individualmente por
   `checkout_and_send`/`pay_order` y el preview consolidado debe coincidir con el cobro. Distinto
   de la invariante 4 (rondas de una misma mesa, que sí comparten un instante porque se cobran
   juntas).
5. **Una venta ya emitida no se recalcula jamás**, sin importar qué tan vencida esté hoy la
   promoción que aplicó — `Sale.promotion_evaluated_at` es lo que permite explicar ese descuento
   después (FR-011a/SC-009), no lo que dispara ningún recálculo (FR-011).
6. **El instante es aware UTC de punta a punta** — columna `DateTime(timezone=True)`, helper,
   `build_sale`, `SaleResponse`. Nunca naive: `local_now()` (`promotions/service.py`) interpreta
   un `datetime` naive como hora local del tenant, lo que desplazaría la evaluación de la franja
   por el offset (−5 en Colombia).
7. **Ningún chequeo previo evalúa el total con su propia aritmética** (adenda US7 — FR-021/FR-023).
   El chequeo del "monto recibido" de `confirm_cash_payment_attempt` deja de usar `_order_total`
   (que sumaba `unit_price × cantidad + domicilio` sin descuento) y pasa a
   `compute_checkout_preview(...).total`, que aplica `promotion_evaluation_instant` +
   `auto_discount` igual que el paso 3 — de modo que "lo que el panel muestra", "lo que el chequeo
   valida" y "lo que `build_sale` registra" son el mismo número. `_order_total` se elimina
   ([research.md D13](../research.md)).

## Anomalía A-70 (Principio II de la Constitución)

Antes de implementar el paso 3 del flujo anterior (que deroga el comportamiento actual de evaluar
siempre con la hora del cobro), debe existir en
`specs/000-reconocimiento/registro-de-anomalias.md` una entrada:

> **A-70 — [DECISIÓN DE NEGOCIO — spec 073]** Qué cambia: la vigencia temporal de las promociones
> se evalúa contra el instante de creación del pedido, no contra la hora del cobro. Por qué:
> el precio que se le cantó al cliente al tomar el pedido es el que se le debe cobrar. Quién y
> cuándo: dueño/desarrollador del proyecto, 2026-09-02. Afecta: `auto_discount`/
> `evaluate_variant_sets` en todos sus call sites (checkout.py, table_sessions/service.py,
> tables_advanced.py — research.md D3). No incluye congelar el **estado** de la promoción
> (FR-009a).

(Estado 2026-09-02: la entrada **ya está registrada** en
`specs/000-reconocimiento/registro-de-anomalias.md` como `### A-70 — [DECISIÓN DE NEGOCIO —
spec 073]`, justo después de `A-69` — tarea T001, completada antes de implementar la Fase 6. La
entrada real desarrolla más que el resumen de arriba: alcance de los 8 call sites, mesas
fusionadas por pedido (FR-018a) y el campo `sales.promotion_evaluated_at`.)
