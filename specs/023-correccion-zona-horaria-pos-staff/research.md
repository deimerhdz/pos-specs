# Research: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — se resolvió leyendo
directamente `pos-backend` y `pos-heladeria` (los cuatro puntos de invocación, el endpoint que ya
consumen, la configuración CORS existente y el harness de tests de ambos repos). Este documento
registra las decisiones de diseño.

## Decisión 1 — Cómo el cliente obtiene la hora del servidor: header en `GET /promotions`, no un endpoint nuevo ni el cuerpo de `Page[T]`

- **Decisión**: agregar un header de respuesta `X-Server-Time` (UTC, `datetime.now(timezone.utc)
  .isoformat()`) únicamente en `list_promotions` (`app/api/v1/promotions/router.py:37`), y listarlo
  en `expose_headers` de `CORSMiddleware` (`app/main.py:95`), junto a `ETag`/`Retry-After` que ya
  siguen exactamente este patrón (comentario existente en el propio archivo: "hay que listar cada
  cabecera que el JS del navegador pueda leer").
- **Rationale**: `PromotionService.activeQuery` (`promotion.service.ts`) ya llama a
  `GET /promotions?status=active&size=100` como parte de `PosTerminalStore.init()`
  (`pos-terminal.store.ts:525`) — es la llamada que el propio POS de staff hace para obtener los
  datos que `combos`/`productDiscountBadges`/`cartView` necesitan de todas formas. Añadir el header
  ahí cumple RF2 ("sin requerir... una llamada dedicada solo a este fin") sin ninguna petición de
  red adicional. `api.skeilopos.com` es un origen distinto al de cada subdominio de tenant
  (`environment.ts`: `apiBaseUrl: 'https://api.skeilopos.com/api/v1'`) — CORS real, no same-origin —
  así que un header no listado en `expose_headers` sería invisible para el JS del navegador aunque
  viaje en la respuesta; de ahí que el cambio en `main.py` sea obligatorio, no opcional.
- **Alternatives considered**:
  - **Agregar el campo al cuerpo de `Page[PromotionResponse]`** (`app/core/pagination.py`) —
    descartado: `Page[T]` es el envoltorio de paginación genérico que usan decenas de endpoints no
    relacionados con promociones; tocar su schema para un dato que solo le interesa a este endpoint
    ensancha el radio de cambio más allá de A-09 (Principio III) sin necesidad, cuando un header
    específico de esta ruta logra lo mismo sin tocar código compartido.
  - **Endpoint nuevo dedicado (`GET /server-time` o similar)** — descartado explícitamente por
    RF2: introduciría una llamada de red adicional que el POS tendría que orquestar (cuándo pedirla,
    con qué frecuencia) solo para este fin, cuando ya existe una llamada que se puede aprovechar.
  - **Agregar el dato a `TenantInfoService`/`GET /tenant`** — descartado: ese endpoint se carga una
    sola vez al iniciar sesión administrativa (`TenantInfoService.load()`), no en el ciclo de vida
    del POS de staff (`PosTerminalStore.init()` no lo llama), y no se refresca periódicamente — un
    offset de reloj capturado una sola vez al día no protege contra el drift ni contra una recarga
    de terminal a media jornada.
  - **Leer el header estándar `Date` que HTTP ya envía en toda respuesta** — descartado: `Date` no
    está en la lista de cabeceras "CORS-safelisted" que el navegador expone a JS sin
    `Access-Control-Expose-Headers` explícito, así que igual habría que tocar `main.py`; usar un
    header propio (`X-Server-Time`) es más explícito sobre su propósito y evita depender de un
    detalle de bajo nivel del stack HTTP que no es parte del contrato de la API.

## Decisión 2 — Dónde vive el desfase de reloj: `PromotionService`, no un servicio nuevo compartido

- **Decisión**: `PromotionService` gana una signal `serverTimeOffsetMs = signal<number | null>
  (null)`, actualizada dentro del `queryFn` de `activeQuery` (el único que se pide con la cadencia
  y el propósito correctos — ver Decisión 1), más un método `now(): Date` y un `computed ready:
  Signal<boolean>` (`offset !== null`).
- **Rationale**: `PromotionService` ya es el dueño de todo lo relacionado con vigencia de
  promociones en el frontend (`activePromotions`, y por extensión el módulo que la spec 012 ya
  documenta como dueño de A-07/A-09/A-10); `PosTerminalStore` ya lo inyecta
  (`pos-terminal.store.ts:177`), así que los cuatro sitios corregidos no necesitan una inyección
  nueva. Mantener el offset ahí, y no en un servicio "reloj global" nuevo, evita crear un módulo
  compartido cuyo único consumidor real, hoy, es el propio POS de staff — si el panel de
  administración (`promotions-page.component.ts`) llegara a necesitarlo en una delta futura (fuera
  de alcance de esta spec), ya lo tendría disponible sin fricción, porque ya inyecta
  `PromotionService`.
- **Alternatives considered**: un `ServerClockService` nuevo en `core/` — descartado por
  sobre-ingeniería: hoy solo un consumidor (`PosTerminalStore`, vía las promociones activas)
  necesita esta hora, y crear una abstracción compartida para un único consumidor viola la guía del
  proyecto de "no diseñar para requisitos hipotéticos futuros".

## Decisión 3 — Qué se prueba con `TestBed` completo y qué se prueba como característica aislada

- **Decisión**: dos niveles de prueba, no uno.
  1. **`promotion.service.spec.ts`** (`TestBed` + `provideHttpClientTesting()` +
     `provideTanStackQuery()`, mismo harness que `product.service.spec.ts`): caracteriza la lógica
     *nueva* de verdad — que `activeQuery` lee `X-Server-Time` de la respuesta simulada
     (`HttpTestingController.flush(body, { headers: { 'X-Server-Time': ... } })`), calcula el offset
     correctamente contra un `Date.now()` fijado (`vi.setSystemTime`), y que `now()`/`ready`
     reflejan ese estado antes y después de la primera respuesta.
  2. **`pos-terminal.store.spec.ts`**: los cuatro sitios corregidos (`combos`,
     `productDiscountBadges`, `cartView`, `orderSubtotal`) **no cambian su lógica de cálculo** —
     siguen delegando en `isPromoActiveNow`/`bestProductDiscount`/`discountedUnitPrice`, que ya
     están cubiertas exhaustivamente por `promotion-pricing.util.spec.ts` con un `now` explícito
     inyectado por el test. Lo único nuevo en el store es *de dónde* sale ese `now` y la guarda de
     "aún sin sync" — así que el test nuevo de este fichero no reconstruye los cuatro flujos
     completos con datos de catálogo/carrito reales, sino que verifica el contrato puntual: con
     `promotionService.now`/`ready` mockeados (un doble simple, no HTTP real), (a) antes del primer
     sync (`ready() === false`) los cuatro resultados no muestran vigencia alguna (FR-004), y (b)
     tras el sync, cada uno de los cuatro usa el valor de `now()` del mock, no `Date.now()` real —
     verificable fijando el reloj del sistema de pruebas a un instante que solo produciría el
     resultado correcto si el store usó el mock y no el reloj real.
- **Rationale**: `pos-terminal.store.spec.ts` hoy **no** instancia `PosTerminalStore` con
  `TestBed` — solo prueba funciones puras exportadas (`deriveTableStatus`, `newPendingIds`); no hay
  precedente en este repositorio de montar el store completo (13 dependencias inyectadas) para un
  test. Reconstruir ese harness completo para las cuatro `computed`/método corregidos sería
  desproporcionado frente al riesgo real: la lógica de cálculo de vigencia ya está probada a fondo
  en `promotion-pricing.util.spec.ts` (sin cambios en esta delta) y la lógica de obtención de la
  hora ya queda probada a fondo en `promotion.service.spec.ts` (Decisión 3.1). Lo único que este
  fichero necesita demostrar es la **integración puntual** entre ambos — un doble simple de
  `PromotionService` para el store, no un `TestBed` completo con HTTP real, sigue siendo
  characterization test válido (Principio II) sin la sobrecarga de mockear las 12 dependencias
  restantes del store que A-09 no toca.
- **Alternatives considered**: montar `PosTerminalStore` completo vía `TestBed.inject` con
  providers reales/mockeados para las 13 dependencias — descartado por desproporcionado (ver
  Rationale) y porque introduciría un harness nuevo y pesado que ninguna otra spec de este
  repositorio necesitó hasta ahora, sin que A-09 lo requiera para demostrar su propio contrato.

## Decisión 4 — Comportamiento antes del primer sync: sin insignias, no reloj sin verificar (FR-003/FR-004)

- **Decisión**: mientras `PromotionService.ready()` sea `false` (offset aún no calculado — arranque
  en frío, corte de red antes de la primera respuesta), los cuatro sitios de `pos-terminal.store.ts`
  se comportan como si no hubiera ninguna promoción vigente: `combos` devuelve `[]`,
  `productDiscountBadges` devuelve un `Map` vacío, `cartView`/`orderSubtotal` calculan el precio
  **sin** descuento de previsualización (el mismo camino que ya usan hoy cuando `activePromotions()`
  está vacío, sin lógica nueva de "modo degradado").
- **Rationale**: RF3/RF4 exigen que el POS nunca bloquee la pantalla del carrito y que nunca use el
  reloj del dispositivo sin corregir — la única opción que cumple ambas a la vez es no evaluar
  vigencia en absoluto hasta tener una hora en la que confiar. El estado resultante (sin insignias)
  es indistinguible, para el cajero, de "no hay promociones activas ahora mismo" — un estado que la
  UI ya sabe mostrar sin cambios, porque ya ocurre hoy cada vez que `activePromotions()` está vacío.
  No es una regresión de UX nueva que diseñar: es un estado ya soportado, solo con una causa
  adicional (offset desconocido) que lo activa.
- **Alternatives considered**: usar el reloj del dispositivo como *fallback* mientras no haya sync —
  descartado explícitamente: es exactamente el comportamiento que A-09 corrige; usarlo aunque sea
  "temporalmente" reintroduce el defecto en la ventana entre el arranque del terminal y la primera
  respuesta del backend, que es además el escenario de mayor riesgo (un terminal recién desempacado
  o que perdió la hora tras un reinicio de fábrica, exactamente el caso citado en
  `contradiccion-01-motor-promociones-frontend-backend.md`).

## Decisión 5 — El endpoint del panel de administración (`getPromoDisplay`) queda fuera, no se "arregla de paso"

- **Decisión**: `promotions-page.component.ts` (`getPromoDisplay`, `findOverlaps`) sigue usando
  `new Date()` sin corregir; no se toca en esta spec.
- **Rationale**: ya documentado como Out of Scope en `spec.md` — es una superficie back-office
  distinta (no POS de venta), sin la misma exposición económica directa que A-09 describe. Añadirlo
  aquí expandiría el alcance de la corrección más allá de lo que la reapertura de A-09 autorizó
  (Principio I: solo se autorizó lo citado en `registro-de-anomalias.md`).
- **Alternatives considered**: corregirlo también, ya que `PromotionService.now()` queda disponible
  y el cambio sería trivial — descartado por disciplina de alcance (Principio III, Principio I): es
  una anomalía distinta, sin autorización de negocio propia, y mezclarla aquí complicaría revertir
  o auditar cada corrección de forma independiente si hiciera falta.
