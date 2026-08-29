# Research: Panel derecho unificado — tipo de orden, mesas y pedido en la creación de orden manual

Todas las incógnitas de esta spec son de diseño técnico dentro del frontend `pos-heladeria`
(Angular, módulo `tables`, componente `manual-order-page.component.ts`); no hay dependencias
externas nuevas que investigar ni cambios de backend. La decisión de fondo (mover los controles al
panel derecho) ya la tomó el dueño/desarrollador vía `AskUserQuestion` antes de esta spec
(spec.md, Clarifications) — las decisiones de aquí son sobre cómo implementarla.

## D1 — Reestructurar el template: qué bloques se mueven y en qué orden

**Decision**: La barra superior (`manual-order-page.component.ts:34-96`) se reduce a un único
bloque con "← Volver a la Terminal" (línea 38-44 actual, sin el `<h2>Tipo de Orden</h2>` ni las
pestañas ni el listado de mesas). El bloque de las tres pestañas de tipo de orden
(líneas 47-70 actuales) y el bloque "Mesas" + listado (líneas 75-95 actuales) se trasladan, en ese
mismo orden, al inicio de la columna derecha (línea 155 actual, `<div class="w-full sm:w-[320px]
...">`), como dos nuevos bloques `shrink-0` antes del bloque "Nueva orden" (línea 156-161 actual)
que ya encabeza esa columna.

**Rationale**: es exactamente la Clarification 1 de spec.md ("todo al panel derecho") — un
`git mv` conceptual de dos bloques de template completos, sin tocar su lógica interna
(`store.selectTable`, `store.selectedTableId`, `t.statusLabel`, las pestañas deshabilitadas), solo
su ubicación. No hay ningún cambio de store: `selectTable`, `tablesView`, `selectedTableId` ya
están disponibles en el mismo componente.

**Alternatives considered**: ninguna — la decisión de ubicación ya se resolvió con el usuario antes
de esta spec; esta sección solo documenta el mapeo línea a línea para la implementación.

## D2 — Cómo acomodar el listado de mesas en un panel más angosto (Clarification 2)

**Decision**: Cambiar el contenedor del listado de mesas de `flex gap-2 overflow-x-auto pb-1`
(scroll horizontal, pensado para la barra de ancho completo) a `grid grid-cols-4 gap-2` (rejilla
de 4 columnas, sin scroll horizontal), y quitar `shrink-0 w-20` de cada botón de mesa (línea 82
actual) — la celda de la rejilla ya define su ancho, no hace falta un ancho fijo por botón.

**Rationale**: con el ancho nuevo del panel (D3, 400px) y su padding (`p-4`, 16px por lado), el
área útil es ~368px; 4 columnas de ~86px cada una (con `gap-2`, 8px) reproducen casi exactamente el
`w-20` (80px) que cada botón de mesa ya tenía en el diseño original — un ancho ya validado
visualmente en 051, no uno nuevo. Con 8 mesas (el caso real de este tenant, según `tablesView()` en
capturas ya vistas) esto da 2 filas completas, todas visibles sin scroll horizontal, cumpliendo
SC-003. Si el tenant tuviera más mesas, la rejilla simplemente agrega filas (crece verticalmente,
dentro del `overflow-y-auto` que ya tiene el resto del panel derecho más abajo — ver D4).

**Alternatives considered**:
- *Mantener el scroll horizontal dentro del panel angosto*: rechazado — un carrusel horizontal
  dentro de una columna de 400px de ancho solo alcanzaría a mostrar 3-4 mesas a la vez, empeorando
  la visibilidad respecto al diseño original (spec.md, Edge Cases).
- *Grid de 3 columnas en vez de 4*: rechazado — con 8 mesas produciría 3 filas en vez de 2 sin
  ninguna ventaja de legibilidad, y cada celda quedaría más ancha de lo necesario (~120px) para un
  botón que solo muestra "M{n}" y un estado corto.

## D3 — Nuevo ancho del panel derecho (FR-007, SC-002)

**Decision**: `w-full sm:w-[320px]` (línea 155 actual) pasa a `w-full sm:w-[400px]`.

**Rationale**: 400px es el mínimo que evita que las tres pestañas de tipo de orden
("🍽️ En Mesa", "🛍️ Para Llevar", "🛵 Domicilio") necesiten envolver a una segunda línea dentro
del padding del panel (`p-4`, área útil ~368px) — las tres pestañas miden en conjunto
~300-320px con sus `gap-1.5`, cabiendo en una sola fila. Es "un poco más de ancho" (spec.md,
Assumptions: incremento moderado, no la mitad de la pantalla) y no compite en exceso con el ancho
del catálogo, que sigue siendo `flex-1` (todo el ancho restante).

**Alternatives considered**:
- *320px (sin cambio)*: rechazado — ya se sentía angosto antes de este cambio (input directo del
  usuario) y ahora debe alojar dos secciones adicionales completas.
- *480px o más*: rechazado por alcance — spec.md, Assumptions pide un incremento moderado; un salto
  mayor reduciría de forma notoria el espacio del catálogo sin que ninguna sección lo necesite para
  no recortarse (FR-007 ya se satisface en 400px, ver cálculo de D2/D3 arriba).

## D4 — Scroll del panel derecho tras agregar dos secciones nuevas

**Decision**: Los dos bloques nuevos ("Tipo de Orden", "Mesas") se agregan como `shrink-0` (igual
que el encabezado "Nueva orden" que ya tenía esa clase), **antes** del bloque de carrito que ya es
`flex-1 overflow-y-auto` (línea 163 actual). El resumen de totales + "Confirmar y Enviar"
(línea 188 actual) sigue siendo el último bloque `shrink-0`, fijo al fondo del panel. No se agrega
ningún `overflow-y-auto` nuevo al contenedor raíz de la columna derecha.

**Rationale**: es el mismo patrón de layout que el panel derecho ya usaba (encabezados fijos +
única zona central que crece/scrollea), extendido a dos bloques fijos más arriba en vez de uno —
evita el doble scroll anidado (columna completa + carrito) que se produciría si en cambio se le
agregara `overflow-y-auto` a todo el contenedor.

**Alternatives considered**: hacer que todo el panel derecho scrollee como una sola unidad
(quitando `flex-1 overflow-y-auto` del carrito) — rechazado, el resumen de totales y "Confirmar y
Enviar" dejarían de estar siempre visibles al fondo, que es el comportamiento que la Historia 1 de
spec 052 y toda la spec 036 original ya daban por hecho (siempre poder confirmar sin buscar el
botón).

## Resumen de impacto en tests existentes

`manual-order-page.component.spec.ts` (post-051, 13 casos) tiene varios casos que dependen del
texto/estructura del bloque que se mueve — verificado leyendo el archivo completo tras 051:
- Los casos que buscan botones por `textContent` ("Para Llevar", "Domicilio", "M2"/"M3",
  "Confirmar y Enviar", "Volver a la Terminal") siguen funcionando igual: `querySelectorAll('button')`
  no depende de en qué contenedor viven los botones, solo de su texto.
- El caso nuevo de 051 "el listado de mesas tiene un título propio 'Mesas', distinguible del
  encabezado 'Tipo de Orden'" (`querySelector('h2')`/`querySelectorAll('h3')`) sigue siendo válido
  tal cual: ambos encabezados solo cambian de contenedor padre, no de tag ni de texto.
- Ningún caso existente hace aserciones sobre la posición geométrica (offsetLeft/getBoundingClientRect)
  ni sobre la jerarquía exacta de contenedores padres — todos navegan por texto o por selector de
  tag, así que el traslado de bloques (D1) no rompe ninguno.

0 tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (mismo hallazgo que specs
045-051).
