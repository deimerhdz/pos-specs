# Feature Specification: Almacenar imágenes como key relativa y servirlas por dominio personalizado

**Feature Branch**: `080-imagenes-key-relativa-r2`

**Created**: 2026-09-08

**Status**: Draft

**Input**: User description: "actualmente en base de datos se guardan las imagenes que se cargan al servicio de r2 de la siguiente forma: https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png. esto debe cambiar, se configuro un custom domain en r2 de nombre assets.skeilopos.com, el objetivo es que a partir de ahora las imagenes que se suban no se creen con la url publica, la idea es que solo se guarde la ruta con la key ejemplo de como deberia verse: heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png. esto debe afectar a las imagenes de los productos, donde se sube el logo y donde se almacenan los metodos de pago y la url que vamos a armar debe ser https://assets.skeilopos.com/heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png"

## Aclaraciones

### Sesión 2026-09-08

- P: ¿Qué se hace con las filas que hoy guardan la URL pública anterior (`https://pub-...r2.dev/{key}`)? → R: Migración única que reescribe a key todas las filas emitidas por el bucket gestionado (imagen de producto, logo, valor de imagen de método de pago), con estrategia de reversión declarada y sin tocar ningún objeto del almacenamiento. Además, el código de lectura tolera temporalmente ambas formas (key o URL absoluta anterior) por si alguna fila se escapa de la migración; esa tolerancia puede retirarse más adelante. Las URLs de otros orígenes (no el bucket gestionado) se conservan tal cual (FR-010).
- P: ¿Los comprobantes de pago del comensal (archivo que se sube en la carpeta `comprobantes`, `receipt_file_url`), que usan exactamente el mismo mecanismo de subida a R2, entran en este cambio? → R: No. Quedan fuera de alcance: se siguen guardando y sirviendo como hoy. Solo cambian las imágenes de producto, el logo del negocio y las imágenes de método de pago. La inconsistencia de que los comprobantes usen otro dominio se podrá abordar en una spec aparte (Principio VI, evolución incremental).
- P: Cuando el frontend reenvía al servidor una imagen sin cambiarla (guardar un formulario que no tocó la imagen), puede mandar la URL de visualización del dominio nuevo (`https://assets.skeilopos.com/{key}`); ¿el servidor debe normalizarla a key antes de persistir? → R: Sí. El servidor MUST normalizar a key **cualquier** URL absoluta del bucket gestionado antes de persistir: tanto la forma anterior (`https://pub-…r2.dev/{key}`) como la del dominio personalizado nuevo (`https://assets.skeilopos.com/{key}`). Amplía FR-004; garantiza FR-002 sin depender del comportamiento del cliente.
- P: Si la migración única (FR-009) se interrumpe o hay que relanzarla, ¿debe ser segura de re-ejecutar? → R: Sí. La migración MUST ser idempotente y re-ejecutable con la aplicación en marcha: reescribe solo las filas cuyo valor es una URL del bucket gestionado, deja intactas las que ya son key, las vacías y las de otro origen; relanzarla tras una interrupción converge sin efectos secundarios y no requiere ventana de mantenimiento (FR-009c, SC-009).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Subir una imagen y verla con el nuevo esquema (Priority: P1)

Un administrador del negocio sube una imagen desde el sistema: la foto de un producto, el logo del negocio o la imagen/QR de un método de pago. Hoy, cuando termina la subida, en la base de datos queda guardada una URL pública completa contra el dominio `pub-...r2.dev`. Con esta funcionalidad, en la base de datos queda guardada únicamente la ruta relativa del objeto (la "key": `heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png`), y allí donde antes se veía la imagen se sigue viendo, ahora cargada desde el dominio personalizado `https://assets.skeilopos.com/{key}`.

**Why this priority**: Es el objetivo central de la solicitud. Desacopla el contenido almacenado del dominio que lo sirve: si el dominio cambia de nuevo, no hay que tocar datos. Entrega valor por sí sola: cada imagen nueva ya nace con el esquema correcto.

**Independent Test**: Subir una imagen de producto nueva, confirmar el cambio, y verificar (1) que el campo en base de datos contiene solo la key (sin `http`, sin dominio) y (2) que el producto muestra su imagen en el menú y en el POS, servida desde `assets.skeilopos.com`. Repetir con el logo del negocio y con la imagen de un método de pago.

**Acceptance Scenarios**:

1. **Given** el formulario de producto con una imagen seleccionada, **When** guardo el producto, **Then** el campo de imagen del producto en base de datos queda como `{tenant}/products/{archivo}` (solo la key) y la imagen se ve en el menú QR y en el catálogo del POS.
2. **Given** la pantalla de información del negocio, **When** subo un logo nuevo, **Then** el campo de logo en base de datos queda como `{tenant}/logo/{archivo}` (solo la key) y el logo aparece en la barra lateral y en los recibos.
3. **Given** la edición de un método de pago con imagen (p. ej. el QR de Nequi), **When** subo la imagen y guardo, **Then** el valor de la imagen dentro de los datos del método de pago queda como `{tenant}/payment-methods/{archivo}` (solo la key) y la imagen se muestra al comensal en el checkout.
4. **Given** cualquier imagen recién subida con esta funcionalidad, **When** la aplicación necesita mostrarla, **Then** la carga desde `https://assets.skeilopos.com/{key}` y nunca desde `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/...`.
5. **Given** un producto sin imagen, un negocio sin logo o un método de pago sin imagen, **When** se guardan, **Then** el campo correspondiente queda vacío y no se arma ninguna URL.
6. **Given** un formulario de producto, logo o método de pago cuya imagen no se modificó y el cliente reenvía la URL de visualización recibida (`https://assets.skeilopos.com/{key}`), **When** guardo, **Then** el campo en base de datos queda como key (`{tenant}/{carpeta}/{archivo}`) y nunca como URL absoluta.

---

### User Story 2 - Las imágenes ya cargadas siguen viéndose (Priority: P2)

El negocio ya tiene cientos de productos con foto, su logo y sus métodos de pago con QR cargados, todos guardados hoy con la URL pública anterior (`https://pub-...r2.dev/{key}`). Al desplegar esta funcionalidad se ejecuta una migración única que reescribe esas filas dejando solo la key; ninguna de esas imágenes puede dejar de verse ni requerir que alguien las vuelva a subir a mano, ni durante ni después de la migración.

**Why this priority**: Sin compatibilidad con lo ya almacenado, el cambio rompe el catálogo entero en producción. Es imprescindible, pero es una consecuencia del cambio de la Historia 1, no un valor nuevo por sí mismo.

**Independent Test**: Tomar un producto cuya imagen se cargó antes del cambio (campo con la URL `pub-...r2.dev`), desplegar la funcionalidad, correr la migración y verificar que (1) el campo quedó como key, (2) la imagen se sigue mostrando en todas las pantallas donde aparecía, servida desde `assets.skeilopos.com`. Repetir con el logo y con un método de pago con QR previo. Verificar además que una fila que (hipotéticamente) no alcanzó la migración también se sigue viendo (tolerancia en lectura).

**Acceptance Scenarios**:

1. **Given** un producto cuya imagen en base de datos era `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/{tenant}/products/{archivo}`, **When** se despliega el cambio y se corre la migración, **Then** el campo queda como `{tenant}/products/{archivo}` (key) y la imagen se sigue viendo en el menú QR y en el POS.
2. **Given** un negocio con el logo guardado como URL pública anterior, **When** se corre la migración y genero un recibo, **Then** el campo de logo quedó como key y el logo aparece en el recibo.
3. **Given** un método de pago con la imagen del QR guardada como URL pública anterior, **When** se corre la migración y el comensal llega al paso de pago por transferencia, **Then** el valor de imagen quedó como key y el comensal ve el QR.
4. **Given** un campo de imagen que todavía contiene una URL pública anterior del bucket gestionado (fila que no alcanzó la migración), **When** la aplicación la muestra, **Then** la imagen se ve igual (tolerancia en lectura, FR-009b).
5. **Given** un campo de imagen con una URL que no pertenece al bucket de R2 gestionado (p. ej. una imagen histórica de otro origen), **When** se corre la migración y la aplicación la muestra, **Then** el valor queda intacto y la imagen se carga tal cual, sin romperse.
6. **Given** una migración que se interrumpió tras convertir parte de las filas, **When** se vuelve a lanzar, **Then** convierte las filas que quedaban, deja las ya convertidas y las de otro origen sin cambios, y el resultado final es idéntico al de una única corrida completa (FR-009c).

---

### User Story 3 - Reemplazar una imagen sigue limpiando la anterior (Priority: P3)

Cuando un administrador cambia la foto de un producto o el logo del negocio, el objeto anterior en el almacenamiento debe seguir borrándose para no acumular archivos huérfanos, y ese borrado debe seguir ocurriendo solo si el cambio se confirmó (comportamiento corregido en su momento por la anomalía A-44 / spec 021).

**Why this priority**: Es una garantía ya existente que no puede perderse al cambiar cómo se referencian las imágenes. El impacto de romperla es acumulación silenciosa de archivos y, si se rompe en el otro sentido, referencias a imágenes inexistentes. Esta garantía cubre **producto y logo**; los métodos de pago no borran hoy su imagen anterior y esta spec no lo cambia (FR-011a).

**Independent Test**: Reemplazar la imagen de un producto que ya tenía una (referenciada como URL pública anterior en un caso, como key en otro) y verificar que, tras confirmarse el cambio, el objeto anterior deja de existir en el almacenamiento y el producto apunta al nuevo; forzar un fallo al guardar y verificar que el objeto anterior no se borra.

**Acceptance Scenarios**:

1. **Given** un producto con imagen previa referenciada como URL pública anterior, **When** subo una imagen nueva y el guardado se confirma, **Then** el objeto anterior se borra del almacenamiento y el producto queda con la nueva key.
2. **Given** un producto con imagen previa referenciada como key, **When** subo una imagen nueva y el guardado se confirma, **Then** el objeto anterior se borra y el producto queda con la nueva key.
3. **Given** un reemplazo de imagen en el que el guardado falla por una razón ajena a la imagen, **When** termina la operación, **Then** el objeto anterior NO se borra y el producto conserva su imagen anterior intacta.
4. **Given** un producto cuya imagen previa es una URL de otro origen (no el bucket gestionado), **When** subo una imagen nueva, **Then** no se intenta borrar nada de ese otro origen y el producto queda con la nueva key.

---

### Edge Cases

- **Campo de imagen vacío o nulo**: producto sin foto, negocio sin logo, método de pago sin imagen. Se sigue tratando como "sin imagen"; no se arma ninguna URL ni se intenta ningún borrado.
- **El valor que llega al servidor ya es una key**: se guarda tal cual, sin modificar.
- **El valor que llega al servidor es una URL absoluta del bucket gestionado**: se normaliza a key antes de guardar (FR-004), de modo que el campo nunca vuelve a quedar con dominio. Cubre tanto la URL pública anterior (`pub-…r2.dev`) como la URL del dominio personalizado nuevo (`assets.skeilopos.com`), incluido el caso habitual del frontend reenviando la URL de visualización al guardar un formulario sin tocar la imagen.
- **URL del bucket gestionado con variaciones** (query string, diferencias de mayúsculas en el host): la extracción de la key se basa en reconocer alguno de los prefijos de dominio conocidos del bucket gestionado, no en una coincidencia exacta.
- **Método de pago con varios datos**: dentro de los datos del método de pago solo el campo marcado como imagen (p. ej. la clave `qr`) es una referencia a un objeto; los demás campos (número de cuenta, celular, etc.) no se tocan.
- **Reemplazo de la imagen de un método de pago**: se persiste la nueva key y se ensambla su URL de visualización como con cualquier imagen, pero el objeto anterior en el almacenamiento **NO se borra** (comportamiento actual sin cambios). La limpieza best-effort del objeto anterior aplica únicamente a la imagen de producto y al logo del negocio (FR-011a). Retirar esos huérfanos, si se quisiera, sería una spec aparte (Principio VI).
- **URL de otro origen** (imagen histórica que no está en el bucket gestionado): se conserva y se muestra sin modificación; nunca se intenta borrar su objeto.
- **Dominio personalizado caído temporalmente**: las imágenes no cargan mientras dure la interrupción, igual que hoy ocurriría si fallara `pub-...r2.dev`. No es una regresión introducida por esta funcionalidad.
- **Reimpresión de un recibo antiguo**: muestra el logo vigente del negocio, no un logo "congelado" para ese recibo (comportamiento actual sin cambios).
- **Consumidores de la API** (frontend de administración, PWA del comensal, generación de recibos): siguen recibiendo una URL lista para usar; el ensamblado key → URL ocurre en el servidor.

## Requirements *(mandatory)*

### Functional Requirements

#### Almacenamiento como key relativa

- **FR-001**: Al subir una imagen de producto, un logo de negocio o una imagen de método de pago, el sistema MUST persistir únicamente la key del objeto (ruta relativa dentro del bucket, con la forma `{tenant}/{carpeta}/{archivo}`, p. ej. `heladeria3/products/46f1a4d1c4aa4c68ba7b32642334d084.png`), sin esquema ni dominio.
- **FR-002**: El sistema MUST NOT guardar una URL absoluta (ni `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/...` ni `https://assets.skeilopos.com/...`) en el campo de imagen del producto, en el campo de logo del negocio, ni en el valor de imagen dentro de los datos de un método de pago.
- **FR-003**: La key almacenada MUST ser exactamente la misma con la que el objeto se subió al almacenamiento (mismo prefijo de tenant y misma carpeta), de modo que sirva tanto para mostrar el objeto como para borrarlo.
- **FR-004**: Si al servidor llega un valor de imagen que es una URL absoluta del bucket gestionado —tanto la URL pública anterior (`https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/{key}`) como la URL del dominio personalizado nuevo (`https://assets.skeilopos.com/{key}`)—, el sistema MUST convertirlo a su key antes de persistirlo. Este caso incluye el reenvío habitual del frontend al guardar un formulario sin cambiar la imagen: el campo nunca debe quedar persistido con esquema ni dominio (FR-002).

#### Presentación por dominio personalizado

- **FR-005**: Toda imagen de producto, logo y método de pago **cuya referencia sea una key o una URL del bucket gestionado** MUST mostrarse cargándose desde `https://assets.skeilopos.com/{key}`, construida anteponiendo el dominio personalizado configurado a la key almacenada. Las referencias a imágenes de otro origen se muestran sin modificación (FR-010).
- **FR-006**: La construcción de la URL de visualización MUST NOT modificar el valor guardado en base de datos: en base de datos vive solo la key; la URL absoluta existe únicamente en la respuesta hacia el consumidor y en el momento de mostrarla.
- **FR-007**: Los consumidores de la API (frontend de administración, PWA del comensal, generación de recibos) MUST seguir recibiendo una URL lista para usar para cada imagen; el ensamblado key → URL absoluta ocurre en el servidor al responder, no en el cliente.
- **FR-008**: El dominio base de assets MUST ser un valor de configuración desplegable (no un literal incrustado en el código), de modo que un cambio futuro de dominio no requiera tocar datos ni código.

#### Compatibilidad con datos existentes

- **FR-009**: Esta funcionalidad MUST incluir una migración única de datos que reescriba a su key todas las filas que hoy guardan una URL pública del bucket gestionado (`https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/{key}`) en el campo de imagen del producto, el campo de logo del negocio y el valor de imagen dentro de los datos de pago de cada método. Tras la migración, esas filas quedan como key y ninguna imagen deja de verse.
- **FR-009a**: La migración de FR-009 MUST tener estrategia de reversión declarada (reconstruir la URL pública anterior a partir de la key) y MUST NOT alterar, mover ni renombrar ningún objeto en el almacenamiento: solo cambia cómo la base de datos referencia la ubicación del objeto.
- **FR-009b**: Además de la migración, el sistema MUST tolerar en lectura, de forma temporal, que un campo de imagen contenga todavía una URL pública anterior del bucket gestionado (fila que se haya escapado de la migración o que entre después): en ese caso la muestra sin romperse y, si el campo se vuelve a guardar, lo normaliza a key (FR-004). Esta tolerancia puede retirarse en una funcionalidad posterior una vez verificado que no quedan filas con la forma anterior.
- **FR-009c**: La migración de FR-009 MUST ser idempotente y segura de re-ejecutar con la aplicación en marcha: procesa únicamente las filas cuyo valor es una URL del bucket gestionado, y deja sin tocar las que ya son key, las vacías y las de otro origen. Si se interrumpe, relanzarla completa la conversión de las filas restantes sin alterar las ya convertidas ni producir efectos secundarios, y no requiere ventana de mantenimiento.
- **FR-010**: Los valores de imagen que sean URLs ajenas al bucket gestionado (otro origen, imágenes históricas) MUST conservarse tal cual y mostrarse sin modificación; ni la migración (FR-009) ni la escritura (FR-004) MUST intentar convertirlas a key, y el sistema MUST NOT borrar sus objetos.

#### Reemplazo y borrado del objeto anterior

- **FR-011**: Al reemplazar la imagen de un producto o el logo del negocio, el objeto anterior en el almacenamiento MUST seguir borrándose (best-effort, sin bloquear la operación si el borrado falla) una vez confirmado el cambio, tanto si la referencia anterior estaba guardada como key como si era una URL pública anterior.
- **FR-011a**: El reemplazo de la imagen de un **método de pago** MUST NOT borrar el objeto anterior del almacenamiento: la limpieza best-effort de FR-011 cubre solo la imagen de producto y el logo del negocio. Un método de pago cuya imagen se reemplaza deja el objeto anterior en el almacenamiento, igual que hoy. Esta spec no introduce limpieza para ese caso.
- **FR-012**: El borrado del objeto anterior MUST NOT ejecutarse si el cambio no llegó a confirmarse (se conserva el comportamiento corregido por la anomalía A-44 / spec 021: primero confirmar, después borrar).
- **FR-013**: Cuando la referencia anterior sea una URL de otro origen (no el bucket gestionado), el sistema MUST NOT intentar ningún borrado.

#### Alcance de comprobantes de pago

- **FR-014**: Los comprobantes de pago del comensal (`receipt_file_url`, carpeta `comprobantes`) MUST quedar fuera del alcance de esta funcionalidad: se siguen guardando con la URL completa y sirviéndose desde el dominio anterior, sin cambios. Esta funcionalidad MUST NOT modificar cómo se guardan ni cómo se muestran los comprobantes, ni el characterization test que hoy congela ese comportamiento.

#### No regresión del flujo de subida

- **FR-015**: El flujo de subida MUST conservar los mismos pasos que hoy (solicitar una URL de subida firmada → subir el archivo directo al almacenamiento → guardar la referencia). Lo único que cambia es qué se persiste (la key) y cómo se arma la URL de visualización.
- **FR-016**: Las validaciones actuales de subida MUST conservarse: tipos de contenido permitidos, carpetas permitidas (`products`, `logo`, `payment-methods`), y que la key nunca use el nombre de archivo enviado por el cliente.
- **FR-017**: Esta funcionalidad MUST NOT cambiar el proveedor de almacenamiento, el bucket, ni el esquema de nombres de las keys.

### Key Entities *(include if feature involves data)*

- **Imagen de producto**: referencia a la foto de un producto (`Product.image_url`). Cambia de contener una URL pública a contener solo la key.
- **Logo del negocio**: referencia al logo del tenant (`Tenant.logo_url`), usado en la barra lateral y en los recibos. Cambia de URL pública a key.
- **Imagen de método de pago**: valor marcado como imagen dentro de los datos de pago de un método (`PaymentMethod.payment_info`, típicamente la clave `qr`), que el comensal ve al pagar por transferencia. Cambia de URL pública a key.
- **Key de objeto**: ruta relativa del objeto dentro del bucket, con la forma `{tenant}/{carpeta}/{archivo}`. Es lo único que se persiste y sirve tanto para mostrar como para borrar el objeto.
- **Dominio de assets**: `assets.skeilopos.com`, dominio personalizado de R2 que sirve el bucket gestionado. Es la base para construir la URL de visualización a partir de la key. Es configuración, no dato.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100 % de las imágenes subidas después del cambio quedan en base de datos como key (el valor no contiene `http` ni ningún dominio), verificable inspeccionando los campos de imagen.
- **SC-002**: El 100 % de las imágenes que hoy se ven (fotos de producto, logo, imágenes de método de pago) siguen viéndose después del cambio: las del bucket gestionado, cargadas desde `assets.skeilopos.com`; las de otro origen, desde su origen original sin cambio (FR-010).
- **SC-003**: Cero imágenes rotas atribuibles al cambio, tanto para imágenes nuevas como para las ya almacenadas antes del despliegue.
- **SC-004**: Al reemplazar la imagen de un producto queda exactamente un objeto vigente para ese producto en el almacenamiento y ninguna referencia rota (la garantía de la anomalía A-44 se mantiene intacta).
- **SC-005**: Un cambio futuro del dominio que sirve las imágenes se aplica en toda la aplicación tocando solo configuración, sin ninguna migración ni edición de datos, porque en base de datos solo vive la key.
- **SC-006**: Ninguna pantalla que hoy muestra imágenes (menú QR, catálogo del POS, formulario de producto, barra lateral, recibos, checkout del comensal) cambia su comportamiento visible más allá del origen desde el que carga las imágenes.
- **SC-007**: Tras correr la migración, el 100 % de las filas de los tres campos afectados que antes tenían una URL pública del bucket gestionado quedan como key (sin `http`); las filas con URLs de otro origen y los comprobantes de pago quedan sin cambios.
- **SC-008**: La migración es reversible: aplicarla y revertirla deja los tres campos afectados exactamente como estaban antes.
- **SC-009**: Re-ejecutar la migración (tras una interrupción o como segunda corrida) no cambia ninguna fila ya convertida ni ninguna fila de otro origen; el estado final de los tres campos es idéntico al de una única corrida completa.

## Assumptions

- El dominio personalizado `assets.skeilopos.com` ya está configurado y activo en Cloudflare R2, apunta al mismo bucket que se usa hoy, y sirve los objetos por su key tal cual (`{tenant}/{carpeta}/{archivo}`), sin anteponer el nombre del bucket ni ningún otro segmento.
- El bucket y su contenido son compartidos entre todos los tenants; la separación por tenant es el primer segmento de la key (el esquema del tenant). Esta funcionalidad no cambia esa separación.
- La API sigue devolviendo a sus consumidores una URL lista para usar para cada imagen; el ensamblado key → URL ocurre en el servidor. El frontend y la PWA del comensal no arman URLs de imágenes por su cuenta (comportamiento actual: reciben la URL y la usan directamente).
- No se cambia el proveedor de almacenamiento (sigue siendo Cloudflare R2 vía API S3-compatible), ni el mecanismo de subida con URL firmada, ni las carpetas permitidas, ni el esquema de nombres de las keys.
- Los objetos ya subidos al almacenamiento no se mueven, renombran ni vuelven a subir; su ubicación física es la misma y solo cambia cómo la base de datos referencia esa ubicación.
- El logo no se "congela" por recibo: cada recibo (incluida una reimpresión) muestra el logo vigente del negocio, tal como hoy. Ninguna factura ni venta ya emitida cambia por esta funcionalidad; estas referencias de imagen no forman parte del importe ni de la representación contable de una factura (Principio VII).
- Todos los textos visibles nuevos, si los hubiera, se redactan en español de Colombia (Principio XIII).

## Impacto sobre el Sistema Existente

### Impacto sobre funcionalidades existentes

- **Subida de imágenes de producto, logo y método de pago**: cambia el valor que se persiste (la key en vez de la URL pública completa). Es un cambio de comportamiento deliberado y solicitado por el negocio; debe quedar registrado como decisión de negocio en `specs/000-reconocimiento/registro-de-anomalias.md` (nueva entrada `A-73`) con quién y cuándo, **antes** de implementarse.
- **Visualización de imágenes** (menú QR, catálogo del POS, formulario de producto, barra lateral del panel, recibos, checkout del comensal): sin cambio percibido por el usuario. La imagen se sigue viendo; cambia el dominio desde el que se carga (`assets.skeilopos.com` en vez de `pub-...r2.dev`).
- **Borrado del objeto anterior al reemplazar imagen** (producto y logo): se conserva. La lógica que hoy extrae la key de una URL pública para poder borrar el objeto anterior debe ampliarse para aceptar también una key directa, y para ignorar referencias de otros orígenes.
- **Endpoint de URL firmada de subida** (`/uploads/presign`) y los presign directos del comensal: siguen devolviendo la key (ya lo hacen hoy). El campo de "URL pública" de esas respuestas deja de ser la fuente de lo que se guarda; el contrato no se rompe pero su uso cambia.
- **Datos de pago que ve el comensal**: el valor de imagen dentro de los datos del método de pago se entrega ensamblado como URL absoluta contra `assets.skeilopos.com`.

### Impacto sobre datos existentes

- Campos afectados: el campo de imagen del producto (`Product.image_url`), el campo de logo del negocio (`Tenant.logo_url`) y el valor de imagen dentro de los datos de pago de cada método (`PaymentMethod.payment_info`).
- **Migración única** (FR-009/FR-009a/FR-009c): reescribe a key todas las filas de esos tres campos que hoy guardan una URL pública del bucket gestionado. Es idempotente y re-ejecutable con la app en marcha (relanzarla tras una interrupción converge, sin ventana de mantenimiento). Lleva estrategia de reversión (reconstruir la URL anterior desde la key). No toca ningún objeto del almacenamiento; solo cambia la representación en base de datos (Principio VIII).
- Las filas con URLs de otro origen (no el bucket gestionado) quedan intactas.
- El campo `receipt_file_url` de los intentos de pago (comprobantes del comensal) **no** entra en el alcance (FR-014): no se migra ni se modifica.
- Ninguna factura ni venta ya emitida se altera. Estas referencias de imagen no forman parte del importe ni de la representación contable de una factura (Principio VII).

### Decisiones de compatibilidad

- Tras la migración, los campos de imagen guardan la key. En lectura, y de forma temporal, el sistema sigue aceptando que un campo contenga una URL pública anterior del bucket gestionado (fila que se escapó de la migración) y sabe mostrarla; esa tolerancia puede retirarse en una funcionalidad posterior (FR-009b).
- Las URLs de orígenes ajenos al bucket gestionado se conservan intactas y se muestran sin modificación.
- El contrato de la API hacia sus consumidores no cambia: siguen recibiendo una URL lista para usar por cada imagen.
- Los comprobantes de pago del comensal se mantienen exactamente como hoy (FR-014).

## Dependencies

- Requiere que el dominio personalizado `assets.skeilopos.com` esté activo en Cloudflare R2 y sirva el bucket que se usa hoy, con las mismas keys.
- Requiere configuración desplegada del dominio base de assets (equivalente a la variable de entorno actual `R2_PUBLIC_BASE_URL`, ahora apuntando al dominio personalizado).
- Requiere que los datos de pago del método sigan indicando qué campo es una imagen (metadato de formato `image`), para saber qué valor tratar como referencia a un objeto.
