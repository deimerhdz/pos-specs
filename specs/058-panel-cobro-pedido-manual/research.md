# Research: Ajustes al panel de cobro de pedido manual

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-08-29

## Decisión 1 — Patrón de solo-lectura + "editar" para "Facturar a nombre de"

**Decisión**: replicar en `PosCheckoutPanelComponent` exactamente el mismo patrón que ya usa el
campo "Cliente" de `ManualOrderPageComponent` (spec 054,
`manual-order-page.component.ts:164-187`): una señal local de solo interacción (no vive en el
store), un método para activarla y otro para desactivarla al perder foco, y el mismo marcado
(`<input readOnly>` + botón `✏️` posicionado en `absolute right-2`).

- Señal local nueva: `editandoFacturacion = signal(false)` (mismo criterio que `editandoCliente`,
  `manual-order-page.component.ts:336`: estado de interacción de este componente, no del store,
  porque no necesita sobrevivir a un cambio de pedido seleccionado — de hecho **debe** reiniciarse
  al cambiar de pedido, igual que hoy se reinicia `paymentDraft`, ver Decisión 3).
- Métodos nuevos: `toggleEditarFacturacion()` (pone la señal en `true`) y
  `onFacturacionBlur()` (la pone en `false`), reflejo directo de `toggleEditarCliente()` /
  `onClienteBlur()` (`manual-order-page.component.ts:369-374`).
- Template: el `<input>` de `pos-checkout-panel.component.ts:146-152` gana
  `[readOnly]="!editandoFacturacion()"`, `[class.bg-gray-50]="!editandoFacturacion()"`,
  `[class.text-gray-500]="!editandoFacturacion()"`, `(blur)="onFacturacionBlur()"`, envuelto en un
  `<div class="relative">` con el botón `✏️` (`(click)="toggleEditarFacturacion()"`,
  `title="Editar nombre de facturación"`), mismo posicionamiento `absolute right-2 inset-y-0`.
- El binding existente `[value]="store.billingCustomerName()"` /
  `(input)="store.billingCustomerName.set(...)"` no cambia — el dato y su destino
  (`checkoutAndSend()`, `billing_customer_name`) siguen siendo los mismos (FR-003).

**Alternativas consideradas**:
- *Inventar un patrón visual distinto* (p. ej. un modal, o mostrar el nombre como texto plano sin
  ningún input debajo) — rechazado: el spec pide expresamente reusar la opción de editar ya
  conocida por el cajero en la otra pantalla, y la Constitución (Principio V) desalienta introducir
  variantes nuevas cuando ya existe una resuelta para el mismo problema.
- *Guardar `editandoFacturacion` en el store* — rechazado: ningún otro estado de "modo edición"
  puramente visual vive en el store (`editandoCliente` tampoco lo hace); mezclarlo ahí sería
  inconsistente con el propio precedente que se está reutilizando.

## Decisión 2 — Reinicio del modo edición al cambiar de pedido

**Decisión**: `editandoFacturacion` se reinicia a `false` cada vez que cambia
`store.selectedOrderId()`, dentro del mismo `effect()` que ya reinicia `paymentDraft`
(`pos-checkout-panel.component.ts:280-286`).

**Rationale**: sin este reinicio, si el cajero deja el campo en modo edición y cambia de pedido
seleccionado sin que el campo pierda foco (p. ej. haciendo clic en la lista de mesas), el nuevo
pedido heredaría el modo edición del anterior — comportamiento inconsistente con "nace en modo
solo lectura" (FR-001). Es el mismo `effect()` existente, sin componente ni suscripción nueva.

**Alternativas consideradas**: un `effect()` separado solo para esto — rechazado, agregar una línea
al `effect()` que ya existe para el mismo propósito (reiniciar estado por-pedido) es más simple y
evita un segundo efecto redundante.

## Decisión 3 — Texto del botón principal

**Decisión**: cambiar únicamente el literal de la línea 173
(`pos-checkout-panel.component.ts`) de `'Cobrar, Facturar y Enviar a Cocina'` a `'Cobrar'`, dejando
intacto el operador ternario que ya distingue el estado de envío (`store.checkoutSubmitting() ?
'Cobrando…' : '...'`), el binding `(click)="checkout()"`, y el `[disabled]`. Cero cambios en
`checkout()` ni en `store.checkoutAndSend()`.

**Rationale**: es exactamente el ajuste pedido (FR-004/FR-005/FR-006) — un cambio de texto puro,
sin superficie de comportamiento adicional.

## Decisión 4 — Cómo ocultar "Imprimir Factura" y "Liberar Mesa" mientras el cobro está pendiente

**Decisión**: agregar un `computed` nuevo en `PosCheckoutPanelComponent`,
`pendingCheckout = computed(() => this.sidebarMode() === 'terminal-pos' && !!this.store.selectedOrder() && !this.showSessionCharge())`,
y envolver el footer completo (`pos-checkout-panel.component.ts:190-227`, ambos botones) con
`@if (!pendingCheckout())` como condición adicional (anidada dentro del `@if (store.sessionBill();
as bill)` que ya existe hoy).

`pendingCheckout()` es verdadero exactamente en la misma condición que ya decide mostrar la rama de
formulario de cobro editable (línea 120, rama `@else` final: ni `resumen` ni `showSessionCharge`,
con un pedido seleccionado) — es la única rama donde el cobro de este pedido aún no se ha
efectuado. Reutiliza `sidebarMode()` y `showSessionCharge()`, ya computados por el propio
componente; no introduce ningún estado nuevo del store.

**Por qué no condicionar cada botón por separado en vez de todo el footer**: los dos botones ya
tienen sus propias condiciones internas (`selectedOrder()` para Imprimir Factura,
`centralState() !== 'validar-pago'` para Liberar Mesa, spec 046). Añadir `&& !pendingCheckout()` a
cada una por separado duplicaría la misma condición dos veces; envolver el footer entero la
expresa una sola vez y dice explícitamente "este footer completo es una acción post-cobro" — más
legible que repetirla.

**Impacto sobre tests existentes** (relevante para Principio X/Verificación Obligatoria; ninguno de
estos tests tiene el prefijo `"CONGELA comportamiento actual:"`, así que el Principio III no
aplica, pero sí el Principio II: el cambio de comportamiento que estos tests protegían está ahora
autorizado por spec 058, Historia 3):

- `pos-checkout-panel.component.spec.ts:162-180` ("T033/spec 029: ofrece 'Imprimir Factura' para el
  pedido seleccionado, con cuenta de sesión") — hoy fija `sessionBill` sobre un pedido `'recibida'`
  (exactamente el estado "cobro pendiente") y espera que el botón **aparezca**. Con FR-007 debe
  seguir **sin aparecer** en ese mismo estado — la aserción final debe invertirse a
  `toBeUndefined()`. No hace falta un test adicional en otro estado: `showSessionCharge` (segundo
  `describe`) ya cubre por separado que el pedido "en cocina" muestra la cuenta completa, y ahí es
  donde puede agregarse, si se quiere cobertura explícita nueva, una aserción de que "Imprimir
  Factura" sigue apareciendo (FR-008).
- `pos-checkout-panel.component.spec.ts:202-218` ("spec 046, FR-001/SC-001: 'Liberar Mesa' no se
  muestra mientras hay pago pendiente de confirmar") — ya espera `toBeUndefined()`; sigue pasando
  sin cambios (la condición nueva es un `||` adicional sobre un resultado que ya era "oculto" en
  este caso).
- `pos-checkout-panel.component.spec.ts:220-244` ("spec 046, FR-002/SC-003: 'Liberar Mesa' reaparece
  de inmediato al confirmarse el pago pendiente") — este test selecciona todo el tiempo el mismo
  pedido `'o1'` con `status: 'recibida'` (nunca se cobra dentro del test), así que con FR-007
  vigente el botón **ya no reaparece** al confirmarse el pago del otro pedido QR — sigue oculto
  todo el tiempo porque `'o1'` mismo sigue con el cobro pendiente. La aserción final
  (`toBeDefined()`) debe cambiar a `toBeUndefined()`, documentando que la regla nueva (spec 058)
  es más estricta que la de spec 046: ya no basta con que no haya pago QR pendiente en la mesa,
  también hace falta que el pedido propio ya esté cobrado.
- `pos-checkout-panel.component.spec.ts:246-269` ("T035: 'Liberar Mesa' pide la liberación y
  muestra el motivo del 409 si falla") — fija `sessionBill` sobre el mismo pedido `'o1'`
  `'recibida'` (sin segundo pedido QR) y hace clic en "Liberar Mesa". Con FR-007 el botón deja de
  existir en ese estado, así que este test debe **moverse** al segundo `describe`
  ("pedido ya en cocina, cobro por sesión de mesa", que usa `abiertaOrder()` con `status: 'abierta'`
  y ya tiene `sessionBill` configurado en su `beforeEach`) — ahí "Liberar Mesa" sigue apareciendo sin
  cambios (FR-008) y el mismo caso de red/409 sigue siendo válido de verificar.

Ningún otro test del archivo depende de la presencia de estos dos botones en el estado "cobro
pendiente" (confirmado con `grep` sobre el archivo completo).

## Decisión 5 — Alcance de archivos a modificar

**Decisión**: el único archivo de producción a modificar es
`pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts` (template +
señales/métodos nuevos del propio componente). El único archivo de test a modificar es
`pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts` (4 tests
existentes ajustados/movidos, ver Decisión 4, más los tests nuevos de FR-001/FR-002 del modo
edición). Ningún cambio en `pos-terminal.store.ts`, en `manual-order-page.component.ts`, ni en
ningún archivo de `pos-backend`.

**Rationale**: los tres ajustes son de presentación sobre un único componente ya identificado en
spec.md ("Alcance concreto sobre el sistema actual"); ni el dato (`billingCustomerName`,
`checkoutAndSend`) ni el contrato de backend cambian.
