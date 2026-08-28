# Research: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

Todos los puntos de la Technical Context del plan quedaron resueltos por investigación directa del
código de `pos-heladeria`, hecha **antes** de escribir la spec (ver spec.md, Input/Assumptions) —
la única decisión de negocio genuina (pestañas vs. vista combinada) ya se resolvió con el dueño del
producto antes de este plan. Este documento consolida esa investigación en el formato
Decisión/Justificación/Alternativas.

## 1. Diseño del estado nuevo en `pos-terminal.store.ts`

- **Decisión**: agregar, junto a `centralState` (líneas 447-456, sin tocar su cuerpo), un signal y
  dos computeds nuevos:

  ```ts
  /** Qué pestaña eligió el cajero cuando la mesa tiene a la vez un pago
   *  pendiente de confirmar y un pedido pagado/activo (FR-001). Por defecto
   *  la más urgente; se reinicia junto con el resto del estado transitorio
   *  de la selección en `resetTransient()`. */
  readonly centralPanelTab = signal<'validar-pago' | 'pedido'>('validar-pago');

  /** ¿La mesa seleccionada tiene A LA VEZ algún pago pendiente de confirmar
   *  y algún pedido pagado/activo? Es el gatillo de las pestañas (FR-001). */
  readonly hasPendingAndActiveOrders = computed(() => {
    const tableId = this.selectedTableId();
    if (!tableId) return false;
    return this.pendingOfSelectedTable().length > 0 && this.ordersOfTable(tableId).length > 0;
  });

  /** Qué debe renderizar el panel central: la pestaña elegida cuando hay
   *  ambos tipos de pedido a la vez, o `centralState()` tal cual en
   *  cualquier otro caso (FR-005) — mismos tres valores que ya consume el
   *  `@switch` de contenido, sin ningún estado nuevo que ese switch tenga
   *  que aprender. */
  readonly effectiveCentralView = computed<'validar-pago' | 'mesa-libre' | 'pedido'>(() =>
    this.hasPendingAndActiveOrders() ? this.centralPanelTab() : this.centralState(),
  );
  ```

- **Justificación**: `ordersOfTable(tableId)` (privado, líneas 396-398, filtra `activeOrders()` por
  mesa) ya excluye exactamente lo que hay en `pendingOfSelectedTable()` (misma frontera
  `recibida`+`qr` — `activeOrders`, líneas 389-393, excluye `recibida`+`qr`; `pendingOrders`, líneas
  365-367, es exactamente ese subconjunto) — no hace falta ningún filtro nuevo, solo combinar dos
  computeds ya existentes. `effectiveCentralView` devuelve el mismo tipo de unión que ya consume el
  `@switch` de contenido (`'validar-pago' | 'mesa-libre' | 'pedido'`), así que ese switch no
  necesita ningún caso nuevo — solo cambiar de qué computed lee (ver §2).
- **Por qué no tocar `centralState()` directamente**: `centralState()` sigue siendo la única fuente
  de verdad sobre "qué pasa con esta mesa" para todo lo demás que ya la consume sin relación con
  esta spec (ninguna, según research.md de spec 047 — `tablesView` usa `deriveTableStatus`/
  `tableOrders`, no `centralState`). Mezclar la lógica de "qué pestaña eligió el cajero" dentro de
  `centralState()` la volvería impura (dependería de una elección de UI, no solo de los datos) y
  complicaría cualquier otro consumidor futuro. Mantenerlas separadas (`centralState` = qué hay;
  `effectiveCentralView` = qué se ve) es más simple de razonar y de testear.
- **Reinicio del signal**: `resetTransient()` (privado, líneas 949-956) ya se llama desde
  `selectTable()` (línea 868) y `cancelSelection()` (línea 941) para limpiar otro estado transitorio
  de la selección (`draftLines`, `catalogOpen`, etc.) — es el punto exacto donde agregar
  `this.centralPanelTab.set('validar-pago');`. Importante: `reload()` (líneas 814-818) **no** llama
  a `resetTransient()`, así que un pago pendiente nuevo que llegue mientras el cajero ya está en la
  pestaña "Pedido de la mesa" no lo saca de ahí (spec.md, Edge Cases) — es una consecuencia directa
  de dónde se coloca el reinicio, no requiere ninguna condición adicional.
- **Alternativas consideradas**: convertir `centralState()` en un tipo con cuatro valores (agregar
  `'validar-pago-y-pedido'` o similar) — rechazada porque obligaría a tocar el cuerpo de
  `centralState()` (que hoy es puro, solo de datos) para incorporarle una preferencia de UI, y
  duplicaría en el `@switch` del template la lógica de qué mostrar dentro de ese caso combinado en
  vez de reutilizar el mismo `@switch` ya existente.

## 2. Cambios en `table-sessions.component.ts`

- **Decisión**: el `@switch` de contenido (líneas 119-149) cambia únicamente su expresión de
  `store.centralState()` a `store.effectiveCentralView()` — el cuerpo de los tres `@case`
  (`'validar-pago'`, `'mesa-libre'`, `@default`) no cambia ni una línea. El encabezado (líneas
  104-110, hoy un `<span>` con un `@switch (store.centralState())` de solo texto) gana una rama
  nueva:

  ```html
  <span class="text-sm font-semibold text-gray-500">
    @if (store.hasPendingAndActiveOrders()) {
      <button type="button" (click)="store.centralPanelTab.set('validar-pago')" ...>
        🔔 Pagos por confirmar
      </button>
      <button type="button" (click)="store.centralPanelTab.set('pedido')" ...>
        Pedido de la mesa
      </button>
    } @else {
      @switch (store.centralState()) {
        @case ('validar-pago') { 🔔 Pagos por confirmar }
        @case ('mesa-libre') { Mesa libre }
        @default { Pedido de la mesa }
      }
    }
  </span>
  ```

  El botón de silenciar la campana (línea 111-119) no se toca — sigue en la misma fila, fuera de
  este `@if`/`@switch`, visible "pase lo que pase en el centro" (comentario ya existente en el
  archivo, línea ~91).
- **Justificación**: es el cambio mínimo que satisface FR-001 a FR-006 sin tocar ningún componente
  hijo. `payment-validation-block`/`pos-order-panel`/`pos-checkout-panel` se siguen montando
  exactamente con las mismas entradas que hoy (`[orders]="store.pendingOfSelectedTable()"`,
  `[categories]`, `[cashShiftId]`, `(refresh)="store.reload()"` para el primero; sin `@Input` para
  el segundo, que ya lee del store directamente) — el `@switch` que decide cuál montar no sabe ni le
  importa si está resolviendo `centralState()` o `effectiveCentralView()`, solo lee el mismo tipo de
  valor de siempre.
- **Alternativas consideradas**: un componente de pestañas genérico reutilizable — rechazada por
  sobre-ingeniería para dos botones que solo aparecen en un lugar de la aplicación (Principio V, no
  refactors oportunistas); un `@Input mode` en `payment-validation-block`/`pos-order-panel` para que
  decidan ellos mismos si mostrarse — rechazada porque la decisión de qué se ve es exclusiva del
  contenedor (`table-sessions.component.ts`), no de los bloques hijos, que no necesitan saber nada
  de esta spec.

## 3. Cobertura de tests

- **Decisión**: en `pos-terminal.store.spec.ts`, agregar un bloque de test para
  `hasPendingAndActiveOrders`/`effectiveCentralView` (sin HTTP, mismo patrón que
  `describe('PosTerminalStore.pendingOrders — solo canal qr')`): una mesa con un pedido `'pagada'`
  y otro `'recibida'`+`qr` a la vez debe resolver `hasPendingAndActiveOrders() === true` y
  `effectiveCentralView() === 'validar-pago'` por defecto; tras `centralPanelTab.set('pedido')`,
  `effectiveCentralView() === 'pedido'`; con solo uno de los dos tipos,
  `hasPendingAndActiveOrders() === false` y `effectiveCentralView()` coincide con `centralState()`.
  Agregar también un caso para el reinicio: tras `centralPanelTab.set('pedido')` y luego
  `selectTable(otraMesa)`, `centralPanelTab()` vuelve a `'validar-pago'`.
- En `table-sessions.component.spec.ts`, agregar un bloque nuevo siguiendo el patrón ya usado en
  `describe('TableSessionsComponent — diálogo de éxito sin botón duplicado (spec 029)')` (líneas
  55-101: `store = fixture.componentInstance.store; vi.spyOn(store, 'init').mockResolvedValue(undefined);`
  luego se ponen signals del store directamente y se llama `fixture.detectChanges()`, sin HTTP): con
  una mesa que tiene ambos tipos de pedido, confirmar que aparecen los dos botones de pestaña en el
  encabezado y que, tras hacer clic en "Pedido de la mesa", el panel central deja de mostrar
  `app-payment-validation-block` y pasa a mostrar `app-pos-order-panel` (o su contenido observable),
  y viceversa al volver a "Pagos por confirmar".
- **Justificación**: Principio X (verificación obligatoria) exige cerrar la brecha exacta que
  reportó el usuario — hoy no existe ningún test que ejercite una mesa con ambos tipos de pedido a
  la vez ni el nuevo mecanismo de pestañas.
