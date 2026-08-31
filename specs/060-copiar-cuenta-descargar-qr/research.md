# Research: Copiar número de cuenta y descargar QR en pagos por transferencia

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-08-30

## Decisión 1 — Copiar al portapapeles: API nativa, sin dependencia nueva

**Decisión**: usar `navigator.clipboard.writeText(valor)` (Clipboard API nativa del navegador) desde
un método nuevo en `TransferDetailsStepComponent`, p. ej. `copyField(value: string): Promise<void>`,
envuelto en `try/catch`. Éxito → `this.toast.success('Copiado al portapapeles', 5000)`. Fallo (API no
disponible, permiso denegado, contexto no seguro) → `this.toast.error('No se pudo copiar', 5000)`.

**Rationale**: es la única API estándar para escribir al portapapeles desde el navegador; no
requiere ninguna librería nueva (Principio IX de la Constitución — preferencia por la plataforma
estándar). `navigator.clipboard` exige un contexto seguro (HTTPS o `localhost`), que ya es el caso
de este flujo (`/menu/t/:token/...` se sirve sobre HTTPS en producción; `localhost` en desarrollo
cae dentro de la excepción estándar de "contexto seguro" de los navegadores).

**Alternativas consideradas**:
- `document.execCommand('copy')` (API vieja, basada en seleccionar un `<input>` oculto y ejecutar el
  comando del navegador) — rechazada: deprecada en todos los navegadores modernos, y la Clipboard
  API ya cubre el 100% de los navegadores que soporta hoy la app (sin IE11 ni objetivos legacy).
- Una librería de terceros (`clipboard.js`, etc.) — rechazada: `navigator.clipboard.writeText` ya
  resuelve el caso completo (un string plano) sin ninguna de las complejidades que esas librerías
  existen para resolver (selección de rango, fallback a Flash, etc.).

## Decisión 2 — Descargar el QR: `fetch` + blob + ancla temporal (no el `href` directo)

**Decisión**: al activar la descarga, hacer `fetch(url)` sobre la URL pública del campo de imagen
(`method()!.payment_info?.[f.key]`), convertir la respuesta a `Blob`, crear una URL de objeto
(`URL.createObjectURL(blob)`), dispararla con una `<a>` temporal (`a.download = '...'; a.click()`) y
revocarla después (`URL.revokeObjectURL`) — mismo patrón de "ancla temporal + click" que ya usa
`TableQrComponent.download()` (`table-qr.component.ts:107-116`), pero con un `fetch` previo porque
aquí la imagen **no** es una data URL generada en el cliente: es una URL pública subida por el
administrador a Cloudflare R2 (`payment-method.service.ts:132-139`, `uploadQrImage()`), es decir,
de otro origen (cross-origin) respecto a la SPA.

**Por qué no basta con `a.href = url; a.download = '...'; a.click()` directo (como si fuera local)**:
el atributo `download` de un `<a>` solo fuerza la descarga quel navegador cuando la URL es del mismo
origen, o cuando el servidor responde con la cabecera `Content-Disposition: attachment`. Sobre una
URL cruzada sin esa cabecera (el caso típico de un objeto público de R2/S3), la mayoría de
navegadores **ignora** `download` y en su lugar navega a la imagen o la abre en pestaña nueva — no
se logra el objetivo de la Historia 2 ("sin tener que estar tomando captura de pantalla"). Traer el
archivo como `Blob` primero y descargar desde una URL de objeto (`blob:`) sí es siempre del mismo
origen para el navegador, sin importar de dónde vino el `fetch`.

**Riesgo identificado, sin cerrar en esta fase de planificación**: `fetch()` para leer el cuerpo de
una respuesta cross-origin sí exige que el servidor de origen (R2) responda con cabeceras CORS
(`Access-Control-Allow-Origin`) permisivas para `GET` — a diferencia de un `<img [src]>`, que carga
igual sin esas cabeceras (carga "opaca", sin necesidad de leer el cuerpo). La configuración de CORS
del bucket de R2 vive del lado de Cloudflare (no hay archivo de configuración de CORS de R2 en
`pos-backend` ni en `pos-heladeria`), así que **no se puede confirmar por código estático** si el
bucket ya permite `fetch()` cross-origin sobre estos objetos públicos. **Queda como verificación
obligatoria de implementación** (Principio X): probar la descarga contra un QR real subido en un
tenant de prueba antes de dar la Historia 2 por completada; si el bucket no tiene CORS habilitado
para `GET`, es un ajuste de configuración de infraestructura (Cloudflare), no de este componente.

**Alternativas consideradas**:
- Descargar directo con `a.href` sin `fetch` (como `TableQrComponent`) — rechazada por la razón de
  cross-origin explicada arriba: ese componente funciona porque su imagen ya es una data URL
  generada localmente (`qrDataUrl()`, mismo origen por definición), no una URL remota de R2.
- Abrir la imagen en una pestaña nueva (`window.open(url)`) como mecanismo de "descarga" — rechazada:
  no cumple la Historia 2 tal como la pide el dueño del proyecto (evitar la captura de pantalla); en
  la mayoría de navegadores móviles eso solo muestra la imagen, sin guardarla, dejando al comensal en
  el mismo punto de partida (tener que capturarla manualmente).
- Pedirle al backend un endpoint de descarga propio que haga proxy del archivo con
  `Content-Disposition: attachment` — rechazada por ahora: agrega una ruta de backend nueva
  (`pos-backend`) para un problema que la técnica `fetch` + blob ya resuelve enteramente del lado
  del cliente, sin tocar el backend ni introducir un contrato de API nuevo (mantiene el alcance de
  esta feature dentro de `pos-heladeria` únicamente, spec.md Assumptions). Si la verificación de
  implementación (arriba) revela que el bucket R2 no admite CORS de lectura, esta alternativa queda
  como la vía de respaldo a evaluar en un spec de seguimiento — no se decide de antemano sin evidencia
  del problema real.

## Decisión 3 — Nombre del archivo descargado

**Decisión**: `qr-<slug-del-metodo>.png` (p. ej. `qr-nequi.png`), derivando el slug del `name` del
método de pago (`method()!.name`, ya disponible) en minúsculas y sin espacios/acentos. Si el método
define más de un campo de imagen (caso hoy no observado en la app, pero el `computed imageFields()`
ya itera un arreglo), se agrega el `label`/`key` del campo al nombre para no colisionar
(`qr-nequi-codigo-qr.png`).

**Rationale**: reutiliza datos que el componente ya tiene en memoria (`method()!.name`, `f.label`,
`f.key`); no depende de parsear la URL de R2 (que no lleva un nombre legible, son claves opacas de
almacenamiento).

## Decisión 4 — Íconos nuevos en el sistema de íconos compartido

**Decisión**: agregar dos casos nuevos al `@switch` de `IconComponent`
(`src/app/shared/icon/icon.component.ts`), `'copy'` y `'download'`, con trazos SVG en el mismo
estilo *single-stroke* (Lucide-style, `stroke="currentColor"`) que ya usan los 24 casos existentes —
mismo criterio ya usado para agregar cualquier ícono nuevo a este switch (p. ej. `'transfer'`,
`'upload'`, ambos ya usados en esta misma pantalla). No se introduce ninguna librería de íconos
externa.

- `'copy'`: dos rectángulos superpuestos (icono estándar "copiar" — el trazo trasero sugiere el
  documento de origen, el delantero la copia).
- `'download'`: una flecha hacia abajo terminando en una bandeja/línea horizontal (icono estándar
  "descargar").

**Rationale**: FR-002 de spec.md exige un ícono "reconocible... consistente con el resto de íconos
ya usados en la aplicación" — el componente ya centraliza exactamente ese contrato (un único
`@switch` reutilizado en toda la app); extenderlo es la única forma de cumplir FR-002 sin duplicar
un SVG suelto dentro de `TransferDetailsStepComponent` (que rompería la consistencia que ya impone
este componente compartido).

**Alternativas consideradas**: usar un emoji (📋/⬇️) como en algunos lugares antiguos del código
(p. ej. el ✕ de `TableQrComponent`, componente más viejo) — rechazada: esta misma pantalla
(`transfer-details-step.component.ts`) ya usa exclusivamente `<app-icon>` para "atrás", "cerrar" y
"subir", así que introducir un emoji aquí sería inconsistente dentro del propio archivo que se está
modificando, no solo con el resto de la app.

## Decisión 5 — Alcance de archivos a modificar; sin componente ni servicio nuevo

**Decisión**: los únicos archivos de producción a modificar son
`pos-heladeria/src/app/modules/tables/pages/checkout/transfer-details-step.component.ts` (template +
métodos nuevos `copyField()`/`downloadImage()` sobre `textFields()`/`imageFields()`, ya existentes) y
`pos-heladeria/src/app/shared/icon/icon.component.ts` (Decisión 4). Se crea un archivo de test nuevo,
`transfer-details-step.component.spec.ts` — hoy **ningún** componente de
`src/app/modules/tables/pages/checkout/` tiene test (confirmado: `find ... -iname "*.spec.ts"` no
devuelve resultados en ese directorio), así que esta feature no modifica ningún test existente, solo
agrega el primero para este componente. Ningún cambio en `pos-terminal.store.ts`,
`checkout-progress.store.ts`, `diner.interface.ts`, ni en ningún archivo de `pos-backend`.

**Rationale**: ambas historias son ajustes de presentación/interacción sobre datos que el componente
ya recibe (`payment_info`, `fields`) — no requieren un servicio nuevo (el "copiar" y "descargar" son
llamadas puntuales a APIs del navegador, no lógica de negocio reutilizable en otra pantalla, ver
spec.md Assumptions: esta mejora es específica de esta única pantalla).

## Decisión 6 — Duración de los toasts: 5000 ms explícitos en ambas llamadas

**Decisión**: pasar `5000` como segundo argumento explícito en las cuatro llamadas nuevas
(`toast.success(..., 5000)` en copiado y descarga exitosos; `toast.error(..., 5000)` en ambos
fallos), en vez de confiar en los valores por defecto de `ToastService` (`success` por defecto usa
3000 ms, `error` ya usa 5000 ms por defecto — `toast.service.ts:20,24`).

**Rationale**: FR-007 de spec.md exige exactamente 5 segundos para **ambas** notificaciones de
éxito pedidas por el usuario, sin depender de que el valor por defecto de `success()` coincida por
casualidad (hoy es 3000 ms, distinto). Pasar el valor explícito en las cuatro llamadas mantiene la
regla de negocio de esta feature visible en el propio call site, sin cambiar el valor por defecto de
`ToastService` para el resto de la aplicación (que no fue pedido y afectaría otras pantallas sin
autorización — Principio V, no mezclar cambios no relacionados).

## Decisión 7 — Alcance por campo: cada campo de texto y cada campo de imagen, de forma independiente

**Decisión**: el ícono de copiar se agrega dentro del `@for (f of textFields(); ...)` ya existente
(`transfer-details-step.component.ts:57-61`), uno por cada campo con valor; la opción de descargar se
agrega dentro del `@for (f of imageFields(); ...)` ya existente (líneas 62-71), una por cada campo de
imagen con valor. Ninguno de los dos `computed` (`textFields`, `imageFields`) cambia su lógica de
filtrado.

**Rationale**: documentado ya en spec.md Assumptions — generalizar a "cada campo mostrado" es
simétrico con cómo el propio componente ya trata los campos hoy (genérico por `format`, no
hardcodeado a una clave particular como `numero_celular`), y evita introducir un campo de
configuración nuevo (p. ej. un flag "copiable" por campo) que el backend no expone
(`PaymentMethodField` no tiene ese campo — `diner.interface.ts:117-123`) y que nadie pidió.
