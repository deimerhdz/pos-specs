# Implementation Plan: Catálogo de Métodos de Pago Administrado por el Super Admin

**Branch**: `032-catalogo-metodos-pago` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/032-catalogo-metodos-pago/spec.md`

## Summary

Hoy `PaymentMethod` (`app/models/payment.py`, esquema `tenant`) es una tabla libre: cualquier
Tenant Admin crea un método con `name` arbitrario (único solo dentro de su propio esquema),
`type` (`cash`/`card`/`transfer`/`other`, usado por el arqueo) y `payment_info` JSONB sin
esquema fijo (spec 024). No existe ningún catálogo de plataforma — nada distingue "qué tipo de
método de pago es" de "cómo lo configuró un tenant".

Esta spec agrega un catálogo nuevo, administrado por el Super Admin, en el esquema `shared`
(mismo esquema de `Tenant`/`Role`/`User`, `app/core/models.py`): `payment_method_catalog`, cada
fila con `name`, `type`, `active` (a nivel plataforma) y `fields` (JSONB: qué campos de
integración requiere, cuáles son obligatorios/opcionales y su formato esperado). El
`PaymentMethod` de cada tenant gana una referencia (`catalog_id`) a ese catálogo y dos columnas
derivadas: `is_complete` (recalculada al guardar `payment_info`, validando contra
`catalog.fields`) y `type`/`name` denormalizados desde el catálogo en el momento de la
activación (así el arqueo, que agrupa por `PaymentMethod.type`/`.name` en
`app/api/v1/cash/service.py`, sigue funcionando sin tocarse). La pantalla de cobro dejará de
listar el `PaymentMethod` en crudo y consumirá una variante filtrada
(`active AND is_complete AND catalog.active`) que además oculta `payment_info` (FR-012a,
clarificación 2026-08-24).

## Technical Context

**Language/Version**: Backend — Python 3.14.4 (venv `pos-backend/env`). Frontend — TypeScript
5.9.2 (Angular 21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync), Alembic 1.18.4, Pydantic 2.13.4, boto3
  1.43.48 (Cloudflare R2 vía `app/core/storage.py`, ya usado para imágenes de producto). Ninguna
  dependencia nueva (Principio IX no aplica).
- Frontend: Angular 21 (standalone + signals), `@tanstack/angular-query-experimental` 5.101.4,
  Tailwind 4.1.12. Ninguna dependencia nueva.

**Storage**: PostgreSQL 16, multi-tenant schema-per-tenant.
- Tabla nueva `payment_method_catalog` en el esquema `shared` (mismo esquema de
  `shared.tenants`/`shared.roles`/`shared.users`), vía Alembic plano (sin
  `@for_each_tenant_schema`, mismo patrón que `c2d3e4f5a6b7_tenant_logo_url.py`).
- Columnas nuevas en `tenant.payment_methods` (`catalog_id`, `is_complete`), vía
  `@for_each_tenant_schema` (`app/scripts/tenant.py`), mismo patrón que
  `c1d2e3f4a5b6_order_payment_attempts.py`.
- Cloudflare R2 (S3-compatible vía `boto3`) para la imagen de código QR, reutilizando
  `app/core/storage.py` (`generate_presigned_put_url`, `build_object_key`, `public_url_for`) con
  un folder nuevo (`"payment-methods"`) en la whitelist de `PresignRequest.folder`
  (`app/api/v1/uploads/schemas.py`, hoy `Literal["products", "logo"]`).

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`
(sin pytest en el repo). `test_sales_payment_methods.py` **no** es un characterization test
protegido — su propio docstring aclara que no lleva el prefijo `"CONGELA comportamiento actual:"`
(es un test de aceptación de la spec 024) — así que se edita directamente para reflejar
`catalog_id`/`payment_info` validado (FR-007/FR-011), sin activar el Principio III. No existe hoy
ningún test `"CONGELA comportamiento actual:"` específico de `payment_methods`; los `CONGELA` más
cercanos (`test_cart_payment_attempts.py`, `test_orders_payment_gate.py`,
`test_payments_timezone.py`) protegen el flujo de intentos de pago de pedidos, no la configuración
de métodos — no se tocan. Se agregan fixtures y tests nuevos por historia de usuario. Frontend:
Vitest (`ng test`, Angular 21 build system) + Playwright para e2e, specs colocados (`*.spec.ts`).

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`,
incluyendo el panel de Super Admin ya existente en `src/app/modules/super-admin/`).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. El único requisito de rendimiento
explícito es de UX (SC-001/SC-002: alta de un método nuevo visible en <5 min/<2 min,
respectivamente, sin desplegar código).

**Constraints**:
- `tenant.payment_methods` nunca se borra físicamente (ya es así hoy): desactivar a nivel
  plataforma o a nivel tenant es siempre `active=false`, nunca `DELETE` — preserva la FK de
  `Payment.payment_method_id` y por tanto el histórico de ventas (Principio VII).
- El arqueo (`app/api/v1/cash/service.py`, agrupa por `PaymentMethod.type`/`.name`) no se
  modifica: `type`/`name` en `tenant.payment_methods` siguen existiendo y se siguen llenando,
  ahora copiados desde `catalog` en vez de tecleados libremente.
- `catalog_id` se agrega **nullable** en `tenant.payment_methods` (no `NOT NULL` en la misma
  migración): forzarlo de una vez sobre datos de producción sin backfill previo violaría el
  Principio VIII (migración sin estrategia de compatibilidad). El backfill (FR-015/FR-015a) puebla
  `catalog_id` fila por fila antes de que la capa de aplicación exija la referencia para escrituras
  nuevas.
- Ninguna venta ni pago ya registrado (`Sale`/`Payment`, `app/models/payment.py` y
  `app/models/sale.py`) se toca — esta spec vive completamente en la configuración de métodos, no
  en el checkout de ventas ya cobradas.
- Fuera de alcance explícito de `spec.md`: integración real con pasarelas/billeteras, que el
  cajero vea los datos de integración (FR-012a), más de una configuración activa por método por
  tenant (FR-017).

**Scale/Scope**: 1 tabla nueva (`shared.payment_method_catalog`), 2 columnas nuevas en
`tenant.payment_methods` (`catalog_id`, `is_complete`) + 1 migración de backfill de datos, ~5
endpoints nuevos + 3 modificados en `pos-backend`; 1 pantalla nueva en el panel de Super Admin
(`pos-heladeria`, `super-admin/`), y una extensión de la pantalla existente de configuración de
métodos de pago del tenant (`sales/pages/payment-methods-page.component.ts`) más un filtrado
nuevo en el store que alimenta el cobro (`pos-terminal.store.ts`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 3 historias priorizadas, 19 FRs (incluye FR-012a/FR-015a) y 3 clarificaciones resueltas (sesión 2026-08-24) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El comportamiento que cambia (`POST /sales/payment-methods` deja de aceptar `name`/`type` libres; el checkout deja de mostrar métodos inactivos/incompletos) está explícitamente definido y autorizado por FR-007/FR-011/FR-012 del propio spec — no es una corrección de una anomalía heredada, es comportamiento nuevo de una fase de evolución funcional (Principio IV), por lo que no requiere entrada en `registro-de-anomalias.md`. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | `test_sales_payment_methods.py` no lleva el prefijo `"CONGELA comportamiento actual:"` (su propio docstring lo aclara) — no es un test protegido, por lo que editarlo para reflejar FR-007/FR-011 no activa este principio. No existe ningún `CONGELA` específico de `payment_methods`; los `CONGELA` relacionados (`test_cart_payment_attempts.py`, `test_orders_payment_gate.py`, `test_payments_timezone.py`) protegen el flujo de intentos de pago de pedidos, ajeno a esta spec, y no se tocan. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Todo el comportamiento nuevo (catálogo, gating por completitud, ocultar `payment_info` al cajero) está definido en `spec.md`; el criterio de éxito es conformidad con la spec, no equivalencia con el pasado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | El arqueo (`cash/service.py`) y el modelo `Payment`/`Sale` no se tocan — comparten "método de pago" con esta spec pero no se refactorizan; solo se le siguen llenando las mismas columnas (`type`/`name`) que ya usan. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec: catálogo del Super Admin (US1) → activación/configuración del tenant (US2) → filtrado del checkout (US3), cada una verificable por separado (ver Project Structure). La migración de datos existentes es su propia unidad, no se mezcla con la implementación del catálogo. | PASS |
| **VII. Compatibilidad con Datos Históricos** | `Payment.payment_method_id` sigue apuntando a la misma fila de `tenant.payment_methods` de siempre; esas filas nunca se borran, solo se desactivan. Ninguna `Sale`/`Payment` existente cambia. | PASS |
| **VIII. Evolución del Modelo de Datos** | Ver data-model.md: tabla nueva (`shared.payment_method_catalog`), columnas nuevas nullable en `tenant.payment_methods` (no rompen filas existentes), estrategia de backfill explícita (FR-015/FR-015a) y de rollback (`op.drop_table`/`op.drop_column`) en research.md. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables, más el characterization test actualizado como red de no-regresión del resto del flujo de ventas. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Dos decisiones técnicas quedan explícitas en research.md para no confundirse con decisiones de negocio: (a) `catalog_id` nullable + backfill en dos pasos, no `NOT NULL` inmediato; (b) `type`/`name` denormalizados en `tenant.payment_methods` en vez de un `JOIN` en cada lectura del arqueo. El spec no exige ninguna de las dos — son la forma de implementar sin romper lo existente. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec) → este `plan.md`/`research.md` (Decisión técnica) → `tasks.md` (Fase 2, no generada por este comando) → implementación → characterization test actualizado + tests nuevos → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

**Re-chequeo post-diseño (Fase 1)**: `research.md` y `data-model.md` no introdujeron ninguna
entidad, dependencia ni decisión que contradiga la tabla anterior — en particular, el patrón nuevo
"catálogo compartido + activación por tenant" (research.md Decisión 1) no tiene precedente en el
código, pero está justificado en la propia spec (RF-1/RF-7) y no requiere una excepción de
Complexity Tracking porque no es complejidad accidental, es la separación que el spec pide
explícitamente en su Objetivo (§1). Gates siguen en PASS.

## Project Structure

### Documentation (this feature)

```text
specs/032-catalogo-metodos-pago/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md        # Fase 1 (/speckit-plan) — entidades, columnas, validaciones, migraciones
├── quickstart.md        # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados
│   ├── super-admin-catalog.md
│   ├── tenant-payment-methods.md
│   └── checkout-payment-methods.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── models/
│   ├── payment_method_catalog.py     # NUEVO — PaymentMethodCatalog (shared.payment_method_catalog)
│   └── payment.py                    # MODIFICADO — PaymentMethod gana catalog_id (FK), is_complete
│
├── api/v1/
│   ├── super_admin/
│   │   ├── router.py                 # MODIFICADO — monta el sub-router de catálogo
│   │   ├── payment_methods_router.py # NUEVO — CRUD + activar/desactivar del catálogo
│   │   └── schemas.py                # MODIFICADO — PaymentMethodCatalogCreate/Update/Response
│   │
│   ├── sales/
│   │   ├── router.py                 # MODIFICADO — GET /payment-methods/catalog (para el tenant);
│   │   │                               POST/PATCH /payment-methods ahora contra catalog_id;
│   │   │                               GET /payment-methods soporta ?available=true (US3)
│   │   ├── service.py                # MODIFICADO — validar payment_info contra catalog.fields,
│   │   │                               recalcular is_complete, guardia "una config activa por
│   │   │                               catalog_id" (FR-017)
│   │   └── schemas.py                # MODIFICADO — PaymentMethodCreate/Update por catalog_id;
│   │                                    PaymentMethodCheckoutOption nuevo (sin payment_info, FR-012a)
│   │
│   └── uploads/
│       └── schemas.py                 # MODIFICADO — folder Literal gana "payment-methods"
│
├── scripts/
│   └── migrate_payment_methods_catalog.py  # NUEVO — script de un solo uso: matchea
│                                              tenant.payment_methods existentes contra el catálogo
│                                              por nombre normalizado, llena catalog_id/is_complete,
│                                              reporta filas sin match (FR-015/FR-015a)
│
├── characterization_tests/
│   ├── test_sales_payment_methods.py   # MODIFICADO — actualiza los casos de creación libre que
│   │                                     FR-007/FR-011 cambian; cita esta spec en el commit
│   ├── payment_catalog_fixtures.py     # NUEVO — helpers para PaymentMethodCatalog en distintos
│   │                                     estados (activo/inactivo, con/sin campos)
│   ├── test_super_admin_payment_catalog.py  # NUEVO — US1 (FR-001/FR-002/FR-003/FR-004)
│   ├── test_sales_payment_methods_catalog.py # NUEVO — US2 (FR-005..FR-011, FR-017)
│   └── test_sales_payment_methods_checkout.py # NUEVO — US3 (FR-012/FR-012a/FR-013)
│
└── alembic/versions/
    ├── {rev}_payment_method_catalog.py       # NUEVO — solo tabla shared.payment_method_catalog
    │                                            (sin seed — el seed es su propia migración, ver
    │                                            abajo, para no mezclar creación de esquema con
    │                                            migración de datos de producción, Principio VI)
    ├── {rev}_tenant_payment_methods_catalog_ref.py  # NUEVO — catalog_id + is_complete en
    │                                                  tenant.payment_methods, vía
    │                                                  @for_each_tenant_schema
    └── {rev}_seed_payment_method_catalog.py   # NUEVO (fase de migración de datos, no Foundational)
                                                  — siembra Efectivo/Nequi/Transferencia Bancolombia
                                                  en shared.payment_method_catalog

# pos-heladeria
src/app/modules/
├── super-admin/
│   ├── interfaces/payment-method-catalog.interface.ts  # NUEVO
│   ├── services/payment-method-catalog.service.ts       # NUEVO — mismo patrón que tenant.service.ts
│   ├── pages/payment-method-catalog-page.component.ts   # NUEVO — listar/crear/editar/(des)activar,
│   │                                                       mismo patrón que tenants-page.component.ts
│   ├── components/payment-method-catalog-form.component.ts # NUEVO — define campos requeridos/
│   │                                                          opcionales y su formato
│   └── routes.ts                                          # MODIFICADO — hijo nuevo junto a
│                                                              `tenants`/`users`; gateado por
│                                                              super-admin-domain.guard.ts (ya
│                                                              cubre todo el árbol de /super-admin)
│
└── sales/
    ├── interfaces/sales.interface.ts        # MODIFICADO — PaymentMethod gana catalog_id,
    │                                           is_complete; CatalogPaymentMethod nuevo
    ├── services/
    │   ├── payment-method.service.ts        # MODIFICADO — create/update contra catalog_id
    │   └── payment-method-catalog.service.ts # NUEVO — GET catálogo activo (para activar)
    └── pages/payment-methods-page.component.ts # MODIFICADO — activar desde catálogo + formulario
                                                    dinámico según catalog.fields; indicador de
                                                    "incompleto". Ruta sin cambios
                                                    (dashboard/ajustes/metodos-pago), ya gateada a
                                                    UserRole.ADMIN por role.guard.ts en la ruta
                                                    padre `ajustes`

└── tables/
    └── services/
        └── pos-terminal.store.ts  # MODIFICADO — `paymentMethods` (checkout) pasa a leer el
                                      endpoint filtrado (?available=true) llamando a un método
                                      nuevo del `PaymentMethodService` ya inyectado (no se crea un
                                      servicio nuevo en `tables/` — `pos-terminal.store.ts` ya
                                      inyecta `sales/services/payment-method.service.ts`, Structure
                                      Decision de esta fase); `payment-draft.util.ts` (validación
                                      de `paymentIssue`) se revisa para confirmar que sigue
                                      validando contra la lista ya filtrada sin cambios propios
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de
los ficheros de arriba (US1 → `super_admin/payment_methods_router.py` + módulo `super-admin/` del
frontend; US2 → `sales/router.py|service.py|schemas.py` (creación/edición) + módulo `sales/` del
frontend; US3 → `sales/router.py` (`?available=true`) + `pos-terminal.store.ts`), consistente con
Principio VI. No se crea ningún paquete nuevo en `pos-backend` más allá de un modelo y su router —
se extiende `super_admin` (donde ya vive la administración de plataforma) y `sales` (donde ya vive
la configuración de métodos de pago del tenant). En `pos-heladeria` se extiende `super-admin`
(mismo patrón que `tenants-page.component.ts`) y `sales` — sin módulo nuevo.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
