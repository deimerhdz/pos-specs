# Research: Imagen de producto en el catálogo y organización del tipo de orden — creación de orden manual

Todas las incógnitas de esta spec son de diseño técnico dentro del frontend `pos-heladeria`
(Angular, módulo `tables`, componente `manual-order-page.component.ts`); no hay dependencias
externas nuevas que investigar ni cambios de backend. Cada decisión cita el código real leído para
tomarla.

## D1 — Cómo mostrar la imagen del producto en la tarjeta del catálogo (FR-001)

**Decision**: Agregar, dentro de cada tarjeta del catálogo
(`manual-order-page.component.ts:126-136`), un contenedor de imagen `@if (p.image_url) { <img> }
@else { <app-icon name="image-off" /> }`, reutilizando exactamente el mismo patrón visual que ya
existe en el catálogo del menú público por QR
(`public-menu.component.ts:359-372`: contenedor `aspect-square bg-indigo-50 flex items-center
justify-center overflow-hidden`, `<img class="w-full h-full object-cover">` cuando hay
`image_url`, ícono `image-off` de 40×40 en gris claro cuando no hay). El texto (nombre y precio,
hoy `manual-order-page.component.ts:134-135`) se reubica debajo del contenedor de imagen, dentro de
su propio padding, en vez del padding uniforme que hoy envuelve toda la tarjeta.

**Rationale**: `MenuProduct` (`product.interface.ts:369-384`, el tipo que devuelve
`store.catalogProductsFiltered()`, línea 819 de `pos-terminal.store.ts`) ya expone `image_url:
string | null` — el mismo campo que `product-select.component.ts:48-51` ya usa para el detalle. No
hace falta ningún cambio de datos ni de store: es un `@if`/`@else` nuevo sobre un dato que el
componente ya recibe. Reutilizar el patrón visual (contenedor + ícono `image-off`) ya validado en
`public-menu.component.ts` evita introducir un segundo criterio visual distinto para "producto sin
imagen" en la misma aplicación (spec.md, Assumptions).

**Alternatives considered**:
- *Mostrar un emoji o inicial del nombre como placeholder*: rechazado — introduciría un tercer
  criterio visual para "sin imagen" (el detalle de producto ya no reserva espacio cuando no hay
  imagen, `product-select.component.ts:47`; el menú público usa el ícono `image-off`); usar el
  criterio ya existente en la misma app es más consistente que inventar uno nuevo solo para esta
  tarjeta.
- *Pedir al backend un campo agregado o una miniatura distinta para el catálogo*: rechazado — el
  campo `image_url` ya es exactamente el que usa el detalle del mismo producto; no hay ninguna
  necesidad de negocio que justifique una imagen distinta para la tarjeta vs. el detalle.

## D2 — Reestructurar el layout de la tarjeta sin romper el badge de descuento

**Decision**: El badge de descuento (`manual-order-page.component.ts:131-133`,
`absolute top-2 right-2`) se mantiene posicionado sobre el contenedor de imagen (que pasa a ser
`relative`, en vez del `<button>` completo), ya que ese contenedor ocupa el ancho completo de la
tarjeta igual que hoy. El `<button>` exterior deja de tener padding uniforme (`p-3`) y pasa a tener
`overflow-hidden` + padding solo en el bloque de texto inferior, mismo criterio que
`public-menu.component.ts:357-374` (`bg-white rounded-2xl ... overflow-hidden` en el botón, `p-3`
solo en el bloque de texto).

**Rationale**: es la misma estructura de dos bloques (imagen a ancho completo + texto con su propio
padding) que ya existe y está probada en `public-menu.component.ts`; adoptarla evita inventar un
layout nuevo para un problema ya resuelto en el mismo repositorio.

**Alternatives considered**: mantener el padding uniforme del `<button>` y agregar la imagen dentro
de él con su propio margen — rechazado, produciría un borde blanco alrededor de la imagen
inconsistente con el resto del catálogo de la aplicación.

## D3 — Imagen en el detalle/opciones del producto (FR-003, Historia 2)

**Decision**: Ningún cambio de código. `product-select.component.ts:48-51` ya muestra
`product.image_url` cuando existe, sin reservar espacio cuando no existe (comentario en línea 47).
Esta parte de la spec se cubre por completo con cobertura de test que confirme el comportamiento ya
existente (Principio X), no con una implementación nueva.

**Rationale**: verificado leyendo el componente real — el modal que abre
`store.openConfig(p)` (`manual-order-page.component.ts:128`) hacia `store.configuringProduct()`
(`manual-order-page.component.ts:200-206`) es exactamente `ProductSelectComponent`, que ya recibe
el mismo objeto `MenuProduct` con `image_url` y ya lo renderiza.

**Alternatives considered**: N/A — no hay ninguna decisión de diseño que tomar sobre un
comportamiento que ya existe.

## D4 — Título propio sobre el listado de mesas (FR-004, FR-005)

**Decision**: Agregar un encabezado nuevo (p. ej. `<h3>Mesas</h3>`, con una jerarquía visual menor
—`text-xs font-semibold uppercase tracking-wide text-gray-400`— a la del `<h2>Tipo de Orden</h2>`
ya existente en línea 44) inmediatamente antes del contenedor del listado horizontal de mesas
(`manual-order-page.component.ts:74`, `<div class="flex gap-2 overflow-x-auto pb-1">`), dentro del
mismo bloque padre (`div.space-y-3`, línea 34) que ya envuelve tanto la fila de pestañas como el
listado de mesas. No se mueve la posición relativa de las pestañas de tipo de orden respecto al
listado de mesas (siguen: pestañas arriba, mesas abajo) — solo se le da al listado de mesas su
propio título, distinto en jerarquía tipográfica del `<h2>` que agrupa las pestañas.

**Rationale**: hoy el único encabezado del bloque es "Tipo de Orden" (línea 44), que antecede solo
a la fila de pestañas (líneas 46-69); el listado de mesas (líneas 74-93) no tiene ningún título
propio, por lo que visualmente se lee como si "perteneciera" a las pestañas. Un segundo encabezado,
con menor peso visual que el `<h2>` (para no competir con él ni sugerir que son el mismo nivel de
navegación), resuelve exactamente la ambigüedad descrita en spec.md (Historia 3, FR-004/FR-005) sin
reordenar nada de lo que ya funciona.

**Alternatives considered**:
- *Renombrar el `<h2>` existente para que cubra ambas secciones (p. ej. "Tipo de Orden y Mesa")*:
  rechazado — seguiría siendo un único encabezado compartido entre dos listas distintas (tipos de
  orden vs. mesas), que es justamente la ambigüedad que la spec pide resolver (FR-005: deben
  percibirse como dos elementos distinguibles).
- *Envolver pestañas y mesas en dos `<fieldset>`/tarjetas separadas con más espacio entre sí*:
  rechazado por alcance — la spec (Assumptions) pide un título distinguible, no un rediseño
  estructural del bloque completo; un cambio más amplio no está autorizado por esta spec.

## Resumen de impacto en tests existentes

`manual-order-page.component.spec.ts` (232 líneas, 8 casos, `pos-heladeria/src/app/modules/tables/
pages/`) no tiene ningún test `"CONGELA comportamiento actual:"` — 0 en todo `pos-heladeria/src/`
(mismo hallazgo que specs 045/046/047/048/049). Ninguno de los 8 casos existentes depende del
markup exacto de la tarjeta de catálogo ni del bloque de tipo de orden/listado de mesas (verificado
leyendo el archivo completo): cubren selección de mesa por parámetro de ruta, habilitación de
pestañas, cambio de mesa, agregar producto al draft, confirmar/enviar y volver a la Terminal — todo
vía `store` (señales/computeds), no vía queries de DOM sobre el bloque que este plan modifica.
