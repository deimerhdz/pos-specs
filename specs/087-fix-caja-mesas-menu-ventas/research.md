# Research: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-09-28 (adenda US8–US11: 2026-09-29)

Este documento resuelve, para cada punto técnico no obvio de la especificación, la decisión
tomada, su justificación y las alternativas descartadas. La especificación (`spec.md`) ya
resolvió todas las ambigüedades de comportamiento/negocio en su sección "Clarifications"; lo
que queda aquí es exclusivamente la capa técnica: cómo se implementa cada requisito sobre el
código real de `pos-backend` y `pos-heladeria`.

## Decisión previa confirmada con el usuario (no estaba en la spec)

Al diseñar el reinicio de numeración por turno (FR-006) se descubrió que el backend ya
soporta múltiples cajas registradoras (`cash_registers`) con turnos abiertos en paralelo (un
turno abierto por caja, `idx_open_shift_per_register`), y que hoy los pedidos de mesa no
están vinculados a ninguna caja/turno. Se preguntó al usuario a qué turno se ata la
numeración: **se confirmó la opción "un solo turno abierto a la vez"** — en la práctica del
negocio solo hay una caja con turno abierto simultáneamente, y un pedido de mesa nuevo toma
el turno abierto más reciente del tenant (`cash_shifts` con `status = 'open'`, el de
`opened_at` más reciente si excepcionalmente hubiera más de uno). El caso de dos turnos
abiertos a la vez queda fuera de alcance de esta spec.

---

## D1 — PDF de cierre de turno sin sidebar/header, con nombre de archivo controlado (FR-002, FR-003)

**Decision**: Reutilizar el patrón ya existente en el propio código (`table-qr-sheet.component.ts:101`,
`pos-heladeria`) de `window.print()` + reglas `@media print` que ocultan todo excepto el
contenido a imprimir. Se agrega ese mismo bloque `@media print` a `cash-report.component.ts`
(módulo `cash-register`), ocultando `app-sidebar`, `app-header` y cualquier control de
navegación/botones, dejando visible únicamente el contenedor del resumen financiero. Para el
nombre de archivo, se fija `document.title` al valor `<tenant-slug>-<DD-MM-YYYY>` justo antes
de invocar `window.print()` y se restaura el título original después — la mayoría de
navegadores basados en Chromium usan `document.title` como nombre sugerido en el diálogo
"Guardar como PDF".

**Rationale**: No añade ninguna dependencia nueva (jsPDF/html2canvas/WeasyPrint), lo que
evita la carga de justificación del Principio IX de la constitución para un caso que el
propio proyecto ya resuelve con CSS. Reutiliza un patrón ya probado dentro del mismo
módulo (`tables`), manteniendo consistencia de codebase. No requiere tocar `pos-backend`,
que hoy no tiene ninguna librería de generación de PDF (`requirements.txt` confirmado sin
weasyprint/reportlab/wkhtmltopdf).

**Alternativas consideradas**:
- **Generar el PDF en el backend** (WeasyPrint/reportlab) — rechazada: requiere una
  dependencia nueva sin otro caso de uso en el proyecto que la justifique, y duplicaría la
  lógica de layout que ya existe en el componente Angular del resumen.
- **Generar el PDF en frontend con jsPDF + html2canvas**, capturando el DOM del resumen y
  disparando la descarga de un Blob con el nombre exacto — más robusta para garantizar el
  nombre de archivo (no depende de que el navegador respete `document.title`), pero
  inconsistente con el patrón ya establecido y añade una dependencia nueva sin caso de uso
  previo en el proyecto. **Riesgo a validar en `quickstart.md`**: si al probar en los
  navegadores objetivo de producción se confirma que el nombre sugerido no es fiable, esta
  alternativa debe reconsiderarse — se documenta aquí para no partir de cero si ocurre.

## D2 — Eliminación completa de Arqueo Parcial, incluyendo historial (FR-001)

**Decision**: Backend: eliminar el modelo `CashPartialCount` (`app/models/cash_partial_count.py`),
el endpoint `POST /cash/shifts/{shift_id}/partial-count` (`app/api/v1/cash/router.py`) y su
servicio; nueva migración Alembic (`@for_each_tenant_schema`, igual patrón que
`f5a6b7c8d9e0_availability_change_partial_count.py`) que hace `op.drop_table("cash_partial_counts", schema=schema)`
tras `op.drop_index(...)`. Frontend: eliminar el modal y estado de `cash-dashboard.component.ts`
(`openPartial`, `submitPartial`, `partialCounted`, `partialNote`, `partialDiff`, `diffClass`,
botón "Arqueo parcial"), el método `partialCount()` de `cash.service.ts` y la interfaz
`PartialCountResponse` de `cash.interface.ts`. **No tocar** `cash-arqueo-modal.component.ts`
(es el arqueo de *cierre de turno*, un flujo distinto que la spec no toca).

**Rationale**: confirmado que no existe ninguna FK entrante hacia `cash_partial_counts` (solo
tiene una FK saliente hacia `cash_shifts` con `ondelete=CASCADE`), por lo que el `DROP TABLE`
es seguro sin arrastrar otras tablas. Arqueo parcial es un registro operativo de caja, no una
factura emitida, por lo que su borrado físico no choca con el Principio VII (inmutabilidad de
facturas) de la constitución.

**Nota de gobernanza (Principio II)**: este es un cambio de comportamiento existente —
requiere una entrada nueva en `specs/000-reconocimiento/registro-de-anomalias.md` (formato
`A-NN — [DECISIÓN DE NEGOCIO — spec 087] ...`, con quién/cuándo/qué cambia/por qué/funcionalidades
afectadas) antes de darse por completada la implementación de esta historia.

## D3 — Numeración estable de pedidos de mesa, reiniciada por turno (FR-006)

**Decision**: Añadir dos columnas nuevas a `tenant.customer_orders`:
- `cash_shift_id UUID NULL` (FK a `cash_shifts.id`, `ondelete="SET NULL"`), poblada solo para
  pedidos con `order_type = 'DINE_IN'`, con el turno abierto vigente del tenant en el momento
  de creación (ver decisión confirmada arriba). `NULL` para `TAKEAWAY`/`DELIVERY` y para todo
  pedido histórico previo a esta migración.
- `table_order_number INTEGER NULL`, asignado **una sola vez**, en el momento de creación,
  como `COUNT(*) + 1` de los pedidos `DINE_IN` ya existentes con el mismo `cash_shift_id`
  (dentro de la misma transacción de creación, para evitar condiciones de carrera entre dos
  creaciones casi simultáneas). `NULL` para `TAKEAWAY`/`DELIVERY` y para pedidos históricos.

El frontend deja de calcular el número por posición de arreglo (`orderTabs()` en
`pos-terminal.store.ts:712-723`, hoy `'Pedido ${i + 1}'`) y pasa a leer `table_order_number`
directamente de la respuesta del backend.

**Rationale**: la causa raíz confirmada del bug es que el número mostrado hoy se calcula
100% en el frontend por índice de un arreglo (`tableOrders()`) sin orden garantizado — de ahí
que el pedido más reciente aparezca como #1 al recargarse los datos. Para que el número sea
"asignado una sola vez... y no cambie después" (FR-006) debe persistirse en el backend en el
momento de creación, igual que cualquier otro identificador estable del pedido.

**Alternativas consideradas**:
- **Solo ordenar `tableOrders()` por `created_at` ascendente en frontend, sin cambios de
  backend** — rechazada: no resuelve el reinicio por turno (el frontend no tiene hoy ninguna
  noción de "turno abierto" asociada a cada pedido) y sigue sin garantizar estabilidad si el
  backend cambia el orden de entrega entre recargas.

**Nota de gobernanza (Principio II + VIII)**: cambio de comportamiento (requiere entrada en
`registro-de-anomalias.md`) y cambio de modelo de datos — ver `data-model.md` para
compatibilidad/migración/rollback completos.

## D4 — "Agregar producto" sobre pedidos paralelos por mesa (FR-007, FR-008, Edge Cases)

**Decision**: El endpoint actual `POST /orders/tables/{table_id}/items`
(`app/api/v1/orders/router.py` → `consolidation.add_item_to_table`) asume una única orden
abierta por mesa — ya no es válido una vez que FR-007 permite pedidos paralelos. Se introduce
un endpoint que opera sobre un pedido específico por su identificador, p. ej.
`POST /orders/{order_id}/items`, y el frontend (`pos-order-panel.component.ts`, botón "+
Agregar producto", línea ~325) pasa a invocarlo con el `order_id` ya identificado en pantalla
— nunca con `table_id` solo. El endpoint basado en `table_id` se retira si no tiene otro
consumidor, o se mantiene como alias que exige que la mesa tenga como máximo un pedido
abierto (rechazando con error explícito si hay más de uno) — a decidir en el diseño detallado
de `contracts/`.

**Rationale**: el edge case de la spec es explícito: "Agregar producto" siempre opera sobre
el pedido específico ya identificado en pantalla, "sin ambigüedad de a cuál pedido
pertenece" — con pedidos paralelos habilitados, un endpoint que resuelve "la orden abierta de
la mesa" de forma implícita ya no puede garantizar eso.

**Nota de gobernanza**: cambio de contrato de API existente. Dentro de este proyecto el único
consumidor es `pos-heladeria` (mismo repo, mismo spec), por lo que no hay rotura de contrato
con terceros; aun así, al ser un cambio de comportamiento de un endpoint ya en producción,
también amerita entrada en `registro-de-anomalias.md`.

## D5 — Bloqueo de edición sobre pedidos pagados/cerrados (FR-009)

**Decision**: verificar durante la implementación si `add_item_to_table`/su sucesor ya valida
`status` antes de anexar ítems (los estados no terminales son `recibida`/`abierta`/`bloqueada`;
`pagada`/`cancelada` son terminales). Si falta, añadir la validación explícita devolviendo un
error 409/422 con mensaje claro, que el frontend traduce en un mensaje que guía a "Crear
pedido manual". Esto es una verificación puntual de implementación, no una decisión de diseño
nueva — no se detectó código existente que la contradiga durante la exploración.

## D6 — Precio Promocional + Σ Adicionales (FR-011) y su alcance frente a ventas históricas

**Decision**: antes de tocar código, reproducir el bug con el caso de la spec (Granizado de
Mora, promo $5.000, + Leche condensada $2.000 + Choco-chips $1.500 = $8.500) contra dos
puntos: (a) el cálculo autoritativo de backend, centralizado en `compute_line_price`
(`app/catalog_engine/core.py`) + `evaluate_variant_sets` (`app/api/v1/promotions/service.py`),
reusado literalmente por `cart/service.py`, `orders/checkout.py` (`compute_checkout_preview`,
`compute_draft_preview`, `pay_order`, `checkout_and_send`, `confirm_cash_payment_attempt`,
`approve_payment_attempt` — este último respalda "Pagos por confirmar"); y (b) la vista previa
de frontend en `product-select.component.ts` (`lineTotal()` línea 557, `packageTotal()` línea
607). La hipótesis de trabajo, a confirmar con un test de caracterización antes del fix, es
que el defecto vive en la lógica de "paquete"/promoción (backend o frontend) que reemplaza el
precio en vez de sumarle los adicionales — hay que aislar cuál de los dos lados falla, o si
fallan ambos de forma independiente.

**Nota de gobernanza — CRÍTICA (Principio VII de la constitución)**: si se confirma que el
defecto ya afectó `Sale.total` persistido de ventas ya finalizadas, el fix se aplica
**solamente a checkouts nuevos, a partir del despliegue** — ninguna venta ya finalizada
(factura emitida) se recalcula retroactivamente, sin excepción. El Acceptance Scenario 3 de
la User Story 1 ("una venta ya finalizada... el total registrado coincide con Precio
Promocional + Σ Precio Adicionales") debe interpretarse y probarse con una venta finalizada
**después** de aplicar el fix, nunca recalculando o reescribiendo el `total` de una venta
histórica preexistente. Esta lectura se deriva directamente del Principio VII y se documenta
aquí como restricción de alcance — **debe confirmarse explícitamente con el usuario/negocio
antes de implementar** si existiera intención distinta (p. ej. si el negocio quisiera de
verdad recalcular ventas históricas, eso requeriría una decisión de negocio separada y
explícita, fuera del alcance de esta spec tal como está redactada).

## D7 — Nombre de cliente obligatorio al crear pedidos (FR-005)

**Decision**: no se toca el esquema de `customer_name` (ya existe, nullable, en
`CustomerOrder.customer_name` y en `DiningOrder.customer_name` del frontend) — la
obligatoriedad es una validación de negocio en el momento de creación, no una restricción
`NOT NULL` de base de datos (los pedidos históricos deben seguir existiendo sin el campo).
Backend: rechazar la creación (`POST /orders` y cualquier otro punto de creación de pedidos
de mesa/para llevar/domicilio) si `customer_name` viene vacío/nulo. Frontend: marcar el input
como requerido con validación síncrona en los tres tabs de `manual-order-page.component.ts`
(Mesa línea ~369, Para llevar línea ~414, Domicilio línea ~447) antes de habilitar "Guardar".
**Verificar en implementación** si existe algún otro punto de creación de pedidos de mesa
además de `manual-order-page.component.ts` (p. ej. un flujo directo desde el propio Menú QR
del cliente) que también deba llevar esta validación.

## D8 — Presentación del producto en el snapshot de `Sale` (FR-010, aplicado a Ventas)

**Decision**: verificar en `app/api/v1/sales/builder.py` si `SaleItem.description` (snapshot
de texto libre, no columna estructurada) ya incluye la presentación; si no la incluye,
añadirla con el mismo formato que ya usa el frontend en pedidos (`variantLabel()` de
`menu-lookup.ts`, `"Producto · Presentación"`). No se propone una columna nueva de
presentación en `SaleItem`: es coherente con que el resto del historial de venta ya se
guarda como snapshot de texto.

## D9 — Campo Cambio/Devuelta en detalle de venta (FR-013)

**Decision**: `Sale.paid_amount` y `Sale.change_given` ya existen en el modelo (migración
`f5a6b7c8d9e0`) y ya están declarados (sin usar) en la interfaz TypeScript `Sale`
(`sales.interface.ts` líneas 166-168). Solo falta: (a) confirmar que `SaleResponse`
(backend) ya serializa ambos campos — si no, añadirlos al schema de respuesta; (b) agregar la
línea "Cambio: $X" al modal inline de detalle en `sales-page.component.ts` (líneas 140-194),
mostrada únicamente cuando la venta tuvo algún componente de pago en efectivo. Determinar
"tuvo componente en efectivo" iterando `Sale.payments` (ya soporta pago dividido, relación
1-a-muchos a `Payment`) buscando algún método de pago en efectivo con monto > 0, en vez de
inferirlo solo de `change_given != null` (una venta 100% no-efectivo con `change_given = 0`
no debe verse como "no tuvo cambio" por casualidad numérica sino explícitamente excluida por
su método de pago).

## D10 — Adicionales/notas "Nombre xN" siempre visibles (FR-012)

**Decision**: unificar el formato de renderizado de adicionales en los puntos de resumen ya
identificados — `product-select.component.ts` (armado), `pos-order-panel.component.ts`
(detalle de mesa), `order-detail.component.ts` línea 86 (historial admin) y
`public-menu.component.ts` línea 319 (tracking del cliente en Menú QR) — para que siempre
antepongan el multiplicador `xN` (incluyendo `x1`) y omitan el contenedor cuando no hay
adicionales/notas, sin introducir un componente compartido nuevo si el patrón actual ya es
suficientemente simple de replicar en los 4 puntos (evaluar en implementación si vale la pena
extraer un pipe/helper común dado que son 4 puntos, respetando Principio V: no refactorizar
más allá de lo que la funcionalidad exige).

---

# Adenda 2026-09-29 — Historias US8 a US11 (FR-014 a FR-017)

Las 4 correcciones reportadas tras la implementación de US1–US7 se agregaron a esta misma
spec (sesión de clarificación 2026-09-29). Las decisiones D11–D14 salen de leer el código real
de `pos-heladeria` (y `pos-backend` donde aplica); las causas raíz marcadas **[hipótesis]**
no se pudieron reproducir solo leyendo código y su primera tarea es un test que las confirme
antes de corregir.

## D11 — Nombre de presentación ilegible junto a la promoción en el modal del producto (FR-014, US8)

**Corrección 2026-09-29 (causa raíz real)**: el diagnóstico de abajo (truncado por CSS) era solo
una causa **secundaria** y no explicaba que el nombre no apareciera *en absoluto* ni siquiera con
ancho de sobra. La causa principal: desde spec 084 (A-79) `MenuVariantResponse` manda
`presentation_name` y **no** `name`; `diner.service.ts::mapCategory` (Menú QR del comensal) seguía
leyendo `v['name']` → `undefined`, así que la fila pintaba una etiqueta vacía y solo la promoción
(el texto "Llevando Grande x 2…" bajo la lista sí salía porque lo compone el backend).
`menu.service.ts` (Terminal) ya mapeaba `name: v.presentation_name`. Se corrige el mapper
(`name: v['presentation_name'] ?? v['name']`, más `presentation_id`) y se actualiza la respuesta de
ejemplo de `diner.service.spec.ts`, que aún usaba `name` y por eso no atrapó la regresión.
Lección: las respuestas de ejemplo de un spec de mapper deben copiar el contrato real del backend.

**Causa raíz secundaria** (`product-select.component.ts`, filas de "Elige tu presentación"): el nombre de
la presentación se pinta en `<span class="block … truncate">` dentro de un contenedor
`min-w-0`, mientras que la etiqueta de promoción (`inline-flex`, sin límite de ancho) y la
columna de precio (`shrink-0`) no se encogen. En un ancho angosto (Menú QR en celular) el
único elemento que cede espacio es el nombre, que queda reducido a "…" y solo se lee la
promoción — justo lo reportado. `promotion.display_text` (`"<condición corta> · <equivalente
por unidad>"`, ver `promotions/service.py:414`) es largo, lo que agrava el efecto.

**Decision**: el nombre de la presentación deja de truncarse (se ajusta en varias líneas con
`break-words`) y es siempre la etiqueta principal; la etiqueta de promoción pasa a ser
secundaria en su propia línea (`max-w-full`, con `whitespace-normal`) y se omite cuando
`v.promotion` es nulo (ya es así). Cambio solo de clases/plantilla de una fila; sin cambio de
datos ni de lógica de precios (`rowPackagePromo`, `discountFor`, `variantPrice` no se tocan).

**Alternatives considered**: (a) truncar la promoción en vez del nombre — descartada, la
spec exige que la promoción nunca desplace al nombre pero un texto de promoción cortado
tampoco es aceptable (Edge Case de nombre largo + promoción: "baja a otra línea"); (b) mover
la promoción a la columna derecha junto al precio — descartada, agranda la columna
`shrink-0` que ya es la que comprime el nombre.

## D12 — Adicionales que no aparecen en el panel del pedido de la terminal (FR-015, US9)

**Causa raíz [hipótesis]**: el panel ya renderiza `<app-cart-item-options [options]="it.options">`
para cada línea, así que el defecto está en cómo se **arma** `it.options`, en
`pos-terminal.store.ts` (`persistedItemsView` y `cartView`):

1. **Ítems guardados**: los nombres se resuelven con `lookup()` construido solo con
   `menuService.categories()` (el menú *vigente*: `menu/router.py` filtra
   `Product.active`/`available`). `OrderItemOption` guarda únicamente `option_id` + `quantity`;
   si el producto salió del menú, `optionLabel()` devuelve `''`. Antes de US6 el
   `.filter((o) => o.text)` descartaba la opción en silencio (el reporte); hoy
   `formatQuantifiedLabel('', 1)` da `" x1"`, que es truthy y se ve como un fragmento sin
   nombre. En ambos casos el adicional no se muestra correctamente.
2. **Ítems en borrador**: el texto se arma con `c.quantity > 1 ? "2x Nombre" : "Nombre"` en
   lugar de `formatQuantifiedLabel`, contradiciendo FR-012/FR-015 ("Nombre xN", incluso x1).

**Decision**: (a) el backend deja de depender del menú vigente para el *nombre*: la respuesta
de cada opción de un ítem de pedido (`OrderItemOptionResponse`) suma `name` y `group_name`
resueltos en lectura sin columna nueva ni migración: como los `selectinload` existentes solo
cargan `OrderItem.options` (no `Option` ni su grupo), se derivan con un `column_property` con
subconsulta escalar sobre `options`/`option_groups` (ver tasks T066 y `contracts/api-changes.md`
§7). (b) el frontend usa ese nombre como respaldo cuando `lookup()` no encuentra la opción, y
elimina el `.filter((o) => o.text)` que ocultaba adicionales; (c) los borradores usan
`formatQuantifiedLabel`, igual que los guardados. Primera tarea: test que arme un pedido
guardado con una opción ausente del menú y verifique que sigue apareciendo "Nombre xN".

**Alternatives considered**: (a) snapshot del nombre en `order_item_options` (columna nueva) —
descartada por costo de migración multi-tenant y porque las opciones referenciadas no se
pueden borrar (FK a `options.id`), así que el JOIN en lectura siempre resuelve; (b) cargar en
el frontend el menú completo incluyendo inactivos — descartada, no existe ese endpoint y
expondría productos ocultos al Menú QR.

## D13 — "TOTAL ORDEN" desactualizado al editar un pedido manual (FR-016, US10)

**Causa raíz** (`manual-order-page.component.ts` + `pos-terminal.store.ts`):
1. El `effect` que recalcula el desglose depende solo de `draftLines()`, `orderTypeTab()` y
   `deliveryFee()`; no de los ítems **guardados** del pedido seleccionado, que cambian al
   guardar/anular y al refrescar por sondeo/tiempo real.
2. `draftPreviewPayload()` envía únicamente los borradores. Con ítems ya guardados, el
   `draft-preview` del backend evalúa las promociones sobre un subconjunto, y al quedar el
   borrador vacío tras guardar, `loadDraftPreview()` pone `draftPreview = null` y el panel
   cae en `totals()` (subtotal local, `discount = 0`, sin promociones reevaluadas).
3. Las respuestas de `loadDraftPreview()` pueden llegar fuera de orden; no hay guarda de
   respuesta obsoleta, por lo que un preview viejo puede pisar al nuevo.

**Decision**: el desglose se calcula sobre el **conjunto vigente completo** (ítems guardados
no anulados + borradores): (a) `draftPreviewPayload()` incluye los ítems guardados de tipo
producto del pedido seleccionado junto con los borradores (el endpoint ya acepta la misma
forma `OrderItemIn`; **sin cambio de contrato**); (b) el `effect` pasa a depender también de
los ítems guardados del pedido seleccionado, de modo que se recalcula cada vez que cambia la
lista; (c) se añade un contador de secuencia para descartar respuestas obsoletas; (d) con cero
ítems vigentes el total muestra $0 (Edge Case), no el anterior. Los combos (líneas
`kind: 'combo'`) siguen la ruta actual del borrador (no se envían al `draft-preview`);
**verificar en implementación** que el total mostrado los sume, y si hoy no lo hace, cubrirlo
con el mismo subtotal local de combos que ya usa `comboDisplaySubtotal`, sin ampliar el
alcance a otro rediseño.

**Alternatives considered**: (a) llamar a `GET /orders/{id}/checkout-preview` tras guardar —
descartada, cubre solo el estado posterior al guardado pero no la edición previa (borradores
sobre ítems guardados) y añade una segunda fuente de verdad; (b) agregar `order_id` al body
de `draft-preview` para que el backend fusione — descartada, cambia el contrato y la
autorización del endpoint por algo que el frontend ya puede armar con datos que tiene.

## D14 — Nota por producto reforzada, un único estilo (FR-017, US11)

**Causa raíz / punto de cambio**: la nota se pinta con `text-[11px]` en `pill` cursiva
(`bg-[#fffbeb] text-[#b45309]`) en `cart-item-options.component.ts`, y el bloque está
**duplicado** en las dos ramas de la plantilla (`@if (grupos.length > 0)` y `@else if
(notes)`). Ese componente compartido lo usan `pos-order-panel`, `manual-order-page` y
`public-menu` (historial del cliente) — exactamente las tres pantallas de FR-017.

**Decision**: definir el estilo de la nota una sola vez dentro del componente (colapsando
las dos ramas duplicadas en una sola plantilla de nota) con: `text-base` (16px), `font-semibold`,
sin cursiva, fondo de alto contraste con texto oscuro (paleta ámbar ya presente, p. ej.
`bg-amber-100 text-amber-900` con borde), `whitespace-pre-wrap break-words` para notas largas y
`rounded-lg` en vez de `rounded-full` (una pill de 16px se deforma con varias líneas). Se
mantiene `@if (notes)` para no pintar contenedor vacío. `order.notes` (nota general) no se
toca. La colapsada de duplicados está **exigida** por FR-017 ("un único estilo"): mantener el
estilo en dos sitios sería la causa de que vuelva a divergir; no es un refactor oportunista
(Principio V). Contraste a validar ≥ 4.5:1 (WCAG AA).

**Alternatives considered**: (a) reforzar solo la copia de la rama `@else` — descartada,
la nota con opciones seguiría en 11px; (b) clase global de CSS — descartada, el proyecto
usa utilidades Tailwind inline y no hay hoja de estilos de componentes compartidos para esto.

## Testing

**Decision**: backend se verifica con `pytest` (confirmado por 168+ archivos `test_*.py` bajo
`app/`, incluyendo characterization tests en `app/characterization_tests/`); frontend se
verifica con el test runner nativo de Angular 21 (`@angular/build:unit-test`, invocado vía
`ng test` / `npm test`, basado en Vitest). No se introduce ningún framework de testing nuevo.
