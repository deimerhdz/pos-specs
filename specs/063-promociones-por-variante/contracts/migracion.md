# Contrato: migración de lo existente y reescritura de tests

Ver [data-model.md](../data-model.md) §"Migración y rollback" para el SQL y
[research.md](../research.md) D12/D13/D18 para el porqué. Este documento fija (1) el
comportamiento observable de la migración de datos y (2) el **inventario completo** de tests
`CONGELA` afectados que `spec.md` §"Tests CONGELA afectados" pidió completar en `/speckit-plan`.

---

## 1. Migración de datos (FR-024 – FR-027)

**Dos revisiones** ([data-model.md](../data-model.md) §"Migración y rollback", Principio VI): el
**paso de datos** (esta tabla) va en `063a` (aditiva, Incremento A), con las tablas viejas aún
presentes. El **borrado** de `promotion_targets` / `promotion_combo_items` /
`promotion_presentation_rules` / `presentations` / `priority` / `presentation_id` va en `063b`
(destructiva, Incremento F), cuando ningún módulo referencia ya esa estructura.

| Situación previa (producción) | Después de la migración | FR |
|---|---|---|
| Promoción `percent` con `target` de **producto** | `type='percent'`, mismo `value`/`min_qty`/vigencia/estado; conjunto = todas las variantes **activas** de ese producto al momento de migrar (foto fija) | FR-026, US6-CA1 |
| Promoción `percent` con `target` de **categoría** | ídem; conjunto = todas las variantes **activas** de los productos de esa categoría al momento de migrar | FR-026, US6-CA1 |
| Promoción `percent` **global** (sin `targets`) | ídem; conjunto = todas las variantes **activas** del tenant al momento de migrar | FR-026 |
| Promoción `combo` (cualquier estado no terminal) | `status='finished'`, `closed_by_refactor_at=now()`, **`type` sin cambio** (`= 'combo'`, histórico); combo_items borrados con la tabla en `063b`; líneas de venta históricas con `combo_id` **intactas** | FR-024, FR-025, US6-CA2 |
| Promoción `qty_price` (producto/categoría) no terminal | `status='finished'`, `closed_by_refactor_at=now()` | FR-025, US6 |
| Promoción `qty_price_presentation` (spec 040) no terminal | `status='finished'`, `closed_by_refactor_at=now()` | FR-025, US6-CA3 |
| Promoción `fixed` no terminal | `status='finished'`, `closed_by_refactor_at=now()`; **no** se convierte en `percent` ni `package_price` | FR-025, FR-026, US6-CA4 |
| Promoción ya `finished` de cualquier tipo | sin cambio (`closed_by_refactor_at` queda `NULL`) | — |
| Entidad `Presentation`, `product_variants.presentation_id`, módulo `/dashboard/presentations`, integración en `api/v1/catalog/` (`_resolve_presentation_id`, `VariantCreate/Update/Response.presentation_id`), y en `pos-heladeria` `product.service.ts` / `diner.service.ts` | eliminadas (Incremento F, `063b` + T061c/T061d) | FR-027, A-63 |
| `Sale` / `SaleInvoice` ya emitida (cualquier tipo de descuento) | `discount`, `total`, factura **sin cambio**; `applied_promotions` nace `'[]'` | FR-021, US6-CA5, Principio VII |

**Aviso "recrea a mano"** (FR-025): las promociones con `closed_by_refactor_at IS NOT NULL` se
listan vía `GET /promotions?closed_by_refactor=true`; el frontend muestra un banner descartable.

**No se migra automáticamente ninguna `combo`/`fixed`/`qty_price`/`qty_price_presentation`**: sus
`value` no equivalen a un porcentaje ni a un precio de paquete de "N unidades cualesquiera del
conjunto"; traducirlas cambiaría el importe cobrado en silencio (clarification 2026-08-31).

---

## 2. Inventario de tests `CONGELA` afectados

`spec.md` §"Tests CONGELA afectados" listó 10 y pidió completar aquí "el inventario concreto" de
los que congelan `fixed` / `qty_price` y de los de spec 040. Este es el inventario completo. Cada
reescritura **cita esta spec y la decisión A-58…A-65 en el mismo commit** (Principio III, FR-028).

### 2.1 Backend — tests `"CONGELA comportamiento actual:"` que se reescriben

| Test | Archivo | Motivo | Re-congela |
|---|---|---|---|
| `test_add_item_combo` | `test_cart_service.py` | Se elimina la selección de combo (FR-024). | — (se elimina el test) |
| `test_serialize_cart_combo_no_recibe_descuento_adicional` | `test_cart_service.py` | Ídem. | — (se elimina) |
| `test_serialize_cart_discounted_total_sin_promocion` | `test_cart_service.py` | Montaje con `make_promotion_target`. | Invariante `discounted_total = None` sin promo, con conjunto de variantes. |
| `test_serialize_cart_discounted_total_con_promocion_activa` | `test_cart_service.py` | Usa alcance por **categoría** (`make_promotion_target(category_id=...)`). | `percent` sobre conjunto de variantes → `discounted_line_total` y `discounted_total`. |
| `test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito` | `test_cart_service.py` | Spec 038 (no CONGELA): el snapshot debe cuadrar con `evaluate_variant_sets`. | Snapshot de descuento por `OrderItem` = `by_line` del motor nuevo. |
| `test_add_item_to_table_combo_expande_componentes_a_precio_normal` | `test_orders_consolidation.py` | Se elimina la expansión de combo (FR-024). | — (se elimina) |
| `test_promo_lines_for_camino_feliz_y_sin_promocion_aplicable` | `test_orders_checkout.py` | `promo_lines_for` deja de traer `product_id`/`category_id`; agrega `description`. | Forma de `promo_lines` en el modelo nuevo. |
| `test_pay_order_construye_sale_real_con_promocion_activa` | `test_orders_checkout.py` | El motor cambia a cálculo por conjunto; `Sale` ahora registra `applied_promotions` (FR-021). | `pay_order` con `percent`/`package_price` sobre conjunto; `Sale.discount` + `applied_promotions`. |
| `test_pay_order_dos_combos_distintos_a29_promotion_id_none` | `test_orders_checkout.py` | Se elimina combo; A-29 se resuelve vía `applied_promotions` (FR-021, A-64). | Dos promociones distintas en una venta → `promotion_id` NULL **pero** `applied_promotions` con las dos. |
| `test_close_session_unified_a29_promotion_id_no_registra_combos_multiples` | `test_table_sessions_service.py` | Ídem A-29 sin combos. | Cierre unificado con dos promociones → `applied_promotions` en `Sale` y `CustomerOrder`. |

### 2.2 Backend — tests `"CONGELA comportamiento corregido:"` afectados

| Test | Archivo | Motivo | Re-congela |
|---|---|---|---|
| `test_group_bill_aplica_promocion_percent_vigente_sin_terminales_historia_2_escenario_1` | `test_orders_tables_advanced.py` | Montaje con target de categoría → conjunto de variantes. | `group_bill` aplica `percent` sobre conjunto en mesa fusionada. |
| `test_group_bill_aplica_combo_vigente_sin_terminales_fr_002` | `test_orders_tables_advanced.py` | Se elimina combo (FR-024). | — (se elimina) |
| `test_group_bill_a01_camino_c_excluye_pagadas_y_aplica_promocion_vigente` | `test_orders_tables_advanced.py` | Usa promo `percent` con target. | Mismo comportamiento A-01 camino C con conjunto de variantes. |
| `test_a08_fuera_de_ventana_en_hora_local_no_descuenta` | `test_menu_router.py` | Montaje con `make_promotion_target`. | A-08 (vigencia en hora local) con conjunto de variantes — la corrección de zona horaria **se conserva**. |
| `test_a08_dentro_de_ventana_en_hora_local_si_descuenta` | `test_menu_router.py` | Ídem. | Ídem. |
| `TestListPromotionsA09::test_expone_x_server_time_en_utc` | `test_promotions_router.py` | La respuesta pierde `overlaps`/`priority`; el enum se reduce. | `X-Server-Time` en UTC intacto; forma nueva de `PromotionResponse`. |
| `test_el_header_no_cambia_la_forma_de_la_respuesta` | `test_promotions_router.py` | Ídem. | Ídem. |

### 2.3 Backend — clase `TestMenuPromotionsAnnouncementUS5` (spec 040, `test_menu_router.py`)

No lleva prefijo CONGELA. Se **reescribe**: el anuncio ahora describe el conjunto de variantes
(`variant_set_condition_text`), no una presentación. Casos: vigente en el instante → aparece;
fuera de ventana → no aparece (SC-007); zona horaria del tenant.

### 2.4 Backend — tests de la spec 040 que se **eliminan** (sin prefijo CONGELA)

`test_promotions_presentation_pricing.py`, `test_promotions_presentation_rules.py`,
`test_presentations_service.py`, `presentation_fixtures.py`. Los casos nuevos de spec 040 dentro
de `test_orders_checkout.py` (`test_una_linea_recibe_una_sola_promocion_la_de_menor_total`,
`test_recalculo_del_pool_x_al_producto_y_a_precio_normal`,
`test_fr023_nunca_deja_la_linea_peor_que_sin_promocion`) se **sustituyen** por los del motor por
conjunto (FR-007/FR-008/FR-009 ya no hay "pool" ni reconciliación).

### 2.5 Backend — script de CI `app/scripts/test_promotions_rules.py`: reescritura completa

**Único script de promociones en CI** (no lleva prefijo CONGELA pero es gate de deploy). Hoy
ejercita, todo eliminado por el refactor:
- §3 / §3b `_line_discount` de `qty_price` + `_matching_target` (targets) → **se van**.
- §4 `priority` decide el conflicto → **se va** (A-58).
- §6 `_line_discount` de `fixed` → **se va** (A-62).

Lo que **entra** (funciones puras del modelo nuevo, sin sesión):
- consumo codicioso descendente (`_greedy_units`): el grupo toma las unidades más caras (FR-008);
- grupos completos + remanente a precio normal (FR-007);
- `package_price`: descuento = `Σ normal − value`, topado en 0 (FR-006, FR-009);
- `percent`: descuento = `round(value% × Σ normal)` a peso (FR-006);
- reparto `_distribute_group_discount` con **división no exacta** → residuo a la variante de id
  más alto; `Σ descuentos por línea == descuento del grupo` al peso, en cualquier orden (FR-008a,
  SC-005);
- un grupo nunca encarece (FR-009);
- vigencia en hora local + ventana que cruza medianoche + **atribución de día al cruzar
  medianoche (A-57 se conserva)**;
- `type` admite exactamente `{percent, package_price}`.

### 2.6 Fixtures

| Fixture | Cambio |
|---|---|
| `cart_fixtures.py` / `orders_fixtures.py` / `table_sessions_fixtures.py` :: `make_promotion` | pierde `priority`; `type` default `percent`. |
| `make_promotion_target` | **se elimina** (targets fuera). |
| `make_combo_item` | **se elimina** (combo fuera). |
| `add_variant_to_promotion(db, promo, variant)` | **nuevo** — inserta en `promotion_variants`. |
| `presentation_fixtures.py` | **se elimina** (spec 040). |

### 2.7 Frontend — specs afectadas

`promotions-page.component.spec.ts`, `promotion-pricing.util.spec.ts`,
`promotion.service.spec.ts`, `scope-picker.component.spec.ts` (se **elimina** con el componente),
`presentations-page.component.spec.ts` (se **elimina** con el módulo). Se reescriben con los dos
tipos, el selector de conjunto y los diálogos de FR-014 / FR-016 / FR-018. Si tienen specs que
tocan `presentation_id` de la variante: `product.service.spec.ts` / `product-form.component.spec.ts`
(pierden el campo, T053) y cualquier spec de `diner.service` que arme el anuncio con
`presentation_name` (forma nueva `{text, variant_count, …}`, T059).

---

## 3. Tests **nuevos** por historia de usuario

| Historia | Archivo (nuevo) | Cobertura |
|---|---|---|
| US1 | `test_promotions_rules_admin.py` | conjunto vacío rechazado (CA3); `percent` value > 100 rechazado (CA4); resumen `variants[].unit_price` (CA1); el filtro por categoría guarda la lista concreta, una variante nueva no entra sola (CA2); nace en `Borrador` (CA5); FR-016 bloquea al guardar. |
| US2 | `test_promotions_service.py` | los 10 Acceptance Scenarios: 2 variantes distintas → $12.000 (CA1); 3 unidades → $20.000 con suelta determinista (CA2); otro orden → idéntico (CA3); `percent` min_qty 1 mixto → $20.700 (CA4); "15% llevando 3 medianos" → $33.500 con reparto (CA5); no alcanza min_qty → sin descuento (CA6); día no incluido (CA7); ventana horaria (CA8); variante desactivada no cuenta (CA9); condición visible + descuento efectivo al 2º ítem (CA10). Edge: remanente > grupo; conjunto con precios distintos → $18.000; descuento tope al normal. **SC-005**: división no exacta cuadra al peso en cualquier orden. |
| US3 | `test_promotions_rules_admin.py` | activar 2ª promo con variante compartida y ventanas que se cruzan → 409 nombrando conflicto (CA2); ventanas que no se cruzan (08–15 / 15–22) → permitido (CA3); ventana 22:00–02:00 aceptada (CA4); dimensión abierta (sin franja) se cruza con 15:00–17:00 → 409 (CA5); happy hour que cruza medianoche descuenta la madrugada del día de inicio (CA1, A-57). |
| US4 | `test_menu_router.py` (clase reescrita) + `test_cart_service.py` | anuncio visible con carrito vacío dentro de ventana (CA1); no visible fuera de ventana (CA2); carrito con 1 de 3 → condición pero precio normal; al 3º → precio de paquete (CA3). |
| US5 | `test_promotions_rules_admin.py` | editar nombre/fin/días/horas de una `Activa` (CA1); intentar cambiar valor/conjunto → 422 (CA2); reactivar `Finalizada` → 409 (CA3); duplicar → copia `Borrador` con mismo conjunto (CA4); cajero no puede crear/editar → 403 (CA5). |
| US6 | `test_promotions_migration.py` | `percent` de categoría → conjunto foto fija, estado y vigencia conservados (CA1); `combo` → `Finalizada` + aparece en el aviso, líneas históricas intactas (CA2); `qty_price_presentation` → `Finalizada` + aviso (CA3); `fixed` → `Finalizada`, **no** se convierte (CA4); `Sale` previa con descuento → `discount`/`total`/factura sin cambio (CA5). |

---

## 4. Verificación de no-regresión (Principio X)

Tras el refactor, con **cero** promociones `active`, todos los totales de cobro son idénticos a
la línea base (aditividad segura). Los `CONGELA` no afectados de `test_orders_checkout.py`,
`test_table_sessions_service.py`, `test_orders_tables_advanced.py`, `test_cart_service.py`,
`test_orders_consolidation.py` (los que no tocan promociones) pasan **sin editar**. El script de
CI reescrito pasa en verde. `quickstart.md` §"Verificación final" corre la suite completa.
