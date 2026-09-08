# Implementation Plan: Almacenar imágenes como key relativa y servirlas por dominio personalizado

**Branch**: `080-imagenes-key-relativa-r2` | **Date**: 2026-09-08 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/080-imagenes-key-relativa-r2/spec.md`

## Summary

Hoy `Product.image_url`, `Tenant.logo_url` y el valor de imagen dentro de `PaymentMethod.payment_info` (clave `qr`) guardan una **URL pública completa** contra `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev/{key}`, construida por `app/core/storage.py::public_url_for` a partir de `R2_PUBLIC_BASE_URL`. Esta funcionalidad hace que **en base de datos viva solo la key** (`{tenant}/{carpeta}/{archivo}`) y que la URL absoluta se **arme en el servidor al responder**, anteponiendo el dominio personalizado nuevo `https://assets.skeilopos.com/{key}` (configuración desplegable `ASSETS_BASE_URL`, FR-008). El contrato hacia los consumidores no cambia: siguen recibiendo una URL lista para usar por cada imagen (FR-007), así que **el frontend no se toca**.

Tres piezas en `pos-backend`:

1. **Helpers centrales en `app/core/storage.py`** (única fuente de la regla key ↔ URL): `asset_display_url(value)` (key/URL del bucket gestionado → `https://assets.skeilopos.com/{key}`; otro origen → intacto, FR-010), `normalize_asset_ref(value)` (para persistir: URL absoluta del bucket gestionado —dominio viejo **o** nuevo— → key; key → intacta; otro origen → intacto, FR-004) y `object_key_for_deletion(value)` (reemplaza `key_from_public_url`: acepta key directa **o** URL del bucket gestionado; otro origen → `None`, FR-011/FR-013). El reconocimiento del "bucket gestionado" mira **dos** prefijos: `ASSETS_BASE_URL` y el `R2_PUBLIC_BASE_URL` heredado.
2. **Escritura**: normalizar a key en el borde de entrada (validador de esquema en `ProductCreate/Update.image_url`, `TenantUpdate.logo_url`, `PaymentMethod{Create,Update}.payment_info`), **antes** de la comparación "¿cambió la imagen?" que dispara el borrado del objeto anterior — de lo contrario un formulario reenviado sin tocar la imagen (que manda la URL de visualización) se vería como un cambio y borraría el objeto en uso.
3. **Lectura**: ensamblar key → URL al responder, vía un tipo `Annotated` reutilizable (`AssetUrl`) en `ProductResponse.image_url`, `MenuProductResponse.image_url`, `TenantInfoResponse.logo_url`, `MenuBusinessResponse.logo_url`; y un mapeo explícito de las claves `format: "image"` de `payment_info` en `DinerPaymentMethod` (checkout del comensal) y `PaymentMethodResponse` (panel de administración).

**Migración única de datos** (FR-009): script `app/scripts/migrate_image_keys_relative.py` (mismo patrón que `migrate_payment_methods_catalog.py`, **no** una migración de Alembic — no hay cambio de esquema), idempotente y re-ejecutable con la app en marcha, con `--report-only` / escritura / `--revert`. Reescribe a key las filas de los tres campos cuyo valor es una URL del bucket gestionado; deja intactas las que ya son key, las vacías y las de otro origen (FR-009c, FR-010). Reversión: reconstruir `{R2_PUBLIC_BASE_URL}/{key}` (SC-008).

**Comprobantes del comensal (`receipt_file_url`, carpeta `comprobantes`) quedan fuera de alcance (FR-014)**: `R2_PUBLIC_BASE_URL` se conserva apuntando al dominio viejo y las funciones `presign_payment_receipt` / `presign_receipt` / `attach_receipt` de `cart/service.py` no se tocan — siguen guardando y sirviendo la URL completa como hoy, y su characterization test sigue en verde.

Sin dependencias nuevas (Principio IX). El cambio de comportamiento en la subida (se persiste la key, no la URL pública) es una **decisión de negocio** solicitada por el propietario y exige la entrada **`A-73`** en `specs/000-reconocimiento/registro-de-anomalias.md` **antes** de implementar (Principio II) — este plan la deja identificada; `/speckit-tasks` la secuencia como prerrequisito.

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend`, imagen `python:3.12-slim`, sin cambio). `pos-heladeria` (Angular 20) **no se toca**.

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50, Pydantic v2, boto3 (cliente R2 S3-compatible, ya presente). **Ninguna dependencia nueva** (Principio IX): `app/core/storage.py` ya usa boto3 y ya expone `public_url_for` / `key_from_public_url` / `delete_object` / `build_object_key`; esta funcionalidad amplía esas primitivas, no añade herramientas.

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin cambios de esquema, sin migración de Alembic.** Los tres campos ya existen y sus tipos siguen sirviendo: `products.image_url` `String(500)`, `shared.tenants.logo_url` `String(500)`, `payment_methods.payment_info` `JSONB`. Una key (`{tenant}/{carpeta}/{uuid}.{ext}`, ~60 chars) es más corta que la URL que reemplaza, así que no hay que ensanchar ninguna columna. El objeto físico en Cloudflare R2 **no se mueve, renombra ni vuelve a subir** (FR-009a, FR-017).

**Testing**: `unittest` (biblioteca estándar) vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` en `pos-backend` (no hay `pytest`). Se añaden tests nuevos junto a los existentes; se actualiza **una** aserción en `test_products_service.py` (ver Constitution Check, fila III).

**Target Platform**: Linux server (contenedor Docker existente de `pos-backend`). El script de migración corre como `python -m app.scripts.migrate_image_keys_relative` en el mismo contenedor, igual que `migrate_payment_methods_catalog`.

**Project Type**: web — solo cambia el backend FastAPI (`pos-backend`). El frontend Angular (`pos-heladeria`) y la PWA del comensal no cambian: el contrato de la API (URL lista para usar por imagen) se preserva byte a byte para el consumidor (FR-007).

**Performance Goals**: sin objetivo de rendimiento nuevo. El ensamblado key → URL es concatenación de strings en el serializador de respuesta (coste despreciable, sin I/O). La migración es un job puntual fuera de la ruta de request; recorre `shared.tenants` una vez y `products` + `payment_methods` por cada schema de tenant, en lotes por schema con `commit` por schema (patrón de `migrate_payment_methods_catalog`).

**Constraints**:
- **Contrato del consumidor inmutable** (FR-007, "Decisiones de compatibilidad"): `GET /products`, `GET /products/{id}`, respuestas de guardado de producto, `GET /tenant`, `GET /menu`, `GET /cart/payment-methods` y `GET /sales/payment-methods` siguen devolviendo, por cada imagen, **una URL absoluta lista para renderizar**. Lo único que cambia es el dominio (`assets.skeilopos.com` en vez de `pub-…r2.dev`) y que esa URL ya no coincide con lo que hay en base de datos.
- **En base de datos nunca queda esquema ni dominio** (FR-002): la normalización a key en escritura cubre tanto la URL pública anterior como la del dominio nuevo que el frontend reenvía al guardar un formulario sin tocar la imagen (FR-004).
- **Normalizar antes de comparar** (FR-011/FR-012, US3): en `update_product` y `update_tenant`, la comparación "¿cambió la imagen?" que decide si se borra el objeto anterior debe hacerse **key contra key**. Si se compara la key almacenada contra la URL absoluta entrante, un guardado que no tocó la imagen se interpreta como cambio y borra el objeto vigente (imagen rota). El borrado sigue siendo best-effort y post-commit (A-44 / spec 021, intacto).
- **Otro origen intacto** (FR-010, FR-013): un valor cuya URL no empieza por ninguno de los prefijos del bucket gestionado (imagen histórica de Supabase u otra fuente) se conserva tal cual en migración y en escritura, se muestra sin modificar, y nunca se intenta borrar su objeto.
- **Comprobantes fuera de alcance** (FR-014): `R2_PUBLIC_BASE_URL` se mantiene apuntando a `pub-…r2.dev`; `cart/service.py` (presign y attach de comprobante) no se modifica; `test_cart_payment_attempts.py` no se toca.
- **Tolerancia de lectura temporal** (FR-009b): aunque una fila se escape de la migración y siga con la URL vieja, `asset_display_url` la muestra igual (reconoce el prefijo viejo y lo reescribe al dominio nuevo al vuelo). Esta tolerancia puede retirarse en una spec posterior una vez verificado que no quedan filas con la forma anterior.

**Scale/Scope**:
- `pos-backend`: `app/core/storage.py` (3 helpers nuevos, 1 renombrado), `app/core/config.py` + `.env.example` (1 setting nuevo `ASSETS_BASE_URL`), 1 tipo `Annotated` compartido (`app/core/schema_types.py` o junto a los helpers), esquemas de request/response de `products`, `tenant`, `menu`, `sales`, `cart` (anotaciones de campo, sin lógica nueva), `products/service.py` y `tenant/router.py` (usar el helper de borrado nuevo; la normalización llega ya hecha desde el esquema), `sales/service.py` + `sales/router.py` (normalizar `payment_info`, ensamblar en la respuesta admin), 1 script de migración nuevo, tests nuevos + 1 aserción actualizada.
- `pos-heladeria`: **0 archivos**.
- 3 historias de usuario priorizadas (P1 subida con el esquema nuevo · P2 compatibilidad con lo ya cargado + migración · P3 borrado del objeto anterior se preserva), entregables en ese orden. P2 incluye el script de migración y la tolerancia de lectura.

## Constitution Check

*GATE: Debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (ver final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `specs/080-imagenes-key-relativa-r2/spec.md` aprobado, con 5 aclaraciones registradas (sesión 2026-09-08) y checklist de calidad en verde. |
| II. El Comportamiento Existente Sigue Protegido | ⚠️ Pass **condicionado** — la subida de imágenes de producto, logo y método de pago cambia de comportamiento de forma deliberada y solicitada por el negocio: se persiste la **key** en vez de la URL pública completa. La spec (§"Impacto sobre el Sistema Existente") ya lo declara. Requiere la entrada **`A-73`** en `specs/000-reconocimiento/registro-de-anomalias.md` con quién/cuándo/qué cambia/por qué/funcionalidades afectadas, creada **antes** de implementar la Historia 1 — `/speckit-tasks` la ordena como T00x prerrequisito y los commits de esa fase la citan. La **visualización** de imágenes no cambia de comportamiento observable (misma imagen, distinto dominio de origen — FR-005/SC-006), y los **comprobantes** quedan explícitamente fuera (FR-014), por lo que solo la subida entra en este condicionamiento. |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ⚠️ Pass **condicionado** — un único test se ve afectado: `test_products_service.py::test_a44_fallo_de_delete_object_no_revierte_el_cambio_de_imagen` (prefijo `"CONGELA comportamiento corregido:"`, spec 021 / A-44) afirma `result.image_url == NEW_URL`, donde `NEW_URL` es, bajo la config de test (`R2_PUBLIC_BASE_URL = https://example.invalid`), una URL del bucket gestionado. Con esta spec el valor persistido pasa a ser su **key**. La aserción se actualiza a `result.image_url == <key de NEW_URL>` citando la spec 080 y `A-73` en el mismo commit. **El comportamiento que ese test congela —el borrado best-effort no revierte el cambio ya persistido— se preserva exactamente** (es precisamente la Historia 3 de esta spec, US3/FR-012); solo cambia la forma del valor con el que se compara. Los otros dos tests A-44 (`test_a44_fallo_de_commit_no_deja_referencia_rota`, `test_a44_camino_feliz_borra_despues_del_commit`) no afirman la forma del valor y **no se tocan**. `test_cart_payment_attempts.py` (comprobantes, `"CONGELA"`) no se toca (FR-014). Ningún test congela hoy la serialización de `image_url`/`logo_url`/`payment_info.qr` como URL absoluta (verificado 2026-09-08). Se añaden tests nuevos junto a los existentes. |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — el comportamiento nuevo (persistir key, ensamblar URL en el servidor, dominio personalizado, migración) está definido y acotado en el spec; el criterio de éxito es la conformidad con él (SC-001…SC-009) más la ausencia de regresiones no autorizadas en los consumidores (contrato de URL lista para usar, FR-007). |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — cada cambio se asocia a un FR. Renombrar `key_from_public_url` → `object_key_for_deletion` y ampliar su semántica es lo mínimo que exige FR-011/FR-013 (borrar aceptando key directa e ignorando otros orígenes), no una generalización especulativa. No se toca `build_object_key`, ni el esquema de nombres de keys, ni el mecanismo de presign, ni el proveedor (FR-015/FR-016/FR-017). El frontend no se toca. |
| VI. Evolución Incremental | ✅ Pass — una sola clase de cambio (cómo se referencia una imagen: key en BD, URL ensamblada en respuesta), dividida en 3 historias verificables por separado. La migración de datos es su propia unidad (P2), con `--report-only` y `--revert`, separada de la lógica de lectura/escritura (P1). Sin cambio de arquitectura, sin cambio de proveedor de almacenamiento. |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — ninguna factura ni venta emitida se recalcula ni se re-representa. Estas referencias de imagen no forman parte del importe ni de la representación contable de una factura; el logo se resuelve vigente en cada recibo, como hoy (no "congelado" por recibo — Assumptions). La migración cambia la **representación de la ubicación** de un objeto en BD, no el objeto ni ningún dato histórico contable (FR-009a). |
| VIII. Evolución del Modelo de Datos | ✅ Pass (con artefacto) — sin entidades, campos ni relaciones nuevas. Cambia el **contenido** de tres campos existentes (URL → key). [`data-model.md`](./data-model.md) especifica: forma de valor antes/después, tabla de verdad de normalización/visualización/borrado, predicado de selección de la migración, transformación, **estrategia de reversión** (`{R2_PUBLIC_BASE_URL}/{key}`) e **idempotencia** (FR-009c, SC-009). Compatibilidad con datos existentes: migración única + tolerancia de lectura temporal (FR-009b) + otros orígenes intactos (FR-010). |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica justificación) — cero dependencias nuevas. Se reutiliza boto3 / `app/core/storage.py` (ya en producción) y las primitivas de Pydantic v2 (`Annotated` + `AfterValidator` / `PlainSerializer`, ya usadas en el proyecto, p. ej. `UtcDatetime` en `app/core/timezone.py`). |
| X. Verificación Obligatoria | ✅ Pass (planificado) — [`research.md`](./research.md) §12 define la matriz: tests unitarios de los 3 helpers (todas las formas de valor de `data-model.md`); tests de escritura (la key es lo que se persiste, forma vieja y nueva se normalizan, "otro origen" intacto); test de que el guardado sin tocar la imagen **no** dispara borrado (US3); tests de la migración (idempotencia, reversión, "otro origen" y comprobantes intactos); characterization existentes en verde (con la aserción A-44 actualizada); criterios de aceptación vía [`quickstart.md`](./quickstart.md), incluida la corrida completa de la suite de `pos-backend`. |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — decisiones de negocio (persistir key; dominio `assets.skeilopos.com`; comprobantes fuera; migración con reversión; tolerancia de lectura temporal; otros orígenes intactos) ya fijadas en el spec y sus aclaraciones. Este plan solo resuelve el "cómo" (helpers centrales, validador de esquema, tipo `Annotated`, script de migración). `A-73` registra el cambio de comportamiento. |
| XII. Trazabilidad | ✅ Pass — Necesidad (pedido del propietario, 2026-09-08) → `spec.md` + Aclaraciones → este plan → `research.md` / `data-model.md` / `contracts/` / `quickstart.md` → `tasks.md` (`/speckit-tasks`) → tests. `A-73` cierra la cadena del cambio de comportamiento. |
| XIII. Todo en Español de Colombia | ✅ Pass — todos los artefactos de este feature en español de Colombia. La funcionalidad no introduce textos visibles nuevos al usuario final (SC-006): solo cambia el origen de una imagen ya visible. |

**Complexity Tracking**: sin violaciones de diseño que justificar — la sección no aplica. Las dos filas condicionadas (II: registrar `A-73` antes de implementar; III: actualizar una aserción citando esa decisión) son prerrequisitos de proceso, no alternativas de arquitectura más simples que se hayan descartado.

**Re-chequeo post Fase 1**: tras generar `research.md`, `data-model.md`, `contracts/asset-reference.md`, `contracts/uploads-presign.md`, `contracts/data-migration.md` y `quickstart.md`, ninguna decisión de diseño introdujo una dependencia nueva, una migración de esquema ni una desviación de la tabla anterior. El diseño (helpers centrales en `storage.py` como única fuente de la regla; normalización en el validador de esquema **antes** de la comparación de borrado; ensamblado vía tipo `Annotated` en las respuestas; script de migración con `--revert`, espejo de `migrate_payment_methods_catalog`) confirma que **no hace falta tocar el frontend ni ningún consumidor**, y que las dos filas condicionadas se resuelven en `/speckit-tasks` + `/speckit-implement`, no aquí.

## Project Structure

### Documentation (this feature)

```text
specs/080-imagenes-key-relativa-r2/
├── plan.md                       # Este archivo (/speckit-plan)
├── research.md                   # Fase 0 (/speckit-plan)
├── data-model.md                 # Fase 1 (/speckit-plan)
├── quickstart.md                 # Fase 1 (/speckit-plan)
├── contracts/                    # Fase 1 (/speckit-plan)
│   ├── asset-reference.md         #   Contrato de los 3 helpers: normalizar / mostrar / key-para-borrar
│   ├── uploads-presign.md         #   POST /uploads/presign — qué cambia y qué no en la respuesta
│   └── data-migration.md          #   Script de migración: selección, transformación, reversión, idempotencia
├── checklists/
│   └── requirements.md            # (ya existente)
└── tasks.md                      # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repositorio `pos-backend`, independiente de `pos-specs`)

```text
pos-backend/
├── .env.example                              # + ASSETS_BASE_URL (dominio personalizado de assets)
├── app/
│   ├── core/
│   │   ├── config.py                         # + ASSETS_BASE_URL: str = Field(..., env="ASSETS_BASE_URL")
│   │   ├── storage.py                        # + asset_display_url(value) -> str | None
│   │   │                                     # + normalize_asset_ref(value) -> str | None
│   │   │                                     # + object_key_for_deletion(value) -> str | None  (reemplaza key_from_public_url)
│   │   │                                     # + _managed_bucket_prefixes() -> tuple[str, ...]  (ASSETS_BASE_URL + R2_PUBLIC_BASE_URL)
│   │   │                                     #   public_url_for / build_object_key / delete_object: SIN CAMBIOS (comprobantes los siguen usando)
│   │   └── schema_types.py                   # NUEVO (o al pie de storage.py): AssetUrl = Annotated[str | None, PlainSerializer(asset_display_url)]
│   │   │                                     #   y AssetRefIn = Annotated[str | None, AfterValidator(normalize_asset_ref)]
│   ├── api/v1/
│   │   ├── uploads/
│   │   │   ├── router.py                     # public_url ahora se arma con asset_display_url(key) (dominio nuevo); resto igual
│   │   │   └── schemas.py                    # PresignResponse.public_url: docstring aclara que ya no es "lo que se guarda"
│   │   ├── products/
│   │   │   ├── schemas.py                    # ProductCreate/Update.image_url: AssetRefIn ; ProductResponse.image_url: AssetUrl
│   │   │   └── service.py                    # update_product: comparar key vs key (data.image_url ya viene normalizada);
│   │   │                                     #   old_key = object_key_for_deletion(old_image_url). create_product: sin cambios de lógica.
│   │   ├── tenant/
│   │   │   ├── schemas.py                    # TenantUpdate.logo_url: AssetRefIn ; TenantInfoResponse.logo_url: AssetUrl
│   │   │   └── router.py                     # update_tenant: comparar key vs key; old_key = object_key_for_deletion(old_logo_url)
│   │   ├── menu/
│   │   │   ├── schemas.py                    # MenuProductResponse.image_url: AssetUrl ; MenuBusinessResponse.logo_url: AssetUrl
│   │   │   └── router.py                     # SIN CAMBIOS — el serializador del tipo AssetUrl actúa al responder
│   │   ├── sales/
│   │   │   ├── schemas.py                    # PaymentMethod{Create,Update}.payment_info: normalización por-valor (validador de dict);
│   │   │   │                                 #   PaymentMethodResponse: payment_info se ensambla en el router (necesita catalog.fields)
│   │   │   ├── service.py                    # create/update_payment_method: payment_info ya viene normalizada desde el esquema
│   │   │   └── router.py                     # list/create/update payment-methods: construir PaymentMethodResponse ensamblando
│   │   │                                     #   las claves format="image" con asset_display_url (helper payment_method_response())
│   │   └── cart/
│   │       ├── schemas.py                    # DinerPaymentMethod: @model_validator(mode="after") ensambla payment_info[k] para
│   │       │                                 #   cada k con fields[*].format == "image"
│   │       └── service.py                    # SIN CAMBIOS — list_payment_methods ya carga selectinload(catalog); presign_* de
│   │                                         #   comprobantes NO se tocan (FR-014)
│   ├── scripts/
│   │   └── migrate_image_keys_relative.py    # NUEVO — espejo de migrate_payment_methods_catalog.py:
│   │                                         #   --report-only | (escribe) | --revert ; idempotente; con_db(None) para
│   │                                         #   shared.tenants.logo_url + with_db(schema) para products.image_url y
│   │                                         #   payment_methods.payment_info ; solo toca valores con prefijo del bucket gestionado
│   └── characterization_tests/
│       ├── test_products_service.py          # ACTUALIZAR 1 aserción (test_a44_fallo_de_delete_object_...): NEW_URL -> su key,
│       │                                     #   citando spec 080 + A-73. Los otros 2 tests A-44: SIN TOCAR.
│       ├── test_storage_asset_refs.py        # NUEVO — tabla de verdad de asset_display_url / normalize_asset_ref /
│       │                                     #   object_key_for_deletion (todas las formas de data-model.md §2)
│       ├── test_uploads_presign_key.py       # NUEVO — /uploads/presign devuelve key + public_url con dominio nuevo
│       ├── test_products_image_key.py        # NUEVO — al guardar, image_url persiste como key (forma vieja y nueva);
│       │                                     #   guardar sin tocar la imagen NO llama delete_object (US3); respuesta trae URL absoluta
│       ├── test_tenant_logo_key.py           # NUEVO — equivalente para logo_url
│       ├── test_payment_methods_qr_key.py    # NUEVO — payment_info["qr"] persiste como key; DinerPaymentMethod y
│       │                                     #   PaymentMethodResponse lo devuelven como URL absoluta; campos no-imagen intactos
│       └── test_migrate_image_keys_relative.py # NUEVO — idempotencia (2 corridas = 1), reversión exacta, "otro origen" y
│                                             #   comprobantes intactos, filas ya-key / vacías sin cambio
```

**Structure Decision**: aplicación web; **solo se toca `pos-backend`**. El núcleo de la regla vive en un solo lugar (`app/core/storage.py`) y se aplica en los bordes mediante dos tipos `Annotated` reutilizables (entrada: normaliza a key; salida: ensambla URL) más dos puntos explícitos para `payment_info` (que necesita la metadata `fields` del catálogo para saber qué clave es imagen). El script de migración replica el patrón ya probado de `app/scripts/migrate_payment_methods_catalog.py` (recorrido por schema de tenant + `shared`, `--report-only`, idempotente). No se crean paquetes ni módulos de alto nivel nuevos. `pos-heladeria` no aparece en este plan porque el contrato hacia el consumidor (URL absoluta lista para usar por imagen) se preserva (FR-007).

## Complexity Tracking

*Sin violaciones del Constitution Check — sección no aplica.* Las filas II y III de la tabla son prerrequisitos de proceso (registrar `A-73` antes de implementar; actualizar una aserción de test citando esa decisión), no violaciones de diseño que exijan una alternativa más simple.
