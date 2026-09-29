# Implementation Plan: Integridad de las referencias a archivos en Cloudflare R2

**Branch**: `088-integridad-referencias-r2` | **Date**: 2026-09-29 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/088-integridad-referencias-r2/spec.md`

## Summary

Tras la spec 080, en base de datos vive solo la **key** de cada archivo (`{esquema}/{carpeta}/{nombre}`), pero nada garantiza que esa key apunte a un archivo real, que sea del negocio correcto ni que el archivo anterior siga sin uso cuando se borra. Hoy, en `pos-backend`:

- `ProductService.update_product` y `tenant/router.py::update_tenant` interpretan **cualquier** `image_url`/`logo_url` distinto del vigente como "cambio de imagen" y borran el objeto anterior (`delete_object`), sin verificar que el nuevo exista, que pertenezca al negocio ni que otra fila use el viejo. Un formulario desactualizado que reenvía la imagen ya borrada deja el producto con una referencia rota **y** borra la imagen vigente.
- `sales/service.py::update_payment_method` reemplaza `payment_info` completo sin ninguna validación de la key del QR.
- El comprobante del comensal (`cart/service.py::submit_cart` y `attach_receipt`) es texto libre de hasta 500 caracteres, se guarda como URL absoluta y nadie comprueba prefijo, carpeta ni existencia.
- No hay forma de detectar lo que ya quedó roto (borrados manuales en Cloudflare, historial).

El diseño añade **cuatro piezas**, todas en `pos-backend`, más un cambio pequeño y compatible en `pos-heladeria`:

1. **Primitivas en `app/core/storage.py`** (`validate_asset_key`, `object_exists` con fallo cerrado, `deletable_key`) y un módulo nuevo **`app/core/asset_refs.py`** (decisión de cambio de imagen por concurrencia optimista, chequeo "¿otra fila lo usa?" sobre las 4 fuentes reales, y un candado consultivo de PostgreSQL por key para cerrar la carrera guardar↔borrar). Es la **única fuente de la regla**, igual que `storage.py` lo es de key↔URL desde la spec 080.
2. **Escritura de imágenes** (producto, logo, QR): los esquemas de actualización aceptan una **imagen base** (`image_url_base`, `logo_url_base`, `payment_info_base`); los servicios resuelven el cambio con la regla FR-002 (validar forma y dueño → comparar base → verificar existencia), guardan y solo después borran el archivo anterior si nadie más lo usa.
3. **Comprobantes**: `submit_cart` y `attach_receipt` aceptan key o URL absoluta del bucket gestionado, validan (prefijo del negocio + carpeta `comprobantes` + existencia), **guardan la key** y las tres respuestas hacia el cajero/comensal ensamblan la URL con `AssetUrl` (tolerancia de lectura para filas antiguas). Script de migración idempotente `migrate_receipt_keys.py` (simulación por defecto, `--apply`, `--revert`).
4. **Reporte de reconciliación** de solo lectura `reconcile_r2_references.py` (referencias sin archivo, archivos huérfanos con ventana de gracia, keys fuera de convención; CSV/JSON opcional; `--tenant` para acotar).

En `pos-heladeria` el cambio es **aditivo y desplegable antes que el backend**: los tres formularios envían la imagen base y dejan de reenviar la imagen sin tocar. Sin dependencias nuevas, sin variables de entorno nuevas, sin migración de Alembic.

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend`, sin cambio). Angular 20 (`pos-heladeria`, sin cambio).

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50, Pydantic v2, boto3 (cliente R2 S3-compatible, ya presente). **Ninguna dependencia nueva** (Principio IX): `head_object` y `list_objects_v2` ya son parte de boto3, que `app/core/storage.py` usa en producción desde la spec 021.

**Storage**: PostgreSQL 16 schema-per-tenant + Cloudflare R2. **Sin cambios de esquema y sin migración de Alembic.** Cambia el **contenido** de una columna existente: `order_payment_attempts.receipt_file_url` pasa de URL absoluta a key (migración de datos por script, ver [data-model.md](./data-model.md)). El objeto físico en R2 nunca se mueve ni se renombra.

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (no hay `pytest`). Las sesiones de test son **SQLite en memoria** (`fixtures.py::new_session`), por eso: (a) el candado consultivo es específico de PostgreSQL y se omite en otros dialectos; (b) `object_exists` y `delete_object` se sustituyen con `mock.patch`; (c) el chequeo de referencias sobre `shared.tenants` necesita esa tabla en el fixture (ver research D9). Frontend: `ng test` (Karma/Jasmine ya configurado) para los `.spec.ts` de los servicios tocados.

**Target Platform**: Linux server (contenedor Docker existente). Los scripts corren como `python -m app.scripts.<nombre>` en el mismo contenedor, igual que `migrate_image_keys_relative`.

**Project Type**: web — `pos-backend` (FastAPI) + `pos-heladeria` (Angular). La spec vive en `pos-specs`.

**Performance Goals**: sin objetivo nuevo. El costo se paga **solo** cuando el administrador cambia realmente una imagen o el comensal envía un comprobante: un `HEAD` a R2 (~50–150 ms) y 3–4 consultas pequeñas de referencias (tablas de un negocio). El resto de las peticiones de guardado no hacen I/O adicional.

**Constraints**:
- **Fallo cerrado** (FR-003): timeout de conexión 3 s, lectura 5 s, máximo 2 intentos en el cliente boto3 compartido; cualquier error distinto de "no existe" → 503 sin guardar ni borrar.
- **Orden de evaluación fijo** (FR-002): forma/dueño → base → existencia. Nada del registro se modifica hasta haber resuelto la imagen (los errores 422/503 dejan el registro intacto).
- **Borrado post-commit, mejor esfuerzo** (FR-007, A-44 intacto): el candado consultivo protege la ventana entre "nadie lo usa" y `delete_object`, sin cambiar que un fallo de borrado solo deja un huérfano.
- **Tolerancia de lectura** (spec 080, FR-009b): filas con URL absoluta antigua siguen viéndose.
- **Sin variables de entorno nuevas**: la ventana de gracia del reporte es un argumento de línea de comandos (`--grace-hours`, 24 por defecto); los timeouts de R2 son constantes de `storage.py`.

**Scale/Scope**:
- `pos-backend`: `core/storage.py` (3 primitivas nuevas + timeouts del cliente), `core/asset_refs.py` (NUEVO), `products/{schemas,service}.py`, `tenant/{schemas,router}.py`, `sales/{schemas,service}.py`, `cart/{schemas,service,router}.py`, `orders/schemas.py` (2 campos → `AssetUrl`), 2 scripts nuevos, tests nuevos + actualización de tests existentes listados en la fila III.
- `pos-heladeria`: `product.service.ts` + `product-form.component.ts`, `tenant-info.service.ts`, `payment-method.service.ts` + `payment-methods-page.component.ts`, y (opcional, último) `transfer-details-step.component.ts` para enviar `presign.key`.
- 6 historias: P1×3 (formulario desactualizado, existencia, aislamiento entre negocios) · P2×2 (borrado solo si nadie lo usa, comprobantes) · P3 (reporte). Se entregan en ese orden.

## Constitution Check

*GATE: debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `spec.md` aprobado, 13 aclaraciones (sesión 2026-09-29) y checklist de calidad. |
| II. El Comportamiento Existente Sigue Protegido | ⚠️ Pass **condicionado** — cambia comportamiento de forma deliberada y pedida por el negocio: (a) guardar producto/logo/QR con una key inexistente, ajena o mal formada deja de aceptarse (hoy se acepta y produce una referencia rota o un borrado ajeno); (b) una imagen enviada por un formulario desactualizado deja de contar como cambio; (c) el archivo anterior deja de borrarse si otra fila lo usa; (d) el comprobante deja de ser texto libre y se guarda como key. Requiere **dos entradas nuevas** en `specs/000-reconocimiento/registro-de-anomalias.md`, creadas **antes** de implementar: **A-92** (imágenes de producto/logo/QR: validación de dueño y existencia, imagen base, borrado protegido) y **A-93** (comprobantes: solo archivos del propio negocio, key en base de datos). `/speckit-tasks` las secuencia como T001–T002 y los commits las citan. La visualización de imágenes y el flujo feliz de subida no cambian para el administrador. |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ⚠️ Pass **condicionado** — se ven afectados tests que **codifican la forma de la petición, no el comportamiento congelado**: (1) `test_products_service.py`, tres tests A-44 (`"CONGELA comportamiento corregido:"`): llaman `update_product(..., ProductUpdate(image_url=NEW_URL))` **sin imagen base**; con la regla nueva esa petición se ignora. Se les añade `image_url_base=<imagen vigente>` y `mock.patch` de `object_exists`/`is_key_referenced`. El comportamiento que congelan —el borrado ocurre **después** del commit y un fallo de borrado o de commit no deja referencia rota— se preserva exactamente; se citan spec 088 + A-92 en el mismo commit. (2) `test_products_image_key.py`, `test_tenant_logo_key.py`, `test_payment_methods_qr_key.py` (tests de la spec 080, sin prefijo CONGELA): agregan la base a las ediciones legítimas. (3) `test_cart_payment_attempts.py`, `test_orders_payment_gate.py` (líneas ~505–520) y `test_order_audit_log.py` (`COMPROBANTE = "https://example.invalid/comprobantes/abc123.jpg"`, l.57): usan una URL fuera de convención como comprobante (ahora 422); pasan a una key `tenant_test/comprobantes/…` con `object_exists` simulado y citan A-93. En `test_order_audit_log.py`, el test FR-012 de la spec 074 ("el mismo comprobante produce el mismo hash en sus tres eventos") pasa a calcular el hash esperado sobre la **key** persistida: la igualdad entre eventos del mismo intento se preserva (research D11). Ningún otro test `CONGELA` cambia; la suite completa de `pos-backend` corre como verificación (SC-008). |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — el comportamiento nuevo (validación, base, borrado protegido, key de comprobante, reporte) está definido y acotado; el criterio de éxito es SC-001…SC-009. |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — cada cambio se ata a un FR. No se toca `build_object_key`, el presign, la convención de carpetas ni el proveedor. `object_key_for_deletion` (spec 080) queda **intacta**: se añade `deletable_key` que la compone, para no reabrir los tests de la 080. |
| VI. Evolución Incremental | ✅ Pass — una sola clase de cambio ("una referencia a archivo nunca queda rota"), en 6 historias verificables por separado. La migración de datos de comprobantes es su propia unidad (P2) y el reporte es de solo lectura (P3). Sin cambio de arquitectura. |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — no se recalcula ni se re-representa ninguna factura ni venta. `receipt_file_url` no forma parte del importe ni de la representación contable. La migración cambia la **ubicación** del objeto en base de datos, no el objeto ni ningún dato contable. El `receipt_hash` de auditoría (spec 074) de eventos ya emitidos **no se reescribe**; los eventos nuevos hashean la key (ver research D11). |
| VIII. Evolución del Modelo de Datos | ✅ Pass (con artefacto) — sin entidades, campos ni relaciones nuevas. [`data-model.md`](./data-model.md) especifica el **antes/después** del contenido de `receipt_file_url`, el predicado de selección, la transformación, la **idempotencia**, la **estrategia de reversión** (`--revert`, reconstruye `{R2_PUBLIC_BASE_URL}/{key}`) y la compatibilidad (lectura tolerante a ambas formas ⇒ el código nuevo funciona con datos sin migrar). |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica) — cero dependencias nuevas. `head_object`/`list_objects_v2` son API del boto3 ya instalado; el candado consultivo es SQL estándar de PostgreSQL vía SQLAlchemy. |
| X. Verificación Obligatoria | ✅ Pass (planificado) — [`research.md`](./research.md) §D12 fija la matriz; [`quickstart.md`](./quickstart.md) recorre los 9 criterios de éxito con comandos concretos, incluida la corrida completa de la suite de `pos-backend`. |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — las decisiones de negocio (ignorar en silencio, 422 vs 503, comprobante sin garantía de autoría, huérfanos aceptables, sin borrado automático) ya están en la spec. Este plan solo resuelve el "cómo". **Dos puntos donde el plan fija un valor por defecto que la spec dejaba abierto** y se marcan para confirmación del negocio al registrar A-92: D7 (el QR sigue permitiendo quitarse por el mecanismo actual de `payment_info` completo) y D8 (no se añade borrado del QR anterior). |
| XII. Trazabilidad | ✅ Pass — Necesidad (problema de integridad detectado, 2026-09-29) → `spec.md` + Aclaraciones → este plan → `research.md` / `data-model.md` / `contracts/` / `quickstart.md` → `tasks.md` → tests. A-92/A-93 cierran la cadena del cambio de comportamiento. |
| XIII. Todo en Español de Colombia | ✅ Pass — artefactos y mensajes de error visibles en español de Colombia (ver contratos, sección "Mensajes"). |
| XIV. Estrategia y Convención de Ramas | ✅ Pass (planificado) — antes de tocar código se crean `feat/088-r2-reference-integrity` en `pos-backend` y en `pos-heladeria`, desde la rama actual (`develop` en ambos). `/speckit-tasks` lo deja como T003. |
| XV. Política de Commits | ✅ Pass (planificado) — commits pequeños por historia, en inglés, Conventional Commits, sin marcas de IA, y solo cuando el usuario lo pida. |

**Complexity Tracking**: sin violaciones de diseño que justificar. Las filas II y III son prerrequisitos de proceso (registrar A-92/A-93 antes de implementar; actualizar tests citando la decisión), no alternativas más simples descartadas.

**Re-chequeo post Fase 1**: tras generar `research.md`, `data-model.md`, `contracts/*` y `quickstart.md`, el diseño no introdujo dependencias, variables de entorno ni migraciones de esquema, y no alteró la tabla anterior. Dos hallazgos del diseño quedaron **absorbidos** sin cambiar el veredicto: (1) el orden de despliegue **debe** ser frontend primero, luego backend (D10), porque un frontend viejo contra el backend nuevo perdería en silencio las subidas legítimas; (2) las sesiones de test son SQLite, por lo que el candado consultivo y el chequeo sobre `shared.tenants` requieren las salvaguardas de D9.

## Project Structure

### Documentation (this feature)

```text
specs/088-integridad-referencias-r2/
├── plan.md                        # Este archivo (/speckit-plan)
├── research.md                    # Fase 0 (/speckit-plan)
├── data-model.md                  # Fase 1 (/speckit-plan)
├── quickstart.md                  # Fase 1 (/speckit-plan)
├── contracts/                     # Fase 1 (/speckit-plan)
│   ├── asset-guard.md             #   Primitivas: validar key, existencia, key-para-borrar, decisión de cambio, referencias
│   ├── image-write-endpoints.md   #   PATCH /products/{id}, PATCH /tenant, POST/PATCH /sales/payment-methods
│   ├── receipt-endpoints.md       #   POST /cart/submit, POST /cart/payment-attempts/{id}/receipt + salidas
│   └── scripts.md                 #   reconcile_r2_references y migrate_receipt_keys (CLI)
├── checklists/
│   └── requirements.md            # (ya existente)
└── tasks.md                       # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repositorios `pos-backend` y `pos-heladeria`, independientes de `pos-specs`)

```text
pos-backend/
├── app/
│   ├── core/
│   │   ├── storage.py                    # + constantes de carpeta (products/logo/payment-methods/comprobantes)
│   │   │                                 # + validate_asset_key(value, tenant_schema, folder) -> str  (pura, lanza AssetKeyError)
│   │   │                                 # + object_exists(key) -> bool  (HEAD; lanza StorageUnavailable si no responde)
│   │   │                                 # + deletable_key(value, tenant_schema, folder) -> str | None  (compone object_key_for_deletion + convención)
│   │   │                                 # + get_r2_client(): BotoConfig con connect_timeout=3, read_timeout=5, retries max 2
│   │   │                                 #   object_key_for_deletion / delete_object / normalize_asset_ref / asset_display_url: SIN CAMBIOS
│   │   ├── asset_refs.py                 # NUEVO — is_key_referenced(db, key)  (products, payment_methods, order_payment_attempts, shared.tenants)
│   │   │                                 #         asset_key_lock(db, key)     (pg_advisory_xact_lock; no-op fuera de PostgreSQL)
│   │   │                                 #         decide_image_change(...) -> ImageDecision  (KEEP / IGNORE / APPLY; FR-002)
│   │   │                                 #         ensure_image_exists(key)  (existencia → 422 / 503)
│   │   │                                 #         resolve_receipt_key(value, tenant_schema) -> str  (FR-008)
│   │   └── schema_types.py               # SIN CAMBIOS (AssetUrl / AssetRefIn ya existen)
│   ├── api/v1/
│   │   ├── products/
│   │   │   ├── schemas.py                # ProductUpdate.image_url_base: AssetRefIn  (se distingue "no enviada" de null vía model_fields_set)
│   │   │   └── service.py                # create_product: validar + verificar imagen (sin base). update_product: resolver la imagen PRIMERO,
│   │   │                                 #   guardar, y borrar el anterior solo con deletable_key + candado + is_key_referenced
│   │   ├── tenant/
│   │   │   ├── schemas.py                # TenantUpdate.logo_url_base: AssetRefIn
│   │   │   └── router.py                 # update_tenant: misma resolución; sesión de negocio (with_db(tenant.schema)) para is_key_referenced
│   │   ├── sales/
│   │   │   ├── schemas.py                # PaymentMethodUpdate.payment_info_base: dict[str, str] | None (normalizado por valor)
│   │   │   └── service.py                # create/update_payment_method: por cada clave format="image" del catálogo → validar/decidir/verificar
│   │   ├── cart/
│   │   │   ├── schemas.py                # DinerPaymentAttempt.receipt_file_url: AssetUrl
│   │   │   ├── service.py                # submit_cart / attach_receipt: resolve_receipt_key(...), persistir la key; hash de auditoría sobre la key
│   │   │   └── router.py                 # pasar ctx.tenant.schema a submit_cart / attach_receipt
│   │   └── orders/schemas.py             # CurrentPaymentAttemptSummary / PaymentAttemptResponse .receipt_file_url: AssetUrl
│   ├── scripts/
│   │   ├── reconcile_r2_references.py    # NUEVO — solo lectura: --tenant, --grace-hours, --output, --format
│   │   └── migrate_receipt_keys.py       # NUEVO — simulación por defecto, --apply, --revert; idempotente
│   └── characterization_tests/
│       ├── fixtures.py                   # make_tenant_stub(schema=...) + tabla shared.tenants para is_key_referenced (research D9)
│       ├── test_storage_asset_guard.py   # NUEVO — validate_asset_key (tabla de verdad), object_exists (404 / timeout), deletable_key
│       ├── test_asset_refs.py            # NUEVO — decide_image_change (matriz FR-002), is_key_referenced (4 fuentes), resolve_receipt_key
│       ├── test_products_image_integrity.py   # NUEVO — US1–US4 sobre producto (stale, existencia, ajena, compartida, concurrencia)
│       ├── test_tenant_logo_integrity.py      # NUEVO — mismas reglas para logo
│       ├── test_payment_methods_qr_integrity.py # NUEVO — mismas reglas para QR (payment_info + payment_info_base)
│       ├── test_receipts_integrity.py         # NUEVO — US5: key/URL gestionada/otro origen/ajena/inexistente/409; respuestas con AssetUrl
│       ├── test_reconcile_r2_references.py    # NUEVO — US6 con R2 simulado; verifica solo lectura
│       ├── test_migrate_receipt_keys.py       # NUEVO — simulación, aplicación, idempotencia, revert, filas intactas
│       ├── test_products_service.py           # ACTUALIZAR 3 tests A-44 (añadir base + mocks), citando spec 088 + A-92
│       ├── test_products_image_key.py         # ACTUALIZAR ediciones legítimas (añadir base)
│       ├── test_tenant_logo_key.py            # ACTUALIZAR ediciones legítimas (añadir base)
│       ├── test_payment_methods_qr_key.py     # ACTUALIZAR ediciones legítimas (añadir payment_info_base)
│       ├── test_cart_payment_attempts.py      # ACTUALIZAR comprobantes a key válida + object_exists simulado, citando A-93
│       ├── test_orders_payment_gate.py        # ACTUALIZAR ~505–520 (attach_receipt con key válida), citando A-93
│       └── test_order_audit_log.py            # ACTUALIZAR COMPROBANTE (l.57) a key válida; hash esperado sobre la key, citando A-93

pos-heladeria/
└── src/app/
    ├── core/tenant/tenant-info.service.ts             # uploadLogo: enviar logo_url + logo_url_base (logo cargado al abrir); update(): nunca reenvía logo_url
    ├── modules/products/
    │   ├── services/product.service.ts                # ProductDraft.image_url_base; toProductPayload: enviar image_url solo si cambió, siempre image_url_base (null explícito si no había)
    │   └── pages/product-form.component.ts            # capturar la imagen base al cargar (toDraft); no reenviar la imagen sin tocar
    ├── modules/sales/
    │   ├── services/payment-method.service.ts         # update(): payment_info_base
    │   └── pages/payment-methods-page.component.ts    # openFieldsForm: guardar la base (payment_info al abrir); submitFields: enviarla
    └── modules/tables/pages/checkout/transfer-details-step.component.ts  # OPCIONAL y último: enviar presign.key en vez de public_url
```

**Structure Decision**: aplicación web; se tocan los dos repositorios. La regla vive en un solo lugar (`storage.py` para lo puro y con I/O de R2, `asset_refs.py` para lo que consulta la base de datos y decide), y los servicios existentes solo la invocan — no se crean paquetes de alto nivel nuevos. Los scripts siguen el patrón de `migrate_image_keys_relative.py` (recorrido por schema con `with_db`, idempotente). El frontend solo cambia **qué envía**, nunca cómo se ve.

## Complexity Tracking

*Sin violaciones del Constitution Check — sección no aplica.* Las filas II y III son prerrequisitos de proceso, no violaciones de diseño.
