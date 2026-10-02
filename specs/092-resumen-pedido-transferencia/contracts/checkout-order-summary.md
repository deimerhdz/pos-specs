# Contrato de interfaz: `app-checkout-order-summary`

**Spec**: [spec.md](../spec.md) · **Plan**: [plan.md](../plan.md) · **Research**: [research.md](../research.md)

## Qué clase de contrato es este

Esta funcionalidad **no expone ninguna interfaz externa nueva**: no hay endpoint nuevo, no hay cambio en el contrato de `GET /cart`, no hay comando de CLI y no hay evento nuevo. El único contrato que esta spec crea es **interno al frontend**: la interfaz del componente compartido que FR-021 convierte en el artefacto único del resumen para los dos pasos del checkout.

Se documenta aquí porque ese contrato es justamente el mecanismo por el que FR-021, FR-022 y RN-004 se cumplen o se incumplen. Si el contrato está mal diseñado, los dos pasos divergen; si está bien diseñado, no pueden divergir.

**Archivo**: `pos-heladeria/src/app/modules/tables/pages/checkout/checkout-order-summary.component.ts`
**Selector**: `app-checkout-order-summary`
**Naturaleza**: `standalone`, `ChangeDetectionStrategy.OnPush`, **puramente presentacional** — no inyecta `DiningCartService`, no inyecta `Router`, no hace peticiones, no emite eventos.

## Entradas

| Input | Tipo | Requerido | Default | Requisito que lo obliga |
|-------|------|-----------|---------|-------------------------|
| `lines` | `CartLine[]` | sí | — | FR-005, FR-007 — las líneas a listar, en el orden en que llegan (el componente **no** reordena) |
| `total` | `number` | sí | — | FR-004, FR-014 — total vigente, ya con descuento aplicado |
| `count` | `number` | sí | — | FR-003, FR-006 — unidades totales del pedido (no líneas) |
| `savings` | `number` | no | `0` | FR-012 — `0` significa "sin promoción": la fila "Ahorro" no se pinta |
| `collapsible` | `boolean` | no | `false` | FR-008, FR-022 — `true` sólo en el paso de datos de pago; el paso de revisión lo deja en `false` |
| `showSavings` | `boolean` | no | `true` | Válvula de escape de la decisión pendiente de research.md D6. Si el negocio decide que el paso 1 no muestre la fila "Ahorro", ese paso pasa `false` y nada más cambia |

**Por qué `collapsible` y `showSavings` tienen default seguro**: el default (`collapsible: false`, `showSavings: true`) es el comportamiento del paso de revisión, que es el consumidor que **no debe cambiar**. Olvidar un input en ese paso produce el comportamiento correcto, no una regresión.

**Por qué los datos entran por input y no se inyecta el servicio**: un componente que inyecta `DiningCartService` sería imposible de probar con un pedido de 4 líneas sin montar medio checkout, y quedaría atado a *una* fuente de datos. Con inputs, el test construye el caso límite de FR-008 en dos líneas de código.

## Salidas

**Ninguna.** El componente no declara ningún `@Output`.

Esto no es una omisión, es el contrato: **FR-010** exige que expandir o contraer no modifique el pedido ni dispare carga, envío o cobro. Un componente sin salidas no le puede pedir nada a nadie, así que FR-010 no se puede violar desde aquí ni por accidente ni por un cambio futuro descuidado. Cualquier propuesta posterior de añadirle un `@Output` (por ejemplo "editar línea") es un cambio de alcance que necesita su propio spec — *Fuera de Alcance* ya prohíbe explícitamente la edición desde esta pantalla.

## Constantes exportadas

| Constante | Valor | Para qué |
|-----------|-------|----------|
| `UMBRAL_EXPANDIDO_POR_DEFECTO` | `3` | FR-008. Decisión de producto con un único punto de cambio (research.md D8). Los tests de frontera la importan en vez de repetir el literal |

## Contrato de render

### Cuando `collapsible === false` (paso de revisión, paso 1)

Renderiza la lista y los totales **planos**, sin `<details>`, sin `<summary>`, sin control de colapso y sin chevron (FR-022). El contenido y el formato son **idénticos** a los del paso 3 (FR-021).

### Cuando `collapsible === true` (paso de datos de pago, paso 3)

Envuelve el **mismo** contenido en un `<details>` con un `<summary>` titulado `"Resumen del pedido"` seguido del conteo de productos (FR-005, FR-006).

El atributo `open` inicial se resuelve una sola vez, en `ngOnInit`, como `lines.length <= UMBRAL_EXPANDIDO_POR_DEFECTO`, y **nunca se reafirma** después (FR-008 + FR-009, research.md D2).

### Invariantes de render — válidas en los dos modos

1. **Un solo markup**: la lista y los totales se definen una vez en un `<ng-template>` y se usan desde las dos ramas con `[ngTemplateOutlet]`. Duplicarlos reintroduce la divergencia que FR-021 prohíbe (research.md D4).
2. **Un solo conteo**: el bloque de total del paso 3 y el `<summary>` de este componente leen la misma composición, `formatProductCount` de `checkout-product-count.ts`, no dos composiciones paralelas (FR-006, RN-004). La pluralización es `"1 producto"` / `"N productos"` (research.md D11).

   La función vive en su propio archivo, y no como propiedad de este componente, porque el bloque de total (US1) debe poder entregarse **antes** de que este componente exista (Principio VI). El componente la expone igualmente como `countLabel` para que su plantilla la lea; el paso 3 la llama directo. Desvío menor respecto a `plan.md` → *Project Structure*, trazado en `implementation-notes.md` (Principio XII).
3. **Ninguna cifra se calcula aquí**: `total`, `savings` y `lineTotal` se pintan tal como llegan, formateados con `MoneyPipe`. El componente no suma, no resta y no redondea (FR-014, RN-001, y el supuesto de que el formato de moneda no cambia).
4. **Fila "Ahorro" condicional**: se pinta si y sólo si `showSavings && savings > 0`. Nunca en cero, nunca vacía (FR-012).
5. **Ninguna fila de impuestos ni de subtotal**, en ningún modo y bajo ninguna condición (FR-013, RN-003, SC-007).
6. **Campos opcionales de la línea**: el renglón de adicionales no se pinta si `optionNames` está vacío; el de la nota no se pinta si `notes` es `null` o cadena vacía (FR-007).
7. **Sin desborde horizontal**: nombres, adicionales y notas se recortan (`truncate` + `min-w-0`) en lugar de ensanchar la caja, desde 320 px (FR-019, SC-009).
8. **Colores y tipografía existentes**: se conservan los del resumen actual del paso 1 — tarjeta blanca redondeada con borde gris, cifras en gris oscuro, secundarios en gris claro (FR-019). El componente no introduce paleta nueva.

## Contrato de accesibilidad

| Requisito | Cómo lo cumple el contrato |
|-----------|----------------------------|
| FR-011 / SC-006 — operable con teclado | El `<summary>` es focusable por defecto y responde a Enter y Espacio. **No se añade `tabindex`** |
| FR-011 / SC-006 — estado anunciado | Lo expone el navegador a partir de `<details open>`. **No se añade `aria-expanded` a mano** mientras el lector anuncie el estado por su cuenta: duplicarlo puede desincronizarse del estado real del elemento. **Excepción autorizada**: si la verificación con lector real (T021) demuestra que el estado no se anuncia, T021-b permite añadirlo sincronizado con el evento nativo `(toggle)`. El contrato prefiere la semántica nativa, pero SC-006 manda sobre la preferencia: un estado que ningún lector anuncia no es accesible por elegante que sea el markup |
| Marcador nativo oculto | `list-none` + `[&::-webkit-details-marker]:hidden` en el `<summary>`, y chevron propio rotado con `group-open:rotate-180` (research.md D3). Ocultar el marcador **no** quita la semántica: el estado lo lleva el atributo `open`, no el triángulo |

## Contrato con `DiningCartService`

El componente no lo inyecta, pero el contrato de lo que los pasos le pasan sí depende de dos miembros **nuevos y aditivos** del servicio:

| Miembro | Tipo | Estado |
|---------|------|--------|
| `lines` | `Signal<CartLine[]>` | existente, sin cambios |
| `total` | `Signal<number>` | existente, **semántica sin cambios** (total vigente, con descuento) |
| `count` | `Signal<number>` | existente, sin cambios |
| `grossTotal` | `Signal<number>` | ⭐ **nuevo** — `Number(CartResponse.total)`, hoy descartado en `apply()` |
| `savings` | `Signal<number>` | ⭐ **nuevo** — `computed(() => Math.max(0, grossTotal() - total()))` |

Detalle y justificación en [data-model.md](../data-model.md) y research.md D5. **Aditivos a propósito**: `public-menu.component.ts` y `cart.component.ts` consumen `total()` y `count()` y están fuera de alcance (Principio V) — ninguno se ve afectado.

## Uso esperado por cada consumidor

**Paso 1 — `review-step.component.ts`** (FR-022): pasa `lines`, `total`, `count` y `savings`, y **no** pasa `collapsible`. El `<h1>"Tu pedido"</h1>` y el botón "Elegir método de pago" se quedan en el paso; el componente sólo reemplaza la tarjeta del resumen.

**Paso 3 — `transfer-details-step.component.ts`** (FR-005 a FR-016): pasa los mismos cuatro datos **más** `collapsible` en `true`, y lo ubica entre el bloque índigo de datos bancarios y la zona de comprobante (FR-016). El bloque de total destacado **no** es parte de este componente: vive en el paso, arriba de todo, porque FR-002 exige que esté fuera de cualquier zona colapsable — y meterlo dentro del componente compartido lo llevaría también al paso 1, que no lo pidió.

## Contratos que esta funcionalidad NO cambia

| Contrato | Estado |
|----------|--------|
| `GET /cart` → `CartResponse` | Sin cambios. No se pide ningún campo nuevo |
| `POST /cart/submit` (envío del pedido con comprobante) | Sin cambios (FR-018) |
| Presign y subida del comprobante a R2 | Sin cambios (FR-018) |
| `CheckoutProgressStore` (paso, método, comprobante subido) | Sin cambios. El resumen no lee ni escribe progreso |
| `checkoutHydrationGuard` | Sin cambios. El resumen depende de que ya recargue el carrito, no lo modifica |
| Pantallas del cajero, mesero y administrador | Sin cambios (FR-023). Ninguna superficie de cobro se ve afectada porque ningún cálculo se toca |
