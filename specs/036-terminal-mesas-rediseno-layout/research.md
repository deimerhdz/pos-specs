# Research: Rediseño de Layout de la Terminal de Mesas (actualizado sobre prototipos 2026-08-26)

Todos los puntos "NEEDS CLARIFICATION" de la Technical Context del plan quedaron resueltos por
investigación directa del código de `pos-heladeria` (no había ambigüedades de negocio pendientes —
esas ya se resolvieron en `spec.md`, sección Clarifications, incluida la sesión durante
`/speckit-plan` que corrigió el supuesto sobre "Dividir la cuenta"). Este documento registra las
decisiones técnicas necesarias para pasar de la spec actualizada a un diseño concreto.

## 1. Grilla de mesas con pestañas de tipo de orden ("Mesas" / "Domicilios" / "Para llevar")

- **Decisión**: se agrega un signal nuevo en `pos-terminal.store.ts`, p. ej.
  `orderTypeTab = signal<'mesas' | 'domicilios' | 'para-llevar'>('mesas')`, independiente del `filter`
  de ocupación (`TableFilter`, `pos-terminal.store.ts:92,236`) que ya existe y no cambia. Cuando
  `orderTypeTab() !== 'mesas'`, tanto la grilla de mesas como la sección "Pagos por confirmar" (ver §2)
  DEBEN resolver a una lista vacía — no se filtra ningún dato real, porque no existe todavía ninguna
  orden de esos tipos (spec.md, FR-003). El componente `pos-tables-panel.component.ts` gana esta
  pestaña por encima de su filtro de ocupación ya existente (`Todas/Libres/Ocupadas/Pendientes`,
  líneas 68-73), sin tocar la lógica de ese filtro.
- **Justificación**: mantiene el filtro de ocupación (dato real, ya implementado) y la pestaña de tipo
  (hoy siempre vacía salvo "Mesas") como dos ejes independientes, tal como los muestra el prototipo —
  evita mezclar ambos en un único `TableFilter` ampliado, lo que complicaría reutilizar `tablesView()`
  sin cambios.
- **Alternativas consideradas**: extender el enum `TableFilter` existente con `'domicilios'`/
  `'para-llevar'` — rechazada porque ese filtro ya tiene un significado de ocupación bien definido
  (spec 028, FR-014) y mezclarlo con tipo de orden rompería su semántica actual y las pruebas
  existentes de `deriveTableStatus()`/`tablesView()`.

## 2. Sección "Pagos por confirmar" (agregación entre mesas)

- **Decisión**: `pos-terminal.store.ts` ya expone `pendingOrders` (líneas 347-349) — un `computed` que
  filtra **todas** las órdenes con `status === 'recibida' && channel === 'qr'`, sin acotarse a la mesa
  seleccionada (a diferencia de `pendingOfSelectedTable`, líneas 394-397, que sí filtra por selección).
  Se agrega un `computed` nuevo, p. ej. `pendingPaymentsView`, que enriquece `pendingOrders()` uniendo
  cada orden con su mesa (`tables()`) para obtener número/nombre, reutilizando el mismo dato que ya
  alimenta `elapsedLabel`/`totalLabel` en `tablesView()`. La confirmación de pago (efectivo) y la
  aprobación/rechazo de transferencia siguen llamando exactamente a los mismos métodos del store que
  hoy usa `payment-attempt-review-panel.component.ts` — no se duplica lógica de negocio, solo se
  renderiza en dos lugares (el panel de la mesa seleccionada y la nueva lista agregada).
- **Componente**: se evalúa reutilizar `payment-attempt-review-panel.component.ts` en un modo
  "compacto" (una tarjeta por orden, sin el layout completo de dos columnas que usa hoy dentro de
  `payment-validation-block`) vs. crear un componente nuevo y delgado que solo orqueste la tarjeta y
  delegue la acción de confirmar/rechazar al store. Se prefiere la segunda opción si el layout completo
  de `payment-attempt-review-panel` no es fácilmente componible en una tarjeta angosta tipo lista — a
  decidir en la fase de tareas/implementación tras revisar el HTML exacto del componente.
- **Justificación**: no existe hoy ningún campo persistido `hasPendingPayment` en `Table`/
  `dining_table` — el estado es siempre derivado (`deriveTableStatus()`, líneas 147-183) — así que la
  fuente de verdad correcta es el mismo `pendingOrders()` reactivo ya probado en
  `pos-terminal.store.spec.ts` (líneas 274-309), no un campo nuevo de backend (no hay cambio de modelo
  de datos, Principio VIII N/A).
- **Alternativas consideradas**: filtrar `tablesView()` por un enum de estado — rechazada porque
  `tablesView()` solo expone `statusLabel`/`chipClass` ya formateados (cadenas), no el enum
  `TableDisplayStatus` crudo; unir contra `pendingOrders()` es más directo y ya está probado.

## 3. Panel central: lista de ítems del pedido + "+ Agregar producto" embebido

- **Decisión**: `pos-order-panel.component.ts` (estado `'pedido'` de `store.centralState()`,
  `table-sessions.component.ts:98-115`) ya implementa casi exactamente el patrón objetivo: una lista
  compacta de ítems con acciones por línea y un botón final "＋ Agregar producto" que llama a
  `store.openCatalog()` (líneas 115-118). Hoy `openCatalog()` solo activa el signal `catalogOpen`, que
  `pos-catalog-drawer.component.ts` consume para mostrarse como overlay `fixed inset-0 z-40` (líneas
  17-18). El cambio necesario es: (a) retirar el wrapper de overlay de `pos-catalog-drawer` y montarlo
  dentro del mismo contenedor del panel central cuando `catalogOpen()` sea verdadero, alternando con
  `pos-order-panel` (no superponiéndose); (b) agregar un signal `catalogSearchText` y combinarlo con el
  `catalogCategoryId` ya existente en un nuevo computed (p. ej. `catalogProductsFiltered`) que extiende
  `catalogProducts` (líneas 618+).
- **Justificación**: minimiza el cambio — la única funcionalidad genuinamente nueva es el buscador por
  nombre (pedido explícitamente por el usuario); el resto es reubicación visual de estado y marcado ya
  existentes (`catalogOpen`, `catalogCategoryId`, `cartView()`).
- **Alternativas consideradas**: reescribir `pos-catalog-drawer` como componente nuevo desde cero —
  rechazada por Principio V (evitar refactorizaciones no relacionadas) cuando la lógica de filtrado por
  categoría ya es reutilizable con cambios mínimos.

## 4. Colapsar/expandir el menú de navegación global en escritorio

- **Decisión**: `layout.service.ts` no cambia su API pública (`sidebarOpen`, `open()`/`close()`/
  `toggle()`, línea 5-17, ya existen y son suficientes). Hoy `sidebarOpen()` solo controla el
  slide-over móvil: `sidebar.component.ts` (líneas 20-27) fuerza `md:relative md:translate-x-0` de
  forma incondicional desde el breakpoint `md`, y el backdrop de `dashboard-layout.component.ts`
  (líneas 19-24) es `md:hidden`. Se hace la visibilidad/ancho del `<aside>` condicional a
  `sidebarOpen()` también en escritorio, y se ajusta el margen/ancho del contenido en
  `dashboard-layout.component.ts` para ocupar el espacio liberado cuando el sidebar está colapsado.
- **Justificación**: es el cambio mínimo para que el toggle ya existente tenga efecto visible en
  escritorio; no se introduce un servicio ni un estado paralelo.
- **Alternativas consideradas**: un signal nuevo específico de "colapso de escritorio" separado de
  `sidebarOpen` — rechazada por duplicar estado ya existente y complicar la sincronización entre vista
  móvil y de escritorio sin necesidad.

## 5. "Dividir la cuenta entre varias personas" y "Facturar a nombre de"

- **Decisión**: ninguna investigación adicional cambia comportamiento — se confirma (ver spec.md,
  Clarifications, sesión durante `/speckit-plan`) que ambos ya están completamente implementados:
  `billingCustomerName` (`pos-checkout-panel.component.ts:172-183`) y el botón que abre
  `split-bill-panel.component.ts` (líneas 98-102 y 159-165 de `pos-checkout-panel.component.ts`), que
  asigna ítems/unidades por comensal vía `participant_id` (spec 010) y alimenta el cobro dividido de
  `session-bill-panel.component.ts`. Esta spec solo reubica visualmente ambos elementos junto con el
  resto del panel derecho (FR-009, FR-010); no se toca `split-bill-panel.component.ts` ni
  `session-bill-panel.component.ts` más allá de su disposición dentro del layout, si acaso.
- **Justificación**: evita duplicar o reimplementar una funcionalidad que ya existe y ya tiene
  cobertura de tests (`split-bill-panel.component.spec.ts`, `session-bill-panel.component.spec.ts`).
- **Nota de compatibilidad**: el botón combinado "Cobrar, Facturar y Enviar a Cocina" solo aplica hoy a
  pedidos aún no enviados a cocina (`checkout-and-send`, comentario en
  `pos-checkout-panel.component.ts` líneas 83-94); para pedidos ya en cocina, el cobro sigue el camino
  separado de `session-bill-panel` ("Cobrar y cerrar mesa"). Esta rama de decisión existente
  (`showSessionCharge()`) **no cambia** con el rediseño — FR-009 solo reordena visualmente ambos modos,
  sin alterar cuál se activa ni cuándo.

## 6. Cobertura de tests

- **Decisión**: agregar `pos-tables-panel.component.spec.ts` (hoy inexistente) cubriendo la nueva
  pestaña de tipo de orden y la persistencia del filtro de ocupación ya existente, siguiendo el patrón
  de `sidebar.component.spec.ts` (mock parcial del store) o el de `pos-checkout-panel.component.spec.ts`
  (setup con `TestBed` + store real/parcial). Agregar un spec para el nuevo componente de "Pagos por
  confirmar" y extender `pos-terminal.store.spec.ts` con casos para el nuevo computed
  `pendingPaymentsView` (reutilizando los mismos fixtures que ya prueban `pendingOrders`, líneas
  274-309). `pos-order-panel.component.spec.ts` y `pos-checkout-panel.component.spec.ts` existentes
  deben seguir en verde sin cambiar su intención (solo ajustes de selector si cambian por el reacomodo
  visual).
- **Justificación**: Principio X (verificación obligatoria) exige cobertura de la funcionalidad nueva;
  no tocar la intención de los specs existentes preserva la garantía de que el comportamiento de cobro,
  división de cuenta y validación de pago (specs 010, 024, 028) no cambió.
