# Fase 0 — Research: Planes de Suscripción por Tenant

Todas las decisiones de esta fase son **técnicas** (Principio XI): el spec (`spec.md`) define el
comportamiento; este documento define cómo implementarlo sobre lo que ya existe en `pos-backend`
y `pos-heladeria`, sin contradecir el spec ni inventar requisitos nuevos.

## Decisión 1 — `Plan` con ocho columnas fijas, no un modelo de características extensible (EAV)

**Decisión**: `Plan` (`app/models/plan.py`, `shared.plans`) tiene ocho columnas fijas: cinco
límites numéricos nullable (`mesas_limit`, `cajas_limit`, `usuarios_limit`, `productos_limit`,
`metodos_pago_activos_limit`, todas `Integer`, default `0`) y tres accesos booleanos
(`inventario_access`, `compras_access`, `promociones_access`, todas `Boolean`, default `false`).
`NULL` en una columna de límite significa "ilimitado"; el propio default `0`/`false` de cada
columna ya implementa FR-002 ("toda característica no configurada explícitamente se trata como
bloqueada") sin necesitar distinguir "no configurada" de "configurada explícitamente en 0" — el
spec no exige esa distinción, ambas producen el mismo bloqueo.

**Alternativas consideradas**:
- Un modelo de características tipo EAV (`plan_features`, una fila por `(plan_id, feature_key,
  value)`, como el propio spec describe conceptualmente en su Key Entity "Característica de
  Plan"). Descartada: las Assumptions del propio `spec.md` fijan explícitamente el conjunto de
  ocho características como cerrado para esta fase ("agregar un tipo de característica nuevo en
  el futuro es una extensión posterior, no parte de esta spec") — construir la flexibilidad de un
  modelo de filas dinámicas para un conjunto que hoy es fijo es complejidad sin uso actual
  (contradice la preferencia del proyecto por evitar abstracciones prematuras). Si una fase futura
  amplía el catálogo de características, ese spec futuro decide entonces si migra a EAV.
- Tres estados explícitos por columna (`not_configured` / `limited(n)` / `unlimited`), vía una
  columna adicional de bandera por característica. Descartada: el spec no distingue "no
  configurada" de "configurada en 0" en ningún requisito ni criterio de aceptación — ambas
  producen exactamente el mismo bloqueo (FR-002/FR-006), así que la tercera bandera no tendría
  ningún caso de uso observable.

## Decisión 2 — `Tenant.plan_id` reemplaza la columna heredada `Tenant.plan`, no coexiste con ella

**Decisión**: se agrega `Tenant.plan_id: Mapped[uuid.UUID]` (FK `NOT NULL` → `shared.plans.id`)
y se elimina la columna `Tenant.plan: Mapped[str]` (heredada, `default="basic"`) en la misma fase
de migración de datos. Tras la ampliación de precio/duración/renovación (Clarifications, sesión
de ampliación), `Tenant` gana además `ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` — ver
Decisión 10 para por qué estas tres también viven en `Tenant` y no en una tabla de asignación
separada.

**Por qué es seguro eliminarla, no solo dejar de usarla**: `plan: str` no se lee en ningún
endpoint, servicio ni validación de negocio (confirmado por búsqueda en todo `pos-backend`); el
único lugar que la toca es `test_tenant_timezone.py`, que la instancia como ruido obligatorio del
constructor de `Tenant`, sin aserciones sobre su valor. En el frontend, `Tenant.plan: string`
existe en la interfaz TypeScript (`tenant.interface.ts`) pero ningún componente la lee ni la
edita. No hay dato de negocio real que preservar.

**Alternativas consideradas**:
- Dejar `Tenant.plan` intacta y agregar `plan_id` como columna independiente. Descartada:
  mantener un campo fantasma que ya nadie usa, junto a su reemplazo real, es exactamente la clase
  de comportamiento no intencionado que la constitución de esta fase busca evitar (Principio II:
  "el comportamiento actual continúa considerándose válido por defecto" — pero aquí el
  "comportamiento actual" de esa columna es no tener ningún efecto, así que no hay comportamiento
  que proteger). Conservarla solo agregaría una pregunta futura ("¿por qué hay dos campos de
  plan?") sin ningún beneficio.
- Renombrar `Tenant.plan` a texto libre visible ("nombre de plan legible") en vez de una FK real.
  Descartada: no satisface FR-005/FR-007/FR-008 (validar límites y accesos en tiempo real requiere
  un valor consultable, no texto arbitrario).

**Key Entity "Asignación de Plan por Tenant" del spec**: se realiza como la columna
`Tenant.plan_id` misma, no como una tabla separada — ver Decisión 3 más abajo, justificado por las
Assumptions del spec ("no se requiere un historial auditable de cambios de plan en esta fase").

## Decisión 3 — Migración: seed determinístico + backfill + `NOT NULL` en una sola fase de datos

**Decisión**: la migración de datos para `plan_id` sigue tres pasos, pero — a diferencia del
backfill de spec 032 (`catalog_id`, que quedó nullable indefinidamente por depender de un match
difuso de nombres que exige revisión humana) — aquí los tres pasos pueden completarse en una
sola migración porque el backfill es **100% determinístico**: todo tenant existente recibe el
mismo plan transicional, sin ambigüedad que resolver.

1. `{rev}_plans_table.py` — crea `shared.plans` (solo esquema, incluye `precio_mensual`/
   `precio_anual` nullable desde el principio — no necesitan backfill porque nacen nullable y así
   se quedan, Decisión 11).
2. `{rev}_seed_transitional_plan.py` — siembra una única fila: `name="Ilimitado (transición)"`,
   los cinco límites en `NULL` (ilimitado), los tres accesos de módulo en `true`, y ambos precios
   en `NULL` (sin precio — no aplica a un plan que no se cobra) — reproduce exactamente el
   comportamiento sin restricciones que todo tenant ya tenía antes de esta spec (Principio VII/II:
   el despliegue no debe cambiar el comportamiento observable de ningún tenant existente el mismo
   día).
3. `{rev}_tenant_plan_assignment.py` (renombrada de `tenant_plan_id` al ampliar su alcance) —
   agrega `plan_id` nullable **y** `ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en`
   (nullable para siempre, Decisión 12), ejecuta `UPDATE shared.tenants SET plan_id = (SELECT id
   FROM shared.plans WHERE name = 'Ilimitado (transición)') WHERE plan_id IS NULL` (dejando
   `ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` en `NULL` — ningún tenant existente
   puede vencer el día del despliegue, Decisión 12), luego `ALTER COLUMN plan_id SET NOT NULL` y
   `DROP COLUMN plan` — las cuatro columnas nuevas y las tres operaciones de `plan_id` en la misma
   migración porque, a diferencia de `catalog_id`, no hay ningún caso en que el backfill deje
   filas sin resolver: el `UPDATE` cubre el 100% de `shared.tenants` en una sola sentencia, y las
   tres columnas de ciclo/fechas no necesitan `NOT NULL` ni backfill — nacen nullable y ese es su
   estado final válido para cualquier tenant sin vencimiento (no solo durante la migración).

**Alternativas consideradas**:
- Reproducir exactamente el patrón de spec 032 (columna nullable indefinida, backfill como script
  reejecutable aparte, sin forzar `NOT NULL`). Descartada: ese patrón existe en 032 específicamente
  porque el backfill requiere juicio humano (nombres personalizados que no matchean el catálogo,
  FR-015a de esa spec) — aquí no existe esa ambigüedad, forzar la misma cautela sería alargar sin
  necesidad la ventana en la que `Tenant.plan_id` podría ser `NULL` y violaría FR-003 ("todo
  tenant tiene, en todo momento, exactamente un plan vigente") más tiempo del estrictamente
  necesario para aplicar la migración.
- Requerir que el Super Admin cree y asigne manualmente un plan a cada tenant existente antes de
  permitir el despliegue (sin plan transicional sembrado). Descartada: violaría FR-003 durante la
  ventana de migración y, más grave, cambiaría de golpe el comportamiento de tenants en producción
  el día del despliegue (todos quedarían bloqueados a 0 en las cinco características, Principio
  II) — el plan transicional sin restricciones es lo que preserva la compatibilidad exigida.

**Estrategia de rollback** (Principio VIII): revertir `{rev}_tenant_plan_assignment.py` re-agrega
la columna `plan` (`String(100)`, default `"basic"`) y elimina `plan_id`/`ciclo_facturacion`/
`plan_iniciado_en`/`plan_vence_en`; revertir `{rev}_seed_transitional_plan.py` borra la fila
sembrada; revertir `{rev}_plans_table.py` hace `op.drop_table("plans")`. Ninguna de las tres
operaciones toca `sales`/`payments`/`invoices`.

## Decisión 4 — Recursos "mesas/cajas/usuarios/productos" cuentan todas las filas; "métodos de pago" cuenta solo las activas

**Decisión**: el conteo de uso contra el límite del plan sigue exactamente la clarificación de
`spec.md`: `dining_tables`, `cash_registers`, `users` (filtrado por `tenant_id`), `products`
cuentan **todas** las filas existentes sin filtrar por `active`; `payment_methods` cuenta solo
`WHERE active = true`. Esto se codifica como configuración declarativa en
`app/core/plan_limits.py` (una entrada por recurso: tabla/modelo, columna de límite en `Plan`,
si filtra por `active` o no), no como cinco funciones de conteo separadas y divergentes.

**Alternativas consideradas**:
- Duplicar la lógica de conteo dentro de cada uno de los cinco endpoints de creación
  (`orders/router.py`, `cash/router.py`, `users/router.py`, `products/router.py`,
  `sales/router.py`). Descartada: los cinco endpoints comparten exactamente la misma forma
  ("lockear tenant, contar filas con un filtro parametrizable, comparar contra un límite,
  bloquear o insertar") — duplicarla cinco veces es la clase de repetición que este proyecto evita
  cuando el patrón ya es idéntico, no una abstracción prematura sobre algo que todavía varía.

## Decisión 5 — Lock sobre la fila de `shared.tenants`, no una tabla de contadores nueva

**Decisión**: `enforce_plan_limit(db, tenant, resource_key)` bloquea la fila del tenant
(`select(Tenant).where(Tenant.id == tenant.id).with_for_update()`) antes de contar las filas
existentes del recurso y antes del `insert` que crea el nuevo registro, dentro de la misma
transacción/sesión de la request (mismo patrón que `InvoiceCounter._next_number()` y
`table_sessions._load(..., lock=True)`, ambos ya en `pos-backend`). Esto satisface FR-015
(garantía estricta bajo concurrencia): dos requests simultáneas para el mismo tenant serializan en
esa fila — la segunda espera a que la primera haga commit/rollback antes de poder contar, así que
nunca ambas ven el mismo cupo libre a la vez.

**Alternativas consideradas**:
- Una tabla de contadores nueva por `(tenant, resource_key)` (como `InvoiceCounter`, pero
  genérica). Descartada: `InvoiceCounter` existe porque el número de factura es un valor que se
  **asigna e incrementa** (no se puede recalcular contando filas hacia atrás sin romper huecos ya
  usados); el conteo de mesas/cajas/usuarios/productos/métodos de pago activos, en cambio, es
  siempre derivable con un `COUNT(*)` directo sobre la tabla real — agregar una tabla de
  contadores paralela introduciría una segunda fuente de verdad que puede desincronizarse (ej. si
  una fila se borra fuera del flujo normal), sin necesidad: el `COUNT` real ya es barato y
  correcto.
- Confiar únicamente en una constraint `CHECK`/trigger de base de datos para el límite. Descartada:
  el límite es un valor configurable en tiempo de ejecución (`Plan.*_limit`, editable por el Super
  Admin, FR-014: "se aplica de inmediato... sin requerir ninguna acción adicional") — una
  constraint declarada en el esquema no puede leer dinámicamente el plan vigente de cada tenant sin
  volverse, en la práctica, el mismo código de aplicación reescrito como trigger PL/pgSQL, con peor
  legibilidad y sin los mensajes de error explicativos que exige FR-006.

## Decisión 6 — Gating de "compras" a nivel de endpoint, no de router completo

**Decisión**: `require_module_access("inventario")` se aplica a nivel de router completo en
`app/api/v1/inventory/router.py` (`dependencies=[...]` del `APIRouter`); `promotions/router.py`
recibe el mismo tratamiento con `"promociones"`. Para "compras" — que hoy vive como endpoints
específicos (`POST /inventory/purchases`, `POST /inventory/purchases/order`,
`POST /inventory/purchases/{id}/receive`, `GET /inventory/purchases`) dentro del **mismo** router
que "inventario" — la dependencia `require_module_access("compras")` se agrega individualmente a
esos cuatro endpoints, no al router.

**Alternativas consideradas**:
- Extraer "compras" a un router/módulo Python separado antes de aplicar el gating. Descartada:
  sería una refactorización de la organización de módulos existente no pedida por el spec
  (Principio V) — el spec pide que el acceso se valide, no que el código se reorganice; separar el
  router es una decisión de arquitectura que esta spec no necesita tomar para cumplir sus FRs.
- No distinguir "compras" de "inventario" y tratarlos como un único módulo gateado. Descartada:
  contradice explícitamente el Key Entity del spec ("inventario, compras, promociones" listados
  como tres accesos de módulo independientes) y su Historia de Usuario 4 (ejemplos separados).

## Decisión 7 — Un solo endpoint (`GET /plan`) sirve la pantalla de consumo y el gating de navegación del frontend

**Decisión**: `app/api/v1/plan/router.py` expone `GET /plan` (tenant-scoped,
`require_tenant_admin`... reconsiderado: ver nota abajo), que devuelve el nombre del plan, el
consumo de cada límite (`{used, limit}` o `{used, limit: null}` para ilimitado) y el estado de
cada acceso de módulo. El frontend usa la misma respuesta para dos fines: pintar la pantalla "mi
plan" (US6/FR-013) y decidir, antes de navegar, si oculta/bloquea las rutas de inventario/
promociones y la pestaña de compras (`plan-module.guard.ts`).

**Nota sobre el rol que puede consultarlo**: `spec.md` (US6) dice "Tenant Admin", pero la
información que expone (consumo agregado del tenant, sin datos de otros tenants) no es sensible
para un CASHIER del mismo tenant, y el guard de navegación (Decisión 6 del Structure Decision en
`plan.md`) necesita poder consultarlo para cualquier usuario autenticado del dashboard, no solo
ADMIN — de lo contrario un CASHIER nunca podría recibir el mensaje de "tu plan no incluye este
módulo" y solo vería un 403 crudo. Se usa `get_current_user` (cualquier rol autenticado del
tenant), no `require_tenant_admin`, para este endpoint específico — más permisivo que lo que el
texto de la historia sugiere, pero necesario para que FR-009 ("mensaje claro" al intentar acceder)
funcione para cualquier rol, no solo ADMIN. Esto es una decisión técnica, no de negocio: no cambia
qué puede *hacer* un CASHIER, solo qué puede *leer* sobre el estado de su propio tenant.

**Alternativas consideradas**:
- Dos endpoints separados: uno para la pantalla de consumo (`GET /plan`, solo ADMIN) y otro más
  liviano para el gating de navegación (ej. `GET /plan/access`, cualquier rol). Descartada:
  ambos necesitan exactamente los mismos datos subyacentes (el plan vigente y sus características)
  — separar la respuesta en dos formas distintas del mismo dato es complejidad sin beneficio
  observable; un único endpoint accesible a cualquier usuario autenticado del tenant cubre ambos
  casos.

## Decisión 8 — Bloqueo por límite y por módulo responde `403`, no `409`

**Decisión**: tanto `enforce_plan_limit` (límite alcanzado) como `require_module_access` (módulo
no incluido) responden `HTTPException(status.HTTP_403_FORBIDDEN, detail=...)` con un mensaje que
menciona el límite/módulo (FR-006/FR-009), no `409 Conflict`.

**Alternativas consideradas**:
- `409 Conflict`, como ya usa `create_table`/`create_register` para duplicados (número de mesa
  repetido, etc.). Descartada: `409` en este código ya tiene un significado establecido ("el
  recurso que intentas crear choca con uno existente") — el bloqueo por plan no es un conflicto
  con otro recurso, es una restricción de autorización sobre lo que el tenant puede hacer, misma
  familia semántica que `403` ya usado por `require_tenant_admin`/`get_current_super_admin` en
  este mismo backend. Reusar `403` evita introducir un tercer significado para el mismo código de
  estado.

## Decisión 9 — Cambio de plan del tenant: nuevo endpoint en `super_admin/router.py`, sin sub-router propio

**Decisión**: `PATCH /super-admin/tenants/{id}` (body `{plan_id}`) se agrega directamente en
`app/api/v1/super_admin/router.py`, junto a los `GET /users`/`GET /tenants` ya existentes ahí —
no se extrae a un `tenants_router.py` separado (a diferencia de `payment_methods_router.py` en
spec 032, que sí se extrajo por ser un conjunto de endpoints más grande, CRUD completo).

**Alternativas consideradas**:
- Extraer un `tenants_router.py` nuevo, moviendo también `list_all_tenants` a ese archivo, para
  mantener simetría con `payment_methods_router.py`. Descartada: mover `list_all_tenants` (código
  existente, sin relación funcional con esta spec más allá de vivir en el mismo archivo) sería una
  reorganización no pedida por el spec (Principio V); un único endpoint `PATCH` nuevo no justifica
  por sí solo la extracción de un archivo nuevo.

---

Las Decisiones 10-17 se agregaron tras la sesión de ampliación de clarificaciones (2026-08-24 —
precio, duración y renovación), que extendió `spec.md` con FR-016 a FR-021 y las Historias de
Usuario 5 y 6.

## Decisión 10 — Ciclo de facturación y fechas de vencimiento viven en `Tenant`, no en una tabla de asignación

**Decisión**: `ciclo_facturacion` (`String(10)` nullable, `CHECK IN ('mensual','anual')` o
`NULL`), `plan_iniciado_en` y `plan_vence_en` (ambas `DateTime` nullable) se agregan como columnas
directas de `Tenant`, junto a `plan_id` — no se crea una tabla `tenant_plan_assignments` aunque
ahora la "asignación" tiene más de un campo.

**Alternativas consideradas**:
- Crear la tabla `tenant_plan_assignments` ahora que la asignación dejó de ser un único FK y pasó
  a tener cuatro campos relacionados (plan, ciclo, inicio, vencimiento). Descartada: las
  Assumptions de `spec.md` siguen exigiendo explícitamente "no se requiere un historial auditable
  de cambios de plan en esta fase" — el criterio que en Decisión 2 ya justificaba no crear esa
  tabla (una sola fila vigente por tenant, sin historial) no cambió por agregar más columnas a esa
  misma fila; agregar una tabla ahora sería anticipar una necesidad de historial que el spec sigue
  descartando explícitamente, no resolver un problema real de esta fase.
- Guardar solo `plan_vence_en` (calculada) y descartar `plan_iniciado_en`/`ciclo_facturacion` tras
  el cálculo. Descartada: sin `plan_iniciado_en`/`ciclo_facturacion` persistidos, la pantalla de
  consumo (US6) no podría mostrar "ciclo mensual, venció el 24 de septiembre" de forma consistente
  si el Super Admin quisiera auditar manualmente (aunque no haya historial, sí hay que poder leer
  el estado vigente completo) — y recalcular el vencimiento en cada lectura a partir de una fecha
  de inicio no guardada sería imposible.

## Decisión 11 — Precios como `Numeric(12, 2)`/`Decimal`, mismo patrón que el resto del dinero en el sistema

**Decisión**: `Plan.precio_mensual`/`Plan.precio_anual` son `Numeric(12, 2)` nullable en el
modelo, `Decimal` en los schemas Pydantic (`Field(..., ge=0, max_digits=12, decimal_places=2)`) —
exactamente el mismo tipo que ya usan `ProductVariant.price` (`app/models/product_variant.py`) y
todas las columnas monetarias de `Sale` (`subtotal`, `tax`, `tip`, `total`, `app/models/sale.py`).

**Alternativas consideradas**:
- `Integer` en centavos (patrón común en otros sistemas de facturación). Descartada: no es el
  patrón que ya usa este proyecto en ningún punto — introducir un segundo esquema de
  representación de dinero (centavos enteros vs. `Numeric(12,2)`) solo para el precio del plan
  sería inconsistente sin ninguna ventaja real, y obligaría a convertir en la UI donde el resto
  del sistema ya muestra pesos colombianos directamente.
- `Float`. Descartada por la razón usual (errores de redondeo en aritmética de punto flotante para
  dinero) — y porque el propio proyecto ya evitó `Float` para todo lo demás.

## Decisión 12 — `plan_vence_en` nullable, mismo patrón que `SessionParticipant.expires_at`; `NULL` = nunca vence

**Decisión**: `Tenant.plan_vence_en: Mapped[Optional[datetime]]` (`DateTime`, sin
`timezone=True`, almacenando siempre UTC naive — convención del proyecto entero, `app/core/
timezone.py`, spec 030) es nullable, y `NULL` significa "esta asignación nunca vence" — mismo
patrón exacto que `SessionParticipant.expires_at` (`app/models/session_participant.py:52`), cuyo
`NULL` ya es el sentinel de "no hace falta refrescar" en `app/core/qr_context.py:119`.

**Alternativas consideradas**:
- Una fecha "infinita" convencional (ej. `9999-12-31`) en vez de `NULL`, para evitar lógica
  condicional de "si es NULL, nunca vence" en cada validación. Descartada: el proyecto ya tiene un
  precedente idéntico resuelto con `NULL` (`SessionParticipant.expires_at`) — usar una fecha
  centinela introduciría una segunda convención para el mismo concepto ("nunca vence") sin
  necesidad, y una fecha mágica es más fácil de teclear mal por accidente en un backfill futuro
  que un `NULL` explícito.
- Una columna booleana separada `vencimiento_activo` en vez de que `NULL` en la fecha ya sea
  suficiente. Descartada: sería redundante — el propio `NULL`/no-`NULL` de `plan_vence_en` ya
  codifica exactamente ese booleano, igual que `NULL` en los límites numéricos de `Plan` ya
  codifica "ilimitado" sin una bandera aparte (mismo estilo que Decisión 1).

## Decisión 13 — Cálculo del vencimiento con `python-dateutil.relativedelta`, no aritmética manual

**Decisión**: `plan_vence_en = plan_iniciado_en + relativedelta(months=1)` (ciclo mensual) o
`+ relativedelta(years=1)` (ciclo anual), usando `python-dateutil` (ya en `requirements.txt`,
`2.9.0.post0`, instalado como dependencia transitiva pero nunca importado directamente en código
de aplicación hasta esta spec).

**Alternativas consideradas**:
- Aritmética manual con `datetime.replace(month=..., year=...)` y manejo a mano de desbordes de
  mes/día (ej. 31 de enero + 1 mes). Descartada: es exactamente el tipo de lógica de calendario
  que `relativedelta` ya resuelve de forma correcta y probada (maneja meses de distinta longitud y
  29 de febrero sin código adicional) — reimplementarla a mano es la clase de trabajo repetido que
  el proyecto prefiere evitar cuando la biblioteca estándar del ecosistema ya está instalada y
  disponible (Principio IX: preferencia por bibliotecas ya presentes antes que reinventar).
- `timedelta(days=30)`/`timedelta(days=365)`. Descartada: no es lo que "un mes"/"un año" significa
  para un Super Admin capturando una fecha de renovación — un plan mensual asignado el 31 de enero
  debe vencer el último día de febrero, no 30 días después; `timedelta` no representa esa
  semántica de calendario en absoluto.

## Decisión 14 — El vencimiento se evalúa al vuelo en cada request, no con un job en segundo plano

**Decisión**: `ensure_plan_not_expired(tenant)` en `app/core/plan_limits.py` compara
`tenant.plan_vence_en` contra `utc_now()` en el momento de cada validación (llamada internamente
por `enforce_plan_limit` y `require_module_access`, nunca por un caller externo) — no existe
ningún proceso programado (Celery beat, cron) que recorra tenants y cambie su estado de forma
proactiva cuando vencen.

**Alternativas consideradas**:
- Un job periódico que marque tenants vencidos con una columna `bloqueado: bool` explícita, leída
  después por las validaciones. Descartada: agregaría un estado derivado que puede desincronizarse
  del dato real (`plan_vence_en`) si el job falla o se atrasa — y el propio proyecto ya tiene
  infraestructura de tareas asíncronas (`app/celery_task.py`, usada para el correo de bienvenida)
  cuya disponibilidad no está garantizada en el camino crítico de una validación de límite/módulo
  (ver `tenant_create()`, que ya trata el fallo de Celery como no bloqueante). Comparar dos
  `datetime` en cada request es más barato y más confiable que depender de que un worker en
  segundo plano haya corrido a tiempo.
- Verificar el vencimiento solo en el login/JWT, cacheando el resultado en el token. Descartada:
  un JWT ya emitido seguiría siendo válido después de que el plan venciera a mitad de sesión —
  contradice FR-019 ("desde ese momento") y el criterio de aplicación inmediata ya establecido
  para el resto del sistema de planes (FR-014/FR-010, SC-005).

## Decisión 15 — `ciclo_facturacion` es un campo obligatorio que acepta `null` como "sin vencimiento"

**Decisión**: en los schemas Pydantic (`TenantCreateWithUser`, `TenantPlanUpdate`),
`ciclo_facturacion: Literal["mensual", "anual"] | None` se declara **sin valor por defecto** —
el Super Admin debe incluir la clave explícitamente en el request, y `null` es un valor válido
que produce una asignación sin vencimiento (research.md Decisión 12).

**Alternativas consideradas**:
- `ciclo_facturacion: Literal["mensual", "anual"] | None = None` (con default implícito).
  Descartada: contradice literalmente FR-004/FR-017 ("no existe ningún mecanismo de plan por
  defecto implícito" / "el sistema MUST exigir un ciclo de facturación... como parte de toda
  asignación"): con un default, un Super Admin que omite el campo por descuido crearía sin darse
  cuenta un tenant que nunca vence, en vez de recibir un error de validación que lo obligue a
  decidir. Sin default, Pydantic rechaza el request con `422` si la clave falta — la ausencia de
  vencimiento solo ocurre cuando alguien la elige activamente enviando `null`.
- Dos campos separados: `ciclo_facturacion: Literal["mensual","anual"]` obligatorio (sin `None`) +
  `sin_vencimiento: bool` aparte. Descartada: introduce un estado inconsistente representable
  (`sin_vencimiento=true` con un `ciclo_facturacion` que de todas formas exige un valor) sin
  necesidad — un único campo que acepta `null` como tercer valor explícito no tiene esa
  ambigüedad.

## Decisión 16 — Renovar reutiliza `PATCH /super-admin/tenants/{id}`, no un endpoint nuevo

**Decisión**: no existe un endpoint `POST .../renovar` separado. `PATCH
/super-admin/tenants/{id}` con el mismo `plan_id` que el tenant ya tenía (y el mismo u otro
`ciclo_facturacion`) es, por definición, una renovación: siempre reinicia `plan_iniciado_en =
utc_now()` y recalcula `plan_vence_en` a partir de ese momento, sin importar si `plan_id` cambió o
no.

**Alternativas consideradas**:
- Un endpoint `POST /super-admin/tenants/{id}/renovar` distinto del `PATCH` de cambio de plan.
  Descartada: "cambiar de plan" y "renovar el mismo plan" son, en términos de datos, exactamente
  la misma operación (recalcular `plan_iniciado_en`/`plan_vence_en` a partir de un `plan_id` y un
  `ciclo_facturacion`) — la única diferencia es si `plan_id` coincide con el valor anterior o no,
  algo que el propio `PATCH` ya puede expresar sin necesitar un segundo endpoint ni duplicar su
  validación de precio/ciclo (FR-017).

## Decisión 17 — La migración de datos también neutraliza el vencimiento para tenants existentes

**Decisión**: la misma migración que sembró el plan transicional sin límites (Decisión 3) deja
`ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` en `NULL` para todo tenant existente al
momento del despliegue — no solo `plan_id` apuntando al plan transicional.

**Alternativas consideradas**:
- Calcular una fecha de vencimiento "razonable" (ej. un mes desde el despliegue) para tenants
  existentes en vez de dejarlos sin vencimiento. Descartada: violaría directamente el Principio II
  — un tenant que hoy nunca se bloquea empezaría a arriesgarse a bloquearse por una fecha que el
  Super Admin nunca eligió, sin ninguna decisión de negocio que lo autorice. `NULL` (sin
  vencimiento) es la única opción que dejar el comportamiento observable de un tenant existente
  exactamente igual el día del despliegue, consistente con el resto de la migración transicional.
