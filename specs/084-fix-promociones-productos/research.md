# Research: Corrección de Bugs en Promociones y Productos

**Input**: [spec.md](./spec.md), verificación directa del código en `../pos-backend` y
`../pos-heladeria` (ambos en `develop`, 2026-09-17).

## D0 — Verificación de código real (punto de partida)

No hay ningún `NEEDS CLARIFICATION` de tecnología: ambos repos ya existen, con stack fijo
(FastAPI/SQLAlchemy/Alembic + Angular 21). El trabajo de esta fase fue **verificar el código real**
antes de diseñar, no elegir tecnología. Hallazgos clave que cambian el diseño respecto a una
lectura ingenua de `spec.md`:

- **Bug 1 no requiere tocar el backend.** `menu_variant_promotion`
  (`app/api/v1/promotions/service.py:357-411`) ya calcula `display_text`/`unit_equivalent` para
  **cualquier** `min_qty` (no solo 1) y ya viaja en `MenuVariantResponse.promotion`
  (`app/api/v1/menu/schemas.py:41`, servido desde `app/api/v1/menu/router.py:184`). La función que
  el frontend usa hoy para decidir qué mostrar en la tarjeta (`menu_unit_discount`,
  `service.py:318-354`) es la que **solo** cubre `min_qty == 1` — es la causa raíz exacta del bug,
  y ya hay un dato mejor (`menu_variant_promotion`) sin consumir. Ver D6.
- **Bug 3 ya tiene una implementación parcial, con semántica distinta a la pedida.**
  `addRuleRow()` (`promotions-page.component.ts:1656-1706`) ya combina los productos candidatos del
  Paso 1 que comparten una presentación — pero genera **una sola regla compartida** (un
  `PromotionRuleForm` cuyo `variantIds` es la unión de variantes de N productos), no N reglas
  independientes editables por separado. `spec.md` FR-017/FR-019 exige lo segundo. Esto es un
  **reemplazo de comportamiento ya en producción** (spec 083, mergeada), no una función nueva desde
  cero — de ahí la anomalía A-77 (D7). Ver D4.
- **El botón "Configurar" no tiene ninguna guarda hoy.** `promotions-page.component.ts:317-323` no
  tiene `[disabled]`; `openEdit(p)` (línea 1528) tampoco valida `p.status`. El patrón "estado real,
  no badge" que `spec.md` FR-008 exige **ya existe** en `canDelete(p)` (línea 1458-1460:
  `return p.status !== 'active'`) — se replica el mismo criterio, no se inventa uno nuevo.
- **El backend ya expone `status` real** en `PromotionResponse` (`schemas.py:226`) — bug 4 no
  necesita ningún cambio de contrato de API para el `[disabled]` del listado.
- **`PATCH /promotions/{id}/shape` ya bloquea `active`/`finished`** a nivel de servicio
  (`update_shape`, `service.py:781`: `if promo.status not in ("draft", "paused")`) — esa guarda
  cubre la **forma** (reglas), no necesariamente un endpoint separado de nombre/vigencia si existe
  uno. Marcado como verificación pendiente en tasks (no bloquea el diseño, ver Constraints de
  `plan.md`).
- **No existe hoy ninguna guarda de exclusividad por producto completo.** Solo
  `_guard_variant_overlap` (`service.py:579-632`), que compara **la misma variante exacta** entre
  promociones cuya vigencia se cruza (`draft/active/paused`). La exclusividad de producto de FR-021
  es código enteramente nuevo, con un criterio de estado más angosto (solo `active`).
- **El menú de acciones ⋮ no usa CDK Overlay ni `mat-menu`.** Es un `<div class="absolute ...">`
  dentro de una celda con `overflow-x: auto` en el contenedor de la tabla — la causa técnica exacta
  del recorte: fijar `overflow-x: auto` sin fijar `overflow-y` fuerza a `overflow-y: auto` también
  (regla del spec CSS de `overflow`), recortando cualquier descendiente absoluto que se extienda
  verticalmente fuera del contenedor. `@angular/cdk` (`^21.2.14`) ya está instalado, sin ningún uso
  de `Overlay`/`Menu` en el repo — no hay patrón previo que replicar ni romper.
- **El patrón de menú ⋮ con recorte solo existe en `promotions-page.component.ts`.**
  `products-page.component.ts:192-230` usa botones de icono inline, sin dropdown — confirma que el
  alcance de bug 5 (FR-026–028) no necesita tocar el listado de productos hoy (el Assumption del
  spec ya lo dejaba condicionado a "si existieran otros listados con el mismo patrón").

## D1 — Modelo de la asociación variante↔presentación: columna vs. tabla puente

> **Vigente**, con un ajuste (D10): la columna `presentation_id` sigue siendo la decisión correcta,
> pero pasa a `NOT NULL`.

**Decisión**: columna nueva `product_variants.presentation_id` (FK nullable a `presentations.id`),
relación 1:1 por variante.

**Rationale**: una variante representa como máximo una presentación (spec.md FR-006: dos variantes
del mismo producto no pueden compartir presentación) — es exactamente la cardinalidad de una FK
simple, igual que `product_variants.product_id`. No hay atributos propios de la asociación (fecha,
quién la creó, etc.) que justifiquen una tabla intermedia.

**Alternatives considered**: tabla puente `variant_presentations` (espejo de
`category_presentations`) — descartada por sobre-modelar una relación 1:1; habría exigido un JOIN
adicional en cada lectura de variante (recibos, menú, promociones) para un beneficio que una FK
simple ya cubre.

## D2 — Sincronización de nombre: columna copiada + cascada vs. nombre derivado (JOIN)

> **Superada por D10 (2026-09-20, A-79)**: el propietario pidió eliminar la columna `name`, que es
> exactamente la alternativa que esta decisión descartó. Se conserva el texto como registro de lo
> que se implementó primero (migración `a9f7d0310f6b`).

**Decisión**: `product_variants.name` sigue siendo una columna propia, almacenada. Al asociar o
reasociar una presentación (`FR-002`/`FR-003`), el backend copia `Presentation.name` dentro de la
misma transacción. Al renombrar una `Presentation` (`PATCH /presentations/{id}`, `FR-004`), el
backend ejecuta `UPDATE product_variants SET name = :nuevo_nombre WHERE presentation_id = :id` en
la misma transacción que el `UPDATE` de `Presentation`.

**Rationale**: `product_variants.name` ya es `NOT NULL` con `UniqueConstraint(product_id, name)` y
lo leen directamente recibos, el motor de promociones, el menú QR y exportes — derivarlo vía JOIN
obligaría a tocar todos esos puntos de lectura para un beneficio marginal (evitar una cascada de
`UPDATE`, que en la práctica son unas pocas filas por tenant). La cascada es la operación
equivalente a la que ya hace, por ejemplo, cualquier catálogo con nombre desnormalizado en este
código base (mismo patrón que otros catálogos del panel).

**Alternatives considered**: nombre derivado en tiempo de lectura (sin columna `name` propia,
`COALESCE(presentation.name, variant.own_name)` vía JOIN) — descartado por el alto radio de
impacto en código que hoy asume `variant.name` como columna simple, para un caso de uso (cambiar el
nombre de una presentación ya usada) que en la práctica es infrecuente.

## D3 — Exclusividad de producto entre promociones: guarda nueva vs. extender `_guard_variant_overlap`

**Decisión**: función nueva y separada (`_guard_product_overlap`, nombre de trabajo) junto a
`_guard_variant_overlap` en `app/api/v1/promotions/service.py`, invocada desde los mismos puntos
(`create()`, `update_shape()`).

**Rationale**: el criterio de estado es distinto — `_guard_variant_overlap` compara contra
`draft/active/paused` con cruce de vigencia (fechas/días/horas); la exclusividad de producto
(`spec.md` FR-021, confirmada en Clarifications) compara **solo** contra `status=active`, sin
evaluar vigencia horaria. Son dos guardas con semántica distinta que deben poder evolucionar por
separado; fusionar ambas en una sola función mezclaría dos criterios de estado diferentes y haría
más difícil verificar cada uno de forma aislada (Principio VI).

**Alternatives considered**: extender `_guard_variant_overlap` para que, además de la variante
exacta, también compare por `product_id` — descartada porque cambiaría el comportamiento de una
guarda ya protegida (spec 063 FR-014, con su propio criterio de vigencia) sin que ningún FR de esta
spec pida modificarla.

## D4 — `addRuleRow()`: refactor a N reglas independientes vs. agregar checkboxes sobre la regla combinada

**Decisión**: refactorizar `addRuleRow()`/`resolvedVariantIdsForLabel` para que, cuando más de un
producto candidato comparte la presentación elegida, genere **una `PromotionRuleForm` por
producto** (cada una con la única variante de ese producto), en vez de una regla combinada con
`variantIds` de varios productos. La lista de checkboxes (FR-016) decide **para cuáles** productos
se genera esa fila antes de confirmar.

**Rationale**: `spec.md` FR-017 y FR-019 exigen que cada fila resultante sea **editable o
eliminable de forma individual sin afectar las demás** — imposible si varias variantes de distintos
productos viven dentro de la misma `PromotionRuleForm`/`PromotionRule`. Agregar solo checkboxes
sobre el comportamiento actual no resolvería esto: seguiría existiendo una única regla combinada
para todos los productos marcados.

**Alternatives considered**: mantener la regla combinada y agregar checkboxes solo para decidir qué
variantes entran en su `variantIds` — descartada porque no cumple FR-019 (edición/eliminación
individual por producto) ni refleja lo que el propietario del producto confirmó en Clarifications
("cada fila de regla independiente con su propia lista explícita de variantes").

## D5 — Menú de acciones (bug 5): Angular CDK Overlay vs. fix CSS manual

**Decisión**: reemplazar el `<div>` absoluto por Angular CDK Overlay (`OverlayModule`/`CdkMenu` de
`@angular/cdk`), ya instalado en `pos-heladeria` sin uso previo.

**Rationale**: CDK Overlay renderiza el menú en un contenedor global (`cdk-overlay-container`),
fuera del árbol DOM de la tabla — no queda sujeto al `overflow` del contenedor con scroll (causa
raíz confirmada en D0) y ya resuelve el reposicionamiento al hacer scroll/resize (`FR-027`/`FR-028`)
sin código manual de cálculo de posición. Es el patrón estándar de Angular para este problema
exacto, y no hay ningún uso previo de CDK en el repo que este cambio pueda romper.

**Alternatives considered**: fix puramente CSS (`overflow-y: visible` explícito en el contenedor de
la tabla, manteniendo `overflow-x: auto`, más un `position: fixed` calculado a mano con
`getBoundingClientRect()` en scroll/resize) — descartado porque replicaría a mano exactamente lo
que CDK Overlay ya resuelve, con más superficie de bugs futuros (recalcular posición en cada evento
de scroll/resize manualmente) para un ahorro de dependencia que no aplica (`@angular/cdk` ya está
instalado).

## D6 — Precio mínimo de la tarjeta del menú QR: cálculo en frontend vs. campo agregado en backend

**Decisión**: calcular el precio mínimo (`FR-013`) en frontend, iterando `product.variants[].promotion`
(ya presente en `MenuProductResponse`) y tomando el `unit_equivalent` más bajo; sin agregar ningún
campo nuevo a `MenuProductResponse` ni tocar `app/api/v1/menu/`.

**Rationale**: el dato por variante (`display_text`, `unit_equivalent`) ya viaja completo en cada
respuesta del menú (confirmado en D0); agregar un campo agregado en el backend sería redundante —
el frontend ya recibe todo lo necesario para reducirlo a un mínimo. Mantiene el alcance de bug 1
100% frontend, coherente con `spec.md` ("no reabre el motor de cálculo").

**Alternatives considered**: campo `MenuProductResponse.min_promo_price` calculado en el backend —
descartado por redundante y porque hubiera exigido tocar `app/api/v1/menu/router.py`/`schemas.py`
sin ninguna necesidad de cálculo que el frontend no pueda hacer con el dato que ya recibe.

## D7 — Anomalías pendientes de registrar antes de implementar (Principio II)

Próximo número disponible en `registro-de-anomalias.md`: **A-76** (el último registrado es A-75,
spec 083).

- **A-76** — Bloqueo total de "Configurar" en promociones `active`: reemplaza, para ese estado, la
  edición parcial (nombre/fin de vigencia/días/horas editables) que spec 083 FR-018 dejó vigente.
  Cita `spec.md` §Clarifications (pregunta 2, bug 4) como decisión de negocio.
- **A-77** — Reemplazo de la regla combinada de `addRuleRow()` (comportamiento en producción desde
  el merge de spec 083, PR #83) por N reglas independientes, una por producto. Cita `spec.md`
  §Clarifications (pregunta 1 y pregunta de `/speckit-clarify` sobre checkboxes, bug 3).
- **A-78** — Exclusividad de producto entre promociones `active`: restricción nueva que puede
  rechazar, hacia adelante, selecciones de producto que hoy son válidas (un producto en dos
  promociones activas simultáneas, mientras cubran variantes distintas). Cita `spec.md`
  §Clarifications (preguntas sobre exclusividad de producto, bug 3). No retroactiva (FR-025).

Estas tres entradas deben existir en `registro-de-anomalias.md` **antes** de mergear los commits
que implementan cada una — mismo tratamiento que A-74/A-75 de spec 083.

## D8 — Alcance de bug 5 confirmado por código

Confirmado en D0: el patrón de menú ⋮ con recorte solo existe hoy en
`promotions-page.component.ts`. `products-page.component.ts` usa botones de icono inline sin
dropdown — no hay nada que corregir ahí. El Assumption de `spec.md` ("si existieran otros
listados con el mismo patrón, se corrigen al encontrarlos") queda sin efecto práctico por ahora:
el alcance real de FR-026–028 es un solo archivo.

## D9 — Endpoint de nombre/vigencia de una promoción `active` (confirmado tras `/speckit-analyze`)

**Confirmado**: sí existe un endpoint separado — `PATCH /promotions/{promotion_id}`
(`app/api/v1/promotions/router.py:81`, `update_promotion`), que llama a `service.update()`
(`app/api/v1/promotions/service.py:762-778`). Esa función edita `name`, `description`, `ends_at`,
`days_of_week`, `start_time`, `end_time` — **sin ninguna guarda de `status` hoy** (a diferencia de
`update_shape`, que sí bloquea `active`/`finished`). Esto confirma que el bug 4 reportado es real y
exacto: hoy se puede editar el nombre/vigencia de una promoción `active` sin restricción.

**Tests de characterization que protegen este comportamiento hoy** (`app/characterization_tests/
test_promotions_rules_admin.py`, clase `TestUS5DuplicarEditarEstados` y
`TestUS5MantenimientoPorLote`):

- `test_ca1_editar_escalares_de_una_activa` (línea 280): crea una promoción, la activa
  (`change_status(..., "active")`), y llama `service.update()` con `name`/`ends_at`/
  `days_of_week`/`start_time`/`end_time` nuevos — **asertando que se aplican con éxito**.
- `test_editar_vigencia_de_promocion_multi_regla_afecta_a_todas_con_una_accion` (línea 363): mismo
  patrón, con una promoción `active` de 6 reglas — asertando que `service.update(..., ends_at=...)`
  tiene éxito y afecta a las 6 reglas de una sola vez.

**Implicación (Principio III)**: implementar FR-008/FR-011 (bloquear `service.update()` cuando
`promo.status == "active"`) rompe **directamente** estos dos tests tal como están escritos hoy. No
es opcional actualizarlos — son characterization tests protegidos, y el Principio III exige que su
actualización sea explícita, cite la decisión de negocio (**A-76**) y evidencie que no afecta otros
comportamientos protegidos del mismo archivo (en particular, `test_ca2_cambiar_reglas_de_una_activa_bloquea`,
`test_ca4_duplicar_copia_borrador_con_las_mismas_reglas` y los de `TestUS5MantenimientoPorLote`
sobre pausar/reactivar deben seguir en verde sin tocarse — no dependen de `service.update()` en
estado `active`).

**Guarda correcta** (no la de `update_shape`): FR-009 exige que "Configurar" — y por extensión
`service.update()` — siga editable en `Borrador`, `Pausada` **y `Finalizada`**, no solo en
`draft`/`paused`. La condición de `update_shape` (`status not in ("draft", "paused")`) es más
amplia de lo que pide esta spec — bloquear con ella también dejaría `Finalizada` sin poder
editarse, violando FR-009. La guarda nueva en `service.update()` debe ser específica:
`if promo.status == "active": raise HTTPException(409, ...)`.

---

## D10 — Eliminar `product_variants.name` (enmienda 2026-09-20, A-79)

**Decisión**: se elimina la columna `name` y `UNIQUE(product_id, name)`; `presentation_id` pasa a
`NOT NULL` con FK `ON DELETE RESTRICT`; el nombre que ve cualquier consumidor sale de un JOIN a
`presentations`. La API deja de aceptar y devolver `name` de variante (devuelve `presentation_id` +
`presentation_name`). Confirmado por el propietario (spec.md §Clarifications, sesión 2026-09-20).

**Rationale**: con US1 la variante ya no tenía un nombre "propio" sino una copia del de su
presentación, mantenida por una cascada y una guarda de colisión (FR-004) que existían solo para
sostener esa copia. Quitarla elimina un dato redundante, una cascada, un caso límite y un 409.
D2 descartó esta opción por el radio de impacto (~11 archivos backend, ~7 frontend leen el
nombre); ese costo se acepta ahora porque la decisión ya no es "sincronizar o no", sino "el nombre
no es un dato de la variante". El JOIN es barato: `presentations` es un catálogo pequeño y
`ProductVariant.presentation` se declara `lazy="joined"`.

**Inventario de impacto (2026-09-20)**: backend, 11 archivos con lecturas de `ProductVariant.name`
(tabla completa en contracts/variante-sin-nombre.md); frontend, `product-form.component.ts` (8),
`product.service.ts` (5), `promotions-page.component.ts` (4), `dining-cart.service.ts` (5),
`menu.service.ts`, `menu-lookup.ts`, `product-select.component.ts`, `review-step`/`cart` (reciben
`variantName` ya resuelto del carrito). Los `v.name` de `cash-report`/`cash-dashboard` **no** son de
variante (son de método de pago); no se tocan. No hay columnas de `order_items`/`cart_items` que
guarden el nombre (verificado en los modelos), y `sale_items.description` ya es texto inmutable
(`models/sale.py:132`), así que el historial de ventas no cambia.

**Riesgo principal — contrato no retrocompatible con clientes en caché**: el menú QR es una PWA
(`ngsw-config.json`); un cliente con el bundle anterior en caché leería `v.name` de una respuesta
que ya no lo trae y mostraría variantes sin nombre hasta que el service worker se actualice.
**Mitigación — despliegue en tres pasos**, cada uno desplegable por separado:

1. **Paso A, backend aditivo (sin migración)**: las respuestas ganan `presentation_id` y
   `presentation_name` **manteniendo** `name`; `VariantSaveIn.name`/`VariantCreate.name` pasan a
   opcionales; `presentation_id` nulo **y** `name` ausente ⇒ "Presentación única" (D11). Con `name`
   presente rige el comportamiento actual, para que el frontend anterior siga funcionando.
2. **Paso B, frontend**: lee `presentation_name`, elimina la columna "Nombre", envía siempre
   `presentation_id` (nulo solo sin tamaños) y nunca `name`. Cuando esta versión lleva un ciclo
   de despliegue publicada, ningún cliente vivo depende de `name`.
3. **Paso C, backend destructivo**: migración (enlazar los nombres libres pendientes, `NOT NULL`,
   quitar columna y unicidad), retiro de `name` de los schemas y de la rama de compatibilidad del
   paso A. Como la migración repite el enlazado, absorbe cualquier variante de nombre libre que el
   frontend anterior haya creado durante los pasos A–B.

En desarrollo los tres pasos viven en la misma rama; la separación importa para el orden de
despliegue, no para el orden de escritura del código (tasks.md los marca).

**Alternatives considered**: (a) mantener `name` en la API como campo calculado — descartada por el
propietario (menos cambios, pero deja dos nombres para lo mismo en el contrato); (b) dejar la
columna y solo ocultarla en el formulario — descartada por el propietario ("eliminarlo del modelo de
la base de datos"); (c) `presentation_id` opcional con nombre calculado `COALESCE` — descartada
(D11): reintroduce la variante "sin nombre" que la decisión quiere eliminar.

## D11 — "Presentación única" como fila del catálogo y atajo `presentation_id: null`

**Decisión**: la variante de un producto sin tamaños se asocia a una fila normal del catálogo de
presentaciones llamada "Presentación única" (literal de A-74), creada por get-or-create al
necesitarla (por nombre exacto, sin filtrar `active`). No se agrega columna `is_system` ni se
siembra en la migración de tenants. En el payload de guardado, `presentation_id: null` significa
"usar la Presentación única".

**Rationale**: (1) el aprovisionamiento de tenants nuevos no garantiza que una migración de datos
los alcance; get-or-create funciona igual en tenants viejos y nuevos. (2) El frontend no necesita
conocer el id de esa presentación para guardar un producto sin tamaños, ni esperar a que
`PresentationService.allPresentations` cargue. (3) Sin marca de "sistema" no hay un caso especial
que mantener: si el administrador la renombra o desactiva, las variantes existentes conservan su
enlace y su nombre actual; solo un producto **nuevo** sin tamaños crearía otra fila con el literal.
Es un caso raro con consecuencia mínima (dos presentaciones de nombre distinto en el catálogo).

**Riesgo aceptado**: `null` como atajo hace que una fila de un producto **con** tamaños que llegue
sin presentación se guarde silenciosamente como "Presentación única". El frontend lo impide
(`canSave()`), y el backend no puede distinguir los dos casos sin un campo extra; a cambio, dos
filas nulas del mismo producto sí chocan por unicidad (409). Si el descuido se vuelve frecuente,
la salida es exigir `presentation_id` en el backend y que el frontend lo resuelva desde el
catálogo — cambio local, no de modelo.

**Alternatives considered**: `is_system BOOLEAN` en `presentations` + siembra en migración —
descartada por el problema (1); `presentation_id` obligatorio en el payload con el id resuelto en
el frontend — descartada por (2), queda como salida de respaldo; endpoint `GET /presentations/default`
— sobra con el atajo.

## D12 — Orden de la tarjeta "Tamaños del producto"

**Decisión**: el bloque "Maneja inventario" (interruptor + aviso) se mueve **después** de la tabla
de tamaños y de la lista de presentaciones desactivadas, y **antes** del detalle del tamaño activo.
Con tamaños apagados no hay tabla y queda justo bajo el encabezado, como hoy.

**Rationale**: el interruptor solo gobierna el bloque de insumos fijos y la parte de inventario de
"Sabores a elegir" (comentario del template, spec 027/064); ponerlo entre el encabezado y la tabla
lo presenta como el control de la tabla. Debajo de la tabla y encima del detalle queda adyacente a
lo que sí gobierna, y el recuadro "Activa «Maneja inventario» arriba…" sigue diciendo la verdad.

**Punto no dictado literalmente por el propietario**: pidió que la tabla aparezca "debajo de
Tamaños del producto"; que el interruptor quede entre la tabla y el detalle (y no, p. ej., al pie de
toda la tarjeta) es la lectura más coherente con el resto del formulario y se documenta como tal —
`contracts/formulario-tamanos-orden.md` la fija; si se prefiere otra posición, es un cambio de
orden de bloques sin efecto en datos.

**Alternatives considered**: interruptor al final de la tarjeta, debajo del detalle — descartada:
el recuadro gris del detalle dice "arriba" y quedaría falso, y el interruptor quedaría lejos de lo
que habilita.

## D7 (actualización) — Anomalía adicional

- **A-79** — registrada 2026-09-20: eliminar `product_variants.name`, presentación obligatoria y
  `name` fuera de la API (D10/D11). Existe en `registro-de-anomalias.md` antes de implementar. El
  reordenamiento de la tarjeta (D12) es de presentación pura, sin cambio de comportamiento
  observable de datos ni de reglas, y **no** requiere anomalía propia.

## D13 — Selector de presentación del Paso 2: catálogo vs. variantes de los productos (A-80)

**Decisión**: el selector lista las presentaciones activas del catálogo
(`PresentationService.allPresentations`, ya cargado en otras pantallas), y la regla se resuelve al
configurar: por cada producto seleccionado se busca su variante con esa `presentation_id` y se
guarda como fila independiente con una lista explícita de variantes (sin cambios en backend).

**Rationale**: tras A-79 toda variante tiene `presentation_id`, así que el emparejamiento por id es
exacto y la etiqueta "Presentación única" para productos de una variante (un residuo de cuando ese
nombre era un literal guardado) deja de tener sentido. Nada del backend depende de la etiqueta: el
motor y las guardas (`_guard_variant_overlap`, `_guard_product_overlap`) trabajan con variantes.
El menú público ya devuelve `presentation_id` por variante (A-79), por lo que no hay endpoint
nuevo: `menu.service.ts` solo lo propaga al modelo del cliente.

**Alternatives considered**: modo dinámico (la regla guarda `presentation_id` + productos y el
motor resuelve en cada venta) — descartado por el propietario: exige tabla/columnas nuevas, cambio
del motor de cálculo, del menú QR (precio mínimo, spec 084 US3) y de ambas guardas de
exclusividad, y reabre A-65. Selector solo con presentaciones usadas por algún producto —
descartado por el propietario en favor de mostrar todo el catálogo activo.

## D14 — Regla por presentación en la UI, expandida a reglas por producto al guardar (A-81)

**Decisión**: el formulario de configuración guarda `presentationRules` (presentación, unidades,
valor) y la selección de productos como estado propio; `form.rules` (lo que recibe el backend) se
deriva de ellos con `rebuildRules()`: una regla de backend por cada par (regla de presentación,
producto seleccionado con esa presentación), con solo esa variante. Al abrir una promoción
guardada se hace la operación inversa (`hydrateFromRules`), agrupando por presentación + valor +
unidades y recuperando los productos de las variantes.

**Rationale**: reutiliza el modelo y el motor de spec 063/084 sin tocarlos — cada regla de backend
sigue siendo un conjunto explícito de variantes (A-65) y sigue sin mezclar productos (A-77) — y da
al administrador la lista corta que pidió. Las guardas del backend (`_guard_variant_overlap`,
`_guard_package_is_discount`, `_guard_product_overlap`) siguen aplicando a cada regla expandida.
La comprobación local de precio de paquete usa el precio más barato entre los productos
seleccionados con esa presentación, que es conservadora respecto de la del backend por regla.

**Consecuencias asumidas**: (1) el backend no persiste la regla por presentación sin variantes, así
que una regla sin ningún producto seleccionado que la tenga no se guarda; (2) una regla antigua
con variantes de varios productos (anterior a A-77) se muestra como una regla de presentación y se
normaliza a una regla por producto en el siguiente guardado; (3) reglas guardadas de igual
presentación pero con valor o unidades distintos entre productos (datos antiguos) aparecen como
dos reglas de la misma presentación y el formulario las marca como conflicto (variante repetida)
hasta que se corrijan.

**Alternatives considered**: modelo dinámico (la regla guarda la presentación y el motor resuelve
las variantes en cada venta) — descartado antes (D13) y sigue descartado: no es necesario para el
flujo pedido, que se resuelve al configurar. Persistir `presentation_id` en `promotion_rules` para
conservar reglas sin productos — se descarta por ahora; sería un cambio de modelo aditivo si la
limitación (1) resulta molesta.

