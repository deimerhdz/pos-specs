# Research: Rediseño de Layout de la Terminal de Mesas

Todos los puntos "NEEDS CLARIFICATION" de la Technical Context del plan quedaron resueltos por
investigación directa del código de `pos-heladeria`/`pos-backend` (no había ambigüedades de negocio
pendientes — esas ya se resolvieron en `spec.md`, sección Clarifications). Este documento registra las
decisiones técnicas necesarias para pasar de la spec a un diseño concreto.

## 1. Barra visual de tiempo transcurrido (franja de "Órdenes Recientes")

- **Decisión**: la barra de tiempo es un elemento visual nuevo, independiente del cálculo de badge de
  estado. Su color reutiliza la misma categorización de estado ya calculada por `deriveTableStatus()` /
  `STATUS_META` en `pos-terminal.store.ts` (la misma fuente que ya pinta el `chipClass` del badge), en
  vez de inventar un umbral de tiempo nuevo. Su longitud/relleno se calcula a partir del mismo dato que
  ya alimenta el texto `elapsedLabel` existente (`DiningOrder.created_at`, vía el pedido más antiguo de
  la mesa/orden — `pos-terminal.store.ts`, función `tablesView()`).
- **Justificación**: evita introducir una regla de negocio nueva no pedida por el usuario (a qué minuto
  exacto una orden se considera "demorada") — decisión que le correspondería al negocio, no a este
  plan (Principio XI de la Constitución). Reutilizar la categorización de estado ya existente mantiene
  la barra consistente con el badge de texto+color que la acompaña (FR-001).
- **Alternativas consideradas**: (a) un umbral de minutos fijo e inventado por el plan — rechazado por
  ser una decisión de negocio no autorizada; (b) omitir el color y usar un solo color neutro — rechazado
  porque el diseño de referencia sí usa color para comunicar urgencia, y el badge de texto ya garantiza
  la accesibilidad exigida por spec 028 FR-014 independientemente del color de la barra.

## 2. Navegación tipo carrusel (flechas izquierda/derecha)

- **Decisión**: implementación nativa sin librería — un `ElementRef` sobre el contenedor de tarjetas,
  desplazado con `scrollBy({ left: ±container.clientWidth, behavior: 'smooth' })` por cada clic de
  flecha (un "tramo" = un ancho de contenedor, consistente con la Assumption ya registrada en el spec:
  "no una tarjeta a la vez"). El estado deshabilitado de cada flecha se deriva de `scrollLeft` y
  `scrollWidth` del contenedor (señal actualizada en el listener `(scroll)`), sin estado adicional en
  el store.
- **Justificación**: no existe ningún patrón de scroll horizontal con flechas en todo `pos-heladeria`
  hoy (los únicos `overflow-x-auto` encontrados —`sales-page.component.ts`, `public-menu.component.ts`—
  son wrappers simples sin controles); tampoco hay ningún componente de carrusel en `@angular/cdk` que
  resuelva esto listo para usar. Construirlo con las primitivas del DOM ya disponibles evita agregar una
  dependencia nueva (Principio IX) para una necesidad de UI simple.
- **Alternativas consideradas**: (a) agregar una librería de carrusel (p. ej. Swiper) — rechazada por
  Principio IX (dependencia nueva no justificada para un requisito tan simple); (b) desplazar tarjeta
  por tarjeta — rechazada porque la Assumption del spec ya fija "tramo fijo", y un tramo de un ancho de
  contenedor es más predecible para el cajero que contar tarjetas individuales.

## 3. Colapsar/expandir el menú de navegación global desde la Terminal de Mesas

- **Decisión**: `LayoutService` (`layout.service.ts`) no cambia su API pública (`sidebarOpen`, `open()`,
  `close()`, `toggle()` ya existen y son suficientes — el nuevo botón simplemente los inyecta y llama).
  Lo que sí cambia es `sidebar.component.ts`: hoy fuerza `md:relative md:translate-x-0` de forma
  incondicional a partir del breakpoint `md`, por lo que en escritorio el sidebar es siempre visible
  sin importar `sidebarOpen()`. Se hace esa clase condicional también en escritorio
  (`sidebarOpen() ? 'md:relative md:translate-x-0' : 'md:hidden'` o equivalente), y se ajusta el margen
  del contenido en `dashboard-layout.component.ts` para que ocupe el espacio liberado.
- **Justificación**: es el cambio mínimo necesario para que el toggle ya existente tenga efecto visible
  en escritorio (hoy solo controla el slide-over móvil); no se introduce un servicio ni un estado
  paralelo.
- **Alternativas consideradas**: crear un signal nuevo específico para "colapso de escritorio" separado
  de `sidebarOpen` — rechazada por duplicar estado que ya existe y complicar la sincronización entre
  vista móvil y de escritorio sin necesidad.

## 4. Menú central embebido (reemplazo del catálogo superpuesto)

- **Decisión**: se reutiliza tal cual la lógica de filtrado por categoría ya implementada en
  `pos-terminal.store.ts` (`catalogCategoryId`, `setCatalogCategory()`, `catalogProducts` computed) y el
  marcado de pestañas de categoría de `pos-catalog-drawer.component.ts`. Se agrega un signal nuevo de
  texto de búsqueda (p. ej. `catalogSearchText`) y se amplía el computed existente (o se agrega uno
  nuevo `catalogProductsFiltered`) para combinar categoría + coincidencia de nombre. Se retira el
  wrapper de overlay (`fixed inset-0 ... bg-black/40` y la animación de panel deslizante) para que el
  grid quede embebido de forma permanente en el panel central.
- **Justificación**: minimiza el cambio — la única funcionalidad genuinamente nueva es el buscador por
  nombre (pedido explícitamente por el usuario); todo lo demás es reubicación visual del mismo estado y
  marcado ya existentes.
- **Alternativas consideradas**: reescribir el catálogo como un componente nuevo desde cero — rechazada
  por Principio V (evitar refactorizaciones no relacionadas con la funcionalidad pedida) cuando la
  lógica existente ya es reutilizable con cambios mínimos.

## 5. Cobertura de tests

- **Decisión**: agregar `pos-tables-panel.component.spec.ts` (hoy inexistente) cubriendo los 3 filtros y
  el comportamiento del carrusel antes de modificar el componente, siguiendo el patrón ya usado en
  `pos-checkout-panel.component.spec.ts`. Igual para el nuevo panel de catálogo embebido. Los specs
  existentes de `pos-checkout-panel.component.spec.ts` y `pos-terminal.store.spec.ts` deben seguir en
  verde sin modificarse en su intención (solo ajustes si cambian selectores de DOM por el reacomodo
  visual, nunca sus aserciones de comportamiento).
- **Justificación**: Principio X (verificación obligatoria) exige cobertura de la funcionalidad nueva; y
  no tocar la intención de los specs existentes preserva la garantía de que el comportamiento de cobro
  (spec 028) no cambió.
