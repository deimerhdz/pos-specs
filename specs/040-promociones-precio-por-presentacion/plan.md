# Implementation Plan: Promociones de Precio por Cantidad Configuradas por Presentación

**Branch**: `040-promociones-precio-por-presentacion` | **Date**: 2026-08-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/040-promociones-precio-por-presentacion/spec.md`

## Summary

Hoy una promoción de "precio por cantidad" (`Promotion.type == "qty_price"`,
`app/models/promotion.py:30`) apunta a un **producto** o a una **categoría** vía `PromotionTarget`
(`product_id` XOR `category_id`, `promotion.py:122-175`), y el descuento se calcula **línea por
línea** en `promotions/service.py::_best_line_match` (`service.py:238-276`). Eso falla en los dos
frentes que describe `spec.md`: un mismo producto con presentaciones de precio distinto no se
distingue, y dos sabores distintos en la misma presentación no combinan para un paquete.

Esta spec introduce una **modalidad nueva** —no un reemplazo (FR-016)— con tres piezas:

1. **Entidad de catálogo `Presentation`** (`presentations`, schema `tenant`), a la que las
   `ProductVariant` de distintos productos pueden apuntar mediante una columna nueva
   `product_variants.presentation_id` (nullable). Hoy no existe ningún concepto compartido entre
   productos (`spec.md` §"Naturaleza de esta spec"); modelarlo es evolución del modelo de datos
   (Principio VIII) y por eso vive en esta fase, no en la spec.
   ([data-model.md](./data-model.md), [research.md](./research.md) D1/D2).

2. **Tipo de promoción `qty_price_presentation`** con sus reglas en una tabla hija propia
   `promotion_presentation_rules` (una fila = `(presentación, cantidad_mínima, precio_paquete)`,
   FR-001), a semejanza de cómo `promotion_combo_items` es la tabla hija de los combos —
   [research.md](./research.md) D3. El motor automático línea-por-línea (`AUTO_TYPES`,
   `service.py:47`) **excluye** el tipo nuevo, igual que ya excluye `combo`.

3. **Motor de evaluación por presentación**: una función nueva
   `presentation_package_discount_for_lines(db, lines, now)` —hermana de
   `combo_discount_for_lines` (`service.py:399-448`)— que agrupa las unidades elegibles por
   `presentation_id` sin importar de qué variante/producto vengan, arma paquetes completos, calcula
   un **precio unitario normal único** por presentación (el menor vigente entre las variantes
   elegibles que aportan unidades, FR-011) y devuelve un **desglose por línea** para que el reparto
   cuadre al peso (FR-011, SC-005). La coexistencia con la promoción heredada a nivel de producto
   trata esta modalidad como candidata más del mecanismo de "mejor promoción por línea" del motor
   (spec 012 FR-003/FR-004), con un **recálculo del pool de líneas del paquete hasta punto fijo**
   cuando una línea se adjudica al motor línea-por-línea — "menor total por línea" (FR-013, FR-023),
   sin reescribir `_best_line_match` — [research.md](./research.md) D5/D6.

Sobre esto se añaden: la validación de solape entre reglas (dentro de una promoción y contra otras
promociones activas del **mismo tipo**, FR-006), la verificación de uniformidad de precio con
confirmación explícita (FR-017/FR-018) y el aviso de "no es descuento real" con el mismo patrón
(FR-022), el bloqueo de baja de una presentación referenciada por una regla activa (FR-020), la
entrada automática de variantes nuevas por referencia a la presentación (FR-007/FR-019, sin editar
la promoción — sale gratis del diseño, no requiere código), y el anuncio en el menú QR público
mientras la promoción esté **vigente en ese momento** (FR-021).

**Sin cambio de comportamiento sobre lo existente, con una excepción registrada**: ningún test
`"CONGELA comportamiento actual:"` cambia de cuerpo (research.md D14), las promociones `qty_price` a
nivel de producto siguen intactas (FR-016). La modalidad de descuento en sí no requiere entrada en
`registro-de-anomalias.md` (es una modalidad que se suma). **La única excepción** es FR-004: corrige
`_valid_now` (atribución de día al cruzar medianoche) para todos los tipos de promoción, cambio de
comportamiento observable registrado como **A-55** (research.md D18), no retroactivo. Última entrada
previa: A-54, spec 039.

## Technical Context

**Language/Version**: Backend Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`).
Frontend Angular (`../pos-heladeria`, "pos-frontend" para el negocio). Esta spec toca **ambos
repositorios** (Constitución §Alcance).

**Primary Dependencies**: FastAPI + SQLAlchemy 2.0 (sync, `Mapped`/`mapped_column`, sentencias Core
`select`), Alembic (`@for_each_tenant_schema`, head actual `187e491e597a`), Pydantic v2. Angular +
signals, `promotion-pricing.util.ts` (port del motor Python). **Ninguna dependencia nueva**
(Principio IX no aplica): todo lo que se necesita —una tabla nueva, una columna FK, un tipo de
promoción más, una función de agrupación análoga a `combo_discount_for_lines`— se construye con lo
que el proyecto ya usa.

**Storage**: PostgreSQL 16, **schema por tenant** (`{"schema": "tenant"}`). Entidades nuevas /
modificadas, todas en `tenant`:
- `presentations` — **tabla nueva**.
- `promotion_presentation_rules` — **tabla nueva** (hija de `promotions`).
- `product_variants` — **columna nueva** `presentation_id` UUID nullable, FK → `presentations.id`
  `ondelete="SET NULL"`.
- `promotions` — `CheckConstraint` de `type` ampliado con `qty_price_presentation` (sin columnas
  nuevas).
- `sales` / `order_items` / `cart_items` — **sin cambio**: su `promotion_id` (FK `ondelete="SET
  NULL"`) sirve igual para la promoción nueva; el descuento nunca se persiste (FR-014, igual que hoy).

**Testing**: `unittest` vía `python -m unittest` (sin pytest, sin `conftest.py`). Characterization
tests sobre SQLite en memoria. Script de CI `app/scripts/test_promotions_rules.py` (el único de los
12 scripts que corre en CI). Se agregan tests nuevos por historia de usuario (research.md D14,
[quickstart.md](./quickstart.md)); ningún test `CONGELA` se modifica. Frontend: `*.spec.ts` con el
runner ya configurado del repo.

**Target Platform**: Linux server (`../pos-backend` en producción) + SPA Angular
(`../pos-heladeria` en producción).

**Project Type**: Web application (API FastAPI + frontend Angular), dos repos independientes.

**Performance Goals**: Sin objetivo nuevo. `presentation_package_discount_for_lines` corre una vez
por cálculo de cobro/preview, sobre las líneas de un pedido (decenas, no miles), con el mismo perfil
que `combo_discount_for_lines` ya tiene hoy. El menú QR gana una consulta acotada por el índice
`ix_promotions_status_ends_at` ya existente (research.md D12).

**Constraints**:
- **FR-014 (no persistir descuento)**: el desglose se calcula en cada cobro, recalculando desde el
  estado del pedido — igual que el resto del motor. Ninguna tabla nueva guarda montos calculados.
- **FR-011 (reparto determinista, cuadra al peso)**: el residuo del redondeo y las unidades
  sobrantes se asignan por identificador de variante más alto (desempate: identificador de línea
  más alto), sin depender del orden de las líneas — SC-005 lo verifica.
- **FR-013/FR-023 (excluyente por línea, nunca empeora)**: una línea recibe el descuento de una
  sola promoción; entre la de producto y la de presentación gana la de menor total; nunca se aplica
  si deja la línea peor que sin promoción.
- **FR-007/FR-019 (alcance por referencia, no por nombre)**: el conjunto de variantes de una regla
  se resuelve SIEMPRE por `product_variants.presentation_id == regla.presentation_id`, nunca
  comparando `ProductVariant.name`.
- **Principio VII (datos históricos)**: ninguna venta/factura ya emitida se recalcula; la promoción
  nueva solo actúa en cobros futuros.
- **Compatibilidad hacia atrás**: `product_variants.presentation_id` nace `NULL` para todo lo
  existente (FR-008: una variante sin presentación no entra en ninguna regla). Sin backfill.

**Scale/Scope**: 2 tablas nuevas + 1 columna nueva + 1 `CHECK` ampliado (1 migración
`@for_each_tenant_schema`). 1 tipo de promoción nuevo. ~1 función de evaluación nueva + su
integración en los **~10 puntos** que hoy llaman `promotions.evaluate` /
`combo_discount_for_lines` / `best_line_discount` (research.md D7, [contracts/cobro-y-preview.md](
./contracts/cobro-y-preview.md)). CRUD nuevo de presentaciones (backend + frontend). Ampliación del
formulario de promociones y del formulario de producto/variante. Endpoint hermano nuevo
`GET /menu/promotions` + clave aditiva `promotions` en el dict del flujo QR + su render (`_build_menu`
sin tocar). Corrección puntual de `_valid_now` (FR-004, A-55). Cero migraciones de datos, cero
cambios **incompatibles** a endpoints existentes (la clave `promotions` es aditiva; flags de
confirmación opcionales, research.md D9).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe y está aprobado: 5 historias priorizadas, 23 FR, 6 SC, 8 clarifications resueltas, checklist `requirements.md` 100%. Este plan es posterior. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | La modalidad de descuento se **suma**; las promociones `qty_price` de producto siguen intactas (FR-016). **Una excepción registrada**: FR-004 corrige `_valid_now` (atribución de día al cruzar medianoche) para **todos** los tipos de promoción — cambio de comportamiento observable en promociones existentes, registrado como **A-55** en `registro-de-anomalias.md` (research.md D18), no retroactivo (Principio VII). El resto de `spec.md` §"Naturaleza de esta spec" se mantiene; research.md D14 verifica qué tests `CONGELA` y de CI quedan intactos. | PASS (con A-55) |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún test `"CONGELA comportamiento actual:"` se edita en su cuerpo. `_build_menu` **NO se toca** (es entrada del `CONGELA` de `test_menu_router.py` y de `cart_fixtures.py:379`): el anuncio del menú se añade con `_build_menu_promotions` + `GET /menu/promotions` (research.md D12). `app/scripts/test_promotions_rules.py` (CI): `_in_time_window`/`_line_discount`/`best_line_discount` sin cambio; **`_valid_now` cambia de cuerpo** por FR-004 (A-55, research.md D18) — los `check()` actuales del script (ventanas normales) siguen verdes y se agrega uno para la combinación con cruce de medianoche. research.md D14 lo detalla test por test. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El descuento por presentación es comportamiento nuevo definido por `spec.md` (FR-001–FR-013, FR-021–FR-023). El criterio de éxito es conformidad con esos FR/SC + ausencia de regresión en los ~10 caminos de cobro, no equivalencia con el pasado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | No se migra el resto de `evaluate` → `evaluate_detailed` (deuda pendiente de spec 012, fuera de alcance). No se unifican los ~10 call-sites en un helper "de una vez" más allá de lo mínimo para no duplicar la nueva llamada (research.md D7 elige el punto de enganche más conservador). No se toca `promotion-pricing.util.ts` más allá de reconocer el tipo nuevo como no-previsualizable (research.md D13, mismo trato que `qty_price` ya recibe). | PASS |
| **VI. Evolución Incremental** | Entrega en 5 incrementos verificables por separado (research.md D17): (A) entidad `Presentation` + FK en variante + CRUD + baja bloqueada; (B) tipo `qty_price_presentation` + reglas + validaciones de forma/solape/uniformidad; (C) motor de evaluación + integración en los caminos de cobro; (D) anuncio en menú QR; (E) superficies de preview de staff/comensal. Cada incremento no mezcla migración + refactor + cambio de comportamiento en una sola unidad. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`SaleInvoice`/`Payment` se recalcula ni se altera. El descuento nunca se persiste (FR-014). La migración solo agrega estructura vacía; `presentation_id` nace `NULL` para todo el catálogo existente, sin backfill (data-model.md). | PASS |
| **VIII. Evolución del Modelo de Datos** | [data-model.md](./data-model.md) especifica entidades nuevas, columnas, FKs, `ondelete`, valores por defecto, `CHECK`s, unicidad, compatibilidad con datos existentes, estrategia de migración `@for_each_tenant_schema` (con guarda `_has_table`) y de **rollback** (`downgrade` que elimina tabla/columna sin pérdida de dato histórico, porque nada histórico se escribió). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | Ninguna dependencia nueva (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de `spec.md` tiene "Independent Test"; [quickstart.md](./quickstart.md) los traduce a `unittest` ejecutables + verificación manual del menú QR y del panel. Se ejercita el reparto por línea (SC-005), el determinismo por orden (CA-5), el solape (SC-002), la entrada automática (SC-004) y la ventana de anuncio (SC-006). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las decisiones de negocio están en `spec.md` (bloquear al crear/activar ante solape, bloquear baja de presentación, excluyente por línea con "menor total", precio de referencia único = menor vigente, reparto determinista). Las decisiones técnicas —tabla hija propia vs. reutilizar `PromotionTarget`, tipo nuevo vs. extender `qty_price`, función hermana de combos, punto de enganche en los call-sites— están en [research.md](./research.md), cada una con su alternativa descartada. | PASS |
| **XII. Trazabilidad** | Cadena: `spec.md` (Necesidad + contrato) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (decisión técnica) → `tasks.md` (Fase 2, `/speckit-tasks`, no en este comando) → implementación → tests nuevos por historia + verificación de no-regresión sobre los `CONGELA` y el script de CI → `quickstart.md` (verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos, los nombres de tests nuevos, los mensajes de error de negocio (uniformidad, solape, baja bloqueada, "no es descuento real") y los mensajes de commit se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones que justificar en Complexity Tracking. Un punto de atención documentado, no una
violación: el "riesgo residual" que `spec.md` §Assumptions señala —que "mejor promoción por línea"
de spec 012 se evalúa hoy línea por línea y este descuento nace de agrupar varias líneas— se
resuelve en research.md D6 definiendo cómo la modalidad entra como candidata en ese mecanismo, sin
reescribir el mecanismo.

## Project Structure

### Documentation (this feature)

```text
specs/040-promociones-precio-por-presentacion/
├── plan.md               # Este fichero (/speckit-plan)
├── research.md            # Fase 0 — 17 decisiones técnicas y alternativas descartadas
├── data-model.md          # Fase 1 — tablas nuevas, columna FK, CHECKs, migración y rollback
├── quickstart.md          # Fase 1 — validación ejecutable por historia de usuario
├── contracts/             # Fase 1
│   ├── presentaciones-api.md                 # CRUD nuevo de presentaciones + FK en variante
│   ├── promociones-precio-por-presentacion.md # tipo, reglas, validaciones, flags de confirmación
│   ├── cobro-y-preview.md                    # efecto sobre los ~10 caminos de cobro/preview
│   └── menu-qr-anuncio.md                    # endpoint hermano GET /menu/promotions (sin tocar _build_menu)
├── checklists/
│   └── requirements.md    # Ya existente, 100%
└── tasks.md               # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Rutas relativas a la raíz de cada repo. La spec vive en `pos-specs`; el código en `../pos-backend`
y `../pos-heladeria` (Constitución §Alcance).

```text
# ../pos-backend
app/
├── models/
│   ├── presentation.py                # NUEVO — Presentation (id, name único, active, timestamps),
│   │                                     schema "tenant".
│   ├── promotion.py                   # MODIFICADO — PROMOTION_TYPES += "qty_price_presentation";
│   │                                     CheckConstraint ck_promotion_type ampliado; clase nueva
│   │                                     PromotionPresentationRule (promotion_id FK CASCADE,
│   │                                     presentation_id FK, min_qty >= 1, pack_price Numeric(12,2)
│   │                                     >= 0, UNIQUE(promotion_id, presentation_id)); relación
│   │                                     Promotion.presentation_rules (cascade all, delete-orphan).
│   └── product_variant.py             # MODIFICADO — columna presentation_id UUID nullable,
│                                         FK → presentations.id ondelete="SET NULL", index.
│
├── api/v1/presentations/              # NUEVO paquete — CRUD de presentaciones
│   ├── router.py                      #   GET (lista) / POST / PATCH / DELETE. DELETE y PATCH
│   │                                     active=false: 409 si alguna promoción ACTIVA la referencia
│   │                                     (FR-020), con la lista de promociones en el detalle.
│   ├── schemas.py
│   └── service.py                     #   incluye applicable_variants(db, presentation_id) →
│                                         variantes activas que la referencian (para el panel
│                                         "Productos Aplicables" y el resumen, FR-005).
│
├── api/v1/promotions/
│   ├── schemas.py                     # MODIFICADO — PromotionType += QTY_PRICE_PRESENTATION;
│   │                                     PresentationRuleIn (presentation_id, min_qty>=1,
│   │                                     pack_price>=0); PromotionCreate/PromotionShapeUpdate
│   │                                     aceptan presentation_rules; flags opcionales
│   │                                     confirm_precio_no_uniforme / confirm_sin_descuento
│   │                                     (research.md D9). PromotionResponse expone
│   │                                     presentation_rules + applicable_count por regla.
│   ├── service.py                     # MODIFICADO — AUTO_TYPES SIN CAMBIO (el tipo nuevo NO entra
│   │                                     al motor línea-por-línea, igual que combo). Funciones
│   │                                     nuevas:
│   │                                       · presentation_package_discount_for_lines(db, lines, now)
│   │                                         → desglose por línea (hermana de
│   │                                         combo_discount_for_lines, service.py:399).
│   │                                       · _presentation_reference_unit_price(...) (FR-011).
│   │                                       · validación de solape de reglas (FR-006), invocada en
│   │                                         create/update_shape y en change_status→active.
│   │                                       · validación de uniformidad (FR-017) y de "no es
│   │                                         descuento real" (FR-022) con confirmación.
│   │                                     _valid_now: MODIFICADO (FR-004, A-55, research.md D18) —
│   │                                     atribución de día cuando la ventana cruza medianoche;
│   │                                     afecta a TODOS los tipos de promoción.
│   │                                     find_overlaps / active_discount_promotions: ajuste acotado
│   │                                     para reconocer el tipo nuevo donde aplique (research.md
│   │                                     D8/D12).
│   └── router.py                      # MODIFICADO — 409 legibles para solape y confirmaciones;
│                                         sin endpoints nuevos.
│
├── api/v1/orders/
│   ├── checkout.py                    # MODIFICADO — promo_lines_for(...) agrega presentation_id por
│   │                                     línea; los 4 call-sites (pay_order 275, checkout_and_send
│   │                                     466, approve_payment_attempt 849, confirm_cash 969) suman
│   │                                     presentation_package_discount_for_lines al descuento total
│   │                                     y reconcilian con evaluate por línea (research.md D6/D7).
│   │                                     compute_bill (136) idem.
│   └── tables_advanced.py             # MODIFICADO — group_bill (155) idem.
│
├── api/v1/table_sessions/service.py   # MODIFICADO — compute_bill (186), release_paid_session (656)
│                                         y _close_split (753): suman el descuento por presentación.
│                                         Unidad de agrupación = las líneas evaluadas juntas en ese
│                                         cobro (research.md D16: en split, por comensal).
│
├── api/v1/sales/service.py            # MODIFICADO — venta de mostrador (279) idem.
│
├── api/v1/cart/service.py             # MODIFICADO — serialize_cart (269) / submit_cart snapshot
│                                         (~635): el carrito del comensal refleja el descuento por
│                                         presentación (preview), sin persistirlo.
│
├── api/v1/menu/
│   ├── schemas.py                     # MODIFICADO — clase nueva MenuPromotionAnnouncement
│   │                                     (condición legible por regla). MenuCategoryResponse y el
│   │                                     response_model de public_menu SIN CAMBIO.
│   └── router.py                      # MODIFICADO — función nueva _build_menu_promotions(db, now)
│                                         (promociones de presentación VIGENTES EN ESE INSTANTE:
│                                         _valid_now, hora local del tenant, FR-021). _build_menu
│                                         NO se toca (CONGELA de test_menu_router.py,
│                                         cart_fixtures.py:379). Endpoint hermano nuevo
│                                         GET /menu/promotions + clave "promotions" en el dict del
│                                         flujo QR con token (research.md D12).
│
├── alembic/versions/
│   └── XXXX_presentations_y_reglas_por_presentacion.py   # NUEVO — down_revision 187e491e597a.
│                                     @for_each_tenant_schema con guarda _has_table: crea
│                                     presentations, promotion_presentation_rules, agrega
│                                     product_variants.presentation_id, amplía ck_promotion_type.
│                                     downgrade simétrico.
│
└── characterization_tests/           # tests NUEVOS por historia (research.md D14); ninguno CONGELA
    ├── test_presentations_service.py          # NUEVO — CRUD + FR-020 (baja bloqueada)
    ├── test_promotions_presentation_rules.py  # NUEVO — FR-006 solape, FR-017 uniformidad, FR-022
    ├── test_promotions_presentation_pricing.py# NUEVO — FR-009/FR-010/FR-011 cálculo + SC-005
    │                                             reparto + CA-5 determinismo por orden
    ├── test_menu_router.py                    # MODIFICADO — caso nuevo: anuncio dentro/fuera de
    │                                             ventana (FR-021, SC-006); los CONGELA de este
    │                                             fichero NO se tocan
    └── test_orders_checkout.py                # MODIFICADO — caso nuevo: coexistencia con qty_price
                                                  de producto (FR-013/FR-023); CONGELA intactos

# ../pos-heladeria
src/app/
├── modules/presentations/             # NUEVO módulo — CRUD de presentaciones (lista + form)
│   ├── pages/presentations-page.component.ts
│   ├── services/presentation.service.ts
│   └── interfaces/presentation.interface.ts
│
├── modules/products/pages/product-form.component.ts   # MODIFICADO — selector de presentación
│                                     (opcional) por variante/tamaño; la variante "Single" y las
│                                     que no eligen presentación quedan fuera de toda regla (FR-008).
│
├── modules/promotions/
│   ├── interfaces/promotion.interface.ts   # MODIFICADO — PromotionType += 'qty_price_presentation';
│   │                                          PresentationRule; el tipo nuevo NO se agrega a
│   │                                          AUTO_TYPES (paridad con el backend).
│   ├── pages/promotions-page.component.ts  # MODIFICADO — formulario DEDICADO de
│   │                                          `qty_price_presentation` (template `#presentationForm`):
│   │                                          reproduce el bento del mockup de spec §Assumptions —
│   │                                          tarjetas "Información General" + "Configuración de
│   │                                          Reglas", paneles "Productos Aplicables" + "Resumen de
│   │                                          la Regla" (FR-005). Aplica al crear
│   │                                          (`@case ('presentation')`) y al editar borrador
│   │                                          (`@case ('edit')` con isDraft); las promos ya activas
│   │                                          en solo lectura. Diálogos de confirmación FR-017/FR-022;
│   │                                          diálogo del 409 de solape (FR-006).
│   │                                          Ajuste posterior (T047): la tarjeta "Paquete"
│   │                                          (`qty_price`) sale del selector de creación; las
│   │                                          `qty_price` existentes se siguen editando (tipo en
│   │                                          `editableTypes`, `@case ('pack')` intacto).
│   └── services/promotion-pricing.util.ts  # MODIFICADO (mínimo) — getPromoDisplay reconoce el tipo
│                                              nuevo; queda FUERA de la previsualización de precio
│                                              del cliente, igual que 'qty_price' ya lo está
│                                              (research.md D13).
│
├── modules/tables/pages/public-menu.component.ts   # MODIFICADO — render del banner de anuncio de
│                                     promociones de presentación (FR-021).
│
└── core/services/menu.service.ts     # MODIFICADO — tipado del campo promotions nuevo.
```

**Structure Decision**: la modalidad se apoya en dos entidades nuevas (`presentations`,
`promotion_presentation_rules`) y un tipo de promoción nuevo, sin tocar la semántica de
`PromotionTarget` ni de `qty_price` (research.md D1/D3). El cálculo vive detrás de una única
función nueva (`presentation_package_discount_for_lines`) invocada en los mismos ~10 puntos donde
`combo_discount_for_lines` ya se invoca hoy, con el mismo patrón "descuento aparte que se suma"
(research.md D7). El frontend gana un módulo `presentations` y amplía dos formularios existentes; el
`promotion-pricing.util.ts` se toca lo mínimo porque el tipo nuevo —como `qty_price`— no se
previsualiza en el cliente. No se crea ningún "motor de promociones extraído" (eso sería otra spec,
Principio V).

**Ajuste de UI posterior (2026-08-27, decisión del propietario — T047):** el formulario de
`qty_price_presentation` se reimplementó siguiendo el **mockup adjunto en `spec.md` §Assumptions**
(las dos tarjetas + los dos paneles laterales), reproduciendo su **estructura y composición** con el
Tailwind existente de `pos-heladeria` (índigo/gris, iconos SVG inline). **No se adopta Material 3**
(ni Google Fonts ni Material Symbols): el `primary` del mockup (`#4648d4`) ya coincide con el
`indigo-600` de la app y el resto del panel mantiene su estilo. Además, la forma de promoción
**`qty_price`** (producto/categoría) sale del **selector de creación** — sigue soportada en el
backend y editable para las promociones que ya existen.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
