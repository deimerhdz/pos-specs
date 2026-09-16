# Fase 0 — Investigación: Estandarización de Íconos en el Panel de Administración

**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

## Contexto investigado en el código real (`pos-heladeria`)

- El componente SVG artesanal vive en `src/app/shared/icon/icon.component.ts` (selector `app-icon`,
  standalone, `OnPush`, host `inline-flex w-full h-full`). Un único `@Input({ required: true }) name!: string`.
  Renderiza un `<svg>` inline por cada `@switch (name)`, con `stroke="currentColor" stroke-width="2"
  aria-hidden="true"` fijo. El tamaño y el color los define hoy el padre vía clases Tailwind
  (`w-5 h-5`, `text-gray-400`, etc.) aplicadas al host. 35 nombres soportados: `dashboard, sales,
  reports, orders, tables, sessions, products, categories, promotions, option-groups, units,
  inventory, suppliers, cash, payment-methods, users, eye, eye-off, settings, tenants, home, receipt,
  cart, exit, search, layers, transfer, upload, close, back, check-circle, alert-circle, image-off,
  copy, download`. El caso `@default` cae a un círculo simple (nunca queda vacío).
- **Hallazgo crítico**: `app-icon` **no es exclusivo del panel de administración**. Lo usan también
  `public-menu.component.ts` y los cuatro pasos de `checkout/*-step.component.ts` (flujo público de
  menú QR, fuera de alcance por FR-010). Ver spec.md, Clarifications, sesión durante `/speckit-plan`,
  para la decisión que esto obligó a tomar (Decisión D2 abajo).
- **Segundo hallazgo crítico**: los componentes `cart.component.ts`, `product-select.component.ts`,
  `payment-attempt-review-panel.component.ts` y `pos-catalog-drawer.component.ts` (con emojis 🛒 ✕ 💳
  🏷️ usados como íconos) son compartidos entre `manual-order-page.component.ts` (panel de
  administración, dentro de alcance) y el menú público del cliente (fuera de alcance). Ver spec.md,
  misma sesión de Clarifications, para la decisión (Decisión D6 abajo).
- El proyecto ya tiene `@angular/service-worker` (`^21.1.0`) registrado en `src/app/app.config.ts`
  (`provideServiceWorker('push-sw.js', ...)`, un wrapper de `public/push-sw.js` sobre el
  `ngsw-worker.js` generado, usado hoy solo para push notifications — spec 077). `ngsw-config.json`
  ya incluye un grupo de assets `lazy` con el glob `woff|woff2|otf|ttf`. Cualquier archivo de fuente
  colocado bajo `public/` queda cubierto automáticamente por ese grupo — no se requiere tocar
  `ngsw-config.json` para que la fuente de Material Icons quede disponible sin conexión.
- No existe hoy ningún `<link>` a una fuente externa en `src/index.html`, ni `@font-face` en
  `src/styles.css` (solo `@import "tailwindcss";`). No hay precedente de autoalojamiento de fuentes
  en este repositorio — se establece con esta feature.
- No hay script `lint` ni configuración de ESLint/Prettier-como-gate en `package.json` — no hay gate
  de lint que integrar en Fase 2. El único comando de verificación de frontend es `ng test` (Vitest,
  vía el builder `@angular/build:unit-test`).
- Búsqueda de `"CONGELA comportamiento actual:"` en todo `src/app` (`pos-heladeria`): **0
  coincidencias**. No existe ningún characterization test protegido en este repositorio que esta
  funcionalidad pueda poner en riesgo (Principio III de la constitución, ver Constitution Check en
  `plan.md`).

## Decisiones

### D1 — Librería de íconos y mecanismo de autoalojamiento

**Decisión**: Instalar el paquete npm `material-icons` (fuente + hojas de estilo autoalojables,
variante **Outlined**), copiar únicamente los archivos de fuente de esa variante a
`public/fonts/material-icons/`, y declarar un `@font-face` + una clase utilitaria
(`.material-icons-outlined`) en `src/styles.css`. Los íconos se renderizan como texto-ligadura
(`<span class="material-icons-outlined">table_restaurant</span>`), heredando tamaño y color del
elemento padre igual que el patrón `currentColor` que ya usa `app-icon` hoy (mismo comportamiento
visual de theming, solo cambia el mecanismo de renderizado interno).

**Razón**: satisface el pedido explícito ("la librería que vas a instalar... es material icons") y el
requisito de funcionamiento sin conexión (spec.md, Clarifications) sin depender de una fuente cargada
por CDN en tiempo de ejecución. El archivo de fuente autoalojado queda cubierto automáticamente por el
grupo de cacheo `lazy` ya existente en `ngsw-config.json` (ver Contexto arriba), sin tocar
configuración del service worker. La variante Outlined es la de trazo más fino, la más parecida
visualmente a los íconos SVG actuales (`stroke-width="2"`, sin relleno).

**Alternativas consideradas**:
- **Cargar la fuente desde Google Fonts (CDN)** — rechazada: viola directamente el requisito de
  funcionamiento sin conexión ya decidido en la sesión de Clarificación de `/speckit-clarify`.
- **Instalar `@angular/material` completo y usar `mat-icon`** — rechazada: trae consigo un sistema de
  theming y un conjunto de módulos de componentes que esta feature no necesita ni pidió nadie; viola
  el Principio IX (preferencia por la solución más simple) y el Principio V (no mezclar
  funcionalidad no relacionada) al introducir superficie nueva no justificada por el alcance de esta
  spec.
- **Material Symbols** (la fuente variable más reciente de Google, sucesora de Material Icons) —
  rechazada: quien pidió la funcionalidad nombró explícitamente "material icons", no "material
  symbols"; además la tipografía variable exige herramientas de subsetting adicionales no justificadas
  para el alcance de esta spec.
- **Mantener SVG inline por ícono (patrón actual)** — rechazada: es exactamente el patrón que esta
  spec pide eliminar del panel de administración (FR-005).

### D2 — El componente SVG artesanal (`app-icon`) no se elimina del código

**Decisión**: `src/app/shared/icon/icon.component.ts` permanece sin modificar. Se crea un componente
**nuevo**, con selector distinto, para todo ícono del panel de administración. Ningún archivo del
flujo público de menú QR se toca para lograr esto.

**Razón**: `app-icon` sigue siendo usado hoy por `public-menu.component.ts` y los pasos de
`checkout/*-step.component.ts`, fuera de alcance por FR-010. Borrar o renombrar el componente
rompería ese flujo, que esta misma spec prohíbe modificar. Ver spec.md, Clarifications, sesión
durante `/speckit-plan`, para el registro completo de este hallazgo y su resolución. FR-005 y SC-002
ya fueron actualizados en `spec.md` para reflejar que "eliminado" significa "sin referencias activas
dentro del panel de administración", no "borrado del repositorio".

**Alternativas consideradas**:
- **Reescribir `app-icon` en el mismo archivo para que use Material Icons por dentro** — rechazada:
  cambiaría el rendering (SVG → ligadura de fuente) también para el flujo público sin que ninguna
  historia de usuario de esta spec lo pida ni lo autorice; sería modificar el flujo público por la
  puerta de atrás, violando FR-010 literalmente en vez de solo su efecto visual acordado (D6).

### D3 — Contrato de accesibilidad del nuevo componente

**Decisión**: El nuevo componente acepta un input opcional (p. ej. `ariaLabel`). Si se provee, el
elemento raíz se renderiza con `role="img"` y `aria-label="<valor>"`. Si no se provee, se renderiza
`aria-hidden="true"` (ícono puramente decorativo, ya envuelto por texto visible o por un contenedor
con su propia etiqueta accesible). Ver `contracts/icon-component-contract.md` para el detalle
completo de esta API.

**Razón**: cumple FR-002/FR-011 (descripción accesible para íconos sin texto visible) sin forzar una
etiqueta redundante cuando el ícono ya está acompañado de texto visible o cuando el elemento
interactivo que lo envuelve (p. ej. un `<button>`) ya expone su propio nombre accesible — evita
lectores de pantalla anunciando el mismo texto dos veces.

### D4 — Reemplazo neutro de los íconos con temática de heladería

**Decisión**: El cono de helado (🍦) usado en `sidebar.component.ts:41` (marca por defecto del
tenant) y en `admin-dashboard.component.ts:50,179` (tarjeta/acceso "Productos") se reemplaza por el
ícono `storefront` de Material Icons en el primer caso (marca genérica de negocio) y por
`shopping_bag` en el segundo (entidad "producto"). El escudo (🛡️, super-admin) se conserva
semánticamente pero se migra al mismo componente nuevo usando `admin_panel_settings`.

**Razón**: `storefront`/`shopping_bag` son íconos neutros de uso genérico en cualquier tipo de
negocio, sin ninguna referencia a helados, cumpliendo FR-006/SC-003 directamente.

### D5 — Alcance estructural (admin vs. público) según el routing real

**Decisión**: El alcance del panel de administración corresponde exactamente a las rutas
cargadas por `modules/dashboard/routes.ts` (protegidas por `authGuard`/`tenantDomainGuard`/
`passwordChangeGuard`, más `roleGuard` por ruta) en `src/app/app.routes.ts`: panel principal, caja,
categorías, productos, promociones, mesas/terminal (`tables-page`, `table-sessions`,
`table-qr-sheet`, `manual-order-page`), órdenes, inventario, proveedores, usuarios, reportes,
configuración/cuenta/plan. Fuera de alcance: `modules/tables/pages/{public-menu, diner-shell,
expired-qr, checkout/*}` (ruta pública `path: ''` → `DinerShellComponent`, sin guard, por token) y
`modules/super-admin/**` (rama de nivel superior independiente, no forma parte del "panel de
administración" del tenant).

**Razón**: es la única frontera verificable en el código (guards de autenticación vs. rutas
públicas por token), consistente con la Asunción ya documentada en `spec.md` ("panel de
administración" = aplicación autenticada de uso interno).

### D6 — Íconos en componentes compartidos entre panel de administración y menú público

**Decisión**: Se migran igual que cualquier otro ícono dentro de alcance. Ver spec.md, Clarifications,
sesión durante `/speckit-plan`, para el registro completo de este hallazgo y su resolución (FR-004 >
alcance de FR-010 cuando el mismo componente sirve a ambos flujos; el cambio es puramente visual, sin
tocar lógica ni rutas del flujo público).

## Catálogo semántico de íconos

Ver `data-model.md` para la tabla completa nombre-semántico → glifo actual → ícono de Material
Icons elegido, organizada por módulo.

## Hallazgo durante `/speckit-implement`: SVG artesanales fuera de `app-icon`

Al implementar la migración del módulo de caja (T021, cierre de `cash-movement-modal.component.ts`)
se descubrió que su botón de cerrar no usaba el emoji `✕` catalogado, sino un `<svg>` escrito a mano
directamente en la plantilla — un tercer patrón de "ícono artesanal" que ni el componente `app-icon`
ni la búsqueda de emoji de `/speckit-plan` habían detectado. Una búsqueda amplia (`grep -rl "<svg"`)
en las rutas del panel de administración encontró **14 archivos adicionales con 43 íconos SVG
artesanales más**, incluido un tercer punto de temática de heladería no detectado antes (un ícono
con forma de cono/copa de helado usado para "Modificar Toppings" en
`manual-order-page.component.ts`).

Se presentó la decisión al usuario (alcance: expandir esta misma feature vs. dejarlo para una spec
futura vs. arreglar solo donde ya se tocaba el archivo) — respuesta: **expandir el alcance de esta
misma feature**. El catálogo completo de estos 43 íconos queda en `data-model.md`, sección
"Descubiertos durante la implementación"; `tasks.md` se actualizó con las tareas correspondientes
(Fase 3, US1) antes de continuar. No se abrió un anomaly-registry aparte por la misma razón que
research.md D2/D6: es un ajuste de presentación, no una decisión de negocio, y la autorización
(quién: usuario del producto; cuándo: durante esta misma sesión de implementación; qué cambia; por
qué) queda trazada aquí y en `tasks.md`.
