# Research: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-10-02

La spec llegó a esta fase **sin ningún `[NEEDS CLARIFICATION]`**: las dos ambigüedades de producto (estructura del resumen e impuestos/subtotal) se resolvieron con el usuario antes de redactarla y están en `spec.md` → *Clarifications*. Este documento no las revisa.

Lo que queda por resolver en la fase de plan es de tres clases:

1. El **pendiente explícito** que la spec le dejó al plan: el chevron del colapsable (D3).
2. Las **decisiones técnicas** que la spec no podía tomar sin mirar el código: dónde sale el "Ahorro", cómo se fija el estado inicial sin pelear con el usuario, cómo no duplicar markup, qué tests hacen falta (D1, D2, D4, D5, D9, D10, D11).
3. Una **contradicción interna de la spec** detectada al leer el código del paso 1 (D6), más dos límites de alcance que conviene dejar por escrito (D7, D8).

---

## D1 — Mecanismo del colapsable: `<details>` / `<summary>` nativo

**Decisión**: el acordeón es un `<details>` con su `<summary>`, sin librería y sin señal de estado en el componente. El componente nunca conoce si está abierto o cerrado.

**Razón**: no es una preferencia estética, es la forma de volver dos requisitos **inviolables por construcción** en vez de dependientes de que el código esté bien escrito:

- **FR-010** (expandir o contraer MUST NOT modificar el pedido ni disparar carga, envío o cobro): imposible de violar si no hay handler, si no hay estado y si el navegador es el que abre y cierra. Un acordeón con `signal` + `(click)` siempre deja la puerta abierta a que alguien cuelgue un efecto de ese click.
- **FR-011** (operable con teclado y estado anunciado al lector de pantalla): el `<summary>` es focusable por defecto, responde a Enter y Espacio, y los navegadores modernos lo exponen con rol de *disclosure* y su estado expandido/contraído. No hay que escribir `tabindex`, `role`, `aria-expanded` ni `keydown` — y por lo tanto no hay forma de escribirlos mal.

**Alternativas consideradas**:

- **`@angular/cdk` (`CdkAccordion` / `cdkAccordionItem`)**: ya está en el proyecto (`@angular/cdk@^21.2.14`), así que **no habría violado el Principio IX**. Se descartó igual: aporta estado y API que esta pantalla no necesita (un solo panel, sin acordeón múltiple, sin animación pedida por la spec) y nos devolvería la responsabilidad de la semántica ARIA que `<details>` regala.
- **Colapsable propio con `signal<boolean>` + `@if`**: más código, menos accesibilidad gratis, y reintroduce exactamente el riesgo que FR-010 quiere cerrar.
- El proyecto **no tiene hoy ningún componente de acordeón** que reutilizar (`grep '<details'` sobre `src/app` no devuelve nada), así que no se descartó ninguna reutilización existente.

**Consecuencia para los tests**: en jsdom, `<details>`/`<summary>` existe y `details.open` es manipulable, pero el *toggle por click en el summary* no está implementado de forma confiable. Los tests de expandir/contraer fijan `details.open = true/false` directamente y verifican el render, en vez de simular el click. Ver [quickstart.md](./quickstart.md).

---

## D2 — Estado inicial: una sola escritura, nunca una reafirmación

**Decisión**: el componente recibe el estado inicial ya resuelto y lo **congela en un campo plano** (no una señal, no un `computed`) durante `ngOnInit`. La plantilla lo lee con `[attr.open]` sobre una expresión que, por construcción, no vuelve a cambiar en toda la vida del componente.

**Razón**: es la única trampa real de usar `<details>` con un framework. El elemento es *no controlado*: el usuario abre y cierra el DOM sin pasar por Angular. Si el atributo `open` se enlaza a una expresión viva (`[attr.open]="lineas().length <= 3 ? '' : null"`), basta con que esa expresión cambie de valor para que Angular **reescriba el atributo y cierre de un golpe el panel que el comensal acababa de abrir** — una violación directa de FR-009. Hoy el carrito no muta en el paso 3 (FR-010 y el supuesto de "sin recarga en vivo" lo garantizan), así que el bug no se dispararía; pero dejarlo enlazado a una expresión viva es dejar armado un fallo que aparecería el día que alguien añada recarga del carrito a esa pantalla.

Congelarlo en `ngOnInit` elimina la posibilidad: Angular escribe el atributo una vez en el primer ciclo de detección de cambios y nunca más, porque el valor enlazado es literalmente constante. `ngOnInit` del componente hijo corre **antes** de que se evalúen las expresiones de su propia plantilla, así que no hay un primer render con el valor equivocado.

**Alternativas consideradas**:

- **`[open]` como *property binding* sobre una expresión viva**: el fallo descrito arriba.
- **`@ViewChild` + `nativeElement.open = ...` en `ngAfterViewInit`**: funciona, pero añade acceso directo al DOM y un ciclo de vida más para algo que un campo congelado resuelve sin tocar el DOM.
- **Que el componente derive el umbral de su propio input de líneas en cada render**: mismo problema que la expresión viva.

---

## D3 — El chevron: se añade `chevron-down` al set de iconos del proyecto

**Decisión** (este era el pendiente que la spec le dejó al plan): añadir un caso `chevron-down` a `src/app/shared/icon/icon.component.ts`, ocultar el marcador nativo del `<summary>` y rotar el icono 180° cuando el `<details>` está abierto, usando la variante `group-open:` de Tailwind sobre un `group` puesto en el `<details>`.

**Razón**: el set de iconos es **del proyecto** — un `@switch` de SVG inline, sin librería externa ("This is the project's icon set — no external icon library is configured"). Añadirle un caso es seguir la convención del repo, no añadir superficie: es un `<path>` de una línea, aditivo, sin tocar ningún icono existente, y `chevron-down` es el icono genérico que cualquier otra pantalla va a querer después. No hay dependencia nueva, así que el Principio IX no entra en juego.

La alternativa que la spec dejaba abierta — **rotar por CSS el indicador nativo** (el triángulo de `::marker` / `::-webkit-details-marker`) — se descarta porque es el único camino del diseño cuyo resultado depende del navegador: `::marker` acepta muy pocas propiedades, `transform` no es una de ellas de forma confiable, y la forma y el tamaño del triángulo difieren entre Chrome, Safari y Firefox. Un icono propio se ve igual en los tres.

**Detalle de implementación** (para que no se improvise en la fase de tareas):

- `<details class="group ...">`, y dentro del `<summary>` el icono envuelto en un `<span class="... transition-transform group-open:rotate-180">`.
- El marcador nativo se oculta con `list-none [&::-webkit-details-marker]:hidden` en el `<summary>`; al darle `display: flex` para maquetar el título y el chevron, Chrome ya deja de pintarlo, y las dos utilidades cubren Safari y Firefox.
- `cursor-pointer` en el `<summary>`: el navegador no lo pone solo.

---

## D4 — Un solo markup de líneas dentro del propio componente (`ngTemplateOutlet`)

**Decisión**: el componente compartido tiene **dos modos de envoltura** (colapsable y plano) pero **un solo markup de la lista y los totales**, definido una vez en un `<ng-template>` y usado desde las dos ramas con `[ngTemplateOutlet]`.

**Razón**: FR-021 exige que el formato de líneas y totales sea idéntico en los dos pasos, y RN-004 exige una sola fuente de cálculo. Extraer el resumen a un componente compartido y después duplicar su contenido *adentro* (una copia en la rama `@if (colapsable)` y otra en el `@else`) reintroduciría, un nivel más abajo, exactamente la divergencia que FR-021 quiere eliminar: alguien corrige el formato de una línea en una rama y no en la otra, y los dos pasos vuelven a verse distintos sin que ningún test obvio lo note.

**Alternativas consideradas**:

- **Dos componentes hermanos** (uno colapsable, uno plano) que compartan un tercero con la lista: tres archivos para lo que un input resuelve, y el input `colapsable` ya está pedido por la spec (FR-022: "El modo colapsable es una entrada de ese componente").
- **Renderizar siempre el `<details>` y, en el paso 1, dejarlo abierto sin `<summary>`**: un `<details>` sin `<summary>` no se puede cerrar, pero sigue siendo un elemento interactivo para la tecnología asistiva y añade un nivel semántico que el paso 1 no quiere. FR-022 dice "sin control de colapso", no "con un control escondido".

---

## D5 — El "Ahorro" sale de `cart.total` vs. `cart.discounted_total`, no de un cálculo nuevo

**Decisión**: añadir a `DiningCartService` dos miembros **aditivos**:

- `grossTotal: Signal<number>` — el total de lista, `CartResponse.total`, que hoy el servicio **descarta**.
- `savings: Signal<number>` — `max(0, grossTotal() - total())`, derivado con `computed`.

La fila "Ahorro" se pinta sólo cuando `savings() > 0`. `total()` **no cambia de semántica**: sigue siendo el total vigente (con descuento si lo hay), tal como lo consumen hoy `review-step`, `cart.component` y `public-menu`.

**Razón**: el dato ya viaja en la respuesta y ya está resuelto por el backend. `DiningCartService.apply()` hace hoy `this.total.set(effectivePrice(cart.total, cart.discounted_total))`, que se queda con el número con descuento y tira el de lista a la basura. El contrato está fijado por characterization tests de `pos-backend` (`test_cart_service.py`): `discounted_total` es `None` **cuando no hay promoción** (no cero) y es **estrictamente menor** que `total` cuando sí la hay. Es decir: la diferencia *es* el ahorro, y no hay que calcular ni inferir nada. Esto es lo que mantiene a RN-001 y FR-014 en pie — el front no introduce ninguna cifra propia.

El `max(0, ...)` no es desconfianza del backend: es lo que hace que la fila simplemente no aparezca si algún día llega un `discounted_total` igual o mayor, en vez de pintar un "Ahorro: -$500".

**Por qué aditivo y no un refactor de `total`**: cambiar la semántica de `total()` tocaría tres pantallas que no están en el alcance de esta spec (Principio V). Dos miembros nuevos no tocan ninguna.

**Alternativas consideradas**:

- **Sumar `lineTotal` de las líneas y comparar con `total()`**: sería un cálculo propio del front, prohibido por FR-014/RN-001, y daría un número distinto en cuanto el descuento sea de nivel de pedido y no de línea.
- **Pedir un campo nuevo al backend**: innecesario y rompería "sin cambios de contrato con el backend" (spec → *Decisiones de Compatibilidad*).

---

## D6 — La fila "Ahorro" aparece también en el paso 1 ⚠️ *pendiente de confirmación del negocio*

**La contradicción**: la spec dice dos cosas incompatibles sobre el paso de revisión (paso 1):

- **FR-012 + FR-021 + US4 esc. 1** → el resumen es el mismo artefacto en ambos pasos, y "las líneas, el orden, el formato de los montos, **el Ahorro** y el Total son idénticos".
- **"Impacto sobre Funcionalidades Existentes"** → el paso 1 "cambia de forma estructural, no visual… conservando el mismo contenido y el mismo aspecto que hoy. **Cualquier diferencia visible en ese paso respecto a hoy es una regresión, no una mejora.**"

Y el código decide quién tiene razón: `review-step.component.ts` **hoy no pinta ninguna fila de "Ahorro"** — sólo líneas y Total. Así que mostrar el Ahorro en el paso 1 *es* una diferencia visible respecto a hoy.

**Decisión del plan**: la fila "Ahorro" se muestra en **ambos** pasos. Prevalecen FR-012, FR-021 y US4 esc. 1, que son el cuerpo normativo de la spec, sobre la prosa de la sección de impacto, que describe la *intención* ("no quiero que el paso 1 se vea distinto") sin haber advertido que el paso 1 no tenía esa fila.

**Razón**: la alternativa — un resumen que oculta el Ahorro en el paso 1 y lo muestra en el paso 3 — viola de frente US4 esc. 1 y, peor, RN-004 ("el comensal nunca debe ver dos cifras distintas para lo mismo"): el comensal vería el mismo pedido descrito de dos formas distintas en dos pantallas consecutivas, que es exactamente la desconfianza que esta spec vino a eliminar. Además, el caso sólo se manifiesta **cuando hay una promoción vigente aplicada**; sin promoción, el paso 1 se ve idéntico a hoy, que es el caso por defecto.

**Acción pendiente**: esto es una **decisión de negocio**, no técnica (Principio XI), y cambia un comportamiento existente de forma visible, así que el Principio II pide que quede registrada. La fase de tareas debe incluir, **antes de implementar US4**:

1. Confirmar con el usuario que acepta ver la fila "Ahorro" en el paso 1 cuando hay promoción.
2. Si la acepta: registrar la entrada correspondiente en `specs/000-reconocimiento/registro-de-anomalias.md` (la siguiente libre es **A-103**) y corregir la frase de la sección "Impacto sobre Funcionalidades Existentes" de `spec.md` para que no contradiga a FR-012/FR-021.
3. Si **no** la acepta: el escape ya está diseñado y cuesta un booleano — el componente expone el input `showSavings`, y el paso 1 lo pasa en `false`. Eso dejaría US4 esc. 1 parcialmente incumplido, lo cual tendría que quedar registrado como excepción explícita en la propia spec.

El diseño no se compromete con ninguna de las dos salidas: el input existe en cualquier caso y su valor por defecto es `true`.

---

## D7 — `cart.component.ts` queda fuera de alcance, por decisión y no por olvido

**Decisión**: el tercer lugar del frontend que pinta el mismo markup de líneas y Total — `modules/tables/components/cart.component.ts`, el carrito del menú público — **no se migra** al componente compartido.

**Razón**: FR-021 nombra dos pasos ("el paso de revisión del pedido y el paso de datos de pago") y la sección *Fuera de Alcance* prohíbe "mostrar el resumen en cualquier otra pantalla no nombrada en esta spec". Arrastrar el carrito del menú a la extracción sería una refactorización no exigida por la funcionalidad, que es precisamente lo que el Principio V trata como trabajo independiente.

**Consecuencia asumida y declarada**: después de esta spec quedan **dos** fuentes del markup de líneas, no una — la compartida de los dos pasos del checkout, y la del carrito del menú. RN-004 queda satisfecha en su letra (el comensal no ve dos cifras distintas del mismo pedido *en la misma pantalla*, y los dos pasos del checkout sí convergen), pero el riesgo de divergencia con el carrito del menú sigue vivo. Es candidato natural a un spec propio de limpieza; aquí se deja anotado, no hecho.

---

## D8 — El umbral de 3 líneas es una constante exportada del componente

**Decisión**: el umbral vive como una constante exportada del propio componente compartido (p. ej. `UMBRAL_EXPANDIDO_POR_DEFECTO = 3`), no como un número literal en la plantilla ni duplicado en los tests.

**Razón**: la spec es explícita en que "el umbral de 3 líneas es una decisión de producto, no un cálculo de altura" y que ajustarlo "es una decisión de producto que se registra, no un detalle de implementación" (*Assumptions*). Una constante con nombre y un único punto de cambio es lo que hace que ese ajuste sea un cambio de una línea con un test que lo acompaña, en vez de una búsqueda por el repo. Los tests de frontera (3 → expandida, 4 → contraída, FR-008 y el caso límite de la spec) importan la constante en vez de repetir el `3`.

---

## D9 — Presupuesto vertical de la pantalla: el total va arriba del nombre del método

**Decisión**: el orden del paso 3 queda así, de arriba hacia abajo:

1. Encabezado fijo existente (volver · indicador de paso · salir) — **sin cambios**.
2. **Bloque de total a pagar** (nuevo): el total como el texto más grande de la vista, con el conteo de productos debajo.
3. `<h1>` con el nombre del método + su frase de apoyo, y el bloque índigo con datos bancarios y QR — **el `<h1>` baja de posición**.
4. Sección `<details>` **"Resumen del pedido"** (nueva).
5. Zona de comprobante y botón "Enviar pedido" — **sin cambios de comportamiento**.

**Razón**: FR-001 no deja margen — el total debe ser "el primer dato que se lee bajo el encabezado, **por delante del nombre del método de pago**". Hoy el `<h1>` con el nombre del método es lo primero bajo el encabezado, así que mover el `<h1>` hacia abajo no es una libertad de diseño: es el requisito. FR-016 confirma el resto del orden. Este movimiento es un cambio visible de una pantalla existente, documentado por la propia spec (a diferencia de D6, aquí no hay contradicción que resolver).

**Riesgo medido, no estimado**: el bloque de total añade alrededor de 70-80 px y el `<details>` contraído alrededor de 48 px al alto de una pantalla que ya puede llevar un QR de 220 px. SC-003 exige que "Enviar pedido" se alcance con **a lo sumo un** desplazamiento en 360 × 640 px con 6 líneas. El plan **no afirma** que se cumpla: lo deja como medición obligatoria en dispositivo o emulación (ver [quickstart.md](./quickstart.md)). Si no se cumple, RN-002 dice qué cede — prevalece la acción de enviar — y las palancas, en este orden, son: compactar el bloque de total (el conteo en la misma línea que la cifra), reducir el `max-width` del QR, y por último bajar el umbral de D8, que es una decisión de producto que se registra.

---

## D10 — Qué se verifica con tests y qué sólo se verifica en un dispositivo

**Decisión**: tres niveles, sin pretender que el de arriba cubra al de abajo.

| Nivel | Herramienta | Qué cubre |
|-------|-------------|-----------|
| Unitario de componente | Vitest + `TestBed` (jsdom) | FR-003 a FR-009, FR-012 a FR-015, FR-021, FR-022: qué se pinta, con qué cifras, en qué orden, con el `<details>` abierto o cerrado, con y sin promoción, con y sin adicionales y nota. |
| Unitario de servicio | Vitest sobre `DiningCartService` | D5: `grossTotal` y `savings` con `discounted_total` ausente, `null` y menor que `total`. |
| Manual en dispositivo | Chrome DevTools (emulación 360 × 640 y 320 px) + lector de pantalla | SC-003 (desplazamiento real), SC-006 (anuncio del estado), SC-009 (sin scroll horizontal), SC-005 (flujo de comprobante de punta a punta), FR-001 (que el total sea *de verdad* el texto más grande). |

**Razón**: jsdom no hace layout. No tiene alto de viewport, no calcula `scrollHeight` real, no aplica Tailwind y no habla con un lector de pantalla. Escribir un test de Vitest que "verifique SC-003" produciría una prueba verde que no prueba nada — peor que no tenerla, porque induce a creer que el requisito está cubierto. Los tres criterios de layout y accesibilidad se verifican a mano y se registran en `quickstart.md`, que es el artefacto donde esa verificación queda trazable (Principio X y XII).

**Un hallazgo concreto que la fase de tareas no puede olvidar**: `transfer-details-step.component.spec.ts` (261 líneas, de la spec 060) provee hoy `DiningCartService` como `{ clear: vi.fn(), clearDiner: vi.fn() }`. En cuanto la plantilla del paso 3 lea `cart.total()`, `cart.count()`, `cart.lines()` o `cart.savings()`, **todos los tests de ese archivo se caen** por métodos inexistentes. Ampliar ese doble de prueba es una tarea de la implementación, no un imprevisto.

**Base de no-regresión del paso 1**: `review-step.component.ts` **no tiene archivo de test hoy**. Se crea `review-step.component.spec.ts` **antes** de la extracción (US4), fijando el formato de línea, el orden y el Total, para que la extracción se haga contra una red y no contra una inspección visual. Esos tests se escriben **sin** el prefijo `"CONGELA comportamiento actual:"`: ese prefijo, por el Principio III, marca comportamiento que no se toca sin decisión de negocio, y aquí ya sabemos que la spec autoriza una diferencia (la fila "Ahorro", D6). Congelar algo que el mismo spec va a cambiar obligaría a editar el test en el mismo commit que lo crea, que es el anti-patrón que el Principio III existe para evitar.

---

## D11 — Pluralización del conteo: condicional inline, sin dependencia de i18n

**Decisión**: `"1 producto"` / `"N productos"` resuelto con un condicional inline en la plantilla, y expuesto como una sola propiedad del componente compartido para que FR-006 (el mismo número y la misma palabra en el bloque de total y en el título del resumen) se cumpla por construcción, no por coincidencia.

**Razón**: FR-003 y FR-006 piden una cadena, no un sistema de localización. El proyecto no tiene `@angular/localize` configurado, y `i18nPlural` exige un mapa de reglas para dos casos. Lo que sí importa es que la cadena se arme **una vez**: si el bloque de total la compone por su cuenta y el `<summary>` por la suya, nada impide que terminen distintas — y FR-006 existe precisamente para prohibir eso. Una propiedad única consumida por los dos lugares lo cierra.

**Nota de alcance**: el conteo es de **unidades** (`DiningCartService.count()`, que ya suma cantidades), mientras que el umbral del colapso es de **líneas** (`lines().length`). Son dos medidas distintas a propósito (spec → *Assumptions*) y el componente no debe mezclarlas: `count` entra como input, el umbral se mide sobre el arreglo de líneas.

---

## Resumen de salidas

- **Ningún `[NEEDS CLARIFICATION]` pendiente.** El único pendiente que la spec dejó al plan (el chevron) está resuelto en D3.
- **Una decisión de negocio pendiente de confirmar**: D6 (fila "Ahorro" en el paso 1). No bloquea el diseño; está aislada tras un input con valor por defecto.
- **Cero dependencias nuevas, cero cambios de backend, cero migraciones.**
- **Un riesgo a medir, no a estimar**: D9 (presupuesto vertical frente a SC-003).
