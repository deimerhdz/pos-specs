# Contrato: `<app-money-input>` (componente nuevo)

Componente Angular standalone nuevo en
`pos-heladeria/src/app/shared/money-input/money-input.component.ts`. No expone ningún
endpoint — este es el contrato de su interfaz pública (`@Input`/`@Output`/comportamiento como
`ControlValueAccessor`), para que cualquier punto de uso lo integre sin sorpresas.

## Uso típico

```html
<!-- Reactive forms -->
<app-money-input formControlName="unit_cost" [decimals]="2" />

<!-- Template-driven -->
<app-money-input [(ngModel)]="fondoInicial" />
```

## `@Input`s

| Input | Tipo | Default | Descripción |
|---|---|---|---|
| `decimals` | `number` | `0` | Cantidad de decimales que el campo admite (ver `research.md`, Decisión 3). `0` = solo pesos enteros, igual que `formatMoney` hoy. |
| `placeholder` | `string` | `'0'` | Igual que el `placeholder` nativo de un `<input>`. |
| `bordered` | `boolean` | `true` | `false` cuando un contenedor padre ya pone el borde (ej. una caja con el `$` como prefijo fuera del propio `<input>`, patrón ya usado en `product-form.component.ts`) — evita un doble borde. Descubierto durante la migración de T015. |
| `invalid` | `boolean` | `false` | Mismo propósito que en `password-input`/`searchable-select`: colorea el borde en rojo cuando el formulario que lo envuelve lo marca inválido. |
| `sizeClass` | `string` | `'px-3 py-2.5 rounded-xl text-sm'` | Clases de tamaño/padding, igual que en `password-input`. |
| `disabled` | — | — | No es un `@Input` propio: se controla vía `[disabled]` de Reactive Forms o `disabled` del `FormControl`, como cualquier `ControlValueAccessor`. |

## `@Output`s

| Output | Tipo | Descripción |
|---|---|---|
| `blurred` | `EventEmitter<void>` | Se emite cuando el `<input>` interno pierde el foco. Necesario porque el evento nativo `blur` no burbujea hasta el host de un componente — un `(blur)` puesto directamente sobre `<app-money-input>` nunca dispararía por sí solo. Descubierto durante la migración de T019 (un campo de `promotions-page.component.ts` necesitaba marcar "tocado" al perder el foco). |

## Contrato de valor (`ControlValueAccessor`)

- **Valor expuesto al formulario** (`onChange`, lo que llega a `formControlName`/`ngModel`):
  siempre un `number` limpio, o `null` si el campo está vacío. **Nunca** una cadena con
  separador de miles.
- **Valor recibido** (`writeValue`): acepta `number | null | undefined`. `null`/`undefined`
  deja el campo vacío (no lo muestra como `0` — FR-009 de `spec.md`).
- **Formato visible mientras se escribe**: `Intl.NumberFormat('es-CO', { maximumFractionDigits:
  decimals })`, el mismo locale que ya usa `shared/money.ts:formatMoney` — separador de miles
  con punto (ej. `$ 50.000`), consistente con cómo se ve el mismo número ya guardado en el
  resto de la aplicación (Clarification Q2 de `spec.md`).
- **Pegar (paste) o escribir caracteres no numéricos**: se descartan los caracteres inválidos
  al vuelo; el campo nunca queda en un estado que no se pueda seguir editando.

## Lo que este componente **no** hace

- No decide si un campo debe mostrarse como moneda o como otra cosa (por ejemplo, el campo
  dual porcentaje/pesos de `promotions-page.component.ts`) — esa decisión la sigue tomando el
  formulario que lo usa, igual que hoy decide cuál `<input>` renderizar según su propio
  estado (`form.type === 'percent'`).
- No valida reglas de negocio del monto (mínimos, máximos, obligatoriedad) — eso sigue siendo
  responsabilidad de los validadores del `FormControl` que lo envuelve, como con cualquier
  otro campo.
- No formatea con símbolo de moneda dentro del valor editable si eso interfiere con escribir
  — el símbolo `$` puede mostrarse como prefijo visual fuera del área editable del campo, no
  como parte del texto que el usuario edita.

## Compatibilidad

- Reemplaza, punto por punto, el `<input type="number">` crudo de los ~12 usos ya
  identificados (`spec.md`, Assumptions) — ningún `FormControl`/modelo cambia de tipo (sigue
  siendo `number | null`), así que ningún código que consuma esos valores después (envío al
  backend, cálculos) requiere cambios.
- No introduce ninguna dependencia nueva (Principio IX n/a) — construido sobre
  `ControlValueAccessor` y `Intl.NumberFormat`, ambos nativos de Angular/el navegador.
