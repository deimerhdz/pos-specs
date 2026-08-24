# Research: Corrección global de fechas, horas y zonas horarias

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — se resolvió leyendo
directamente `pos-backend` y `pos-heladeria` (las once entidades, sus schemas de respuesta, los dos
filtros de rango existentes, los cinco sitios de `DatePipe`, el formateador manual de
`cash-session.store.ts`, el helper `local_now()`/`_tz()` ya introducido por spec 012, y el registro de
anomalías). Este documento registra las decisiones de diseño y sus alternativas.

## Decisión 1 — Dónde vive el mecanismo central del backend: `app/core/timezone.py`, no dentro de `promotions/service.py`

- **Decisión**: crear `app/core/timezone.py`, junto a `config.py`/`models.py`, con cuatro piezas:
  `resolve_timezone(tenant: Tenant) -> ZoneInfo`, el tipo `UtcDatetime` (`Annotated[datetime,
  PlainSerializer(...)]`), `local_day_bounds_utc(day: date, tz: ZoneInfo) -> tuple[datetime,
  datetime]` y `utc_now() -> datetime`.
- **Rationale**: `promotions/service.py` ya tiene `_tz()`/`local_now()` (líneas 50-63), y su propio
  docstring anticipa exactamente este momento ("cuando `Tenant` tenga su columna `timezone`, este es
  el único punto que cambia"). Pero ese fichero es el motor de evaluación de promociones protegido por
  A-07 — mezclar ahí la serialización de la API (`UtcDatetime`) y el cálculo de límites de filtro
  (`local_day_bounds_utc`), que no tienen nada que ver con evaluar vigencia de promociones, ensancha
  el radio de un fichero protegido sin necesidad (Principio III/V). `app/core/` ya es, en este
  repositorio, el lugar de utilidades transversales sin dueño de dominio único (`config.py`, `db.py`,
  `utils.py`) — es el lugar natural para un mecanismo que once entidades distintas van a importar.
- **Alternatives considered**:
  - **Generalizar `_tz()`/`local_now()` in situ y que todo el resto importe desde
    `promotions/service.py`** — descartado: acopla la serialización de `Sale`/`Order`/`Invoice`/etc.
    a un módulo cuyo contrato protegido es otro (A-07), y cualquier cambio futuro al motor de
    promociones arrastraría revisión de un import que nada tiene que ver con promociones.
  - **Un mecanismo por entidad** (una función de conversión en cada `schemas.py`) — es exactamente lo
    que `FR-002` prohíbe explícitamente ("en lugar de una implementación de conversión distinta por
    módulo, componente o pantalla").

## Decisión 2 — Formato de transporte UTC inequívoco: `Annotated[datetime, PlainSerializer(...)]`, no `DateTime(timezone=True)` ni `json_encoders` globales

- **Decisión**: `UtcDatetime = Annotated[datetime, PlainSerializer(lambda dt: dt.replace(tzinfo=
  timezone.utc).isoformat() if dt.tzinfo is None else dt.astimezone(timezone.utc).isoformat(),
  return_type=str)]`, aplicado como anotación de tipo en cada campo de schema Pydantic que representa
  un instante absoluto (reemplaza `datetime` por `UtcDatetime` en la firma del campo, nada más).
- **Rationale**: satisface `FR-003` (offset explícito, `+00:00` en vez de una cadena naive
  ambigua) sin tocar ninguna columna de base de datos — la lectura de SQLAlchemy sigue devolviendo un
  `datetime` naive que Postgres ya garantiza en UTC (`FR-007`, cero cambio de dato ni de tipo de
  columna); el `.replace(tzinfo=timezone.utc)` es una operación de interpretación en el momento de
  serializar, no de conversión de valor. Es además la unidad de reutilización correcta para `FR-002`:
  un solo tipo, importado y usado como anotación en los ~13 campos de las once entidades, en vez de
  una función de conversión que cada schema tendría que recordar invocar.
- **Alternatives considered**:
  - **Migrar las columnas a `DateTime(timezone=True)`** — descartado explícitamente: aunque
    Postgres seguiría almacenando UTC, cambiar el tipo de columna es un cambio de esquema en las once
    tablas que `FR-007`/Principio VII no requieren y que introduce riesgo de migración (locking,
    downtime) para un problema que es puramente de serialización. La investigación confirmó que el
    dato ya es UTC consistente — no hay nada que migrar.
  - **`json_encoders`/`default_response_class` global en `main.py`** — descartado: Pydantic v2
    desaconseja `json_encoders` (deprecado a favor de `field_serializer`/tipos anotados), y un
    encoder verdaderamente global no distingue entre los campos de instante absoluto (que sí deben
    llevar offset) y cualquier otro campo `datetime`/`date` que no lo sea (ninguno detectado hoy, pero
    un encoder global sería una regla implícita difícil de auditar campo por campo).

## Decisión 3 — `Tenant.timezone`: columna con validación a nivel de modelo (`@validates`), no solo en el endpoint

- **Decisión**: `Tenant` gana `timezone: Mapped[str] = mapped_column(String, nullable=False,
  server_default="America/Bogota")`, más un validador SQLAlchemy `@validates("timezone")` que ejecuta
  `zoneinfo.ZoneInfo(value)` y relanza como error de aplicación si `ZoneInfoNotFoundError` — el valor
  inválido nunca llega a `INSERT`/`UPDATE` (Clarifications, spec.md).
- **Rationale**: la Clarificación de spec.md exige que la validación ocurra "en el momento de
  escribirlo (migración, script o endpoint interno)" — múltiples caminos de escritura posibles. Un
  validador a nivel de modelo protege los tres por igual (la migración usa un valor literal ya válido
  por diseño; el script nuevo y cualquier código ORM futuro pasan por el mismo `@validates` sin
  duplicar la regla en cada uno). Poner la validación solo en un schema Pydantic de un hipotético
  endpoint dejaría el script y cualquier acceso ORM directo sin protección.
- **Alternatives considered**:
  - **Validar solo en el schema del endpoint interno** — descartado por lo anterior: no cubre el
    script ni escrituras ORM directas.
  - **`CHECK` constraint de Postgres con una lista fija de zonas** — descartado: la base de datos IANA
    tiene cientos de nombres y cambia (aunque rara vez); mantener una lista espejo en un `CHECK` de
    SQL es una segunda fuente de verdad que puede desincronizarse de la que Python ya trae
    incorporada (`zoneinfo`), sin ganar nada sobre validar en la capa de aplicación.

## Decisión 4 — Cómo se configura `Tenant.timezone`: script interno, no un endpoint HTTP nuevo

- **Decisión**: `app/scripts/set_tenant_timezone.py`, mismo patrón que `seed_super_admin.py`
  (`argparse` + función pura reusable + `with_db(None)` porque `Tenant` vive en `shared`), ejecutado
  manualmente (`python -m app.scripts.set_tenant_timezone --host <host> --timezone <tz>`). No se
  agrega ningún campo `timezone` a `TenantUpdateRequest`/al `PATCH /tenant` que `TenantInfoService
  .update()` ya expone para otros campos (`name`, `logo_url`, `receipt_message`, etc.).
- **Rationale**: decisión ya tomada en la Clarificación de spec.md ("Backend-only field ...
  sin pantalla nueva de autoservicio"). Excluir `timezone` de `TenantUpdateRequest` explícitamente
  (no solo "no construir una pantalla que lo use") es necesario porque `PATCH /tenant` ya existe de
  forma genérica para otros campos — si `timezone` se agregara a ese schema sin querer, cualquier
  cliente que ya llame ese endpoint (incluida una futura pantalla de configuración del tenant que no
  tenga que ver con esta spec) podría escribirlo sin pasar por la decisión explícita de "sin
  autoservicio en esta spec".
- **Alternatives considered**:
  - **Agregar el campo a `TenantUpdateRequest` pero no construir UI para él** — descartado: un campo
    aceptado por un endpoint ya autenticado y genérico es, en la práctica, autoservicio aunque no
    tenga botón — cualquier llamada directa a la API ya let it through. La Clarificación pide
    explícitamente que el camino sea "vía el proceso de aprovisionamiento/soporte existente", no la
    API pública del tenant.
  - **Endpoint interno nuevo, no genérico** (p. ej. `POST /internal/tenants/{id}/timezone`, protegido
    por rol super-admin) — evaluado y descartado por ahora: agrega superficie de autenticación/rol
    nueva para un caso de uso de baja frecuencia (aprovisionamiento) que el script ya cubre sin abrir
    un endpoint HTTP adicional; si en el futuro el equipo de soporte necesita hacerlo sin acceso a la
    consola del servidor, se puede añadir como spec independiente.

## Decisión 5 — Filtros de rango: `local_day_bounds_utc()` en el backend, sin cambios en qué envía el frontend

- **Decisión**: `local_day_bounds_utc(day: date, tz: ZoneInfo) -> tuple[datetime, datetime]` calcula
  la medianoche local del tenant para `day` (`datetime.combine(day, time.min, tzinfo=tz)`) y su
  siguiente medianoche, ambas convertidas y devueltas como `datetime` naive en UTC (mismo tipo que ya
  espera comparar contra las columnas naive). `sales/service.py::build_sales_query` (190-216) y
  `reports/service.py::_paid_sales_filter` (1-27) reemplazan `Sale.sold_at >= date_from` /
  `< date_to + timedelta(days=1)` por `Sale.sold_at >= local_day_bounds_utc(date_from, tz)[0]` /
  `< local_day_bounds_utc(date_to, tz)[1]`, recibiendo `tz` desde el `tenant` ya resuelto por
  `Depends(get_tenant)` en el router correspondiente (se agrega como parámetro nuevo a ambas
  funciones). El bucketing por día/mes de `reports/service.py` (`func.date_trunc`/`func.date` sobre
  `Sale.sold_at`) se corrige con el doble `AT TIME ZONE` de Postgres:
  `func.timezone(tz_name, func.timezone('UTC', Sale.sold_at))` antes de aplicar `date_trunc`/`date` —
  el idiom estándar para reinterpretar un `timestamp without time zone` que ya es UTC como un instante
  en una zona dada, sin cambiar el valor almacenado.
- **Rationale**: el frontend ya envía `date_from`/`date_to` como `YYYY-MM-DD` puros, sin componente de
  hora ni de zona (`input[type=date]`, sección 6 de la investigación) — no hay ninguna ambigüedad que
  resolver del lado del cliente; el defecto entero está en que el backend interpreta ese `date` contra
  medianoche UTC. Centralizar la interpretación en un solo punto server-side es la lectura más directa
  de `FR-004` ("los filtros ... DEBEN calcular los límites del día usando la zona horaria del
  negocio") y evita duplicar lógica de "medianoche local" en dos lenguajes (Python y TypeScript).
- **Alternatives considered**:
  - **Que el frontend calcule y envíe el instante UTC exacto de la medianoche local** — descartado:
    obligaría al frontend a conocer la zona horaria del tenant *antes* de construir cada petición de
    filtro (orden de carga/carrera con `TenantInfoService.load()`), y duplicaría en TypeScript la
    misma lógica de "medianoche local → UTC" que Python ya necesita para el bucketing de reportes, que
    solo puede vivir en SQL/backend.
  - **Guardar la zona horaria en una variable de sesión de Postgres (`SET timezone`) y dejar que
    Postgres haga la conversión implícita** — descartado: cambiaría el comportamiento de *toda* la
    conexión (incluyendo columnas y funciones no relacionadas con esta spec) por el resto de la
    sesión, un radio de efecto mucho mayor y más difícil de auditar que una conversión explícita por
    consulta.

## Decisión 6 — Mecanismo único del frontend: `TenantDatePipe` envolviendo el `DatePipe` nativo, no un formateador con `Intl` desde cero

- **Decisión**: `TenantDatePipe` (standalone, `src/app/shared/pipes/tenant-date.pipe.ts`) inyecta el
  `DatePipe` de `@angular/common` y `TenantInfoService`, y en `transform(value, format)` llama
  `this.datePipe.transform(value, format, this.tenantInfo.info()?.timezone ?? 'America/Bogota')` — el
  `DatePipe` de Angular acepta un tercer argumento de zona horaria IANA desde hace varias versiones
  mayores. Se usa declarativamente en plantilla (`| tenantDate:'dd/MM/yyyy HH:mm'`, reemplaza los 5
  sitios de `| date` desnudo) y se inyecta directamente (sin plantilla) en `cash-session.store.ts`,
  donde `fmtTime`/`fmtDate` llaman `this.tenantDate.transform(iso, 'HH:mm')` /
  `this.tenantDate.transform(iso, 'dd/MM/yyyy HH:mm')` en vez de `new Date(iso).toLocaleString(...)`.
- **Rationale**: reutiliza el formateador de fechas de Angular, ya probado y usado en 5 sitios del
  propio código, en vez de reimplementar reglas de formato (separadores, orden día/mes,
  `Intl.DateTimeFormat` con opciones manuales) que `DatePipe` ya resuelve correctamente dado un string
  de zona horaria — coincide con la preferencia del proyecto por la solución más simple
  (Principio IX). Una sola clase satisface `FR-002` en el frontend: es literalmente el mismo mecanismo
  en ambos usos (plantilla e inyección directa), no dos implementaciones que coincidan por
  casualidad.
- **Alternatives considered**:
  - **Dos mecanismos, uno declarativo (`Pipe`) y otro programático (`Intl.DateTimeFormat` a mano) que
    "hagan lo mismo"** — es exactamente el patrón que existe hoy (`DatePipe` desnudo en 5 plantillas
    vs. `toLocaleString` a mano en `cash-session.store.ts`) y es la causa por la que hay dos defectos
    de zona horaria en el frontend en vez de uno — descartado por ser la antítesis de `FR-002`.
  - **Servicio nuevo `DateFormatService` en vez de un `Pipe` inyectable** — descartado por
    redundante: en Angular standalone, un `Pipe` ya es una clase inyectable (`@Injectable` implícito
    vía `@Pipe({..., standalone: true})`) — crear un servicio adicional que delegue en el pipe (o
    viceversa) sería una capa de indirección sin necesidad.

## Decisión 7 — "Hoy" en Reportes: `businessToday(timezone)` con `Intl.DateTimeFormat`, no dependiente del calendario del navegador

- **Decisión**: `date-format.util.ts::businessToday(timezone: string): string` devuelve
  `Intl.DateTimeFormat('en-CA', { timeZone: timezone }).format(new Date())` (formato `YYYY-MM-DD`,
  igual que el `fmt()` que `reports.service.ts::getDateRange` ya usaba, pero con el `timeZone`
  correcto en vez de omitido). `getDateRange` (250-279) sustituye
  `new Date(now.getFullYear(), now.getMonth(), now.getDate())` (calendario del navegador) por este
  helper para los períodos `'today'`/`'week'`/`'month'`/`'year'`; el caso `'specific-date'` **no
  cambia** — ya usa el string que el propio usuario eligió en el `input[type=date]`, sin pasar por el
  reloj del dispositivo en absoluto.
- **Rationale**: la investigación confirmó que `getDateRange` computa "hoy" a partir de `new Date()`
  del navegador — si el sistema operativo del terminal tiene una zona distinta a la del tenant, "Hoy"
  puede no coincidir con el día de negocio real, exactamente el escenario que spec.md Edge Cases
  exige no depender de "esa configuración externa". `Intl.DateTimeFormat` con `timeZone` es la única
  API nativa (sin librería nueva) que responde "qué día es en esa zona horaria ahora mismo",
  reutilizando el mismo patrón `en-CA` → `YYYY-MM-DD` que el código ya usaba, solo corrigiendo el
  parámetro que faltaba.
- **Alternatives considered**:
  - **Que el backend calcule "Hoy" y lo exponga en un endpoint** — descartado por desproporcionado:
    "Hoy" en la zona del tenant es una función pura de la hora actual y una cadena de zona horaria ya
    disponible en el cliente (`TenantInfoService.info().timezone`); no hay ninguna razón de negocio
    para hacer un viaje de red solo para preguntarle al backend qué fecha es.

## Decisión 8 — Alcance exacto de `utc_now()` (FR-008): solo los sitios que alimentan una columna `DateTime` naive persistida

- **Decisión**: `utc_now()` (envoltorio trivial de `datetime.now(timezone.utc)`) reemplaza únicamente
  los ocho sitios citados explícitamente en spec.md → Edge Cases (`cash/router.py:121`,
  `checkout.py:811/819/873/925/933`, `qr_context.py:85/179`, `table_sessions/service.py:177/652/739`
  — el valor que terminan asignando a un campo de modelo mapeado a una columna `DateTime` naive
  (`closed_at`, `resolved_at`, `closed_at` de `SessionParticipant`, el "ahora" que
  `promotions.evaluate`/`combo_discount_for_lines` reciben antes de que su resultado alimente
  `Sale`/`CustomerOrder`). Los demás 24 sitios de `datetime.now(timezone.utc)` que la investigación
  encontró (`redis.py`, `scheduler.py`, `qr_token.py`, `db.py:179`, `events.py:128`,
  `menu/router.py:82`, `cart/service.py`, `orders/tables_advanced.py:107`,
  `orders/consolidation.py:192`, `orders/service.py:188`, `promotions/router.py:53`) **no cambian**
  en esta spec.
- **Rationale**: `FR-008` habla explícitamente de "puntos ... donde hoy se construye la hora actual de
  forma independiente para asignarla a un registro" — un criterio de selección preciso: ¿el valor
  termina en una columna persistida de instante absoluto? Los sitios excluidos construyen "ahora" para
  JWT (`qr_token.py`, expiración de token, no una columna de entidad), TTL de Redis (`redis.py`, no
  persistido en Postgres), planificación en memoria del scheduler (`scheduler.py`), o cálculos
  intermedios de negocio que no se guardan tal cual (p. ej. evaluación de disponibilidad en
  `cart/service.py`). Unificar esos también sería exceder el alcance que `FR-008` define y el riesgo
  que Historia 6/`FR-007` busca acotar (ningún valor histórico cambia) — cuantos menos sitios de
  escritura se toquen sin necesidad, menor la superficie de riesgo sobre datos ya persistidos.
- **Alternatives considered**:
  - **Unificar los 32 sitios encontrados** — descartado por desproporcionado frente a `FR-008` y
    riesgo innecesario sobre código de autenticación/caché/scheduler que esta spec no necesita tocar
    para cumplir ningún criterio de aceptación de `spec.md`.

## Decisión 9 — `Payment.paid_at` no está expuesto en `PaymentResponse`: se agrega como parte mínima necesaria de esta spec

- **Decisión**: `PaymentResponse` (`sales/schemas.py:94-100`) gana `paid_at: UtcDatetime`, leyendo el
  campo ya existente en el modelo (`app/models/payment.py:57`, sin cambios).
- **Rationale**: Historia 2/Key Entities de spec.md nombra explícitamente "Payment (Pago): su fecha de
  pago (`paid_at`)" como una de las once entidades cuya visualización esta spec corrige — sin exponer
  el campo en el schema de respuesta, es imposible mostrarlo en ninguna pantalla y por lo tanto
  imposible cumplir `SC-002`/`FR-010` (test por entidad) para Pagos. No es una funcionalidad nueva ni
  una expansión de alcance: es el mínimo necesario para que el contrato ya exigido por la propia spec
  sea alcanzable.
- **Alternatives considered**:
  - **Excluir Payment de esta corrección y dejarlo para una spec futura** — descartado: contradice
    directamente Historia 2 de spec.md, que lista `Payment.paid_at` de forma explícita como parte del
    mismo patrón defectuoso a corregir en esta misma spec.

## Decisión 10 — `promotions/service.py::_tz()`: cambio mínimo aditivo para cerrar A-46, sin tocar el criterio de A-07

- **Decisión**: `_tz(tenant: Tenant | None = None) -> ZoneInfo` — si se pasa `tenant`, devuelve
  `resolve_timezone(tenant)`; si no (compatibilidad con los llamadores actuales que no tienen el
  tenant a mano), conserva el comportamiento de hoy (`ZoneInfo(settings.TENANT_TIMEZONE)`). Los
  callers de `promotions/service.py` que sí resuelven `tenant` vía `Depends(get_tenant)` en su router
  (p. ej. `checkout.py`, `table_sessions/service.py` una vez reciban `tenant` para
  `local_day_bounds_utc` — Decisión 5) empiezan a pasarlo; `active_discount_promotions`,
  `best_line_discount` y el criterio de prioridad/desempate **no cambian ni una línea**.
- **Rationale**: la "Autorización de negocio" de spec.md reabre explícitamente el tratamiento de A-46
  para "introducir la columna de zona horaria por tenant que su propio 'Tratamiento acordado' preveía
  como paso siguiente" — el propio comentario de `_tz()` en el código ya señalaba este punto exacto
  como el único que cambiaría. Hacerlo con un parámetro opcional con valor por defecto es
  100% retrocompatible: ningún caller existente que no pase `tenant` cambia de comportamiento, y para
  el único tenant que existe hoy en producción (`America/Bogota` en ambos lugares) el resultado es
  idéntico exista o no el parámetro — cero riesgo de regresión sobre A-07.
- **Alternatives considered**:
  - **No tocar `promotions/service.py` en esta spec y dejar A-46 reabierta pero sin resolver** —
    descartado: contradice la autorización de negocio explícita de spec.md, que pide cerrar A-46
    como parte de esta misma spec (Historia 4, `FR-005`), no solo documentar la columna nueva sin
    conectarla a ningún consumidor real de zona horaria.
  - **Forzar `tenant` como parámetro obligatorio (rompiendo la firma actual)** — descartado: obligaría
    a tocar cada caller de `_tz()`/`local_now()` en el mismo cambio, incluidos posiblemente algunos
    fuera del alcance de esta spec, violando Principio V/VI (no mezclar más superficie de la
    necesaria en un mismo incremento).

## Decisión 11 — Formularios de fecha (Historia 5/FR-006): verificado con test de regresión, sin cambio de código de producción

- **Decisión**: no se modifica ningún `input[type=date]` de `sales-page.component.ts`,
  `reports-page.component.ts` ni `promotions-page.component.ts` — la investigación no encontró ningún
  punto donde el valor elegido pase por un `new Date(string)` intermedio que pudiera introducir
  corrimiento antes de reenviarse o volver a mostrarse (los tres bindings son string-a-string directos
  vía `ngModel`/`[value]`). Se agrega, en cambio, un test de regresión por cada uno de los tres
  formularios que fije este comportamiento (`FR-010` ya exige verificación explícita, y `Historia 5`
  es, en la práctica, una garantía preventiva más que un defecto activo encontrado).
- **Rationale**: introducir código nuevo para un defecto que la investigación no logró reproducir en
  ningún punto real del código violaría Principio V (no inventar cambios sin una necesidad
  identificada) — el riesgo que Historia 5 anticipa (un `Date` de JavaScript corriendo el día por el
  offset del navegador) es real *en general* para este tipo de defecto, pero específicamente no está
  presente hoy en estos tres sitios porque ninguno construye un objeto `Date` a partir del string
  elegido. Fijar el comportamiento actual correcto con un test cumple el espíritu de Historia 5
  (garantía verificable) sin escribir código que no tiene ningún defecto que corregir.
- **Alternatives considered**:
  - **Envolver los tres inputs en un componente de fecha nuevo "a prueba de corrimiento" de todas
    formas** — descartado por sobre-ingeniería: no hay un defecto que este componente nuevo
    resolvería que el string-a-string actual ya no resuelva; construir una abstracción para un riesgo
    hipotético contradice la guía explícita del proyecto.
