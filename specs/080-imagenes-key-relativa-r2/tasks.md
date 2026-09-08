---
description: "Task list para spec 080 — imágenes como key relativa servidas por dominio personalizado"
---

# Tasks: Almacenar imágenes como key relativa y servirlas por dominio personalizado

**Input**: Design documents de `/specs/080-imagenes-key-relativa-r2/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/asset-reference.md](./contracts/asset-reference.md), [contracts/uploads-presign.md](./contracts/uploads-presign.md), [contracts/data-migration.md](./contracts/data-migration.md), [quickstart.md](./quickstart.md)

**Tests**: incluidos — **no son opcionales aquí**. El plan (Constitution Check, Principio X — "Verificación Obligatoria"; [research.md §12](./research.md)) enumera la matriz completa: tabla de verdad de los 3 helpers, la key es lo que se persiste (forma vieja y nueva se normalizan), guardar sin tocar la imagen **no** dispara borrado (US3), tests de la migración (idempotencia, reversión, "otro origen" y comprobantes intactos), characterization existentes en verde (con **una** aserción A-44 actualizada), y los criterios de aceptación vía [quickstart.md](./quickstart.md). Mismo criterio que [spec 079](../079-paginacion-ordenes-mesas/tasks.md).

**Verificación diferida y manual**: los recorridos de [quickstart.md](./quickstart.md) que requieren stack corriendo + navegador + tenant sembrado (T017, T021, T024, T028) no son ejecutables en esta sesión. Incluyen SC-005 end-to-end (cambiar `ASSETS_BASE_URL`, reiniciar y recargar el menú reapunta todas las imágenes sin migración — la parte unitaria del helper la cubre T004) y SC-002/SC-003 (100 % de imágenes previas siguen viéndose, cero rotas). Responsable: propietario del proyecto, antes del despliegue de cada historia.

**Organización**: por historia de usuario de `spec.md` (US1 → US2 → US3, en orden de prioridad P1 → P2 → P3). **Un solo repositorio de código**, independiente de `pos-specs`:

- `../pos-backend` — FastAPI. Se tocan `app/core/storage.py` (3 helpers), `app/core/config.py` + `.env.example` (1 setting), `app/core/schema_types.py` (nuevo, 2 tipos `Annotated`), esquemas de request/response de `products` / `tenant` / `menu` / `sales` / `cart`, `products/service.py` + `tenant/router.py` (helper de borrado), `sales/router.py` (ensamblado admin), `uploads/router.py` (dominio del `public_url`), 1 script de migración nuevo, tests nuevos + 1 aserción actualizada. **Cero** routers, modelos, migraciones de Alembic o dependencias nuevas.
- `../pos-heladeria` — **0 archivos** ([research.md §9](./research.md)): el contrato hacia el consumidor (URL absoluta lista para usar por imagen, FR-007) se preserva.

El único artefacto fuera de `pos-backend` es la entrada **`A-73`** en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (T002), **bloqueante para la Historia 1** (Principio II). Las tres piezas son 1:1 con las tres historias y verificables por separado:

1. **US1** — la subida de imagen de producto / logo / método de pago persiste la **key**; la respuesta arma la URL absoluta contra `ASSETS_BASE_URL`; `POST /uploads/presign` devuelve el `public_url` contra el dominio nuevo.
2. **US2** — migración única, idempotente y reversible de las filas que hoy guardan una URL del bucket gestionado; la tolerancia de lectura (una fila no migrada se muestra igual) ya la aporta `asset_display_url`.
3. **US3** — el borrado best-effort del objeto anterior al reemplazar imagen (producto y logo) se preserva (A-44 / spec 021), aceptando ahora una key directa e ignorando orígenes ajenos.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1 a US3 — solo en fases de historia de usuario
- Cada tarea lleva la ruta con prefijo de repo (`pos-backend/…`, `pos-specs/…`)

---

## Phase 1: Setup

**Purpose**: línea base verde en `pos-backend` antes de tocar código, y la autorización de proceso (Principio II) para la historia que cambia comportamiento.

- [X] T001 [P] Registrar la línea base de tests de **`pos-backend`**: correr
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` desde `pos-backend/` y
      anotar el conteo verde/rojo, con atención especial a los tests que este feature **NO** debe
      romper: `pos-backend/app/characterization_tests/test_cart_payment_attempts.py` (comprobantes,
      prefijo `"CONGELA"`, FR-014 — **intacto**) y los **tres** tests A-44 de
      `pos-backend/app/characterization_tests/test_products_service.py`
      (`test_a44_fallo_de_delete_object_no_revierte_el_cambio_de_imagen` — **1 aserción se actualiza**
      en T016; `test_a44_fallo_de_commit_no_deja_referencia_rota` y
      `test_a44_camino_feliz_borra_despues_del_commit` — **sin tocar**), para distinguir después
      cualquier regresión de esta spec de un fallo preexistente (Principio X)
- [X] T002 [P] Registrar la anomalía **`A-73`** en
      `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`, como entrada nueva **justo después
      de `A-72`** (antes de la sección `## Nota sobre una entrada de memoria-historica.md deliberadamente
      excluida`), con el **mismo formato que `A-70`/`A-71`/`A-72`** (bloques **Qué cambia**, **Por qué
      cambia**, **Quién tomó la decisión y cuándo**, **Funcionalidades afectadas**, **Clasificación**,
      **Tratamiento acordado**) y el contenido de [research.md §13](./research.md): (1) la subida de
      imagen de producto (`Product.image_url`), logo del negocio (`Tenant.logo_url`) e imagen de método
      de pago (`PaymentMethod.payment_info`, clave `format:"image"`) persiste la **key relativa**
      (`{tenant}/{carpeta}/{archivo}`), **no** la URL pública completa; (2) la URL de visualización se
      arma en el servidor contra `ASSETS_BASE_URL` (`assets.skeilopos.com`); (3) `POST /uploads/presign`
      devuelve `public_url` contra el dominio nuevo; (4) el extractor de key para borrar el objeto
      anterior (`key_from_public_url`) se renombra/amplía a `object_key_for_deletion` para aceptar
      también una key directa e ignorar orígenes ajenos (FR-011/FR-013). **Por qué**: desacoplar el
      contenido almacenado del dominio que lo sirve — un cambio futuro de dominio es solo configuración,
      cero migración (SC-005). **Quién/cuándo**: propietario del proyecto, 2026-09-08, en `spec.md` +
      las 5 aclaraciones de esa fecha. **Funcionalidades afectadas**: subida en `products` / `tenant` /
      `sales`; el borrado best-effort del objeto anterior al reemplazar imagen (producto y logo);
      lectura de imágenes (menú QR, catálogo POS, formulario de producto, barra lateral, recibos,
      checkout del comensal) — misma imagen, distinto dominio de origen, sin cambio observable (SC-006).
      **Comprobantes del comensal (`receipt_file_url`, carpeta `comprobantes`) EXPLÍCITAMENTE FUERA**
      (FR-014). El contrato hacia los consumidores de la API no cambia (FR-007) → `pos-heladeria` no se
      toca. **Clasificación**: DECISIÓN DE NEGOCIO. **Tratamiento**: `specs/080-imagenes-key-relativa-r2/tasks.md`;
      debe existir **antes** de implementar la Historia 1 (los commits de US1 citan `A-73`). **No
      retroactivo** (Principio VII): sin cambio de esquema, sin migración de Alembic; la migración de
      datos (FR-009) solo reescribe cómo la base de datos referencia la ubicación de un objeto, con
      estrategia `--revert` (SC-008); ninguna factura ni venta emitida se altera. Revertir los commits
      de `pos-backend` + correr `--revert` restaura el estado previo. **Bloquea US1**; **US2 y US3 no
      dependen de esta tarea** para su código, pero US1 va antes que ambas

**Checkpoint**: suite de `pos-backend` con conteo base registrado; `A-73` existe y cierra la cadena de trazabilidad (Necesidad → `spec.md` + Aclaraciones → `plan.md` → `research.md` → `tasks.md` → `A-73`).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: la única fuente de la regla key ↔ URL — los tres helpers puros de `app/core/storage.py` y su configuración — que consumen por igual la escritura (US1), la lectura/tolerancia (US1/US2), la migración (US2) y el borrado (US3). Ninguna historia puede empezar hasta que esto esté completo. `public_url_for` / `build_object_key` / `delete_object` **no se tocan** (los siguen usando los comprobantes, FR-014/FR-016/FR-017).

- [X] T003 En `pos-backend/app/core/config.py`, añadir el setting **`ASSETS_BASE_URL: str = Field(..., env="ASSETS_BASE_URL")`** (obligatorio, sin default, coherente con el resto de settings de R2 — un despliegue sin la variable falla al arrancar). Documentarlo en `pos-backend/.env.example` (`ASSETS_BASE_URL=https://assets.skeilopos.com`, con comentario: dominio personalizado de R2 para servir imágenes de producto / logo / método de pago; **no** cambiar `R2_PUBLIC_BASE_URL`, que sigue sirviendo comprobantes y la reversión de la migración). En `pos-backend/app/core/storage.py`, añadir `_managed_bucket_prefixes() -> tuple[str, ...]` = `(f"{settings.ASSETS_BASE_URL.rstrip('/')}/", f"{settings.R2_PUBLIC_BASE_URL.rstrip('/')}/")`, de-duplicado ([contracts/asset-reference.md §Configuración](./contracts/asset-reference.md), [research.md §1](./research.md)). **Bloquea T004, T005.**
- [X] T004 [P] Crear `pos-backend/app/characterization_tests/test_storage_asset_refs.py`: tabla de
      verdad de los tres helpers para **todas** las formas de valor de [data-model.md §2–§3](./data-model.md)
      (`ASSETS_BASE_URL` / `R2_PUBLIC_BASE_URL` fijados en la config de test):
      **`normalize_asset_ref`** — `None`/`""`/`"   "` → `None`; key → misma key; URL `pub-…` → key;
      URL `assets.skeilopos.com` → key; URL con query string → key con query string; URL de otro
      origen → intacta (FR-010).
      **`asset_display_url`** — `None`/`""` → `None`; key → `{ASSETS_BASE_URL}/{key}`; URL `pub-…` →
      `{ASSETS_BASE_URL}/{key}` (tolerancia de lectura, FR-009b); URL `assets.skeilopos.com` → igual;
      otro origen → intacta.
      **`object_key_for_deletion`** — `None`/`""` → `None`; key → misma key; URL `pub-…` → key; URL
      `assets.skeilopos.com` → key; otro origen → `None` (FR-013).
      Más los invariantes de [data-model.md §3](./data-model.md) (idempotencia de
      `normalize_asset_ref∘asset_display_url` para V0/V1/V4; `object_key_for_deletion(x) is None` ⟺ x es
      V0 o V4; ninguno hace I/O).
      Más un caso **SC-005**: con una key fija, cambiar `ASSETS_BASE_URL` en la config de test
      (override/`monkeypatch` del settings) y verificar que `asset_display_url` pasa a devolver el
      dominio nuevo **sin** que cambie ningún valor de entrada — prueba en CI que el dominio de
      visualización es configuración pura (FR-008, SC-005).
      Debe **fallar** antes de T005
- [X] T005 En `pos-backend/app/core/storage.py`, implementar los tres helpers puros (sin I/O), según
      [contracts/asset-reference.md](./contracts/asset-reference.md): `normalize_asset_ref(value: str | None) -> str | None`
      (para persistir, FR-004), `asset_display_url(value: str | None) -> str | None` (para responder,
      FR-005/FR-009b) y `object_key_for_deletion(value: str | None) -> str | None` (para borrar,
      FR-011/FR-013). Los tres reconocen el "bucket gestionado" contra `_managed_bucket_prefixes()`
      (por prefijo, tolerando query string y diferencias de host — Edge Cases del spec).
      `object_key_for_deletion` es la versión **ampliada y renombrada** de `key_from_public_url` (hoy
      reconoce un solo prefijo y devuelve `None` para una key directa); se **añade** como función nueva
      y `key_from_public_url` se deja intacta por ahora (sus 2 llamadores migran en T023, US3). Hace
      pasar T004

**Checkpoint**: los tres helpers disponibles y probados en aislamiento; la config `ASSETS_BASE_URL` cargada. La escritura, la lectura, la migración y el borrado tienen ya su única fuente de verdad.

---

## Phase 3: User Story 1 - Subir una imagen y verla con el nuevo esquema (Priority: P1) 🎯 MVP

**Goal**: al guardar un producto, un logo o un método de pago con imagen, el campo en base de datos
queda como **key** (`{tenant}/{carpeta}/{archivo}`, sin esquema ni dominio — FR-001/FR-002), tanto si
el cliente manda la key como si reenvía una URL absoluta del bucket gestionado (vieja o nueva, FR-004).
Toda respuesta que hoy incluye una imagen la sigue entregando como **URL absoluta lista para usar**,
ahora contra `https://assets.skeilopos.com/{key}` (FR-005/FR-006/FR-007). `POST /uploads/presign`
devuelve `public_url` contra el dominio nuevo. La columna nunca lleva la URL; el ensamblado ocurre en
el serializador de respuesta.

**Independent Test**: subir una imagen de producto nueva y verificar (1) que `products.image_url`
contiene solo la key (sin `http`, sin dominio) y (2) que `GET /products/{id}` y `GET /menu` la
devuelven como `https://assets.skeilopos.com/{key}` y la imagen se ve en el menú QR y en el POS.
Repetir con el logo (`GET /tenant`, barra lateral, recibo) y con un método de pago con QR
(`GET /cart/payment-methods`, `GET /sales/payment-methods`). Guardar un formulario **sin tocar la
imagen** (el cliente reenvía la URL de visualización) y confirmar que el campo sigue siendo key
([quickstart.md §Historia 1](./quickstart.md)). No depende de US2 ni de US3.

**⚠️ Depende de `A-73` (T002)** — la subida que persiste la key en vez de la URL pública es el cambio
de comportamiento que `A-73` autoriza; los commits de esta fase citan `A-73`.

### Tests for User Story 1

- [X] T006 [P] [US1] Crear `pos-backend/app/characterization_tests/test_uploads_presign_key.py`:
      `POST /uploads/presign` con `folder: "products"` (y `"logo"`, `"payment-methods"`) → respuesta
      con `key` = `{tenant}/{folder}/{uuid}.{ext}` (nunca el `filename` del cliente, FR-016) y
      `public_url` = `https://assets.skeilopos.com/{key}` (dominio nuevo, [contracts/uploads-presign.md](./contracts/uploads-presign.md));
      `content_type` fuera de `CONTENT_TYPE_EXTENSIONS` → `422` (sin cambios); `folder: "comprobantes"`
      → sigue **rechazado** (whitelist intacta, FR-016). Debe **fallar** antes de T015
- [X] T007 [P] [US1] Crear `pos-backend/app/characterization_tests/test_products_image_key.py`:
      al `POST`/`PATCH /products` con `image_url` = key **o** = `https://pub-…r2.dev/{key}` **o** =
      `https://assets.skeilopos.com/{key}`, la columna `products.image_url` persiste **la key** en los
      tres casos (FR-002/FR-004, SC-001); `image_url` de otro origen → se persiste **intacta** (FR-010);
      `image_url` vacío/omitido → columna `NULL` (escenario 5). La respuesta (`ProductResponse` y
      subclases `list`/`detail`/`save`) y `MenuProductResponse` devuelven `image_url` como
      `https://assets.skeilopos.com/{key}` mientras la fila leída del ORM sigue con la key (FR-006).
      Debe **fallar** antes de T011/T012
- [X] T008 [P] [US1] Crear `pos-backend/app/characterization_tests/test_tenant_logo_key.py`:
      equivalente a T007 para `logo_url` — `PATCH /tenant` persiste la key (forma vieja y nueva se
      normalizan); `GET /tenant` (`TenantInfoResponse`) y `GET /menu` (`MenuBusinessResponse`)
      devuelven la URL absoluta contra `assets.skeilopos.com`; logo vacío → `NULL`, sin URL en la
      respuesta. Debe **fallar** antes de T013
- [X] T009 [P] [US1] Crear `pos-backend/app/characterization_tests/test_payment_methods_qr_key.py`:
      al `POST`/`PATCH /sales/payment-methods`, el valor de la clave `format:"image"` de
      `payment_info` (típ. `qr`) persiste como **key** (forma vieja y nueva se normalizan, FR-002);
      las claves no-imagen (`celular`, `cuenta`, `titular`) quedan **idénticas** (Edge Cases);
      `DinerPaymentMethod` (`GET /cart/payment-methods`) y `PaymentMethodResponse`
      (`GET /sales/payment-methods` sin `available`, `POST`, `PATCH`) devuelven ese valor como
      `https://assets.skeilopos.com/{key}`; el retorno del **servicio** (ORM) sigue con la key
      (varios characterization de `test_sales_payment_methods_catalog.py` lo comparan — no deben
      romperse). Debe **fallar** antes de T014/T015

### Implementation for User Story 1

- [X] T010 [US1] Crear `pos-backend/app/core/schema_types.py` con los dos tipos `Annotated`
      reutilizables (precedente: `app/core/timezone.py::UtcDatetime`):
      `AssetRefIn = Annotated[str | None, AfterValidator(normalize_asset_ref)]` (entrada: normaliza a
      key) y `AssetUrl = Annotated[str | None, PlainSerializer(asset_display_url, return_type=str | None)]`
      (salida: ensambla URL). Importan de `app/core/storage.py`. Depende de T005
- [X] T011 [US1] En `pos-backend/app/api/v1/products/schemas.py`: `ProductCreate.image_url` y
      `ProductUpdate.image_url` → `AssetRefIn`; `ProductResponse.image_url` → `AssetUrl` (cubre
      `ProductListResponse` / `ProductDetailResponse` / `ProductSaveResponse` por herencia). En
      `pos-backend/app/api/v1/products/service.py::create_product` no hay cambio de lógica (persiste
      `data.image_url`, ya key). `update_product` **no** se toca en esta tarea salvo que la comparación
      siga funcionando (ambos lados ya key gracias a `AssetRefIn`); el swap de `key_from_public_url` es
      T023 (US3). Hace pasar la parte de persistencia y respuesta de T007. **Commit cita `A-73`**.
      Depende de T010
- [X] T012 [P] [US1] En `pos-backend/app/api/v1/tenant/schemas.py`: `TenantUpdate.logo_url` →
      `AssetRefIn`; `TenantInfoResponse.logo_url` → `AssetUrl`. En
      `pos-backend/app/api/v1/menu/schemas.py`: `MenuProductResponse.image_url` y
      `MenuBusinessResponse.logo_url` → `AssetUrl`. `tenant/router.py::update_tenant` y
      `menu/router.py` **sin cambios** (el serializador del tipo actúa al responder). Hace pasar la
      parte de persistencia y respuesta de T008. **Commit cita `A-73`**. Depende de T010
- [X] T013 [US1] En `pos-backend/app/api/v1/sales/schemas.py`: en `PaymentMethodCreate.payment_info`
      y `PaymentMethodUpdate.payment_info`, un `field_validator` (dict) que aplica
      `normalize_asset_ref` a **cada valor** string del dict: `{k: normalize_asset_ref(v) or v for k, v in info.items()}`
      (los valores no gestionados quedan idénticos porque `normalize_asset_ref` solo toca los que
      empiezan por un prefijo del bucket gestionado). `sales/service.py::create_payment_method` /
      `update_payment_method` reciben el dict ya normalizado — sin cambio de lógica. Hace pasar la
      parte de persistencia de T009. **Commit cita `A-73`**. Depende de T005
- [X] T014 [US1] Ensamblado en lectura de `payment_info` (necesita la metadata `fields` del catálogo,
      [research.md §4](./research.md)):
      en `pos-backend/app/api/v1/cart/schemas.py::DinerPaymentMethod`, un
      `@model_validator(mode="after")` que, para cada `f in self.fields` con `f["format"] == "image"`,
      reemplaza `self.payment_info[f["key"]]` por `asset_display_url(...)`;
      en `pos-backend/app/api/v1/sales/router.py`, un helper `payment_method_response(method: PaymentMethod) -> PaymentMethodResponse`
      que ensambla las claves `format=="image"` con `asset_display_url` usando `method.fields`, y hacer
      pasar por él los 3 endpoints que devuelven `PaymentMethodResponse` (`GET /sales/payment-methods`
      sin `available`, `POST`, `PATCH`); añadir `selectinload(PaymentMethod.catalog)` al `list` admin
      del router para no disparar N+1. `cart/service.py` **no se toca** (`list_payment_methods` ya hace
      `selectinload(catalog)`; los presign de comprobantes quedan fuera, FR-014). Hace pasar la parte de
      respuesta de T009. **Commit cita `A-73`**. Depende de T005
- [X] T015 [P] [US1] En `pos-backend/app/api/v1/uploads/router.py`, armar `public_url` con
      `asset_display_url(key)` (dominio nuevo) en vez de `public_url_for(key)`; actualizar el docstring
      del endpoint y de `PresignResponse.public_url` en `uploads/schemas.py`: de *"URL pública final
      para guardar en image_url"* a *"URL de visualización lista para usar. **No** es lo que se
      persiste: en base de datos vive la `key` (spec 080). El servidor normaliza a key cualquier URL
      absoluta del bucket gestionado que reciba (FR-004)."* La whitelist de `folder`
      (`products`/`logo`/`payment-methods`) y `build_object_key` **no cambian** (FR-016). Hace pasar
      T006. **Commit cita `A-73`**
- [X] T016 [US1] En `pos-backend/app/characterization_tests/test_products_service.py`, actualizar
      **una** aserción de `test_a44_fallo_de_delete_object_no_revierte_el_cambio_de_imagen`
      ([research.md §11](./research.md)): `self.assertEqual(result.image_url, NEW_URL)` →
      `self.assertEqual(result.image_url, "tenant/products/new.jpg")  # key de NEW_URL` con un
      comentario que cita `spec 080` y `A-73` y explica que el comportamiento congelado (un fallo de
      `delete_object` no revierte el cambio ya persistido) **se mantiene** — solo cambia la forma del
      valor con el que se compara. En el **mismo commit**, confirmar por escrito que
      `test_a44_fallo_de_commit_no_deja_referencia_rota` y `test_a44_camino_feliz_borra_despues_del_commit`
      siguen **intactos y en verde** (Principio III punto 4). **Commit cita `A-73`**. Depende de T011
- [ ] T017 [US1] Ejecutar [quickstart.md §Historia 1](./quickstart.md) (pasos 1.1–1.9) contra
      `pos-backend` + `pos-heladeria` reales con el tenant sembrado y `A-73` registrada: la key es lo
      que se persiste en los tres campos, la imagen se ve desde `assets.skeilopos.com` en menú QR /
      POS / barra lateral / recibo / checkout, guardar sin tocar la imagen no re-persiste una URL, y
      `POST /uploads/presign` devuelve el `public_url` contra el dominio nuevo.
      → **Pendiente** — recorrido con navegador + datos sembrados (no ejecutable en esta sesión; ver T028)

**Checkpoint**: cada imagen nueva de producto / logo / método de pago nace como key; toda respuesta la entrega ensamblada contra `assets.skeilopos.com`; el contrato del consumidor no cambia y `pos-heladeria` sigue sin tocarse. MVP entregable.

---

## Phase 4: User Story 2 - Las imágenes ya cargadas siguen viéndose (Priority: P2)

**Goal**: una migración única (`app/scripts/migrate_image_keys_relative.py`, espejo de
`migrate_payment_methods_catalog.py`) reescribe a su key todas las filas de `Product.image_url`,
`Tenant.logo_url` y los valores de imagen de `PaymentMethod.payment_info` que hoy guardan una URL del
bucket gestionado (FR-009), dejando intactas las que ya son key, las vacías y las de otro origen
(FR-009c/FR-010). Es **idempotente** y re-ejecutable con la app en marcha (FR-009c/SC-009) y lleva
`--revert` que reconstruye `{R2_PUBLIC_BASE_URL}/{key}` (FR-009a/SC-008). La **tolerancia de lectura**
(una fila que se escape de la migración se muestra igual) ya la aporta `asset_display_url` (T005), sin
código extra (FR-009b).

**Independent Test**: tomar un producto / logo / método de pago cuya imagen se cargó **antes** del
cambio (campo con `https://pub-…r2.dev/{key}`); **antes** de migrar, abrir menú QR / POS / recibo /
checkout y verificar que la imagen se ve (tolerancia de lectura); correr la migración y verificar que
el campo quedó como key y la imagen se sigue viendo; verificar que una fila con URL de otro origen
quedó intacta; volver a correr la migración y verificar que no cambia ninguna fila; correr `--revert`
y comparar con un dump previo ([quickstart.md §Historia 2](./quickstart.md)).

**⚠️ Prerrequisito operativo**: el código de US1 (escritura persiste key, lectura ensambla, tolerancia
activa) debe estar desplegado **antes** de correr la migración ([contracts/data-migration.md §Prerrequisitos](./contracts/data-migration.md)).
El **script** (T019) solo depende de la Fase 2 (T003, T005); el test de tolerancia por endpoint (T020)
depende de US1. US2 **no** requiere `A-73` **en su código** (la migración no cambia comportamiento
observable: solo reescribe la representación de la ubicación de un objeto — Principio VIII).
Operativamente `A-73` ya existe cuando se corre la migración, porque la Historia 1 —que sí la exige—
se despliega antes ([contracts/data-migration.md §Prerrequisitos](./contracts/data-migration.md)).

### Tests for User Story 2

- [X] T018 [P] [US2] Crear `pos-backend/app/characterization_tests/test_migrate_image_keys_relative.py`
      (fixtures multi-schema como `migrate_payment_methods_catalog` si existen; si no, sembrar filas a
      mano en `shared.tenants` + un schema de tenant): sembrar en los tres campos valores V0 (vacío),
      V1 (ya key), V2 (`pub-…r2.dev`), V3 (`assets.skeilopos.com`), V4 (otro origen) y una fila de
      `receipt_file_url`/`comprobantes`. Verificar:
      **G1** tras migrar, V2/V3 → key en los tres campos; `... LIKE 'https://pub-%'` y
      `... LIKE 'https://assets%'` → 0 filas (FR-009, SC-007);
      **G3** segunda corrida → reporte `migradas=0`, ninguna fila cambia (FR-009c, SC-009);
      **interrupción** simulada (cortar entre schemas) + relanzar → converge al mismo estado
      (escenario 6);
      **G4** `migrar` + `--revert` deja los tres campos **byte a byte** como el dump previo
      (FR-009a, SC-008);
      **G6** V4 (otro origen) intacto en ambos modos (FR-010);
      **G7** `receipt_file_url` / `comprobantes` **fuera del recorrido** — sin cambios (FR-014);
      claves no-imagen de `payment_info` intactas. Debe **fallar** antes de T019
- [X] T019 [US2] Crear `pos-backend/app/scripts/migrate_image_keys_relative.py`, espejo de
      `pos-backend/app/scripts/migrate_payment_methods_catalog.py`
      ([contracts/data-migration.md](./contracts/data-migration.md), [data-model.md §5](./data-model.md)):
      modos `--report-only` | (escribe) | `--revert`; recorrido `shared.tenants.logo_url` con
      `with_db(None)` + por cada `schema` de `SELECT schema FROM shared.tenants`: `products.image_url` y
      `payment_methods.payment_info` con `with_db(schema)`, `commit` por schema. **Predicado de
      migración**: procesar un valor solo si empieza por algún prefijo de `_managed_bucket_prefixes()`;
      reescribir a `normalize_asset_ref(v)` (la **misma** función que la escritura, FR-003); dejar
      intactas V0/V1/V4. **`payment_info`**: reescribir el dict con las claves de imagen convertidas,
      copiar el resto sin tocar. **`--revert`**: para cada valor que es key (sin `"://"`) y no vacío en
      los tres campos, reescribir a `f"{settings.R2_PUBLIC_BASE_URL.rstrip('/')}/{key}"`; V0/V4
      intactos. **Reporte** por tenant y campo: `migradas / ya_key / vacias / otro_origen`; código de
      salida `!= 0` si algún tenant falló (se loguea y se continúa). Documentar en el docstring la nota
      operativa de `--revert` (asume que no se subieron imágenes nuevas entre `migrar` y `--revert`).
      Hace pasar T018. Depende de T003, T005
- [X] T020 [P] [US2] Añadir a `pos-backend/app/characterization_tests/test_tenant_logo_key.py` (o al
      de productos) un caso de **tolerancia de lectura** (FR-009b): una fila cuyo campo contiene
      todavía `https://pub-…r2.dev/{key}` (no migrada) → `GET /tenant` / `GET /menu` la devuelve como
      `https://assets.skeilopos.com/{key}` y la imagen "se ve igual"; si esa fila se vuelve a guardar,
      queda como key (FR-004). Depende de T012
- [ ] T021 [US2] Ejecutar [quickstart.md §Historia 2](./quickstart.md) (pasos 2.1–2.9) contra el
      stack real con datos sembrados **antes** del cambio (producto/logo/método con `pub-…r2.dev` +
      un producto con URL de otro origen): tolerancia de lectura antes de migrar, `--report-only`,
      `migrar` (conteo `LIKE 'https://pub-%'` → 0), "otro origen" intacto, segunda corrida sin
      cambios, interrupción + relanzar, `--revert` exacto contra un dump previo.
      → **Pendiente** — requiere base de datos sembrada + varios schemas (no ejecutable en esta sesión; ver T028)

**Checkpoint**: las filas históricas de los tres campos quedan como key; ninguna imagen deja de verse antes, durante ni después; la migración es idempotente y reversible; los comprobantes y los otros orígenes quedan intactos.

---

## Phase 5: User Story 3 - Reemplazar una imagen sigue limpiando la anterior (Priority: P3)

**Goal**: al reemplazar la imagen de un producto o el logo del negocio, el objeto anterior en el
almacenamiento se sigue borrando **best-effort y después del `commit`** (A-44 / spec 021, intacto),
tanto si la referencia anterior estaba guardada como key como si era una URL del bucket gestionado
(FR-011); **no** se borra si el guardado no se confirmó (FR-012) ni si la referencia anterior es de
otro origen (FR-013); y un guardado que **no tocó** la imagen (el cliente reenvía la URL de
visualización) **no** se interpreta como cambio (comparación key vs key tras normalizar,
[data-model.md §4](./data-model.md)).

**Independent Test**: reemplazar la imagen de un producto cuya referencia previa es (a) una key ya
migrada, (b) una URL `pub-…r2.dev`, (c) una URL de otro origen, y verificar que en (a) y (b) el objeto
anterior deja de existir y en (c) no se intenta borrar nada; forzar un fallo al guardar tras elegir
imagen nueva y verificar que el objeto anterior no se borra; guardar sin cambiar la imagen y verificar
que `delete_object` no se invoca. Repetir con el logo ([quickstart.md §Historia 3](./quickstart.md)).
No depende de US2.

**⚠️ Depende de US1** (`AssetRefIn` en los esquemas de `products`/`tenant` es lo que hace la
comparación key vs key) y de `object_key_for_deletion` (T005).

### Tests for User Story 3

- [X] T022 [P] [US3] Extender `pos-backend/app/characterization_tests/test_products_image_key.py` y
      `pos-backend/app/characterization_tests/test_tenant_logo_key.py` con el borrado del objeto
      anterior (mock/espía de `delete_object`, [research.md §10](./research.md)):
      **(a)** reemplazo con referencia previa = URL `pub-…r2.dev` → `delete_object` llamado con la key
      correcta tras `commit` (FR-011);
      **(b)** reemplazo con referencia previa = **key** → `delete_object` llamado con esa key (hoy
      **falla**: `key_from_public_url(key)` → `None`);
      **(c)** reemplazo con referencia previa = URL de **otro origen** → `delete_object` **no** se
      llama (FR-013, hoy pasa por casualidad porque devuelve `None`, pero se fija explícitamente);
      **(d)** guardar reenviando `https://assets.skeilopos.com/{key}` **sin** cambiar la imagen →
      `delete_object` **no** se llama (comparación key vs key, FR-012, [data-model.md §4](./data-model.md));
      **(e)** fallo al guardar (excepción antes del `commit`) tras elegir imagen nueva → `delete_object`
      **no** se llama y el campo conserva la referencia anterior (FR-012, A-44).
      (b) y (d)/(c) deben quedar cubiertos; (b) debe **fallar** antes de T023
- [X] T023 [US3] En `pos-backend/app/api/v1/products/service.py::update_product` y
      `pos-backend/app/api/v1/tenant/router.py::update_tenant`, cambiar `key_from_public_url(old_*_url)`
      por `object_key_for_deletion(old_*_url)` (devuelve la key si `old` es key directa **o** URL del
      bucket gestionado; `None` si es de otro origen — FR-011/FR-013). Confirmar el orden de
      operaciones de [data-model.md §4](./data-model.md): normalización (ya hecha por `AssetRefIn`) →
      comparación key vs key → asignación → `commit` → `delete_object` best-effort (A-44 / spec 021,
      **intacto**). Eliminar `key_from_public_url` de `app/core/storage.py` (sin llamadores). Los
      métodos de pago **no** borran el QR anterior (comportamiento actual; FR-011a — FR-011 solo
      cubre producto y logo). Confirmar los **tres** tests A-44 de `test_products_service.py` en verde (2 sin tocar +
      el de T016). Hace pasar T022. **Commit cita `A-73`** (amplía el extractor de key, punto 4 de la
      entrada). Depende de T011, T012
- [ ] T024 [US3] Ejecutar [quickstart.md §Historia 3](./quickstart.md) (pasos 3.1–3.6) contra el
      stack real: reemplazo con previa = key / = URL vieja / = otro origen; fallo forzado al guardar;
      guardado sin tocar la imagen; repetir con el logo.
      → **Pendiente** — recorrido con navegador + datos sembrados (no ejecutable en esta sesión; ver T028)

**Checkpoint**: reemplazar una imagen deja exactamente un objeto vigente y ninguna referencia rota (SC-004); la garantía de A-44 sobrevive al cambio de cómo se referencia la imagen; los otros orígenes nunca se tocan. Las tres historias funcionan de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns (No regresión — FR-014 a FR-017, SC-006)

- [X] T025 [P] Correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`
      completo en `pos-backend` y confirmar **0 fallos nuevos** frente a la línea base de T001, con
      `test_cart_payment_attempts.py` **sin modificar** y en verde (comprobantes, FR-014),
      `test_products_service.py` con las 2 pruebas A-44 intactas y 1 aserción actualizada (T016), y
      `test_sales_payment_methods_catalog.py` (compara el `payment_info` del retorno del servicio =
      ORM = key) en verde
      → **Hecho**: línea base T001 = 755 tests OK; tras la implementación **826 tests OK, 0 fallos,
      0 errores** (+71 tests nuevos de spec 080). `test_cart_payment_attempts.py` sin tocar y en
      verde; `test_products_service.py` con los 3 tests A-44 en verde (2 intactos + 1 aserción
      actualizada en T016); `test_sales_payment_methods_catalog.py` en verde. `app/scripts/test_*.py`
      también en verde (env de `test_promotions_rules.py` actualizado con `ASSETS_BASE_URL`).
- [X] T026 [P] Verificación de no-intrusión: confirmar que el diff **no** toca
      `pos-backend/app/api/v1/cart/service.py` (`presign_payment_receipt` / `presign_receipt` /
      `attach_receipt`), **no** repunta `R2_PUBLIC_BASE_URL` (sigue en `pub-…r2.dev`), **no** cambia la
      firma ni el comportamiento de `public_url_for` / `build_object_key` / `delete_object`
      (FR-016/FR-017), **no** añade ninguna migración de Alembic (`grep` en
      `pos-backend/alembic/versions/`), y **no** toca ningún archivo de `pos-heladeria`
      → **Hecho**: `cart/service.py` sin tocar; `R2_PUBLIC_BASE_URL=` sin cambiar en `.env.example`
      (solo se añadió el comentario y `ASSETS_BASE_URL`); `public_url_for` conserva firma y cuerpo
      (solo docstring); `key_from_public_url` eliminado (renombrado a `object_key_for_deletion`, sin
      llamadores — T023); `build_object_key` / `delete_object` / `generate_presigned_put_url`
      intactos; `git status alembic/` vacío; `git diff develop` en `pos-heladeria` **vacío** (0
      archivos).
- [X] T027 [P] Correr la suite de `pos-heladeria` (`npx ng test --watch=false`) y confirmar que sigue
      en verde **sin cambios** frente a su estado actual (contrato del consumidor preservado, FR-007 —
      el frontend recibe una URL absoluta lista para usar por imagen, ahora contra `assets.skeilopos.com`)
      → **Hecho**: `pos-heladeria` es **byte a byte idéntico a `develop`** (`git diff develop` vacío,
      0 commits en la rama). La suite reporta `18 failed | 788 passed (806)`; esos 18 fallos son una
      **condición preexistente de la rama base** (p. ej. `transfer-details-step.component.spec.ts`
      espera `fetch(url)` y el componente ya llama `fetch(url, {mode:'cors'})` — anterior a esta
      spec), **no** una regresión de spec 080, que no toca ningún archivo del frontend.
- [ ] T028 Ejecutar el resto de [quickstart.md](./quickstart.md) contra `pos-backend` +
      `pos-heladeria` reales: sección **§Comprobantes del comensal — fuera de alcance** (C.1–C.3:
      `receipt_file_url` sigue siendo URL completa contra `pub-…r2.dev`, se ve igual desde "Pagos por
      confirmar", la migración no lo toca) y **§Cierre — criterios de éxito** (SC-001 a SC-009,
      incluido **SC-005**: cambiar `ASSETS_BASE_URL`, reiniciar y recargar el menú reapunta todas las
      imágenes sin migración ni edición de datos). **Pendiente** — requiere navegador + datos
      sembrados; no ejecutable sin entorno visual. Responsable: propietario del proyecto, antes del
      despliegue

---

## Dependencies & Execution Order

### Phase Dependencies

- **Fase 1 (Setup)**: sin dependencias. **T002 (`A-73`) bloquea US1**, no US2 ni US3.
- **Fase 2 (Foundational)**: T003 → T004 → T005. **Bloquea todas las historias.**
- **Fase 3 (US1)**: depende de la línea base (T001), de `A-73` (T002) y de la Fase 2. Sin dependencia
  de US2 ni US3.
- **Fase 4 (US2)**: el **script** (T018/T019) depende solo de la Fase 2; el test de tolerancia por
  endpoint (T020) y el recorrido (T021) dependen de US1 desplegado. **No** requiere `A-73` en su
  código (US1, que va antes, sí la exige).
- **Fase 5 (US3)**: **depende de US1** (`AssetRefIn` en `products`/`tenant` schemas) y de la Fase 2
  (`object_key_for_deletion`). **Independiente de US2.**
- **Fase 6 (Polish)**: depende de todas las historias en alcance que se vayan a entregar.

### User Story Dependencies

- **US1 (P1)** — MVP. Fase 2 + `A-73` (T002). Independiente de US2/US3.
- **US2 (P2)** — el script es independiente en código (solo Fase 2); operativamente US1 se despliega
  antes de correr la migración. No requiere `A-73` en su código (US1, que va antes, sí).
- **US3 (P3)** — **depende de US1** (extiende sus esquemas/servicios). Independiente de US2 y de la
  migración.

### Within Each User Story

- Los tests preceden a la implementación y deben **fallar antes**: T004 → T005; T006 → T015;
  T007 → T011/T012; T008 → T012; T009 → T013/T014; T018 → T019; T022 (caso b) → T023.
- Foundational: config (T003) antes de los helpers (T005); tabla de verdad (T004) entre medias.
- US1: `schema_types.py` (T010) antes de aplicar los tipos en los esquemas (T011/T012);
  T011 antes de la aserción A-44 (T016).
- US3: los esquemas de US1 (T011/T012) antes del swap del extractor de borrado (T023).
- La tarea de recorrido de `quickstart.md` cierra cada fase de historia (T017, T021, T024).

---

## Parallel Opportunities

- **Fase 1**: T001 (`pos-backend`) y T002 (`pos-specs`) en paralelo — repos distintos.
- **Fase 2**: T004 (test) se escribe en paralelo con T003 (config); T005 los hace pasar.
- **US1 tests**: T006, T007, T008, T009 en paralelo — archivos de test distintos.
- **US1 impl**: T011 (`products/`), T012 (`tenant/` + `menu/`) y T015 (`uploads/`) en paralelo —
  archivos distintos; los tres dependen de T010 (T015 depende de T005). T013/T014 (`sales/` + `cart/`)
  en paralelo con ellos.
- **US2 vs US1**: el script de migración (T018/T019) puede escribirse en paralelo con US1 — solo
  necesita la Fase 2.
- **Fase 6**: T025, T026, T027 en paralelo.

### Ejemplo — reparto por desarrollador

```
Dev A: US1 (escritura + lectura + presign)  →  US3 (borrado del objeto anterior)
Dev B: US2 (script de migración)            en paralelo, tras la Fase 2
```

### Ejemplo — tests en paralelo, US1

```bash
Task T006: "pos-backend/…/test_uploads_presign_key.py — presign devuelve key + public_url dominio nuevo"
Task T007: "pos-backend/…/test_products_image_key.py — image_url persiste como key; respuesta trae URL absoluta"
Task T008: "pos-backend/…/test_tenant_logo_key.py — logo_url persiste como key; GET /tenant y /menu ensamblan"
Task T009: "pos-backend/…/test_payment_methods_qr_key.py — payment_info['qr'] persiste key; consumidores ensamblan"
# Luego T010–T015 hacen pasar sus tests.
```

---

## Implementation Strategy

### MVP (solo US1)

1. Fase 1 (Setup) — línea base + `A-73` (T002, barata, bloquea US1).
2. Fase 2 (Foundational) — config `ASSETS_BASE_URL` + los 3 helpers (T003–T005).
3. Fase 3 (US1) — `AssetRefIn` / `AssetUrl` en los esquemas, ensamblado de `payment_info`, `public_url`
   del presign contra el dominio nuevo, 1 aserción A-44 actualizada.
4. **DETENERSE Y VALIDAR**: [quickstart.md §Historia 1](./quickstart.md) — la key es lo que se
   persiste, la imagen se ve desde `assets.skeilopos.com`, el contrato del consumidor no cambia
   (SC-001, SC-006).
5. Desplegar — cada imagen nueva ya nace con el esquema correcto.

### Entrega incremental

1. Setup + Foundational → base lista.
2. + US1 → subida persiste key, lectura ensambla, presign contra el dominio nuevo (MVP, requiere
   `A-73`). Validar SC-001/SC-006. Desplegar.
3. + US2 → migración de las filas históricas (idempotente, reversible) + tolerancia de lectura.
   Validar SC-002/SC-003/SC-007/SC-008/SC-009. **Correr `--report-only` → `migrar`** tras desplegar.
4. + US3 → el borrado del objeto anterior acepta key directa e ignora otros orígenes. Validar
   SC-004. Desplegar.
5. Fase 6 (Polish) sobre lo entregado — no regresión (FR-014–FR-017, SC-006); comprobantes intactos.

US2 (script) puede adelantarse en paralelo a US1; US3 siempre después de US1.

---

## Notes

- `[P]` = archivos distintos, sin dependencia de una tarea sin terminar.
- La etiqueta `[Story]` mapea cada tarea a su historia para trazabilidad (Principio XII).
- Un commit por tarea (o grupo lógico), citando el `FR`/contrato; los commits de **US1** y el swap del
  extractor de borrado en **US3 (T023)** citan además `A-73`.
- Verificar que los tests fallan antes de implementar donde el test precede a la implementación
  (Principio III/X).
- Detenerse en cada checkpoint para validar la historia de forma aislada.
- Evitar: repuntar `R2_PUBLIC_BASE_URL` (rompería el dominio de los comprobantes, FR-014); tocar
  `cart/service.py` o `test_cart_payment_attempts.py` (FR-014); cambiar `public_url_for` /
  `build_object_key` / `delete_object` (FR-016/FR-017); añadir una migración de Alembic (no hay cambio
  de esquema); guardar la URL absoluta en base de datos (FR-002); comparar URL entrante contra key
  almacenada en `update_product` / `update_tenant` (borraría el objeto en uso, [data-model.md §4](./data-model.md));
  intentar convertir o borrar valores de otro origen (FR-010/FR-013); tocar cualquier archivo de
  `pos-heladeria` ([research.md §9](./research.md)).
