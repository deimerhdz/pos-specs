# Phase 0 Research: Rediseño responsive de la Terminal de Mesas

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — el stack, el storage y el
testing ya estaban determinados por los dos repositorios existentes. Esta investigación se enfoca
en las decisiones técnicas necesarias para traducir las 5 historias de usuario de `spec.md` a una
estrategia de implementación, y en verificar que ninguna choca con comportamiento protegido.

## 1. Cuadrícula responsive (reemplaza el carrusel)

- **Decision**: usar una grilla CSS (`grid-template-columns`, vía utilidades de Tailwind ya
  disponibles en el proyecto) con 4/3/2 columnas según los breakpoints `lg`(1024px)/`md`(768px) ya
  usados hoy en `table-sessions.component.ts:54,73`, en vez de flexbox con `flex-wrap` o de mantener
  el `overflow-x-auto` actual del carrusel (`pos-tables-panel.component.ts:75`).
- **Rationale**: Tailwind 4 ya está en el proyecto (sin dependencia nueva); una grilla CSS nativa
  resuelve directamente FR-001 a FR-004 (N columnas por breakpoint, envolver en filas, scroll
  vertical) sin necesitar JavaScript de layout — a diferencia del carrusel actual, que sí necesita
  JS (`scrollCarousel()`, `pos-tables-panel.component.ts:160-168`) para desplazar por flechas; ese
  código (y los botones ‹/›) se retira junto con el carrusel.
- **Alternatives considered**: (a) mantener flexbox con `flex-wrap` — funciona, pero requiere fijar
  el ancho de cada tarjeta con cálculos porcentuales por breakpoint en vez de declarar columnas
  directamente, más frágil ante cambios de gap/padding; (b) una librería de virtualización de listas
  — rechazada, no se justifica (Principio IX) para 16-30 tarjetas típicas por pestaña, muy por
  debajo de donde la virtualización aporta valor.

## 2. Contadores de ocupación (Historia 2)

- **Decision**: un `computed()` nuevo en `pos-terminal.store.ts` derivado de la misma señal que ya
  alimenta `tablesView()`, expuesto una sola vez y consumido tanto por el resumen superior (FR-009)
  como por los filtros (FR-010) — nunca dos cálculos independientes.
- **Rationale**: satisface directamente FR-011 (ambos contadores deben coincidir siempre, mismo
  origen de datos) sin ningún endpoint ni petición nueva — el dato ya está cargado en memoria.
- **Alternatives considered**: calcular el resumen superior en el componente de presentación en vez
  del store — rechazada, duplicaría la lógica de conteo en dos lugares (componente + filtros que ya
  hoy derivan del store), justo el riesgo que FR-011 busca evitar.

## 3. Indicador de sincronización (Historia 3 / FR-014)

- **Decision**: derivar el indicador de `navigator.onLine` (evento `online`/`offline` del
  navegador) combinado con `store.error()` (ya existente, se pone en `null`/mensaje según el
  resultado de la última carga) — sin abrir ninguna conexión de verificación nueva ni sondeo
  periódico al servidor.
- **Rationale**: el sistema no tiene hoy ningún mecanismo de trabajo sin conexión (confirmado — ver
  Out of Scope de spec.md); estas dos señales ya existen y son suficientes para responder la
  pregunta real que el cajero necesita ("¿lo que veo está actualizado?"), sin inventar un protocolo
  de sincronización nuevo.
- **Alternatives considered**: un `setInterval` que haga ping al backend cada N segundos — rechazada,
  agrega tráfico de red recurrente sin necesidad de negocio declarada (spec.md no pide detectar
  caídas del servidor mientras la red del cliente sigue arriba, solo un indicador de estado general).

## 4. Acceso directo a abrir turno de caja (Historia 3 / FR-015-017)

- **Decision**: reutilizar tal cual los dos endpoints ya existentes del módulo de Caja —
  `GET /api/v1/cash/shifts/current` (`cash.service.ts:117`, para saber si hay turno abierto y quién
  es el cajero) y `POST /api/v1/cash/shifts/open` (`cash.service.ts:107`, ya invocado por
  `CashSessionStore.openShift()`, `cash-session.store.ts:445`) desde la nueva barra superior de la
  Terminal de Mesas.
- **Rationale**: FR-016 exige explícitamente no duplicar la lógica de apertura de turno; ambos
  endpoints ya existen, están en producción y ya tienen su propio flujo de validación (monto de
  apertura, caja registradora) — no hay ninguna razón de negocio para reimplementarlos.
- **Alternatives considered**: abrir el turno con un formulario embebido nuevo dentro de la Terminal
  de Mesas — rechazada, duplicaría validaciones y estados ya resueltos por `CashSessionStore`; el
  patrón más simple y consistente con FR-016 es invocar/navegar al mismo flujo ya implementado.

## 5. Etiqueta de turno ("Turno Mañana: Carlos M.")

- **Decision**: derivar el segmento "Mañana"/"Tarde"/"Noche" de la hora local de
  `CashShift.opened_at` en el frontend (mismo patrón de zona horaria del tenant ya usado por
  `TenantDatePipe`, referenciado en `cash-session.store.ts:27`), no de un campo nuevo.
- **Rationale**: `CashShift` (`cash.interface.ts:12-23`) no tiene ningún campo de nombre de turno
  hoy; agregar uno requeriría migración y captura manual — fuera de alcance (spec.md, Out of Scope).
  Derivarlo del horario evita expandir el modelo de datos mientras conserva el mismo efecto visual
  de la imagen de referencia.
- **Alternatives considered**: mostrar solo la hora exacta de apertura sin ninguna etiqueta de
  "Mañana/Tarde" — más simple, pero se aleja innecesariamente de la imagen de referencia cuando
  existe un default razonable sin costo de alcance adicional.

## 6. Exponer quién atiende el pedido ("Atendido por", Historia 4)

- **Decision**: en el backend, añadir `staff_user_name: str | None` a `OrderResponse`
  (`app/api/v1/orders/schemas.py:197-224`), poblado en el service (`orders/service.py`) mediante un
  `join`/lookup a `shared.users` por `DiningOrder.user_id` (`customer_order.py:115`) — mismo patrón
  ya usado para computar `paid` antes de serializar (`schemas.py:221-224`, comentario "El router lo
  asigna antes de serializar"). En el frontend, agregar el campo equivalente a la interfaz
  `DiningOrder` (`dining.interface.ts`) y consumirlo en `pos-terminal.store.ts` para exponerlo por
  mesa, y en `order-summary-card.component.ts` como nuevo input opcional.
- **Rationale**: cero migraciones (el dato ya existe en la columna `user_id`); reutiliza
  exactamente el patrón de "campo computado, no columna" que el propio `OrderResponse` ya usa para
  `paid` — consistente con el resto del archivo en vez de introducir un mecanismo de serialización
  distinto.
- **Alternatives considered**: exponer `user_id` crudo y resolver el nombre en el frontend con una
  llamada aparte a un endpoint de usuarios — rechazada, obliga a N peticiones adicionales (una por
  usuario distinto visible en la grilla) para un dato que el backend puede resolver en el mismo
  query con un join, sin coste adicional de petición.

## 7. Colapso a una sola vista en móvil al seleccionar (Historia 5, Assumption)

- **Decision**: en el ancho móvil (<768px), seleccionar una tarjeta reemplaza la vista de la
  cuadrícula por el panel de detalle/cobro (patrón maestro-detalle de una sola vista a la vez);
  un control de "volver" regresa a la cuadrícula preservando filtro/pestaña/scroll.
- **Rationale**: no hay espacio horizontal para dos columnas en un ancho móvil; es el patrón
  estándar ya usado por aplicaciones responsive de una sola columna con lista+detalle, y spec.md ya
  documentó esto como Assumption (ninguna de las 3 imágenes de referencia muestra el estado
  seleccionado en móvil).
- **Alternatives considered**: un panel deslizante (bottom sheet) superpuesto sobre la grilla —
  válido visualmente, pero agrega un patrón de interacción nuevo no sugerido por ninguna imagen de
  referencia; se prefiere el patrón más simple y predecible (reemplazo de vista) para no introducir
  comportamiento no solicitado.

## 8. Verificación de tests de characterization protegidos

- **Decision/Finding**: se confirmó (búsqueda de texto) que ningún archivo `*.spec.ts` de los
  componentes/store tocados por esta spec (`table-sessions`, `pos-tables-panel`,
  `pos-checkout-panel`, `pos-terminal.store`) contiene el prefijo `"CONGELA comportamiento actual:"`
  sobre el carrusel, los colores de `STATUS_META`, o cualquier otro comportamiento que esta spec
  modifica.
- **Rationale**: Principio III de la constitución exige autorización explícita + evidencia antes de
  tocar un test con ese prefijo; al no existir ninguno en el área afectada, no aplica ningún paso
  adicional de autorización más allá del ya documentado en `spec.md`.
- **Alternatives considered**: N/A — es un hallazgo de verificación, no una decisión de diseño.
