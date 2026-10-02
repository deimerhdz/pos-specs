# Feature Specification: Resumen del Pedido en la Pantalla de Pago por Transferencia

**Feature Branch**: la spec vive en la carpeta `specs/092-resumen-pedido-transferencia/` de `pos-specs`, rama `main`. Las ramas de código de `pos-heladeria` siguen el Principio XIV y se definen en `tasks.md`

**Created**: 2026-10-02

**Status**: Ready for planning

**Input**: User description: "Mejorar la UX de la pantalla de pago por Transferencia Bancaria (paso 3 del checkout del comensal). Hoy el comensal no puede ver qué está pagando en esa vista: solo ve los datos bancarios y la zona de comprobante. Incorporar un resumen completo y claro del pedido (productos, adicionales y total) en esa pantalla, brindando confianza antes de salir de la app hacia la app bancaria, sin entorpecer el flujo de subida del comprobante."

## Usuarios

- **Comensal (Menú QR, checkout)**: cliente sentado en la mesa que armó su pedido desde el menú por QR y eligió pagar con un método que exige comprobante. Necesita validar el monto exacto y los ítems de su orden antes de salir de la aplicación para entrar a la de su banco.

Esta funcionalidad no tiene impacto en el cajero, el mesero ni el administrador: ninguna de sus pantallas cambia.

## Problema

El paso de datos de pago del checkout del comensal muestra el nombre del método, los datos de la cuenta (banco, número, titular), el QR cuando existe, y la zona para adjuntar el comprobante. **No muestra ni el monto a pagar ni qué se está pagando.**

Eso obliga al comensal a una de dos conductas, ambas malas:

1. Memorizar el total del paso anterior (la pantalla de revisión) antes de avanzar, o
2. Volver atrás dos pasos para releer el total, y volver a avanzar, perdiendo de vista los datos bancarios.

En el momento de mayor desconfianza del flujo — justo antes de transferir dinero desde otra aplicación — el comensal no tiene a la vista ni cuánto debe transferir ni por qué. El riesgo concreto es transferir un monto equivocado, lo que después obliga al cajero a rechazar el pago y rehacer el cobro.

## Clarifications

### Session 2026-10-02

- Q: ¿Cómo se estructura visualmente el resumen para no entorpecer la subida del comprobante en móvil? → A: **Acordeón adaptativo**. Una sección colapsable titulada "Resumen del pedido" que arranca **expandida cuando el pedido tiene 3 líneas o menos** y **contraída cuando tiene más de 3**. El **total a pagar vive fuera del acordeón**, arriba, siempre visible y como elemento de mayor jerarquía de la pantalla. Se descartó la lista estática completa (empuja el botón de enviar fuera de la primera pantalla en pedidos grandes) y la lista truncada con "ver N más" (mismo beneficio, pero exige estado propio y pierde la semántica nativa de expandible).
- Q: El modelo de datos no tiene impuestos ni subtotal, solo total y descuento por promoción. ¿Qué se muestra? → A: **Solo Total + Ahorro**. Se pinta la fila "Ahorro" únicamente cuando hay descuento de promoción vigente, y el Total. **No se muestran impuestos**: el dato no existe en ningún nivel del pedido del comensal y mostrarlo sería inventar información. Modelar impuestos queda explícitamente fuera de alcance de esta spec.

### Hallazgos del reconocimiento (contexto, no decisiones)

- El resumen del pedido **ya existe** en el paso de revisión del checkout (paso 1), con exactamente el contenido que este brief pide para el paso 3: cantidad × producto · presentación, adicionales elegidos, nota del comensal, total de línea y Total. Esta spec no define un resumen nuevo: define que ese mismo resumen aparezca también en el paso de datos de pago.
- El pedido del comensal **no tiene impuestos ni subtotal** en ningún nivel: solo expone el total de cada línea, el total del pedido y, cuando hay promoción vigente, el total ya con el descuento aplicado. No hay campo de impuestos que mostrar.
- Al entrar al checkout, el sistema **siempre recarga el pedido vigente** y devuelve al comensal al menú si está vacío. Consecuencia: el resumen nunca puede aparecer vacío en el paso de datos de pago, ni siquiera si el comensal recarga el navegador directamente en ese paso. Esta spec no necesita definir un estado "resumen vacío".
- El paso de datos de pago **no es exclusivo de un método llamado "Transferencia"**: lo alcanza *cualquier* método de pago que no sea efectivo, es decir, todo método que exija comprobante. El comportamiento de esta spec aplica a todos ellos, no solo al que el negocio haya nombrado "Transferencia Bancaria".
- Los adicionales se cobran **una sola vez por línea** y no escalan con la cantidad del producto (comportamiento fijado por la spec 089). El resumen debe reflejar ese cálculo, no recalcularlo por su cuenta.
- Si el comensal ya había subido un comprobante y vuelve a entrar al paso, la vista previa del comprobante se rehidrata sin pedir el archivo de nuevo. El resumen debe convivir con ese estado sin alterarlo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver cuánto debo transferir, sin buscarlo (Priority: P1)

Como comensal, al llegar a la pantalla de datos de pago quiero ver el total a pagar de inmediato, destacado y sin tocar nada, para saber exactamente cuánto transferir cuando abra la aplicación de mi banco.

**Why this priority**: es el dato sin el cual la pantalla no cumple su función. Un comensal que no sabe cuánto transferir o transfiere un monto equivocado, o abandona el pedido. Entrega valor por sí sola incluso si el resumen de ítems nunca se implementa.

**Independent Test**: armar un pedido, elegir un método de pago que exija comprobante, y comprobar que el total aparece en la pantalla resultante sin desplazarse ni expandir nada, y que coincide con el total del paso de revisión.

**Acceptance Scenarios**:

1. **Given** armé un pedido con varios productos, **When** elijo un método de pago que exige comprobante y llego a la pantalla de datos de pago, **Then** veo el total a pagar como el elemento de mayor jerarquía visual de la pantalla, sin haber tocado nada.
2. **Given** estoy en la pantalla de datos de pago, **When** contraigo la sección del resumen del pedido, **Then** el total a pagar sigue visible: nunca queda escondido dentro de la zona colapsable.
3. **Given** estoy en la pantalla de datos de pago, **When** comparo el total que veo con el del paso de revisión del pedido, **Then** ambos son idénticos, hasta el último dígito.
4. **Given** mi pedido tiene una promoción vigente aplicada, **When** veo el total a pagar, **Then** el monto mostrado es el que ya tiene el descuento aplicado — no el precio de lista.
5. **Given** mi pedido tiene 2 unidades de un producto y 3 de otro, **When** veo el total a pagar, **Then** lo acompaña el conteo "5 productos", y ese mismo número y esa misma palabra son los que aparecen en el título del resumen — nunca dos conteos distintos del mismo pedido.

---

### User Story 2 - Revisar mi pedido sin perder de vista los datos bancarios (Priority: P2)

Como comensal, quiero poder desplegar la lista de productos y adicionales de mi pedido en la misma pantalla de los datos bancarios, para confirmar que mi orden es correcta antes de pagar, sin tener que retroceder en el flujo.

**Why this priority**: resuelve la desconfianza ("¿esto incluye el adicional que pedí?") que hoy obliga a retroceder. Depende de que el total ya esté visible (US1) para que el acordeón pueda estar contraído sin ocultar lo crítico.

**Independent Test**: armar un pedido con adicionales y notas, llegar a la pantalla de datos de pago, expandir "Resumen del pedido" y comprobar que cada línea muestra cantidad, producto, presentación, adicionales, nota y el valor de la línea, sin navegar a ninguna otra pantalla.

**Acceptance Scenarios**:

1. **Given** estoy en la pantalla de datos de pago, **When** busco qué estoy pagando, **Then** encuentro una sección titulada "Resumen del pedido" que indica en su propio título cuántos productos contiene.
2. **Given** mi pedido tiene 3 líneas o menos, **When** llego a la pantalla, **Then** la sección aparece ya expandida, sin que yo tenga que abrirla.
3. **Given** mi pedido tiene más de 3 líneas, **When** llego a la pantalla, **Then** la sección aparece contraída, y al abrirla veo todas mis líneas.
4. **Given** la sección está expandida, **When** reviso una línea que tiene adicionales y una nota, **Then** veo la cantidad, el producto, su presentación, los adicionales elegidos, mi nota y el valor total de esa línea.
5. **Given** mi pedido tiene una promoción vigente, **When** la sección está expandida, **Then** veo una fila "Ahorro" con el descuento, además del Total.
6. **Given** mi pedido no tiene ninguna promoción aplicada, **When** la sección está expandida, **Then** no aparece ninguna fila "Ahorro" — ni con valor cero ni vacía.
7. **Given** estoy en la pantalla de datos de pago, **When** reviso el resumen, **Then** no aparece ninguna fila de impuestos ni de subtotal.
8. **Given** navego con teclado o con lector de pantalla, **When** llego a la sección del resumen, **Then** puedo expandirla y contraerla, y su estado (expandido o contraído) me es anunciado.

---

### User Story 3 - Enviar el comprobante sigue siendo lo fácil de esta pantalla (Priority: P2)

Como comensal, quiero que el resumen del pedido me informe sin estorbarme: después de transferir vuelvo a la aplicación a adjuntar el comprobante, y esa acción debe seguir a mano.

**Why this priority**: es la regla de negocio que condiciona toda la solución y la razón por la que el resumen es colapsable. Un resumen que informe pero que entierre el botón de enviar empeora la pantalla en lugar de mejorarla.

**Independent Test**: en un teléfono, con un pedido de 6 líneas, medir cuánto desplazamiento hace falta desde el inicio de la vista hasta la acción de enviar el pedido, y completar el flujo de adjuntar y enviar de punta a punta.

**Acceptance Scenarios**:

1. **Given** estoy en un teléfono con un pedido de 6 líneas, **When** llego a la pantalla de datos de pago, **Then** alcanzo la zona de adjuntar comprobante y la acción de enviar con a lo sumo un desplazamiento de pantalla.
2. **Given** estoy en la pantalla de datos de pago, **When** adjunto, reemplazo, quito y finalmente envío mi comprobante, **Then** todo el flujo funciona igual que antes de este cambio, sin bloqueos.
3. **Given** ya había subido un comprobante antes y vuelvo a entrar al paso, **When** la pantalla carga, **Then** veo mi comprobante rehidratado y el resumen del pedido, sin que uno interfiera con el otro.
4. **Given** estoy en la pantalla de datos de pago, **When** expando y contraigo el resumen varias veces, **Then** no se modifica mi pedido ni se dispara ninguna operación de envío, carga o cobro.
5. **Given** estoy en un teléfono angosto (320 px de ancho), **When** el resumen está expandido con nombres de producto y notas largas, **Then** el diseño no se rompe ni genera desplazamiento horizontal.

---

### User Story 4 - El mismo resumen en los dos pasos del checkout (Priority: P3)

Como comensal, quiero que el pedido se vea igual en el paso de revisión y en el paso de datos de pago, para no dudar de si estoy mirando dos cosas distintas.

**Why this priority**: es consistencia percibida y, sobre todo, una garantía estructural: dos resúmenes mantenidos por separado terminan divergiendo. Es P3 porque el valor para el comensal ya está entregado por US1 y US2; esta historia protege ese valor en el tiempo.

**Independent Test**: comparar lado a lado el resumen del paso de revisión y el del paso de datos de pago con el mismo pedido, verificando formato de líneas, orden, redacción y cifras.

**Acceptance Scenarios**:

1. **Given** un mismo pedido, **When** comparo el resumen del paso de revisión con el del paso de datos de pago, **Then** las líneas, el orden, el formato de los montos, el Ahorro y el Total son idénticos.
2. **Given** estoy en el paso de revisión del pedido, **When** la pantalla carga, **Then** el resumen se muestra siempre expandido y sin control de colapso — es el contenido principal de esa pantalla.
3. **Given** se corrige o ajusta el formato de una línea del resumen, **When** reviso los dos pasos, **Then** el cambio aparece en ambos, no en uno solo.

---

### Edge Cases

- **Pedido de una sola línea**: la sección aparece expandida; el valor de la línea y el total coinciden. El acordeón no debe verse absurdo por esconder un solo renglón.
- **Frontera exacta del umbral**: con 3 líneas aparece expandida; con 4, contraída.
- **Pedido con muchas líneas (10+)**: contraída al llegar; al expandirla, la lista crece y la pantalla se desplaza — pero el total sigue arriba y la acción de enviar sigue alcanzable al final.
- **Línea con muchos adicionales y nota larga**: el texto se recorta sin romper el ancho de la pantalla ni generar desplazamiento horizontal.
- **Pedido con promoción que cambia mientras el comensal está en la pantalla**: el resumen muestra lo que se cargó al entrar al checkout; esta spec no introduce recarga en vivo del pedido en el paso de datos de pago (ver Assumptions).
- **Recarga del navegador en el paso de datos de pago**: el pedido se recarga al entrar, así que el resumen reaparece con los mismos datos.
- **Pedido vacío**: no se puede llegar a esta pantalla con el pedido vacío — el sistema ya devuelve al comensal al menú. No hay estado vacío que diseñar.
- **Método de pago sin QR ni imagen** (solo datos de texto, p. ej. número de cuenta): la jerarquía de la pantalla se mantiene; el resumen no cambia de posición.
- **Método de pago con QR grande**: el QR ya ocupa un bloque considerable; el resumen contraído es lo que evita que la suma de QR + lista entierre el botón de enviar.
- **Negocio con varios métodos que exigen comprobante** (p. ej. una billetera digital y una transferencia bancaria): los dos llegan a esta pantalla y los dos muestran el total destacado y el resumen, sin importar cómo el negocio haya nombrado cada método (FR-020).
- **Método de pago en efectivo**: no pasa por esta pantalla. Fuera de alcance.

## Requirements *(mandatory)*

### Functional Requirements

**Total a pagar**

- **FR-001**: La pantalla de datos de pago MUST mostrar el total a pagar del pedido en curso como el elemento de mayor jerarquía visual de la pantalla. Verificable así: es el texto de mayor tamaño de la vista y el primer dato que se lee bajo el encabezado, por delante del nombre del método de pago.
- **FR-002**: El total a pagar MUST estar visible sin ninguna interacción previa del comensal y MUST permanecer fuera de cualquier zona colapsable.
- **FR-003**: La pantalla MUST acompañar el total con la cantidad de productos del pedido, expresada en unidades (suma de las cantidades de todas las líneas).
- **FR-004**: El total mostrado MUST ser el total vigente del pedido, ya con el descuento de promoción aplicado cuando exista.

**Resumen del pedido**

- **FR-005**: La pantalla de datos de pago MUST ofrecer una sección titulada "Resumen del pedido" que, expandida, liste todas las líneas del pedido.
- **FR-006**: El título de la sección MUST indicar la cantidad de productos que contiene, con el mismo número y la misma palabra que acompañan al total (FR-003), para que el comensal nunca vea dos conteos distintos del mismo pedido en la misma pantalla.
- **FR-007**: Cada línea listada MUST mostrar: cantidad, nombre del producto, presentación, adicionales elegidos (si tiene), nota del comensal (si tiene) y el valor total de esa línea.
- **FR-008**: La sección MUST aparecer expandida cuando el pedido tiene 3 líneas o menos, y contraída cuando tiene más de 3 líneas. El umbral se mide en líneas del pedido, no en unidades de producto.
- **FR-009**: El comensal MUST poder expandir y contraer la sección cuantas veces quiera.
- **FR-010**: Expandir o contraer la sección MUST NOT modificar el pedido ni disparar ninguna operación de carga, envío o cobro.
- **FR-011**: La sección MUST ser operable con teclado y MUST anunciar su estado (expandido o contraído) a los lectores de pantalla.

**Totales del resumen**

- **FR-012**: El resumen MUST mostrar una fila "Ahorro" con el descuento vigente únicamente cuando el pedido tiene alguna promoción aplicada. Sin descuento, la fila no se muestra — ni en cero ni vacía.
- **FR-013**: El resumen MUST NOT mostrar impuestos ni subtotal. El sistema no calcula ni almacena impuestos en el pedido del comensal; una fila derivada o en cero sería información falsa frente al comensal.
- **FR-014**: Las cifras del resumen (líneas, Ahorro y Total) MUST proceder del mismo cálculo oficial del pedido que ya alimenta el paso de revisión, de modo que ambos pasos no puedan mostrar valores distintos.
- **FR-015**: El total MUST reflejar el cobro de adicionales una sola vez por línea, sin escalar con la cantidad del producto (comportamiento fijado por la spec 089).

**Convivencia con el flujo de comprobante**

- **FR-016**: La pantalla MUST ordenarse verticalmente así: (1) total a pagar destacado, (2) datos de la cuenta bancaria y QR, (3) sección "Resumen del pedido", (4) zona de adjuntar comprobante y acción de enviar el pedido.
- **FR-017**: La zona de adjuntar el comprobante y la acción de enviar el pedido MUST seguir alcanzables con a lo sumo un desplazamiento de pantalla en un teléfono, incluso con pedidos de muchas líneas y con la sección del resumen contraída. Verificable así: con el resumen en su estado inicial, el alto total del contenido de la pantalla no supera **dos veces** el alto del área visible — lo que queda por debajo del primer pantallazo cabe completo en un segundo pantallazo.
- **FR-018**: La incorporación del resumen MUST NOT alterar el comportamiento de adjuntar, reemplazar, quitar ni enviar el comprobante, ni los datos bancarios y el QR que la pantalla ya muestra, ni las acciones de volver y salir sin enviar.
- **FR-019**: El diseño MUST adaptarse a pantallas de teléfono desde 320 px de ancho sin romper el layout ni generar desplazamiento horizontal, y MUST usar los colores y la tipografía que el checkout ya emplea.

**Alcance de aplicación y consistencia entre pasos**

- **FR-020**: El comportamiento definido en esta spec MUST aplicar a la pantalla de datos de pago de **cualquier** método de pago que exija comprobante, no solo al método nombrado "Transferencia Bancaria".
- **FR-021**: El resumen MUST ser el mismo artefacto compartido entre el paso de revisión del pedido y el paso de datos de pago, con idéntico formato de líneas y totales en ambos.
- **FR-022**: En el paso de revisión del pedido, el resumen MUST mostrarse siempre expandido y sin control de colapso: es el contenido principal de esa pantalla. El comportamiento colapsable de FR-008 aplica únicamente al paso de datos de pago.
- **FR-023**: El cambio MUST NOT alterar ninguna pantalla del cajero, del mesero ni del administrador, ni el importe que el cajero ve al confirmar un pago.

### Key Entities

- **Línea del pedido**: una elección del comensal ya consolidada — un producto en una presentación, con sus adicionales, su nota y su cantidad. Su valor total es lo que el resumen muestra a la derecha de cada renglón.
- **Total del pedido**: el monto único que el comensal debe transferir. Es el dato de mayor jerarquía de la pantalla y la única cifra que el comensal necesita para operar en su banco.
- **Ahorro por promoción**: diferencia entre lo que el pedido costaría sin promociones y el total vigente. Existe solo cuando alguna promoción aplica; no es un campo permanente del pedido.
- **Conteo de productos**: unidades totales del pedido (suma de cantidades). Es un dato derivado, usado para orientar al comensal sobre el tamaño de su pedido antes de abrir el resumen.

## Reglas de Negocio

- **RN-001**: El total que ve el comensal en la pantalla de datos de pago es el mismo que el cálculo oficial del pedido, incluido el comportamiento independiente de los adicionales fijado por la spec 089. Esta spec no introduce ningún cálculo nuevo ni ninguna cifra derivada en pantalla.
- **RN-002**: Informar nunca puede costarle al comensal la acción que vino a hacer. El resumen se subordina al flujo de comprobante: si una decisión de diseño mejora la lectura del pedido pero entierra la acción de enviar en móvil, prevalece la acción de enviar.
- **RN-003**: El sistema no muestra cifras que no calcula. Mientras el pedido del comensal no tenga impuestos modelados, ninguna pantalla muestra una fila de impuestos, ni en cero ni como aproximación.
- **RN-004**: El comensal nunca debe ver dos cifras distintas para lo mismo. Un conteo de productos, un total, una sola fuente de cálculo para los dos pasos del checkout.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En el 100% de las entradas a la pantalla de datos de pago, el comensal ve el total a pagar sin desplazarse ni interactuar con nada.
- **SC-002**: Un comensal puede confirmar qué está pagando (productos y adicionales) sin salir de la pantalla de datos de pago: **cero** navegaciones a pasos anteriores necesarias.
- **SC-003**: En un teléfono de 360 × 640 px con un pedido de 6 líneas y el resumen contraído, el alto total del contenido de la pantalla de datos de pago no supera **1.280 px** — dos veces el alto visible, que es lo que significa alcanzar la acción de enviar con **a lo sumo un** desplazamiento desde el inicio de la vista.
- **SC-004**: El total mostrado en la pantalla de datos de pago coincide exactamente con el del paso de revisión y con el importe que el cajero ve al confirmar el pago, en el 100% de los pedidos verificados, con y sin promoción aplicada.
- **SC-005**: El flujo de adjuntar y enviar el comprobante se completa de punta a punta sin bloqueos en el 100% de los recorridos de verificación, incluyendo reemplazar y quitar el comprobante antes de enviar.
- **SC-006**: La sección del resumen se expande y se contrae completamente con teclado, y su estado es anunciado por un lector de pantalla, en el 100% de los recorridos de accesibilidad.
- **SC-007**: Ninguna pantalla del checkout muestra una fila de impuestos ni de subtotal mientras el sistema no los calcule.
- **SC-008**: El resumen del paso de revisión y el del paso de datos de pago muestran el mismo formato de líneas y totales, verificado con el mismo pedido en ambos pasos.
- **SC-009**: En pantallas desde 320 px de ancho, con el resumen expandido y nombres y notas largas, no se produce desplazamiento horizontal ni se recorta contenido de forma ilegible.

## Trazabilidad de los criterios de aceptación del brief

| # del brief | Criterio original | Cubierto por |
|-------------|-------------------|--------------|
| 1 | Se muestra claramente el "Total a pagar", fuera de zona colapsable | FR-001, FR-002, SC-001 · US1 esc. 1-2 |
| 2 | Existe un acordeón que lista productos, adicionales y totales de línea | FR-005, FR-007, SC-002 · US2 esc. 1, 4 |
| 3 | Acordeón abierto con ≤3 líneas, cerrado con más de 3 | FR-008 · US2 esc. 2-3 |
| 4 | Fila "Ahorro" solo con descuento vigente; nunca impuestos | FR-012, FR-013, SC-007 · US2 esc. 5-7 |
| 5 | El mismo resumen en el paso 1 y el paso 3, con idéntico total | FR-014, FR-021, FR-022, SC-004, SC-008 · US4 |
| 6 | Diseño adaptado a móvil, integrado con colores y tipografía existentes | FR-019, SC-009 · US3 esc. 5 |
| 7 | El flujo de comprobante sigue funcionando y el botón sigue alcanzable | FR-017, FR-018, SC-003, SC-005 · US3 esc. 1-3 |
| 8 | Acordeón operable por teclado y anunciado por lector de pantalla | FR-011, SC-006 · US2 esc. 8 |

## Impacto sobre Funcionalidades Existentes

- **Paso de datos de pago del checkout del comensal**: cambia. Gana el bloque de total destacado y la sección del resumen; conserva sin cambios los datos bancarios, el QR, sus acciones de copiar y descargar, la zona de comprobante y las acciones de volver y salir.
- **Paso de revisión del pedido del checkout**: cambia. Su resumen pasa a ser el artefacto compartido descrito en FR-021, siempre expandido y sin control de colapso (FR-022). El cambio es estructural salvo en **un** punto visible, deliberado y autorizado: cuando el pedido tiene una promoción vigente, este paso ahora muestra la fila "Ahorro" que antes no mostraba, porque FR-012, FR-021 y US4 esc. 1 exigen que el Ahorro sea idéntico en ambos pasos y RN-004 prohíbe que el comensal vea el mismo pedido descrito de dos formas distintas en dos pantallas consecutivas. Sin promoción vigente —el caso por defecto— el paso se ve exactamente igual que hoy. **Fuera de esa fila, cualquier otra diferencia visible en este paso respecto a hoy es una regresión, no una mejora.**

  > Decisión de negocio tomada el 2026-10-02 por el dueño de la spec y registrada como **A-103** en `specs/000-reconocimiento/registro-de-anomalias.md` (Principios II y XI). El diseño conserva la válvula de escape: el componente compartido expone el input `showSavings`, así que revertirla cuesta un booleano. Ver [research.md](./research.md) D6.
- **Paso de elección de método de pago y paso de confirmación**: sin cambios.
- **Pantallas del cajero, del mesero y del administrador**: sin cambios (FR-023). En particular, ninguna de las superficies de cobro que validan el total se ve afectada, porque esta spec no toca ningún cálculo.
- **Flujo de pago en efectivo**: sin cambios — no pasa por la pantalla de datos de pago.

## Impacto sobre Datos Existentes

Ninguno. Esta spec es exclusivamente de presentación:

- No crea, modifica ni elimina entidades, campos ni relaciones.
- No requiere migración de datos ni estrategia de rollback de datos.
- No introduce cálculos nuevos: consume el total y los valores de línea que el pedido ya expone.
- No altera, recalcula ni reexpresa ninguna factura ya emitida (Principio VII).

## Decisiones de Compatibilidad

- **Pedidos en curso al momento del despliegue**: un comensal que esté en el paso de datos de pago cuando se despliegue el cambio verá la pantalla nueva al recargar, con su comprobante rehidratado y su pedido intacto. No hay estado de sesión incompatible.
- **Métodos de pago ya configurados**: todos siguen funcionando sin reconfiguración, tengan o no QR, con uno o varios campos de datos.
- **Pedidos sin promoción**: se comportan como el caso normal; la ausencia de la fila "Ahorro" es el estado por defecto, no una excepción.
- **Sin cambios de contrato con el backend**: la pantalla no pide ningún dato nuevo. Si en el futuro el pedido llegara a exponer impuestos, esta spec no bloquea añadir esa fila — solo prohíbe inventarla mientras no exista.

## Assumptions

- **El formato de moneda es el que el checkout ya usa**. Esta spec no introduce un formato, símbolo ni redondeo nuevo.
- **El umbral de 3 líneas es una decisión de producto, no un cálculo de altura**. Se eligió porque 3 renglones de línea más el bloque de total y el bloque bancario todavía dejan la acción de enviar a un desplazamiento en un teléfono típico. Si la verificación en dispositivo muestra que el umbral real debería ser otro, ajustarlo es una decisión de producto que se registra, no un detalle de implementación.
- **El resumen refleja el pedido tal como se cargó al entrar al checkout**. Esta spec no introduce recarga en vivo del pedido ni actualización automática del total mientras el comensal está en la pantalla de datos de pago; ese comportamiento es el que ya existe y no se modifica.
- **Solo el comensal usa esta pantalla**. No se contempla que un cajero o mesero abra el paso de datos de pago del comensal.
- **No se añaden dependencias nuevas** (Principio IX). La sección colapsable se construye con lo que el proyecto ya tiene.
- **La cantidad de productos se expresa en unidades** (suma de cantidades), mientras el umbral de colapso se mide en líneas. Son dos medidas con propósitos distintos, y solo la primera se le muestra al comensal (FR-003, FR-006, FR-008).

## Fuera de Alcance

- **Modelar impuestos** en el pedido del comensal, en cualquier nivel (línea, pedido, método de pago). Es otra funcionalidad, con su propio spec, su propio modelo de datos y su propia decisión de negocio.
- **Mostrar un subtotal** o cualquier otra cifra intermedia derivada en el front.
- **Rediseñar el bloque de datos bancarios o el QR**, sus acciones de copiar y descargar, o el flujo de subida del comprobante.
- **Cambiar el indicador de pasos** del checkout o la cantidad de pasos del flujo.
- **Añadir edición del pedido** desde la pantalla de datos de pago (cambiar cantidades, quitar líneas, editar adicionales). El resumen es de lectura; para editar, el comensal vuelve atrás como hoy.
- **Recarga en vivo del pedido** o del total mientras el comensal está en la pantalla.
- **Mostrar el resumen en el paso de confirmación** o en cualquier otra pantalla no nombrada en esta spec.

## Decisiones de Diseño Ya Confirmadas *(entrada para la fase de plan, no requisitos)*

Esta sección registra las decisiones técnicas que el usuario ya tomó y confirmó el 2026-10-02, antes de redactar esta spec. Se recogen aquí por trazabilidad (Principio XII) y para que la fase de plan no las vuelva a deliberar. **No son requisitos**: los requisitos verificables son los FR de arriba, que están expresados en términos de comportamiento observable y seguirían siendo válidos si estas decisiones cambiaran.

- **Mecanismo del acordeón**: elemento colapsable nativo del navegador (`<details>` / `<summary>`), sin librería y sin estado propio en el componente. Razón: FR-011 (teclado y lector de pantalla) sale gratis con la semántica nativa, y FR-010 (expandir no dispara nada) es imposible de violar si no hay estado que manejar. El proyecto no tiene hoy ningún componente de acordeón previo que reutilizar.
- **Estado inicial**: se controla con el atributo de apertura del elemento nativo, enlazado al conteo de líneas del pedido (FR-008).
- **Artefacto compartido**: extraer el resumen que hoy vive en el paso de revisión (`review-step.component.ts`) a un componente presentacional propio, consumido por ese paso y por el de datos de pago (`transfer-details-step.component.ts`). El modo colapsable es una entrada de ese componente, apagada en el paso de revisión (FR-022). Se descartó explícitamente duplicar el markup.
- **Fuente de datos**: el servicio del pedido del comensal (`DiningCartService`) y el formato de moneda del proyecto (`MoneyPipe`), sin llamadas nuevas al backend.
- **Pendiente para el plan**: el set de iconos del proyecto no tiene un chevron. El plan decide entre añadir un icono `chevron-down` al set o rotar vía CSS el indicador nativo. Cualquiera de las dos cumple los FR; es una decisión de implementación.
- **Alternativas descartadas** (ver Clarifications): lista estática completa siempre visible, y lista truncada con enlace "ver N más".
