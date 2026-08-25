# Research: Vista de Pasos para Revisión y Pago del Menú QR

## Decisión 1 — Dónde vive el progreso recuperable

**Decisión**: el progreso recuperable de la vista (paso alcanzado, método de pago elegido,
referencia del comprobante ya subido) se guarda en `localStorage` del navegador del comensal, con
una clave por sesión/participante (ej.
`pos.diner.checkout_progress.<table_session_id>.<participant_id>`).

**Justificación**: la Clarification Session 2026-08-24 del spec 034 resolvió explícitamente que esta
garantía es local al dispositivo/navegador, no multi-dispositivo. `localStorage` es además el mismo
mecanismo que ya usa `diner-token.store.ts` para el token de sesión del comensal (spec 007, A-21,
`RN-CART-*`) — decisión de negocio ya cerrada ("la forma actual es aceptable, no es prioridad"),
así que extenderlo a este progreso no introduce un mecanismo de almacenamiento nuevo ni sujeto a
revisión aparte.

**Alternativas consideradas**:
- **Persistencia en el backend** (una tabla o campo nuevo ligado al `SessionParticipant`): resuelve
  también el caso multi-dispositivo, pero ese caso quedó explícitamente fuera de alcance en la
  clarificación; hacerlo de todas formas violaría el Principio VI (evolución incremental, no
  construir para un alcance no pedido) y el Principio VIII (evolución del modelo de datos) sin
  necesidad real.
- **`sessionStorage`**: sobrevive a una recarga igual que `localStorage`, pero se limpia al cerrar la
  pestaña — más estrecho que el mecanismo ya usado para el token de sesión. Se descarta para no
  introducir un segundo mecanismo de almacenamiento distinto al ya establecido para datos de la
  misma sesión.

## Decisión 2 — Cómo reconocer un comprobante ya subido tras una recarga

**Decisión**: la referencia devuelta por `POST /cart/payment-receipt/presign` (`public_url`/`key`,
`ReceiptPresignOut`, `pos-backend/app/api/v1/cart/schemas.py:145-149`) se guarda junto con el paso y
el método en el mismo registro de `localStorage`. Al hidratar la vista tras una recarga, si esa
referencia existe, la vista salta directo a reintentar `POST /cart/submit`
(`SubmitCartIn{payment_method_id, receipt_file_url}`, `cart/schemas.py:157-161`) — el mismo endpoint
y el mismo comportamiento de reintento que spec 025 (FR-012) ya define para un fallo de red tras la
subida, ahora disparado por una recarga en vez de por un error de conexión.

**Justificación**: la subida del comprobante ya es un paso separado y persistido (a almacenamiento de
objetos) **antes** de que `submit_cart` cree la `CustomerOrder` (`cart/service.py:491-633`); el
backend no vincula esa subida a un participante hasta que se llama `submit_cart`, así que la única
referencia que existe antes de esa llamada es la que el propio frontend ya recibió del presign. No
hay ninguna pieza de estado adicional que un mecanismo nuevo pudiera "recuperar" del lado del
servidor sin duplicar esa misma referencia.

**Alternativas consideradas**:
- **Nuevo endpoint "consultar comprobante pendiente por participante"**: se descarta — requeriría
  que el backend persista una asociación participante↔comprobante antes de que exista un pedido,
  algo que no existe hoy y que spec 025 nunca necesitó porque el propio cliente ya retenía la
  referencia en memoria para su reintento por fallo de red. Construir esa asociación nueva solo para
  este caso duplicaría lo que ya resuelve guardar la misma referencia en `localStorage`.

## Decisión 3 — Cómo mostrar el código QR como imagen

**Decisión**: se extiende la respuesta que el comensal ya consulta
(`GET /cart/payment-methods` → `DinerPaymentMethod`, `cart/schemas.py:119`) para incluir
`fields: list[PaymentMethodFieldDefinition]` — el mismo tipo que ya usa
`GET /sales/payment-methods/catalog` para el tenant-admin (`sales/schemas.py:57`,
`super_admin/schemas.py:18-26`, con `format: text|numeric|image`). El frontend usa esa metadata para
decidir, por cada clave de `payment_info`, si renderiza `<img>` o texto plano.

**Justificación**: la metadata de formato (`format`) ya existe en `PaymentMethodCatalog.fields`
(spec 032, `models/payment_method_catalog.py:23-26`) — hoy solo llega al tenant-admin, nunca al
comensal. Replicar el mismo patrón que ya usa el catálogo del tenant-admin es la solución más simple
y consistente: no inventa una forma nueva de anotar formato, reutiliza la ya definida.

**Alternativas consideradas**:
- **Adivinar en el frontend si un valor "parece" una imagen** (por extensión de archivo o prefijo de
  URL): se descarta por frágil — el nombre de archivo en el bucket de almacenamiento de objetos no
  necesariamente contiene una extensión reconocible, y ya existe una fuente de verdad explícita
  (`format`) que el catálogo define para exactamente este propósito.

## Decisión 4 — Iconos

**Decisión**: se extiende el componente compartido `app-icon`
(`pos-heladeria/src/app/shared/icon/icon.component.ts:9-13,175-176`, selector `app-icon`, un único
`@Input() name!: string`, `@switch(name)` sobre un set fijo) con los `@case` que falten para este
flujo (transferencia, adjuntar/quitar comprobante, volver, confirmar), reemplazando los emoji del
checkout (`public-menu.component.ts:599` y similares) por `<app-icon name="...">`.

**Justificación**: ya existe un componente propio con estilo vectorial consistente ("Lucide-style"
según su propio comentario) usado en el resto de la aplicación (ej. `sidebar.component.ts:72`);
casos como `cash`/`receipt` ya existen y se reutilizan tal cual. Agregar una librería de iconos
externa solo para este flujo no aporta nada que el componente ya existente no resuelva.

**Alternativas consideradas**:
- **Librería de iconos externa** (ej. `lucide-angular`): se descarta — violaría el Principio IX
  (dependencias nuevas requieren justificar que la solución existente no basta) cuando el propio
  componente `app-icon` ya sirve exactamente ese estilo visual en el resto del producto.

## Decisión 5 — Arquitectura de la vista (modal → ruta)

**Decisión**: la interacción se convierte en una ruta hija/hermana bajo `menu/t/:token` (ej.
`menu/t/:token/checkout`), siguiendo la misma convención de carga perezosa ya usada por
`PublicMenuComponent` (`app.routes.ts:63-69`); el token de la mesa sigue siendo el único mecanismo de
autenticación, sin guardias adicionales.

**Justificación**: una ruta propia (no un modal con estado en query params) es lo que permite que una
recarga de página tenga una URL con sentido propio para hidratar el paso correspondiente, y resuelve
directamente lo pedido en la Historia 2 del spec ("vista propia, no una superposición").

**Alternativas consideradas**:
- **Modal con paso codificado en un query param, sin cambiar de ruta**: se descarta — mezclaría dos
  fuentes de verdad (query param + estado de componente) en vez de una sola (ruta + progreso
  persistido en `localStorage`), y no resuelve la sensación de "vista propia" pedida explícitamente.
