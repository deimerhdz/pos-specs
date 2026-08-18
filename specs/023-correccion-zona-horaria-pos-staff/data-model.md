# Data Model: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

Esta delta no agrega, elimina ni modifica ninguna tabla, columna ni relación de base de datos — es
un cambio de (a) qué header viaja en una respuesta HTTP ya existente y (b) qué valor de tiempo usa
el cliente para evaluar vigencia con funciones puras ya existentes (spec 012). Se documentan aquí
los tipos y roles relevantes para trazabilidad.

## Promotion (spec 012, sin cambios)

| Atributo | Tipo | Rol en esta corrección |
|---|---|---|
| `start_time` / `end_time` | `time` | Ventana horaria contra la que se compara la hora sincronizada — sin cambio de tipo ni de semántica. |
| `status` | `str` | Filtro de estado (`active`), sin cambio. |

## El header nuevo (no es una entidad de base de datos)

| Campo | Tipo | Dónde se genera | Dónde se consume |
|---|---|---|---|
| `X-Server-Time` | `string` (ISO 8601, UTC, `datetime.now(timezone.utc).isoformat()`) | `list_promotions` (`app/api/v1/promotions/router.py:37`), en cada respuesta a `GET /promotions` | `PromotionService.activeQuery` (`promotion.service.ts`), solo en las respuestas de la query `['promotions', 'active']` |

## El estado nuevo del cliente (no es una entidad de base de datos)

| Campo | Tipo | Rol |
|---|---|---|
| `PromotionService.serverTimeOffsetMs` | `Signal<number \| null>` | Diferencia (ms) entre `X-Server-Time` del último `GET /promotions?status=active` recibido y `Date.now()` local en el instante de recibirlo. `null` hasta la primera respuesta exitosa. |
| `PromotionService.now()` | método → `Date` | `new Date(Date.now() + serverTimeOffsetMs())`. Reemplaza `new Date()` en los cuatro puntos de invocación de `pos-terminal.store.ts`. Nunca se llama mientras `ready()` sea `false` (ver Decisión 4 de research.md). |
| `PromotionService.ready` | `Signal<boolean>` | `serverTimeOffsetMs() !== null`. Gobierna si `pos-terminal.store.ts` evalúa vigencia o degrada a "sin promociones" (FR-004). |

## El `Date` que se compara en cada punto de invocación

| Punto | Antes de esta corrección | Después de esta corrección |
|---|---|---|
| `pos-terminal.store.ts:248` (`combos`) | `new Date()` — reloj crudo del dispositivo | `this.promotionService.now()`, con guarda `ready()` (si `false`, `[]`) |
| `pos-terminal.store.ts:262` (`productDiscountBadges`) | `new Date()` | `this.promotionService.now()`, con guarda `ready()` (si `false`, `Map` vacío) |
| `pos-terminal.store.ts:386` (`cartView`) | `new Date()` | `this.promotionService.now()`, con guarda `ready()` (si `false`, sin descuento de previsualización) |
| `pos-terminal.store.ts:1190` (`orderSubtotal`) | `new Date()` | `this.promotionService.now()`, con guarda `ready()` (si `false`, sin descuento de previsualización) |

## Transiciones de estado

No hay una máquina de estados formal en el backend — `list_promotions` es una función de lectura
que agrega un header en cada invocación, sin persistir nada nuevo. En el cliente, la única
transición relevante es `serverTimeOffsetMs`: `null` (arranque, sin sync) → `number` (tras la
primera respuesta exitosa de `activeQuery`) → se **reemplaza** (no se acumula) en cada respuesta
exitosa posterior, para no arrastrar un desfase obsoleto si el reloj del dispositivo deriva con el
tiempo.

## Reglas de validación

Ninguna regla de `Promotion`, `Product`, `ProductVariant` ni `Cart` cambia. Esta delta no introduce
ninguna regla de validación de dominio nueva — solo agrega un dato de transporte (el header) y
sustituye qué instante compara el cliente contra una ventana horaria ya definida.
