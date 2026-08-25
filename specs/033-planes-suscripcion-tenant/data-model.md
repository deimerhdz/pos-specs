# Fase 1 — Data Model: Planes de Suscripción por Tenant

Ver decisiones técnicas detalladas en [research.md](./research.md). Este documento describe las
entidades, campos, validaciones y migraciones — no el DDL completo (eso pertenece a la
implementación).

## Entidades

### `Plan` (nueva — esquema `shared`, tabla `plans`)

Corresponde a "Plan" y "Característica de Plan" en `spec.md` (Decisión 1 de research.md: ocho
características como columnas fijas, no un modelo dinámico). Administrado exclusivamente por el
Super Admin (FR-001).

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID (PK) | |
| `name` | `String(100)`, único | Ej. "Básico", "Pro", "Premium" |
| `description` | `String(500)`, nullable | Texto libre (FR-001) |
| `mesas_limit` | `Integer`, nullable, default `0` | `NULL` = ilimitado (FR-007). `0`/no configurado = bloqueado (FR-002) |
| `cajas_limit` | `Integer`, nullable, default `0` | Ídem |
| `usuarios_limit` | `Integer`, nullable, default `0` | Ídem |
| `productos_limit` | `Integer`, nullable, default `0` | Ídem |
| `metodos_pago_activos_limit` | `Integer`, nullable, default `0` | Ídem — cuenta solo métodos de pago `active=true` del tenant (ver reglas de conteo abajo) |
| `inventario_access` | `Boolean`, default `false` | Acceso al módulo de inventario (FR-008) |
| `compras_access` | `Boolean`, default `false` | Acceso al submódulo de compras (dentro de inventario, ver research.md Decisión 6) |
| `promociones_access` | `Boolean`, default `false` | Acceso al módulo de promociones |
| `precio_mensual` | `Numeric(12, 2)`, nullable | Precio de referencia para el ciclo mensual (FR-016). `NULL` = sin precio definido para ese ciclo — ningún tenant puede elegir ciclo `mensual` con este plan hasta que se defina (FR-017) |
| `precio_anual` | `Numeric(12, 2)`, nullable | Ídem para el ciclo anual, independiente del mensual |
| `created_at` / `updated_at` | `DateTime` | Timestamps estándar del proyecto |

**Reglas**:
- `name` único a nivel de plataforma.
- No existe borrado físico mientras el plan tenga al menos un tenant asignado (`Tenant.plan_id`
  es `NOT NULL` FK sin `ON DELETE CASCADE` — Postgres rechaza el `DELETE` con violación de FK
  mientras exista al menos una fila de `tenants` apuntando a ese plan). El Super Admin debe
  reasignar esos tenants a otro plan antes de poder eliminarlo (Assumptions de `spec.md`).
- Editar cualquier columna de límite/acceso aplica de inmediato a todos los tenants con ese
  `plan_id` — no hay ninguna copia ni caché por tenant (FR-014): cada validación lee `Plan` en
  el momento de la solicitud.

### `Tenant` (modificada — esquema `shared`, tabla `tenants`, ya existente)

Columnas relevantes para esta spec, resaltadas las nuevas/eliminadas.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | Integer (PK) | Sin cambios |
| `name` / `schema` / `host` | — | Sin cambios |
| ~~`plan`~~ | ~~`String(100)`~~ | **Eliminada** — columna heredada sin uso (research.md Decisión 2) |
| **`plan_id`** | UUID (FK → `shared.plans.id`), **`NOT NULL`** | Nueva. Garantiza FR-003 a nivel de esquema: ningún `INSERT`/`UPDATE` puede dejar un tenant sin plan |
| **`ciclo_facturacion`** | `String(10)`, nullable, `CHECK IN ('mensual','anual')` o `NULL` | Nueva. Ciclo elegido para la asignación vigente (FR-017). `NULL` = sin ciclo, y por tanto sin vencimiento (FR-021) — estado válido y permanente, no solo transitorio |
| **`plan_iniciado_en`** | `DateTime` (naive UTC), nullable | Nueva. Momento en que se asignó o renovó por última vez (FR-018/FR-020). `NULL` únicamente cuando `ciclo_facturacion` también es `NULL` |
| **`plan_vence_en`** | `DateTime` (naive UTC), nullable | Nueva. Calculada como `plan_iniciado_en + 1 mes` (ciclo mensual) o `+ 1 año` (ciclo anual), vía `dateutil.relativedelta` (research.md Decisión 13). `NULL` = nunca vence, mismo patrón que `SessionParticipant.expires_at` (research.md Decisión 12) |

**Reglas**:
- `plan_id` es obligatorio desde la creación del tenant (FR-004): `tenant_create()` en
  `app/core/db.py` recibe `plan_id` como parámetro requerido y crea la fila `Tenant` con ese
  valor dentro de la misma transacción que crea el esquema y el primer usuario ADMIN — si
  `plan_id` no referencia un `Plan` existente, la transacción completa hace rollback (mismo
  comportamiento de `IntegrityError` que ya maneja `tenant_create` hoy).
- Cambiar `plan_id` (Super Admin, FR-010) es un `UPDATE` directo, sin efecto sobre los recursos ya
  creados del tenant (FR-011/FR-012) — el nuevo límite/acceso solo se evalúa en la **siguiente**
  creación/acceso, nunca retroactivamente sobre lo ya existente.
- `ciclo_facturacion` MUST elegirse explícitamente en toda asignación o cambio de plan (FR-017);
  el schema Pydantic no le da valor por defecto, así que omitirlo es un `422`, no un `NULL`
  implícito (research.md Decisión 15). Elegir `mensual` cuando `Plan.precio_mensual IS NULL` (o
  `anual` con `Plan.precio_anual IS NULL`) es un `409`.
- Toda asignación o cambio de plan (creación de tenant, `PATCH /super-admin/tenants/{id}`, sea
  para cambiar de plan o para renovar el mismo) recalcula `plan_iniciado_en = utc_now()` y
  `plan_vence_en` a partir de `ciclo_facturacion` — nunca se "extiende" una fecha de vencimiento
  existente sumándole un período; siempre se recalcula desde el momento de la operación
  (FR-018/FR-020, research.md Decisión 16).

**Key Entity "Asignación de Plan por Tenant" del spec**: no existe como tabla propia — se realiza
íntegramente como las cuatro columnas `Tenant.plan_id`/`ciclo_facturacion`/`plan_iniciado_en`/
`plan_vence_en` (research.md Decisiones 2 y 10). No hay historial de cambios de plan en esta fase
(Assumptions de `spec.md`) — solo se conoce la asignación vigente, nunca las anteriores.

### Recursos gobernados por límite (sin cambios de modelo — solo se agrega validación antes del insert)

| Recurso | Modelo / tabla | Regla de conteo (FR-005, clarificaciones) |
|---|---|---|
| Mesas | `DiningTable` / `tenant.dining_tables` | Todas las filas (no tiene noción de "inactiva" a efectos de este conteo) |
| Cajas | `CashRegister` / `tenant.cash_registers` | Todas las filas, sin filtrar por `active` |
| Usuarios | `User` / `shared.users`, filtrado por `tenant_id` | Todas las filas del tenant, sin filtrar por `active` |
| Productos | `Product` / `tenant.products` | Todas las filas, sin filtrar por `active` |
| Métodos de pago activos | `PaymentMethod` / `tenant.payment_methods` | Solo filas con `active = true` |

Ninguno de estos cinco modelos cambia — la única adición es la llamada a
`enforce_plan_limit(db, tenant, resource_key)` antes del `db.add(...)` en cada uno de los cinco
endpoints de creación (ver `plan.md` Project Structure).

### Módulos gobernados por acceso (sin cambios de modelo — solo se agrega gating de acceso)

| Módulo | Router(s) | Clave de `Plan` |
|---|---|---|
| Inventario | `app/api/v1/inventory/router.py` (todos los endpoints excepto los de compras) | `inventario_access` |
| Compras | Mismos 4 endpoints de `/inventory/purchases*` dentro del router de inventario | `compras_access` |
| Promociones | `app/api/v1/promotions/router.py` (router completo) | `promociones_access` |

## Relaciones

```
shared.plans (1) ──< (0..N) shared.tenants   [Tenant.plan_id NOT NULL FK]
shared.tenants (1) ──< (0..N) tenant.dining_tables / cash_registers / products / payment_methods
                                                                        [ya existente, sin cambios]
shared.tenants (1) ──< (0..N) shared.users [FK tenant_id, ya existente, sin cambios]
```

Un `Plan` puede estar asignado a cero, uno o varios tenants a la vez (spec, Key Entity "Plan"). Un
`Tenant` referencia exactamente un `Plan` en todo momento — nunca cero, nunca más de uno (FR-003,
garantizado por la FK `NOT NULL`, sin necesidad de validación de aplicación adicional).

## Transiciones de estado

### Asignación de plan de un tenant

```
[tenant creado] --Super Admin elige plan_id + ciclo_facturacion (FR-004/FR-017)--> plan_id = X,
                                                                    plan_iniciado_en = ahora,
                                                                    plan_vence_en = calculada
                                                                          │
                            Super Admin cambia O renueva (FR-010/FR-020) │ (mismo endpoint,
                                                                          │  research.md Decisión 16)
                                                                          ▼
                                                                    plan_id = Y (o el mismo),
                                                                    plan_iniciado_en = ahora,
                                                                    plan_vence_en = recalculada
                                     (recursos existentes se conservan sin cambios,
                                      FR-011/FR-012 — solo cambia qué se evalúa
                                      en la SIGUIENTE creación/acceso)
```

No existe un estado "sin plan" en ningún punto observable — ni transitoriamente durante la
creación del tenant (la fila `Tenant` no se persiste sin `plan_id` válido, misma transacción).
`ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` sí pueden ser `NULL` de forma permanente
(elección explícita de "sin vencimiento", FR-021) — eso nunca deja al tenant sin plan, solo sin
fecha de vencimiento.

### Bloqueo por vencimiento (previo a cualquier otra validación)

```
Cualquier intento de crear un recurso limitado O acceder a un módulo
        │
        ▼
  ensure_plan_not_expired(tenant)  ── ¿plan_vence_en es NULL?
        │                                    │
        │                                   sí ──► continuar (nunca vence, FR-021)
        ▼
  ¿plan_vence_en < ahora (UTC)?
        │                  \
       no                   sí
        │                    │
        ▼                    ▼
   continuar (sigue     403 — "tu plan venció, debe renovarse" (FR-019),
   con la validación      MISMO bloqueo tanto para límites como para módulos,
   de límite/módulo       sin importar cuál se estaba evaluando
   normal)
```

Esta verificación corre **antes** que la de límite numérico o acceso a módulo, dentro de la misma
`enforce_plan_limit`/`require_module_access` (research.md Decisión 14) — no es un flujo aparte que
los cinco endpoints de creación o los routers de módulo tengan que invocar por su cuenta.

### Bloqueo por límite numérico (por creación de recurso)

```
Tenant Admin intenta crear recurso R
        │
        ▼
  lock shared.tenants (FOR UPDATE) ──► leer Tenant + Plan vigente
        │
        ▼
  ensure_plan_not_expired(tenant) ──► (ver diagrama de arriba; si venció, corta aquí con 403)
        │
        ▼
  ¿Plan.<R>_limit es NULL?
        │                     \
       sí (ilimitado)          no
        │                       │
        ▼                       ▼
   permitir, insertar    contar filas existentes de R
                          (regla de conteo de la tabla de arriba)
                                  │
                                  ▼
                       ¿conteo >= Plan.<R>_limit?
                          │              \
                         no                sí
                          │                 │
                          ▼                 ▼
                    permitir,          403 — "límite de R alcanzado (N),
                    insertar            actualiza tu plan para crear más"
                                        (FR-006), NO se inserta el registro
```

El lock (`with_for_update()` sobre la fila de `shared.tenants`) envuelve todo el flujo hasta el
`commit()` del `insert`, garantizando que dos solicitudes concurrentes para el mismo tenant nunca
lean el mismo conteo "antes" del insert de la otra (FR-015, research.md Decisión 5).

### Bloqueo por acceso a módulo

```
Usuario del tenant intenta acceder al módulo M
        │
        ▼
  leer Plan vigente del tenant (sin lock — solo lectura, no hay concurrencia que resolver)
        │
        ▼
  ensure_plan_not_expired(tenant) ──► (ver diagrama de vencimiento; si venció, corta aquí con 403)
        │
        ▼
  ¿Plan.<M>_access es true?
        │                \
       sí                 no
        │                  │
        ▼                  ▼
    permitir          403 — "tu plan actual no incluye este módulo" (FR-009)
```

Sin estado intermedio de "solo lectura": denegado significa completamente inaccesible, incluyendo
consulta (edge case de `spec.md` — los datos del módulo permanecen en la base sin borrarse, pero
ningún endpoint del módulo responde mientras el acceso esté denegado, ni por falta de acceso ni
por vencimiento).

## Migración de datos existentes

Ver research.md Decisión 3 (ampliada) para la justificación completa de por qué esta migración
puede completarse en una sola fase de datos (backfill determinístico, a diferencia del backfill
fuzzy de spec 032):

1. **Crear tabla** (Alembic, esquema `shared`, sin `@for_each_tenant_schema`): `shared.plans`,
   incluyendo `precio_mensual`/`precio_anual` nullable desde el inicio.
2. **Sembrar plan transicional** (Alembic, datos): una fila `name="Ilimitado (transición)"`, los
   cinco límites en `NULL`, los tres accesos de módulo en `true`, ambos precios en `NULL` —
   reproduce el comportamiento sin restricciones que todo tenant ya tenía.
3. **Columnas + backfill + `NOT NULL` + drop de columna heredada** (Alembic, una sola migración):
   agrega `plan_id` (nullable), `ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` (nullable
   para siempre) en `shared.tenants`; `UPDATE` de todas las filas: `plan_id` al plan transicional,
   las otras tres columnas quedan en `NULL`; `ALTER COLUMN plan_id SET NOT NULL`; `DROP COLUMN
   plan`.
4. **Verificación** (parte de quickstart.md): cero filas en `shared.tenants` con `plan_id IS NULL`
   después de la migración; cada tenant existente sigue pudiendo crear mesas/cajas/usuarios/
   productos/métodos de pago sin bloqueo, accediendo a los tres módulos, y sin fecha de
   vencimiento (`plan_vence_en IS NULL`), hasta que el Super Admin lo baje activamente a un plan
   real con su propio ciclo (SC-006/SC-007, Principio II).
