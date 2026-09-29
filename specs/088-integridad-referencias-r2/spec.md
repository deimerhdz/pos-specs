# Feature Specification: Integridad de las referencias a archivos en Cloudflare R2

**Feature Branch**: `088-integridad-referencias-r2`

**Created**: 2026-09-29

**Status**: Draft

**Input**: User description: "Integridad de las referencias a archivos en Cloudflare R2 (imágenes de producto, logo del negocio, QR de método de pago y comprobantes del comensal). Hoy los archivos se suben directo a R2 con URL firmada y en la base de datos solo se guarda la key (spec 080). El sistema puede quedar con un registro que apunta a un archivo que ya no existe, o que nunca existió, y puede borrar un archivo que sigue en uso. El objetivo es que NUNCA quede una referencia rota por causa del propio sistema, y que ningún negocio pueda tocar los archivos de otro."

## Aclaraciones

### Sesión 2026-09-29

- P: ¿Cómo distingue el sistema una imagen realmente nueva de una imagen reenviada por un formulario desactualizado? → R: Concurrencia optimista. Junto con la imagen nueva, el cliente envía la referencia base: la imagen que el formulario mostraba al abrirse. Si esa base no coincide con la imagen vigente, el formulario está desactualizado y la imagen enviada se ignora. El formulario, además, deja de reenviar la imagen cuando el usuario no la tocó (FR-001).
- P: Si un formulario desactualizado reenvía una imagen cuyo archivo ya no existe, ¿es error o se ignora? → R: Se ignora en silencio: se guardan los demás campos y se conserva la imagen vigente. El error solo aplica a la edición legítima (base coincidente) con un archivo inexistente. La respuesta no avisa que la imagen la cambió otra persona (FR-002).
- P: ¿El mecanismo de formulario desactualizado aplica solo a producto? → R: A los tres: imagen de producto, logo del negocio e imagen (QR) de método de pago (FR-001).
- P: Si el almacenamiento no responde al verificar que el archivo existe, ¿qué pasa? → R: Se falla cerrado: error reintentable (503), sin guardar y sin borrar nada (FR-003).
- P: ¿Qué filas cuentan como "otra referencia" antes de borrar? → R: La spec lista explícitamente las tablas y columnas revisadas (FR-006) y el plan las confirma contra el código.
- P: ¿La revisión de "otra fila lo usa" cruza negocios? → R: Se revisa el esquema del negocio más las tablas globales que puedan usar ese prefijo (FR-006).
- P: ¿Qué pasa con las keys históricas que no cumplen la convención `{esquema}/{carpeta}/`? → R: La validación estricta se aplica solo a keys nuevas. Una key histórica fuera de convención se conserva y se lee, pero nunca se borra al reemplazarla (queda huérfana, lo que es seguro) y el reporte la marca como "fuera de convención" (FR-004).
- P: ¿Cómo entra el comprobante del comensal durante la transición? → R: Se aceptan ambos formatos: la key, o la URL absoluta del bucket gestionado (dominio público anterior o dominio de assets), de la que el sistema extrae la key y valida igual (FR-008).
- P: El comensal no tiene sesión: ¿qué se garantiza sobre un comprobante? → R: Solo prefijo del negocio, carpeta `comprobantes` y existencia del archivo. No se garantiza quién lo subió (FR-008).
- P: ¿Qué se hace con las filas de comprobantes cuya URL absoluta es de otro origen o apunta a un archivo inexistente? → R: Se dejan intactas, se reportan y no hacen fallar la migración (FR-008).
- P: ¿El reporte de reconciliación necesita endpoint? → R: No. Basta un script de línea de comandos, de solo lectura (FR-009).
- P: ¿Cómo se evitan falsos positivos con archivos recién subidos que aún no se guardaron? → R: Ventana de gracia configurable, por defecto 24 horas: los archivos más recientes no se listan como huérfanos (FR-009).
- P: Cuando llega una imagen que se ignoraría por formulario desactualizado o por falta de imagen base, ¿se rechaza primero con 422 si la key tiene forma inválida o es de otro negocio, o se ignora sin validar? → R: Se valida siempre primero la forma y el negocio dueño (FR-004): key inválida o ajena da 422 aunque el formulario esté desactualizado o no traiga base. Solo después se compara la base y, únicamente si la edición es legítima, se verifica la existencia (FR-002).
- P: Si una petición trae una key válida distinta de la vigente pero no trae la imagen base (cliente antiguo o manual), ¿se ignora la imagen o se trata como cambio legítimo? → R: Se ignora en silencio (tras validar FR-004). Un cambio legítimo exige enviar la base; las pruebas por API de las historias 2 y 3 deben incluirla (FR-001).
- P: Al crear un producto o método de pago nuevo con imagen, ¿se exige la base como nula o la imagen se acepta siempre? → R: En una creación no aplica la comparación de base: la imagen se valida (FR-004), se verifica su existencia (FR-003) y se guarda (FR-001).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un formulario desactualizado no deshace ni borra la imagen vigente (Priority: P1)

Dos administradores (o dos pestañas) abren el mismo producto. Uno de ellos cambia la imagen y guarda. El segundo, con su formulario ya desactualizado, cambia solo el precio y guarda. Hoy, la imagen que el segundo formulario reenvía (ya borrada) se interpreta como un cambio: la base vuelve a apuntar a un archivo inexistente y, además, se borra la imagen nueva que sí estaba vigente. Con esta funcionalidad, el precio del segundo administrador se guarda y la imagen vigente se conserva intacta.

**Why this priority**: Es la pérdida de datos más grave y la más probable en la operación real (varias personas administrando el mismo catálogo). Un archivo borrado por error no se recupera, y el producto queda con la imagen rota en el menú del comensal.

**Independent Test**: Abrir el mismo producto en dos pestañas; en la primera subir una imagen nueva y guardar; en la segunda (formulario viejo) cambiar solo el nombre y guardar. Verificar que el nombre se guardó, que la imagen nueva sigue visible en la pantalla del producto y en el menú del comensal, y que el archivo de la imagen nueva sigue abriéndose. Repetir con el logo del negocio y con el QR de un método de pago.

**Acceptance Scenarios**:

1. **Given** el producto "Cono doble" con imagen X, dos administradores con el formulario abierto y el primero ya reemplazó X por Y y guardó (X se borró), **When** el segundo cambia solo el precio y guarda con su formulario viejo (que reenvía X), **Then** el precio se guarda, el producto conserva Y, Y sigue existiendo y no se borra ningún archivo.
2. **Given** un formulario desactualizado que reenvía una imagen cuyo archivo aún existe (el borrado de X falló y quedó huérfano), **When** guarda, **Then** el producto conserva la imagen vigente igualmente, porque la base del formulario no coincide con la vigente; no se restaura X.
3. **Given** un formulario al día (su base coincide con la imagen vigente) en el que el administrador sube una imagen nueva, **When** guarda, **Then** la imagen cambia normalmente y la anterior se limpia (User Story 4).
4. **Given** la creación de un producto nuevo con una imagen recién subida y sin imagen base, **When** guarda, **Then** el producto se crea con esa imagen.
5. **Given** un formulario al día en el que el administrador no toca la imagen, **When** guarda otros cambios, **Then** la imagen no cambia y no se borra ningún archivo.
6. **Given** un formulario desactualizado del logo del negocio o del QR de un método de pago, **When** guarda, **Then** aplican las mismas reglas: se conserva el valor vigente y se guardan los demás campos.
7. **Given** un formulario desactualizado, **When** guarda, **Then** la respuesta es exitosa y no muestra ningún aviso ni error relacionado con la imagen.

---

### User Story 2 - No se puede guardar una imagen que no existe en el almacenamiento (Priority: P1)

Un administrador guarda un producto, el logo o el QR de un método de pago. Antes de aceptar la referencia, el sistema comprueba que el archivo realmente existe en el almacenamiento. Si el archivo no existe (nunca se subió, la subida falló o alguien lo manipuló con un cliente manual), el guardado se rechaza con un error claro y el registro queda exactamente como estaba.

**Why this priority**: Sin esta comprobación cualquier subida fallida o solicitud manipulada deja una referencia rota que el comensal ve como imagen rota en el menú. Es la barrera de entrada de todas las demás garantías.

**Independent Test**: Por API (o con un cliente manual), intentar guardar un producto enviando la imagen base vigente y una key con la forma correcta pero cuyo archivo no se subió. Verificar que la respuesta es un error 422 con mensaje claro, que la imagen anterior sigue mostrándose y que ningún otro campo cambió. Repetir con el logo y el QR.

**Acceptance Scenarios**:

1. **Given** una key con la forma correcta cuyo archivo no existe en el almacenamiento, **When** la edición es legítima (la petición trae la base y coincide con la vigente) y se intenta guardar, **Then** el sistema responde 422 con un mensaje claro y no modifica nada.
2. **Given** el almacenamiento no responde (tiempo agotado o error de red) al verificar, **When** se intenta guardar una imagen nueva, **Then** el sistema responde con un error reintentable (503), no guarda nada y no borra nada.
3. **Given** una imagen recién subida correctamente, **When** se guarda, **Then** el guardado se completa y la imagen se ve.
4. **Given** un valor vacío (nulo o cadena vacía) en el campo de imagen de producto o logo, **When** se guarda, **Then** se interpreta como "no tocar la imagen" (comportamiento actual) y no se verifica nada.
5. **Given** una imagen cuyo valor es una URL de otro origen (no gestionada), **When** se guarda, **Then** el valor se conserva tal cual y nunca se verifica contra el almacenamiento (FR-005).

---

### User Story 3 - Un negocio no puede referenciar ni borrar los archivos de otro (Priority: P1)

Cada negocio tiene su propio espacio dentro del almacenamiento, identificado por el prefijo de su nombre de esquema. Un administrador solo puede guardar referencias que apunten a su propio espacio y a la carpeta que corresponde al campo (productos, logo, métodos de pago, comprobantes). Si envía la key de otro negocio, o de otra carpeta, o una key con segmentos que intenten salirse de su espacio, el guardado se rechaza. Así también queda garantizado que, al reemplazar una imagen, el sistema jamás borra un archivo que no le pertenece.

**Why this priority**: Es un problema de aislamiento entre negocios (multi-tenant): hoy un administrador puede apuntar a un archivo ajeno y, al reemplazarlo, provocar que el sistema borre el archivo del otro negocio. Es una vulnerabilidad, no una comodidad.

**Independent Test**: Con el negocio `acme`, intentar guardar como imagen de producto la key `globex/products/zzz.jpg`. Verificar que la respuesta es 422, que el registro no cambió y que el archivo de `globex` sigue abriéndose en su URL.

**Acceptance Scenarios**:

1. **Given** el negocio `acme`, **When** intenta guardar la key `globex/products/zzz.jpg`, **Then** responde 422 y no se toca ningún archivo de `globex`, con o sin imagen base y aunque el formulario esté desactualizado.
2. **Given** el negocio `acme`, **When** intenta guardar como imagen de producto una key de su propio espacio pero de otra carpeta (por ejemplo `acme/logo/x.png`), **Then** responde 422.
3. **Given** una key con `..`, `//`, `\`, caracteres de control o que no encaja en el patrón `{esquema}/{carpeta}/{nombre}`, **When** se intenta guardar, **Then** responde 422; el sistema no normaliza ni "corrige" la key en silencio.
4. **Given** una key con mayúsculas distintas a las del prefijo del negocio (por ejemplo `ACME/products/x.jpg`), **When** se intenta guardar, **Then** responde 422 (se compara la key tal cual).
5. **Given** una fila histórica cuya key no cumple la convención, **When** se reemplaza su imagen, **Then** la fila cambia normalmente pero el archivo antiguo no se borra (queda huérfano) y el reporte de reconciliación lo marca "fuera de convención".
6. **Given** una fila cuya imagen es una URL de otro origen (no gestionada), **When** se reemplaza, **Then** nunca se intenta borrar el recurso antiguo.

---

### User Story 4 - El archivo anterior solo se borra si nadie más lo usa (Priority: P2)

Cuando un administrador reemplaza una imagen, el sistema limpia el archivo anterior para no acumular basura, pero solo si ningún otro registro (producto, logo del negocio, método de pago u otra tabla que guarde imágenes) sigue apuntando a él. Si otro registro lo usa, el archivo se conserva. El borrado sigue ocurriendo después de confirmar el cambio y sin bloquearlo: si falla, queda un archivo huérfano, nunca una referencia rota.

**Why this priority**: Evita que duplicar o reutilizar una imagen entre dos registros rompa uno de ellos cuando el otro se actualiza. Es una salvaguarda secundaria respecto a las tres anteriores (que evitan el daño más probable), pero necesaria para que "nunca se borra un archivo en uso" sea verdad.

**Independent Test**: Dos productos referencian la misma key K. Reemplazar la imagen del primero: el registro cambia y K sigue existiendo. Reemplazar después la del segundo: K se borra. Verificar también que con un reemplazo simple (un solo registro usa K) K se borra.

**Acceptance Scenarios**:

1. **Given** dos filas que referencian la key K, **When** se reemplaza la imagen de una, **Then** el registro cambia y K no se borra.
2. **Given** el mismo escenario, **When** después se reemplaza también la segunda, **Then** K se borra.
3. **Given** una imagen usada por un único registro, **When** se reemplaza, **Then** la imagen anterior deja de existir en el almacenamiento (flujo normal de reemplazo, sin regresión).
4. **Given** un fallo al borrar el archivo anterior, **When** el reemplazo ya se confirmó, **Then** el cambio se conserva, el usuario no ve error, y el archivo queda huérfano.
5. **Given** dos actualizaciones concurrentes del mismo producto, **When** ambas terminan, **Then** prevalece la última en escribir, y cada actualización solo borra el archivo que ella misma reemplazó, después de confirmar y verificando que nadie más lo usa.

---

### User Story 5 - El cajero solo ve comprobantes válidos del propio negocio (Priority: P2)

El comensal que paga por transferencia sube un comprobante. El sistema solo acepta como comprobante un archivo que exista, que esté en la carpeta de comprobantes y que pertenezca al espacio del negocio donde se está pagando. El cajero, en la pantalla de "Pagos por confirmar", ve entonces únicamente comprobantes reales del propio negocio. Como el comensal no tiene sesión, no se puede garantizar quién subió el archivo, solo que está en el espacio correcto.

**Why this priority**: Hoy el comprobante es texto libre: puede ser una URL cualquiera, de otro negocio o inexistente, y se guarda como URL absoluta en vez de key. Es el único campo de archivos que aún no sigue el esquema de la spec 080.

**Independent Test**: Como comensal, adjuntar un comprobante real subido a la carpeta del negocio: aparece en "Pagos por confirmar" del cajero. Intentar adjuntar una key inventada, una key de otro negocio y una URL ajena: las tres se rechazan.

**Acceptance Scenarios**:

1. **Given** un comprobante recién subido por el comensal, **When** lo adjunta al intento de pago, **Then** se acepta, en base de datos queda solo la key y el cajero lo ve en "Pagos por confirmar".
2. **Given** una key inventada (el archivo no existe) o de otro negocio o de otra carpeta, **When** se intenta adjuntar, **Then** se rechaza con 422 y el intento de pago no cambia.
3. **Given** el cliente envía la URL absoluta del bucket gestionado (dominio público anterior o dominio de assets), **When** se adjunta, **Then** el sistema extrae la key, la valida igual que una key directa y guarda solo la key.
4. **Given** una URL de otro origen, **When** se intenta adjuntar, **Then** se rechaza (un comprobante nuevo siempre debe ser un archivo gestionado).
5. **Given** un intento de pago que ya tiene un comprobante adjunto, **When** se intenta adjuntar otro, **Then** se mantiene el 409 existente.
6. **Given** filas antiguas con URL absoluta del dominio público anterior o del dominio de assets, **When** el cajero las consulta, **Then** siguen viéndose (tolerancia de lectura).
7. **Given** una migración de comprobantes en modo simulación, **When** se ejecuta, **Then** informa qué filas reescribiría sin modificar nada.
8. **Given** una migración aplicada, **When** se ejecuta de nuevo, **Then** no cambia nada (idempotente); las filas con URL de otro origen o con archivo inexistente quedan intactas y se reportan sin hacer fallar el proceso.

---

### User Story 6 - Reporte de reconciliación para el operador (Priority: P3)

La persona que opera la plataforma ejecuta un script de solo lectura que compara lo que dice la base de datos con lo que hay realmente en el almacenamiento. El reporte lista, agrupado por negocio: (a) referencias cuyo archivo no existe (por ejemplo por un borrado manual en el panel de Cloudflare), (b) archivos sin ninguna referencia (huérfanos) y (c) referencias fuera de la convención de carpetas. No borra ni modifica nada, ni en la base de datos ni en el almacenamiento.

**Why this priority**: Da visibilidad para detectar y corregir manualmente lo que las otras garantías no pueden prevenir (borrados manuales en Cloudflare, historial anterior a esta spec). No bloquea a ningún usuario final.

**Independent Test**: En un entorno de prueba, borrar a mano un objeto referenciado por un producto y subir otro sin referencia (de más de 24 horas). Ejecutar el script: el primero aparece como referencia rota y el segundo como huérfano. Verificar que después de ejecutarlo la base de datos y el almacenamiento están idénticos.

**Acceptance Scenarios**:

1. **Given** una referencia cuyo archivo se borró a mano, **When** se ejecuta el reporte, **Then** aparece listada bajo su negocio como "referencia sin archivo".
2. **Given** un archivo del almacenamiento sin referencia y con más antigüedad que la ventana de gracia, **When** se ejecuta el reporte, **Then** aparece listado como "archivo sin referencia".
3. **Given** un archivo sin referencia más reciente que la ventana de gracia (24 horas por defecto, configurable), **When** se ejecuta el reporte, **Then** no aparece listado como huérfano.
4. **Given** una key histórica fuera de convención, **When** se ejecuta el reporte, **Then** aparece marcada "fuera de convención".
5. **Given** cualquier ejecución, **When** termina, **Then** la base de datos y el almacenamiento no cambiaron y el reporte se pudo leer en pantalla y, opcionalmente, escribir a un archivo cuya ruta se indica en la salida.
6. **Given** el argumento para acotar a un solo negocio, **When** se ejecuta, **Then** el reporte cubre solo ese negocio.

---

### Edge Cases

- **Filas antiguas con URL absoluta** del dominio público anterior o del dominio de assets: se siguen leyendo bien (tolerancia de lectura, spec 080).
- **Imagen con URL de otro origen** (no gestionada): se conserva intacta, nunca se verifica ni se borra (FR-005).
- **Vaciar la imagen** (nulo o cadena vacía): en producto y logo se interpreta como "no tocar", igual que hoy; no existe manera de quitar esas imágenes. **Excepción, QR de método de pago:** como `payment_info` se reemplaza completo, una edición legítima (base coincidente) que omite el QR lo elimina, igual que hoy; el archivo no se borra y queda huérfano. Un formulario desactualizado nunca elimina el QR vigente.
- **El almacenamiento no responde** al verificar existencia: 503 reintentable, sin guardar ni borrar nada.
- **Dos actualizaciones concurrentes** del mismo producto: prevalece la última en escribir; cada una borra solo lo que ella misma reemplazó.
- **Key con mayúsculas distintas o con segmentos extra** (`../`): 422, sin normalizar.
- **Comprobante ya adjunto** a un intento de pago: se mantiene el 409 existente.
- **Keys históricas fuera de convención**: se conservan, se leen y nunca se borran.
- **Formulario desactualizado cuyo archivo reenviado aún existe** (el borrado anterior falló): igual se conserva la imagen vigente, porque la base del formulario no coincide.
- **Formulario que no envía la referencia base** (cliente antiguo): se trata como formulario sin imagen nueva declarada; no se considera cambio de imagen salvo que traiga la base y la imagen (ver Assumptions). Aun así, si trae una key de forma inválida o de otro negocio, responde 422 (FR-002, orden de evaluación).
- **Archivo recién subido y todavía no guardado**: no cuenta como huérfano en el reporte durante la ventana de gracia.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Actualizar un producto, el logo del negocio o el QR de un método de pago sin haber cambiado la imagen NUNCA cuenta como cambio de imagen, aunque el formulario esté desactualizado. El sistema MUST detectar el cambio comparando la imagen base que el cliente vio al abrir el formulario contra la imagen vigente: si no coinciden, la imagen enviada se ignora y se conserva la vigente. El formulario MUST dejar de reenviar la imagen cuando el usuario no la modificó. Aplica a los tres campos: imagen de producto, logo del negocio e imagen (QR) de método de pago. La comparación de base solo aplica a ediciones: en la creación de un producto o de un método de pago no existe imagen vigente ni formulario desactualizado, por lo que la imagen enviada no requiere base y se valida (FR-004), se verifica (FR-003) y se guarda. En el QR de método de pago, que viaja dentro de `payment_info` (reemplazo completo), el formulario sigue enviando el valor vigente; la protección contra formularios desactualizados la da `payment_info_base`, no la omisión del valor.
- **FR-002**: Si el formulario está desactualizado y reenvía una imagen que ya no es la vigente sin haberla subido de nuevo, el sistema MUST conservar la imagen vigente, guardar el resto de los campos y NO borrar ningún archivo. Esto MUST ocurrir en silencio: la respuesta no devuelve error ni aviso de que otra persona cambió la imagen. Orden de evaluación de una imagen enviada: (1) forma y negocio dueño (FR-004): si la key es inválida o de otro negocio o carpeta → 422, siempre, aunque el formulario esté desactualizado o no traiga base; (2) comparación de la base contra la vigente: si no coincide (o no se envió base y la key difiere de la vigente) → se ignora la imagen, sin error y sin verificar existencia; (3) solo si la edición es legítima, verificación de existencia (FR-003). Resumen frente a FR-003: (a) key válida, distinta de la vigente + base que no coincide + archivo inexistente → se ignora la imagen, sin error; (b) key válida + base que coincide con la vigente (edición legítima) + archivo inexistente → 422.
- **FR-003**: Antes de guardar cualquier key de imagen (producto, logo, QR), el sistema MUST comprobar que el archivo existe en el almacenamiento. Si no existe, MUST responder 422 con un mensaje claro y no modificar nada. Si el almacenamiento no responde al verificar (tiempo agotado o error de red), el sistema MUST fallar cerrado: responder 503 con un mensaje reintentable, sin guardar ni borrar nada.
- **FR-004**: Toda key nueva MUST empezar por `{esquema_del_negocio}/` y usar la carpeta correspondiente al campo: `products` (producto), `logo` (logo), `payment-methods` (QR de método de pago) o `comprobantes` (comprobante). Si no cumple, MUST responder 422. Esto aplica tanto al guardar como al decidir qué archivo borrar. La key se compara tal cual (sensible a mayúsculas); MUST rechazarse toda key con `..`, `//`, `\`, caracteres de control o que no coincida con el patrón `{esquema}/{carpeta}/{nombre}`, sin normalizar en silencio. La validación estricta aplica solo a keys nuevas: una key histórica fuera de convención se conserva y se lee bien, pero MUST NOT borrarse al reemplazarla, y el reporte de reconciliación la marca "fuera de convención".
- **FR-005**: Las referencias a URLs de otro origen (no gestionadas) MUST conservarse intactas y el sistema MUST NOT intentar verificarlas ni borrarlas (mismo criterio que la spec 080, FR-010).
- **FR-006**: Antes de borrar el archivo anterior, el sistema MUST comprobar que ninguna otra fila lo referencia. Si otra fila lo usa, MUST NOT borrarlo. La revisión cubre el esquema del negocio más las tablas globales que puedan usar ese prefijo. Tablas y columnas que se revisan (a confirmar contra el código en el plan):
  - Producto: la imagen del producto (esquema del negocio).
  - Negocio: el logo del negocio (esquema global compartido).
  - Método de pago del negocio: el valor de imagen dentro de los datos de integración del método de pago (esquema del negocio). El catálogo global de métodos de pago solo define qué campos son de tipo imagen; no almacena archivos, por lo que se revisa pero se espera que no aporte referencias.
  - Intento de pago: el comprobante adjunto (esquema del negocio), como referencia adicional cuando un mismo archivo pudiera coincidir.
  - Cualquier otra tabla con imágenes (por ejemplo presentaciones o promociones de la spec 083): el análisis preliminar del código no encontró columnas de imagen en ellas; el plan debe confirmarlo y, si aparece alguna, incluirla.
- **FR-007**: El borrado del archivo anterior MUST seguir ocurriendo después de confirmar el cambio y sin bloquearlo (mejor esfuerzo): un fallo al borrar deja un archivo huérfano, nunca una referencia rota. Con actualizaciones concurrentes del mismo producto prevalece la última en escribir; cada actualización MUST borrar solo la key que ella misma reemplazó, después de confirmar y comprobando FR-006.
- **FR-008**: Comprobantes. El sistema MUST aceptar como comprobante únicamente una key que pertenezca al negocio, esté en `comprobantes/` y exista en el almacenamiento. La entrada MUST aceptar ambos formatos durante la transición: la key, o la URL absoluta del bucket gestionado (dominio público anterior o dominio de assets), de la cual se extrae la key y se valida igual. Una URL de otro origen se rechaza para comprobantes nuevos. Como el comensal no tiene sesión, el requisito solo garantiza prefijo del negocio + carpeta correcta + existencia; no garantiza quién subió el archivo. En base de datos MUST guardarse la key, no la URL absoluta; la URL se arma al responder, como en la spec 080. Las filas existentes con URL absoluta MUST seguir leyéndose bien (tolerancia de lectura) y MUST migrarse con un script idempotente, con modo de simulación por defecto y aplicación explícita. Las URLs absolutas de otro origen y las que apuntan a un archivo inexistente MUST quedar intactas, reportarse y no hacer fallar el script. Se mantiene el 409 cuando el intento ya tiene comprobante.
- **FR-009**: El sistema MUST ofrecer un reporte de reconciliación de solo lectura, ejecutable como script (sin endpoint), que liste, agrupado por negocio: (a) referencias cuyo archivo no existe en el almacenamiento, (b) archivos del almacenamiento sin ninguna referencia, y (c) referencias fuera de la convención de carpetas. MUST imprimir un resumen legible, opcionalmente escribir su resultado a un archivo (CSV o JSON) indicando la ruta en la salida, y permitir acotar a un solo negocio. MUST aplicar una ventana de gracia configurable (por defecto 24 horas) para no listar como huérfanos los archivos más recientes. MUST NOT borrar ni modificar nada, ni en la base de datos ni en el almacenamiento.

### Key Entities

- **Referencia a archivo**: valor guardado en un registro (imagen de producto, logo del negocio, imagen de método de pago, comprobante de un intento de pago) que apunta a un archivo del almacenamiento. Puede ser una key gestionada, una URL absoluta histórica del bucket gestionado o una URL de otro origen.
- **Key**: ruta relativa del archivo dentro del almacenamiento con la forma `{esquema_del_negocio}/{carpeta}/{nombre}`. Identifica el archivo y el negocio dueño.
- **Imagen base del formulario**: la imagen que el formulario mostraba al abrirse; se envía junto con la imagen nueva para detectar formularios desactualizados.
- **Archivo huérfano**: archivo del almacenamiento que ningún registro referencia (por un borrado fallido o una subida que nunca se guardó). Es aceptable y seguro; el reporte lo detecta.
- **Referencia rota**: registro cuyo archivo no existe. Es lo que esta funcionalidad busca que el sistema nunca produzca por sí mismo.
- **Reporte de reconciliación**: resultado de solo lectura que agrupa por negocio las referencias rotas, los archivos huérfanos y las referencias fuera de convención.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En la prueba de dos pestañas (una con formulario desactualizado), guardar desde la pestaña vieja cambiando solo un campo distinto de la imagen conserva la imagen nueva en el 100 % de los intentos, visible en la pantalla del producto y en el menú del comensal, y ningún archivo se borra.
- **SC-002**: El 100 % de los intentos de guardar una imagen cuyo archivo no existe (probados por API o con un cliente manual) termina en un error claro y deja el registro sin cambios; la imagen anterior sigue mostrándose.
- **SC-003**: El 100 % de los intentos de guardar la key de otro negocio, de otra carpeta o con segmentos que salen del espacio del negocio termina en error, y ningún archivo de otro negocio se modifica ni se borra.
- **SC-004**: Reemplazar la imagen de un producto por el flujo normal (subir, guardar, ver la nueva) sigue funcionando sin pasos adicionales para el administrador, y el archivo anterior deja de existir cuando ningún otro registro lo usa.
- **SC-005**: Cuando dos registros comparten un mismo archivo, reemplazar la imagen de uno nunca deja al otro con una imagen rota.
- **SC-006**: El cajero ve el comprobante subido por un comensal en "Pagos por confirmar" en el flujo normal, y el 100 % de las keys falsas, de otro negocio o de otra carpeta se rechaza.
- **SC-007**: Ejecutar el reporte de reconciliación en un entorno de prueba con un archivo borrado a mano lo lista como referencia sin archivo, y después de la ejecución la base de datos y el almacenamiento permanecen idénticos.
- **SC-008**: Las imágenes de producto, los logos, los QR y los comprobantes ya existentes siguen mostrándose sin intervención manual tras el despliegue (sin regresión respecto a la spec 080).
- **SC-009**: Ejecutar la migración de comprobantes una segunda vez no modifica ninguna fila, y la ejecución en modo simulación no modifica ninguna.

## Assumptions

- **Repositorios afectados**: `pos-backend` (validación, verificación de existencia, control de borrado, reconciliación y migración) y `pos-heladeria` (el formulario envía la imagen base y deja de reenviar la imagen sin cambios en producto, logo y QR; envía la key del comprobante). La spec vive en `pos-specs`.
- **Superficies de cobro**: la spec no cambia ningún cálculo de montos; la superficie relevante es la pantalla "Pagos por confirmar" del cajero (ver memoria de superficies de cobro), que debe seguir mostrando los comprobantes.
- **Clientes antiguos**: un cliente que no envía la imagen base (versión anterior del formulario o llamada directa a la API) no puede declarar un cambio de imagen legítimo; en ese caso la imagen enviada se ignora en silencio si no coincide con la vigente (después de validar su forma y negocio dueño, FR-004). Las pruebas por API de cambios legítimos deben enviar la base. El detalle de compatibilidad se define en el plan, con el despliegue coordinado de `pos-backend` y `pos-heladeria`.
- **Comprobantes sin sesión**: el comensal sube comprobantes sin sesión autenticada; por eso solo se garantiza prefijo del negocio, carpeta y existencia, no autoría.
- **Ubicación de las tablas**: la ubicación exacta de cada columna (esquema del negocio o esquema global) y la lista definitiva de tablas con imágenes la confirma el plan contra el código; esta spec fija el criterio (revisar el esquema del negocio más las tablas globales que puedan usar el prefijo).
- **Vaciar una imagen**: en producto y logo se mantiene "no tocar"; quitar esas imágenes queda fuera de alcance. El QR de método de pago conserva su comportamiento actual (se puede quitar por edición legítima).
- **Archivos huérfanos**: son aceptables y seguros; esta spec no incluye un proceso que los borre, solo el reporte que los detecta. Borrar archivos huérfanos o corregir referencias rotas detectadas queda como acción manual del operador o de una spec futura.
- **Dominio de assets**: `assets.skeilopos.com` y el dominio público anterior del bucket siguen siendo los orígenes gestionados reconocidos (spec 080).
- **Sin cambio de convención de carpetas**: no se renombran carpetas ni se mueven archivos existentes.
- **Mensajes de error**: los mensajes se muestran en español de Colombia (Principio XIII de la constitución).
