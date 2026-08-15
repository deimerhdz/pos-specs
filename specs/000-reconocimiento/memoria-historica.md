# Memoria histórica — POS Heladería

**Fecha**: 2026-08-15
**Alcance**: `../pos-backend` (FastAPI + PostgreSQL 16) y `../pos-heladeria` (Angular 21), según
define la [Constitución](../../.specify/memory/constitution.md).
**Método**: lectura de comentarios de código (`#`, `//`, docstrings, JSDoc) en
`pos-backend/app`, `pos-backend/alembic` y `pos-heladeria/src` vía `grep -rn`, más
`CHANGELOG.md` del backend (única bitácora de cambios que existe en cualquiera de los dos
repos — el frontend no tiene uno). Cada fecha se obtuvo con `git blame -L` sobre la línea
exacta del comentario, no estimada: es la fecha del commit que introdujo ese texto, que
puede no coincidir con la fecha en que ocurrió lo que el comentario narra (un comentario
puede documentar un bug de hace meses el mismo día que se corrige). Los autores que
aparecen en el historial de ambos repositorios son únicamente **Deimer Hernandez**
(`deimerhdz21@gmail.com`) y **Leonardo Gomez** (`leonardogomez306@gmail.com`, commits como
`LeonardoGomezz`); no se encontró ningún "Roberto" ni otro nombre en comentarios, commits o
CHANGELOG. Un commit del backend (`2e94a3ad`, 2026-08-07, reescritura del motor de
promociones) queda registrado bajo el autor genérico **`refactor <dev@local>`**, no bajo un
nombre — se marca explícitamente donde aparece.

Este documento **no propone correcciones** (Principio III de la Constitución): es un
inventario de lo que el propio código dice sobre su pasado, para que el negocio confirme,
corrija o complete cada entrada. El orden es cronológico por fecha de commit del comentario.

---

## Cronología

| # | Fecha (commit) | Qué pasó o se decidió | Implicados | Comentario (fichero:línea) | Pregunta para el negocio |
|---|---|---|---|---|---|
| 1 | **2026-07-17** | Se marca explícitamente **fuera de alcance de v1**: facturación electrónica DIAN (el modelo queda "DIAN-ready" con `cufe`/`dian_status`), notas crédito / anulación fiscal, y notificaciones push al KDS (v1 funciona por *polling*). | Deimer Hernandez (commit `8777acbc`) | `pos-backend/CHANGELOG.md:125-129` | ¿Sigue vigente la lista? DIAN en particular: ¿hay fecha límite regulatoria para tener facturación electrónica, o el negocio opera hoy fuera de esa obligación? |
| 2 | **2026-07-28** | Se eliminaron a propósito dos endpoints legacy del flujo de QR: `POST /sessions` (autenticaba con un UUID plano + header falsificable) y `GET /menu/qr/{qr_token}` (exponía un `table_id` editable por el cliente). El primero además tenía manejo de `IntegrityError` que era código muerto (ninguna constraint lo producía). Los reemplazan `POST /cart/sessions` y `GET /menu/qr-token/{token}`, con tenant y mesa firmados dentro del token. | Deimer Hernandez (commit `5c1db9ed`) | `pos-backend/app/api/v1/orders/router.py:155-160` y `pos-backend/app/api/v1/menu/router.py:187-190` | ¿Quedó algún cliente (app vieja en un dispositivo, enlace impreso) que todavía apunte a las rutas eliminadas? |
| 3 | **2026-07-28** | Se eliminó a propósito `PATCH /{order_id}/status`: permitía asignar cualquier estado a un pedido sin validar la transición ni tocar inventario, así que un pedido podía pasar de `recibida` a `abierta` esquivando `confirm_order` (el único punto que descuenta stock) — inventario sobrestimado sin que nadie se enterara. Cada transición legítima quedó con su propio endpoint. | Deimer Hernandez (commit `5c1db9ed`) | `pos-backend/app/api/v1/orders/router.py:443-450` | ¿Se detectó este patrón (`PATCH /status` saltándose `confirm_order`) en datos reales antes de retirarlo, o fue una revisión preventiva de código? Si fue real, ¿cuánto inventario quedó descuadrado? |
| 4 | **2026-07-28** | Endurecimiento de sesión: el refresh token ya no reutiliza los claims del JWT anterior, sino que relee el usuario en base de datos — de lo contrario una cuenta desactivada o con el rol cambiado seguiría emitiendo access tokens válidos con datos obsoletos durante toda la vida del refresh (hasta 7 días por defecto). | Deimer Hernandez (commit `5c1db9ed`) | `pos-backend/app/api/v1/auth/routes.py:126-129` | ¿Este cambio respondió a un caso real (un empleado desvinculado que siguió con acceso)? |
| 5 | **2026-07-28** | El frontend reconoce por escrito que el diseño documentado del token de sesión del comensal (cookie `httpOnly`) nunca se implementó: el backend devuelve el token en el cuerpo JSON, no con `Set-Cookie`, y mientras eso no cambie el token vive en `localStorage`. | Deimer Hernandez (commit `46ad3eda`) | `pos-heladeria/src/app/modules/tables/services/diner-token.store.ts:15-18` | ¿La cookie `httpOnly` sigue siendo el diseño que se quiere, o `localStorage` se adoptó ya como decisión definitiva? (ver también R11 y R9 del [registro de riesgos](./registro-riesgos.md)). |
| 6 | **2026-07-29** | Decisión de diseño: la factura se emite en un único punto central (`build_sale`) y no en cada camino de cobro, porque hay cuatro formas de cobrar (mostrador, cierre unificado, cierre dividido y el `pay_order` **legacy**) y emitir por separado en cada una "garantizaba que alguna se quedara fuera — que es exactamente lo que pasaba cuando facturar dependía de que alguien pulsara un botón". | Deimer Hernandez (commit `27711065`) | `pos-backend/app/api/v1/sales/service.py` → docstring de `build_sale` en `pos-backend/app/api/v1/sales/builder.py:90-95` | El comentario da a entender que **facturas faltantes ya ocurrieron en producción** antes de este rediseño. ¿Hubo un reclamo o auditoría que lo disparó? Y el `pay_order` legacy que menciona: ¿sigue en uso hoy? |
| 7 | **2026-08-01** | `SESSION_TTL_REFRESH_SLACK_MINUTES` se introduce como optimización (evitar un `UPDATE`+`COMMIT` por cada sondeo del comensal, de ~360/h a ~6/h) pero deja un invariante frágil: debe ser **menor** que `EMPTY_SESSION_TTL_MINUTES` o el barrido de sesiones cierra mesas activas sin pedidos. No hay validación en el arranque que lo garantice (ver R14 del registro de riesgos). | Deimer Hernandez (commit `87e6282c`) | `pos-backend/app/core/config.py:32-40` | ¿Alguna vez se tocó uno de estos dos valores en `.env` de producción sin conocer el invariante? |
| 8 | **2026-08-03** | Se documenta un **bug de inventario real y silencioso**: el descuento por receta sumaba la cantidad de la opción a la del tamaño en vez de dejar que una sola mandara. Con los sabores en 80 g (su valor histórico) y una ensalada pequeña en 60 g, cada venta descontaba 140 g — "nadie se enteraba hasta el conteo físico". Se corrigió para que una sola cantidad (la del tamaño si la define, si no la de la opción) mande siempre. | Deimer Hernandez (commit `03469cad`) | `pos-backend/app/api/v1/catalog/consumption_plan.py:24-26` | ¿Cuánto tiempo estuvo así? ¿Se llegó a hacer un conteo físico que explique una merma de insumos sin causa aparente, y coincide la fecha con el mes en que se corrigió? |
| 9 | **2026-08-03** | Se decide **no** validar por defecto que las selecciones de opciones respeten `min_select`/`max_select` (`STRICT_OPTION_SELECTION=False`), explícitamente porque "el catálogo histórico nunca se validó" — activar el chequeo a ciegas rechazaría combinaciones ya cargadas en el catálogo real. Se deja un script (`opciones_fuera_de_grupo.py`) para limpiar antes de poder encenderlo. | Deimer Hernandez (commit `03469cad`) | `pos-backend/app/core/config.py:55-59` | ¿Se llegó a correr `opciones_fuera_de_grupo.py` alguna vez? ¿Sigue el catálogo sin depurar, casi un año después de esta nota? (ver también R5 del registro de riesgos). |
| 10 | **2026-08-03** | Decisión de negocio explícita: se **deprecia el campo de impuestos editable** en la terminal — el total ya no calcula impuestos, siempre queda en 0 (`const tax = 0`). El commit lo confirma: `fix(tables): deprecate editable tax field in table terminal`. | **Leonardo Gomez** (commit `8166ea9e`) | `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:497` | ¿Por qué se dejó de cobrar impuestos en el sistema — es una decisión fiscal (el negocio no discrimina IVA en el ticket) o quedó pendiente de retomar? Si el negocio sí paga IVA, ¿cómo se está calculando hoy fuera del POS? |
| 11 | **2026-08-04** | Se cierran a propósito, **antes** de dar a los cajeros la capacidad de armar bloques de pago manualmente desde el POS, cuatro huecos de seguridad en el cierre de cuenta dividida que existían "desde que se implementó el split" pero eran difíciles de alcanzar (nadie podía construir un bloque a mano): comensales repetidos en `splits` causaban doble cobro y doble factura; importes en la raíz con `billing_mode='split'` se ignoraban en silencio y el cajero perdía la propina; importes negativos no se validaban; el bloque sin comensal salía sin nombre en la factura. | Deimer Hernandez (commit `42b5dec3`) | `pos-backend/app/scripts/test_split_blindaje.py:1-18` | ¿Esta capacidad de armar bloques manuales ya está en producción? Si es así, ¿desde cuándo, y coincide con la fecha de este endurecimiento (2026-08-04) o quedó una ventana abierta entre que se dio la capacidad y que se blindó? |
| 12 | **2026-08-07** | El **KDS** (pantalla de cocina separada) queda deprecado; el ciclo de vida del ítem (`estado_cocina`) pasa a moverse desde la terminal de mesas. La migración que lo formaliza elimina el estado `entregado` (ya no aporta una decisión distinta de `listo`) y consolida las filas existentes. | Deimer Hernandez (commits `d52f024c`, migración `c5d6e7f8a9b0`) | `pos-backend/app/api/v1/orders/kitchen.py:3-4` y `pos-backend/alembic/versions/c5d6e7f8a9b0_simplify_kitchen_status.py:1-8` | El CHANGELOG de v1.0.0 (2026-07-17, entrada #1 de esta tabla) todavía describía "notificaciones push al KDS" como trabajo futuro, dando a entender que el KDS seguía siendo el plan. Tres semanas después se deprecó por completo. ¿Qué cambió: se descartó la pantalla de cocina como concepto, o se fusionó intencionalmente con la terminal de mesas por decisión operativa (un solo dispositivo en cocina en vez de dos)? |
| 13 | **2026-08-07** | El rol `STAFF` —"una invención del frontend, el backend solo emite `ADMIN` y `CASHIER`"— queda marcado como legado y se mapea a `CASHIER` para no invalidar sesiones JWT vivas. Su pantalla de inicio era justamente el tablero de cocina (KDS) que la entrada #12 deprecó. `STAFF` se había introducido en el commit `9510479` (2026-05-21, "add dashboard layout... for admin, cashier, and staff roles"). | Deimer Hernandez (commits `9510479` y `9d2e4bc8`) | `pos-heladeria/src/app/core/interfaces/user.interface.ts:22-30` | ¿Hubo algún usuario con rol `STAFF` (distinto de cajero/admin) en producción entre mayo y agosto? Si el mapeo automático a `CASHIER` les dio permisos que antes no tenían (o les quitó los que sí tenían), ¿alguien lo notó? |
| 14 | **2026-08-07** | El módulo de **Horarios** (RF-073, `/business-hours`) se retira del todo: el router y la pestaña de Ajustes desaparecen. La tabla `business_hours` y su modelo SQLAlchemy se conservan sin migración de borrado ("no se pierden datos"), quedando huérfanos en el código (ver "zonas oscuras" en [mapa-sistema.md](./mapa-sistema.md)). El mismo commit retira también la pestaña de Ajustes de **Auditoría** (RF-076), aunque ahí el endpoint sigue activo porque `record_audit()` sigue escribiendo desde checkout y promociones. | Deimer Hernandez (commit `1db62bd1`) | `pos-backend/CHANGELOG.md:47-52` | Horarios: ¿se retiró por decisión de producto (no lo pedía nadie) o por un problema técnico sin resolver? ¿Vale la pena una migración que borre la tabla, o hay planes de retomarlo? Auditoría: si ya no hay pestaña para verla, ¿alguien consulta `audit_logs` hoy, y cómo? |
| 15 | **2026-08-07** | Reescritura del motor de promociones con tres cambios estructurales frente a la versión anterior: (1) la vigencia pasa a evaluarse en hora local del tenant — antes se evaluaba en UTC, lo que en UTC-5 corría no solo la ventana horaria sino el día de la semana, el día del mes y el corte de `ends_at` ("un 20% los martes empezaba el lunes a las 19:00 locales, que es justo cuando una heladería vende"); (2) `evaluate_detailed` pasa a devolver un desglose por línea en vez de un escalar; (3) ante dos promociones aplicables, antes ganaba siempre el descuento mayor — ahora gana la de mayor `priority` explícita. | Autor del commit: **`refactor <dev@local>`** (commit `2e94a3ad`) — no un nombre identificable | `pos-backend/app/api/v1/promotions/service.py:1-20` y `pos-backend/app/core/config.py:16-19` | El autor del commit no es una persona identificable en el historial. ¿Quién hizo este cambio y lo revisó? Y el bug de zona horaria: ¿llegó a afectar una promoción real en producción antes del 2026-08-07 (reclamos de clientes o cajeros por una promo que "debía" estar activa y no lo estaba, o viceversa)? |
| 16 | **2026-08-10** | Se documenta que el spec de tests de `ReportsService` llevaba **roto desde que el módulo de reportes migró a `/reports/*`**: seguía probando una versión anterior del servicio que agregaba ventas en el cliente contra `/sales`, `/sales/payment-methods` e `/inventory/items` — URLs que ya nadie llama. El spec quedó así, fallando, hasta que alguien lo notó y lo reescribió. | Deimer Hernandez (commit `33220db4`) | `pos-heladeria/src/app/modules/reports/services/reports.service.spec.ts:10-14` | ¿Desde cuándo corre CI en el frontend? Si corría, ¿por qué un spec roto no se detectó hasta ahora — no hay pipeline, o el pipeline no lo ejecutaba? (ver R6 del registro de riesgos, mismo patrón en el backend). |
| 17 | **2026-07-18** | El campo `cash_sales` del reporte de cierre de caja se marca `DEPRECADO: alias de ventas_efectivo` en ambos lados (respuesta del backend y su schema), mantenido solo por compatibilidad con el frontend. | Deimer Hernandez (commit `927a4606`) | `pos-backend/app/api/v1/cash/service.py:180` y `pos-backend/app/api/v1/cash/schemas.py:112` | ¿Sigue el frontend leyendo `cash_sales` hoy, o ya migró a `ventas_efectivo` y este alias se puede retirar? |

---

## Trabajo pendiente o abandonado (fuera de la tabla cronológica)

Estos elementos no tienen una fecha de comentario tan precisa como los anteriores, pero
el propio código los marca como incompletos o descartados:

- **Facturación electrónica DIAN, notas crédito y push al KDS** (entrada #1): a la fecha de
  este documento (2026-08-15, más de tres semanas después de la última entrada de la
  tabla) no se encontró código nuevo relacionado con ninguno de los tres en `pos-backend`
  ni `pos-heladeria`.
- **Cookie `httpOnly` para el token del comensal** (entrada #5): el diseño documentado
  ("el documento de contrato") nunca se implementó en el backend; `localStorage` es el
  almacén real desde julio y sigue siéndolo hoy.
- **Tabla `business_hours` huérfana** (entrada #14): modelo y datos existen en la base,
  sin router que los exponga ni pestaña que los edite.
- **Impuestos** (entrada #10): el cálculo quedó fijo en 0 sin que el código diga si es
  definitivo o transitorio.

## Notas metodológicas

- No se marca ninguna entrada como `SUPOSICIÓN`: el texto citado en la columna "Comentario"
  es literal (traducido de contexto cuando hace falta) y la fecha viene de `git blame`, no
  de una estimación.
- Este documento se limita a lo que los **comentarios de código** narran sobre el pasado.
  Existen además incidentes visibles solo en el historial de commits sin comentario que los
  explique (p. ej. `pos-backend` commit `8efe0fb`, 2026-07-22, `hotfix/fix-email`: la URL de
  login del correo de bienvenida a un tenant nuevo estaba mal construida) — se dejan fuera
  a propósito porque no hay una nota en el código que narre la causa, solo el diff.
- El registro de riesgos ([registro-riesgos.md](./registro-riesgos.md)) y el mapa del
  sistema ([mapa-sistema.md](./mapa-sistema.md)) documentan el estado *actual* del código;
  este documento documenta lo que el código dice sobre *cómo llegó a ser así*. Varias
  entradas se cruzan (R4, R5, R11, R12, R14) — se referencian en vez de repetirse.
