# Research: Importe fijo para pagos no efectivo y nombre de cliente en el desglose de cobro

Decisiones técnicas para implementar `specs/057-cobro-mesa-importe-y-cliente/spec.md`. Todo el
cambio vive en `../pos-heladeria` (Angular 21, signals) — no hay ningún cambio de backend ni de
modelo de datos (ver spec.md, Assumptions).

## Decisión 1 — Bloquear el importe con `[disabled]` reactivo sobre `<app-money-input>`, sin componente nuevo

- **Decisión**: en `payment-input.component.ts`, agregar `[disabled]="!isCash(draft().methodId)"`
  al `<app-money-input>` ya existente (líneas 52-56). `MoneyInputComponent` ya implementa
  `ControlValueAccessor` completo, incluido `setDisabledState(isDisabled)` (`money-input.component.
  ts:176-178`), que Angular invoca automáticamente cuando el `[disabled]` de un control envuelto en
  `ngModel` cambia — el propio `<input>` interno ya tiene `[disabled]="disabled()"`
  (línea 34) y su clase `disabled:bg-gray-50 disabled:text-gray-400` (línea 37) ya da la señal
  visual de "campo inactivo" sin ningún estilo nuevo que escribir.
- **Rationale**: cero componentes/lógica nueva — el mecanismo de bloqueo ya existe de punta a
  punta en `MoneyInputComponent`, solo nadie lo conecta todavía desde `PaymentInputComponent`. Es
  el mismo patrón (`ControlValueAccessor` + `[disabled]` reactivo) que ya usa el resto de inputs
  reutilizables del proyecto (`shared/password-input/`).
- **Alternatives considered**: `[readOnly]` en vez de `[disabled]` — descartado: `readonly` deja el
  valor foco-able y seleccionable (el navegador permite copiarlo, pero también deja el cursor
  activo, lo que en algunos navegadores móviles dispara el teclado numérico igual) y no tiene
  soporte nativo en el `ControlValueAccessor` de Angular Forms (`setDisabledState` es la API
  estándar) — `disabled` es más explícito y ya está completamente cableado.

## Decisión 2 — El total ya llega precargado y exacto: no hace falta recalcular nada

- **Hallazgo**: `setMethod()` (`payment-input.component.ts:96-99`) ya hace
  `this.patch({ methodId, amount: methodId ? this.total : 0 })` — al elegir **cualquier** método
  (efectivo o no) el importe ya arranca en el total exacto. Con el campo bloqueado para no-efectivo
  (Decisión 1), ese valor precargado simplemente deja de poder tocarse — no se necesita ningún
  código nuevo para "forzar" el importe al total, ya se forzaba al seleccionar el método, solo
  faltaba impedir que se editara después.
- **Confirmado sin cambio necesario**: `paymentIssue()` (`payment-draft.util.ts:60-75`) sigue
  funcionando sin modificaciones — con el importe bloqueado en el total exacto, `missingAmount()`
  siempre da `0` y `nonCashAmount() > total` nunca ocurre para un método no efectivo, así que
  `ready()` (`session-bill-panel.component.ts:244-246`) queda `true` en cuanto se elige el método,
  sin que el cajero tenga que hacer nada más — coincide exactamente con SC-002 de spec.md.
- **`changeDue()`** (`payment-draft.util.ts:39-41`) también sigue igual: con `amount === total`
  para no-efectivo, el vuelto siempre da `0` — el bloque "Vuelto" de `payment-input.component.ts:
  61-65` simplemente no se muestra para esos métodos (correcto: no hay vuelto en una transferencia).

## Decisión 3 — Un único punto de cambio cubre los dos flujos de cobro (FR-003)

- **Hallazgo**: `PaymentInputComponent` es el único componente que captura un importe de cobro en
  todo el proyecto — usado por `session-bill-panel.component.ts:144-148` (cierre de sesión /
  "Cuenta de la mesa") y directamente por `pos-checkout-panel.component.ts:155-159` ("Cobrar
  pedido"/mostrador, `checkout-and-send`). Confirmado por `grep` — ningún otro archivo referencia
  `<app-payment-input`.
- **Decisión**: cambiar únicamente `payment-input.component.ts` (Decisión 1) — ambos flujos heredan
  el comportamiento nuevo automáticamente, sin tocar `session-bill-panel.component.ts` ni
  `pos-checkout-panel.component.ts` para este requisito.
- **Rationale**: exactamente el mismo criterio que ya aplicó spec 046 (retirar la cuenta dividida
  del frontend) — la regla de negocio (no-efectivo = importe fijo) es del **método de pago**, no de
  la pantalla desde la que se cobra, así que corregir el componente compartido es la única
  ubicación correcta (no un atajo): dos implementaciones separadas del mismo bloqueo divergirían
  tarde o temprano.

## Decisión 4 — Nombre de cliente en la línea "Sin asignar (mesero)": reusar el `@Input` ya recibido

- **Decisión**: en `session-bill-panel.component.ts`, cambiar `lineLabel(label: string | null)`
  (líneas 271-273) para que, cuando `label` sea `null`, devuelva `this.customerName.trim()` si no
  está vacío, y solo entonces caiga a `'Sin asignar (mesero)'`:
  ```ts
  lineLabel(label: string | null): string {
    if (label) return label;
    return this.customerName.trim() || 'Sin asignar (mesero)';
  }
  ```
- **Hallazgo que hace esto trivial**: el componente **ya recibe** `@Input customerName = ''`
  (línea 174), y ambos llamadores **ya se lo pasan** como `store.customerName()`
  (`pos-checkout-panel.component.ts:79,101`) — el mismo signal que spec 054/055/056 ya usan y ya
  garantizan no vacío para pedidos creados desde la creación manual ("Consumidor final" por
  defecto). El `@Input` hoy solo se lee en `buildPayload()` (línea ~309) para el `customer_name`
  que se envía al backend al cobrar — nunca en el template. No hace falta ningún dato nuevo, ninguna
  petición nueva, ningún cambio de contrato.
- **Rationale**: reusa exactamente el mismo "nombre de facturación" que este mismo componente ya
  usa para lo que efectivamente se guarda en la venta — mostrar ese mismo nombre en el desglose es
  coherente por construcción, no una fuente de verdad nueva y distinta.
- **Alternatives considered**: derivar el nombre desde `bill` (la cuenta) en vez de `customerName`
  — descartado: `SessionBill`/`bill.split` no trae ningún campo de nombre de orden por línea
  (confirmado en `dining.interface.ts`, `SessionBill`/`BillSplitLine`), así que habría que ampliar
  el contrato del backend para un dato que el componente ya recibe por otro lado — desproporcionado
  frente a reusar el `@Input` existente.

## Decisión 5 — Hallazgo no pedido por la spec: selector de test obsoleto (`input[type="number"]`)

- **Hallazgo** (no anticipado en spec.md, encontrado al preparar los tests de FR-001): 8 tests ya
  fallan hoy, antes de cualquier cambio de esta spec, en `pos-checkout-panel.component.spec.ts`
  (5 casos, líneas 96, 390, 423) y `session-bill-panel.component.spec.ts` (3 casos, líneas 87,
  150) — **7 de los 8** con `TypeError: Cannot set properties of null/undefined (setting
  'value')`. Causa raíz de esos 7: esos selectores buscan `input[type="number"]`, pero
  `MoneyInputComponent` (el componente real que `PaymentInputComponent` ya usa desde antes de esta
  spec) renderiza su `<input>` con `type="text" inputmode="decimal"` (`money-input.component.ts:
  30-31`) — el selector quedó desactualizado de una versión anterior del campo de importe (antes
  de `MoneyInputComponent`) y nunca se corrigió.
  - **Corrección al diagnóstico inicial**: el 8º caso
    (`pos-checkout-panel.component.spec.ts`, `'T032: ofrece "Imprimir Pre-cuenta" cuando hay
    cuenta de sesión'`) falla por una causa **distinta y no relacionada**: el botón busca el texto
    `'Imprimir Pre-cuenta'`, que ya no existe en ningún lado de
    `pos-checkout-panel.component.ts` (la plantilla actual solo tiene "🧾 Imprimir Factura") — un
    test huérfano de una versión anterior de la pantalla, sin ninguna relación con el campo de
    importe ni con `MoneyInputComponent`. **Queda explícitamente fuera de alcance de esta spec**
    (no es un selector de importe, no lo toca ningún FR de spec.md) — se documenta aquí solo para
    que quede registrado como hallazgo aparte, candidato a un spec propio si el dueño del proyecto
    decide corregirlo.
  - Con ambas correcciones aplicadas (research.md Decisión 1, más el ajuste del valor formateado
    "12.000" — `Intl.NumberFormat('es-CO')`, `money-input.component.ts:119-125` — que quedaba
    enmascarado por el propio selector roto), los 7 casos relacionados con el importe quedan en
    verde; el 8º (T032) permanece en rojo, sin que esta spec lo autorice a cambiar.
  - **Fuera de alcance, no confundir**: la suite completa del proyecto tiene además otros 3 fallos
    preexistentes ajenos por completo a esta spec (`app.spec.ts`, `auth.service.spec.ts`,
    `sidebar.component.spec.ts` — nada relacionado con pago/importe/`MoneyInputComponent`) — no se
    tocan, no son parte de este hallazgo ni de esta spec (Principio V).
- **Decisión**: corregir esos selectores a `input[type="text"]` (o un selector más específico, p.
  ej. por clase/atributo de `app-money-input`) como parte de esta spec, no como un ticket aparte —
  es un prerrequisito real, no una limpieza oportunista: **no es posible escribir ningún test nuevo
  de FR-001** (bloqueo del importe para no-efectivo) sin poder ubicar primero el campo de importe
  real en el DOM, y el selector actual no lo encuentra.
- **Rationale**: Principio V de la Constitución permite esta corrección precisamente porque es
  necesaria para la funcionalidad de esta spec (verificarla con un test automatizado), no una
  refactorización no relacionada — sin ella, FR-001 quedaría sin ninguna cobertura de test posible
  en los dos archivos que ya ejercitan este componente.
- **Alcance de la corrección**: solo los selectores que apuntan al campo de importe/con-cuánto-paga
  (`amounts()`, los dos `querySelector('input[type="number"]')` sueltos); no se tocan otros
  selectores de esos archivos que no están rotos.
