---

description: "Tareas de la spec 088 — integridad de las referencias a archivos en Cloudflare R2 (US1–US6)"
---

# Tasks: Integridad de las referencias a archivos en Cloudflare R2

**Input**: Documentos de diseño de `/specs/088-integridad-referencias-r2/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D15), [data-model.md](./data-model.md), [contracts/](./contracts/) (`asset-guard.md`, `image-write-endpoints.md`, `receipt-endpoints.md`, `scripts.md`), [quickstart.md](./quickstart.md)

**Tests**: El proyecto usa *characterization tests* (`app/characterization_tests/`, `python -m unittest`, sesiones SQLite en memoria) como árbitro de comportamiento (Principio III); `pos-heladeria` usa `ng test` (Karma/Jasmine). El plan ya enumera los archivos de prueba, así que las tareas de prueba van **antes** de la implementación de cada historia y deben fallar contra el código actual. Los tests existentes que codifican la *forma de la petición* (no el comportamiento congelado) se actualizan citando spec 088 + A-92/A-93 en el mismo commit (plan.md, fila III).

**Organización**: Las tareas se agrupan por historia de usuario (US1 → US6, mismo orden de prioridad que `spec.md`: P1×3, P2×2, P3×1) sobre una rama `feat/088-r2-reference-integrity` en `../pos-backend` y `../pos-heladeria` (Principio XIV). Todas las rutas fueron verificadas contra el código real de ambos repos el 2026-09-29.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: A qué historia de usuario pertenece (US1–US6)
- Cada tarea incluye la ruta exacta del archivo, relativa a `../pos-backend`, `../pos-heladeria` o `pos-specs` (esta carpeta) según corresponda

## Convenciones de ruta

- `../pos-backend/app/...` — API FastAPI + SQLAlchemy (Python 3.12). Tests en `../pos-backend/app/characterization_tests/`; se ejecutan desde `../pos-backend` con `python -m unittest app.characterization_tests.<modulo> -v`
- `../pos-heladeria/src/app/...` — SPA Angular (TypeScript); pruebas con `ng test --watch=false --include='<glob>'`
- Sin prefijo — archivo dentro de este repo (`pos-specs`)

## Nota sobre gobernanza (Principio II/XI) y numeración de anomalías

Esta spec cambia comportamiento existente y exige **dos entradas nuevas** en `specs/000-reconocimiento/registro-de-anomalias.md` **antes** de implementar (plan.md, Constitution Check, fila II). La última anomalía registrada al 2026-09-29 es **A-91**, por lo que corresponden **A-92** y **A-93**; T001 verifica el número real con `grep` por si otra spec tomó alguno entre tanto. Formato de encabezado del registro: `### A-NN — [DECISIÓN DE NEGOCIO — spec 088] ...`.

**Despliegue (research D10)**: `pos-heladeria` (US1, tareas de frontend) **debe desplegarse antes** que `pos-backend`; el envío de `presign.key` del comprobante (T067) va **después** del backend. Los commits solo se hacen cuando el usuario lo pida (Principio XV).

---

## Phase 1: Setup

**Propósito**: gobernanza y ramas antes de tocar código.

- [X] T001 Registrar **A-92** en `specs/000-reconocimiento/registro-de-anomalias.md` (`### A-92 — [DECISIÓN DE NEGOCIO — spec 088] ...`): imágenes de producto, logo y QR — validación de dueño/carpeta y de existencia, imagen base (concurrencia optimista) que ignora en silencio un formulario desactualizado, borrado del archivo anterior solo si nadie más lo usa. Antes de escribir, confirmar el último número con `grep -oE "^### A-[0-9]+" specs/000-reconocimiento/registro-de-anomalias.md | sort -t- -k2 -n | tail -1`. Incluir las dos decisiones que el plan fijó por defecto y quedan **por confirmar con el negocio** (research D7: el QR sigue pudiéndose quitar por el mecanismo actual de `payment_info` completo; research D8: no se añade borrado del QR anterior), y la consecuencia asumida de D4 (una imagen nueva subida desde un formulario desactualizado se ignora en silencio y queda huérfana). **D7 quedó confirmada como opción A el 2026-09-29** (el QR se puede quitar por edición legítima; excepción documentada en spec.md, Edge Cases y Assumptions): registrarla como decisión confirmada, con quién y cuándo. D8 sigue por confirmar. La entrada incluye quién decidió, fecha, qué comportamiento cambia, por qué y qué funcionalidades se afectan (Principio II); mientras D8 esté abierta, A-92 se registra con esa parte "por confirmar"
- [X] T002 Registrar **A-93** en `specs/000-reconocimiento/registro-de-anomalias.md`: comprobantes del comensal — dejan de ser texto libre; solo se aceptan archivos existentes, de la carpeta `comprobantes` y del propio negocio (sin garantía de autoría, el comensal no tiene sesión), y en base de datos se guarda la key en vez de la URL absoluta; migración idempotente de las filas existentes; el `receipt_hash` de auditoría (spec 074) de los eventos nuevos se calcula sobre la key (research D11). La entrada incluye quién decidió, fecha, qué comportamiento cambia, por qué y qué funcionalidades se afectan (Principio II)
- [X] T003 Crear la rama `feat/088-r2-reference-integrity` desde `develop` en `../pos-backend` y en `../pos-heladeria` (ambos repos están en `develop`; Principio XIV). Verificar con `git branch --show-current` que el árbol de trabajo esté limpio antes de empezar

**Checkpoint**: A-92/A-93 registradas y ambos repos sobre la rama de esta funcionalidad.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: las primitivas que aplican FR-002 … FR-004 y que consumen las 6 historias. Contrato completo en [contracts/asset-guard.md](./contracts/asset-guard.md). `normalize_asset_ref`, `asset_display_url`, `object_key_for_deletion` y `delete_object` (spec 080/021) **no cambian**.

**⚠️ CRITICAL**: ninguna historia puede empezar hasta terminar esta fase.

- [X] T004 Extender `../pos-backend/app/core/storage.py` (sin tocar las funciones existentes): (a) constantes `FOLDER_PRODUCTS="products"`, `FOLDER_LOGO="logo"`, `FOLDER_PAYMENT_METHODS="payment-methods"`, `FOLDER_RECEIPTS="comprobantes"`; (b) excepciones `AssetKeyError` y `StorageUnavailable`; (c) `validate_asset_key(value, tenant_schema, folder) -> str`: pura, `re.fullmatch(rf"{re.escape(tenant_schema)}/{folder}/[A-Za-z0-9][A-Za-z0-9._-]{{0,199}}", value)` y `".." not in value`, sin strip/lower/unquote/normpath (research D2); (d) `object_exists(key) -> bool` con `head_object`: `ClientError` `404`/`NoSuchKey`/`NotFound` → `False`, cualquier otro `ClientError`, `EndpointConnectionError`, `ConnectTimeoutError`, `ReadTimeoutError` o `BotoCoreError` → lanza `StorageUnavailable` (research D3); (e) `deletable_key(value, tenant_schema, folder) -> str | None` que compone `object_key_for_deletion` con la convención (URL de otro origen, key fuera de convención, de otro negocio o de otra carpeta → `None`); (f) `get_r2_client()` con `BotoConfig(signature_version="s3v4", region_name="auto", connect_timeout=3, read_timeout=5, retries={"max_attempts": 2, "mode": "standard"})`
- [X] T005 [P] Crear `../pos-backend/app/characterization_tests/test_storage_asset_guard.py`: (a) tabla de verdad completa de `validate_asset_key` de contracts/asset-guard.md (válidas `acme/products/<hex>.png` y `acme/products/foto.v2.jpg`; inválidas: otro negocio, otra carpeta, `ACME/...`, `..`, `//`, `\`, `\n` final, `\x00`, `%2e%2e`, espacio, sin nombre, `.hidden`, subcarpeta, cadena vacía); (b) `object_exists` con `get_r2_client` parcheado: 200 → `True`, `ClientError` 404/`NoSuchKey`/`NotFound` → `False`, 403, 5xx, `EndpointConnectionError`, `ConnectTimeoutError`, `ReadTimeoutError` → `StorageUnavailable`; (c) `deletable_key` para `None`, vacío, URL de otro origen, key en convención (con y sin URL absoluta gestionada), key histórica fuera de convención y key de otro negocio/carpeta; (d) la configuración del cliente (timeouts 3/5 y 2 intentos)
- [X] T006 [P] Ajustar `../pos-backend/app/characterization_tests/fixtures.py` para que las sesiones de test alcancen `shared.tenants` (research D9): añadir el parámetro `schema` a `make_tenant_stub(...)` y crear la tabla del logo con la opción (a) `ATTACH DATABASE ':memory:' AS shared` + `Tenant.__table__.create` en la conexión del fixture; si no es viable con `schema_translate_map`, usar la opción (b) — parámetro inyectable `shared_check` en `is_key_referenced` (T040) que los tests sustituyen — y dejar constancia de la opción elegida en un comentario del fixture. Los tests existentes deben seguir en verde (correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` antes y después)
- [X] T007 Crear `../pos-backend/app/core/asset_refs.py` con `ImageDecision` (`KEEP` / `IGNORE` / `APPLY`), `decide_image_change(*, tenant_schema, folder, sent, base_provided, base, current, is_creation)` (pura, valores ya normalizados con `normalize_asset_ref`, orden fijo de research D4: vacío → `KEEP`; igual a la vigente → `KEEP`; gestionada con forma inválida → `AssetKeyError` **siempre**; creación → `APPLY`; sin base o base ≠ vigente → `IGNORE`; otro origen → `APPLY` sin verificar; gestionada → `APPLY`) y `ensure_image_exists(key)` (existe → retorna; no existe → `HTTPException(422, "La imagen no se encontró en el almacenamiento. Sube el archivo de nuevo.")`; `StorageUnavailable` → `HTTPException(503, "El almacenamiento de archivos no responde. Intenta de nuevo en unos segundos.")`). "Gestionado" = no contiene `://` tras normalizar. Las demás funciones del módulo (`is_key_referenced`, `asset_key_lock`, `resolve_receipt_key`) se añaden en US4 y US5. Depende de T004
- [X] T008 [P] Crear `../pos-backend/app/characterization_tests/test_asset_refs.py` con la parte de esta fase: (a) `decide_image_change` — las 7 filas de research D4 × creación/edición × base presente/ausente/`null` explícito, incluyendo los resúmenes (a) y (b) de FR-002 (key válida distinta + base que no coincide → `IGNORE` sin llamar a R2; base coincide → `APPLY`), key ajena con formulario desactualizado → `AssetKeyError`, fila histórica con key fuera de convención igual a la vigente → `KEEP` sin validar, una fila con URL absoluta equivale a su key; (b) `ensure_image_exists` con `object_exists` parcheado: existe / 422 / 503

**Checkpoint**: primitivas listas y probadas; las 6 historias pueden empezar.

---

## Phase 3: User Story 1 — Un formulario desactualizado no deshace ni borra la imagen vigente (Priority: P1) 🎯 MVP

**Goal**: guardar producto, logo o QR con un formulario desactualizado conserva la imagen vigente y guarda los demás campos, sin error ni aviso y sin borrar ningún archivo (FR-001, FR-002).

**Independent Test**: quickstart Historia 1 — dos pestañas del mismo producto; la primera sube una imagen nueva; la segunda (formulario viejo) cambia solo el nombre: el nombre se guarda y la imagen nueva sigue visible. Repetir con logo y QR.

### Tests (Principio III — actualizar primero lo que codifica la forma de la petición)

- [X] T009 [P] [US1] Actualizar los tres tests A-44 (`"CONGELA comportamiento corregido:"`) de `../pos-backend/app/characterization_tests/test_products_service.py`: llaman `update_product(..., ProductUpdate(image_url=NEW_URL))` sin imagen base; añadirles `image_url_base=<imagen vigente>` y `mock.patch` de `object_exists` (→ `True`). (La comprobación `is_key_referenced` aún no existe en esta fase; T038 añade el parche de `is_key_referenced` a estos mismos tests cuando US4 la introduzca — así T009 pasa en verde tanto antes como después de US4.) Conservar exactamente lo que congelan: el borrado ocurre **después** del commit y un fallo de borrado o de commit no deja referencia rota. Citar spec 088 + A-92 en el comentario del test y en el commit
- [X] T010 [P] [US1] Añadir la imagen base a las ediciones legítimas de `../pos-backend/app/characterization_tests/test_products_image_key.py`, `test_tenant_logo_key.py` (`logo_url_base`) y `test_payment_methods_qr_key.py` (`payment_info_base`), con `object_exists` parcheado, sin cambiar lo que verifican (spec 080)
- [X] T011 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_products_image_integrity.py` con los escenarios de la US1 sobre `update_product`: (1) formulario desactualizado que reenvía X con base X y vigente Y → el precio/nombre se guarda, imagen Y intacta, `delete_object` **no** llamado; (2) desactualizado cuyo archivo aún existe → igual conserva la vigente; (3) formulario al día que sube una imagen nueva → cambia (`APPLY`); (4) creación con imagen y sin base → se crea con esa imagen; (5) formulario al día sin tocar la imagen (sin `image_url`) → nada cambia, `delete_object` no llamado; (7) la respuesta es exitosa y sin aviso; sin base y key válida distinta → `IGNORE`; base `null` explícito sobre producto sin imagen + key válida → `APPLY` ("no enviada" ≠ `null`, research D5); key ajena con formulario desactualizado → 422 y registro sin cambios
- [X] T012 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_tenant_logo_integrity.py`: mismas reglas para `update_tenant` (escenario 6): con `logo_url` ignorado `receipt_message` e `invoice_prefix` se guardan igualmente
- [X] T013 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_payment_methods_qr_integrity.py`: algoritmo por clave de research D7 sobre `update_payment_method`: `payment_info_base` ausente o `base[k]` ≠ vigente → se conserva el valor vigente de `k` (o se elimina la clave si no había) y las demás claves (`celular`, `cuenta`…) se guardan; edición legítima con `k` vacío/ausente → se elimina (comportamiento actual, el archivo no se borra); `is_complete` se recalcula sobre el `payment_info` **resultante**

### Implementation — backend

- [X] T014 [P] [US1] `../pos-backend/app/api/v1/products/schemas.py`: añadir `image_url_base: AssetRefIn = None` a `ProductUpdate` (solo actualización, no `ProductCreate`; research D5, contracts/image-write-endpoints.md)
- [X] T015 [P] [US1] `../pos-backend/app/api/v1/tenant/schemas.py`: añadir `logo_url_base: AssetRefIn = None` a `TenantUpdate`
- [X] T016 [P] [US1] `../pos-backend/app/api/v1/sales/schemas.py`: añadir `payment_info_base: dict[str, str] | None = None` a `PaymentMethodUpdate`, normalizando por valor con `normalize_asset_ref` (no en `PaymentMethodCreate`)
- [X] T017 [P] [US1] `../pos-backend/app/api/v1/products/service.py::update_product` (l.101–152): resolver la imagen **antes** de asignar cualquier otro campo (research D6): normalizar `sent`/`base`/`current`, `base_provided = "image_url_base" in data.model_fields_set`, llamar `decide_image_change(..., folder=FOLDER_PRODUCTS, is_creation=False)`; `AssetKeyError` → `HTTPException(422, "La imagen no es válida para este negocio.")` (registrar el detalle de la key en el log, no en el mensaje); `KEEP`/`IGNORE` → no asignar `image_url` ni calcular `old_key`; `APPLY` → conservar por ahora la lógica actual de asignación y borrado post-commit (la endurecen US3/US4). Depende de T007, T014
- [X] T018 [P] [US1] `../pos-backend/app/api/v1/tenant/router.py::update_tenant` (l.59–80): misma resolución con `logo_url`/`logo_url_base`, carpeta `FOLDER_LOGO` y `row.schema`; `receipt_message` e `invoice_prefix` se guardan siempre. Depende de T007, T015
- [X] T019 [P] [US1] `../pos-backend/app/api/v1/sales/service.py::update_payment_method` (l.119): por cada clave con `catalog.fields[*].format == "image"` aplicar el algoritmo de research D7 (`decide_image_change` con carpeta `FOLDER_PAYMENT_METHODS`; `IGNORE` restituye el valor vigente o elimina la clave si no había; edición legítima sin valor elimina la clave; el archivo anterior no se borra, research D8); recalcular `is_complete` sobre el `payment_info` resultante. Depende de T007, T016

### Implementation — frontend (desplegar antes que el backend, research D10)

- [X] T020 [P] [US1] `../pos-heladeria/src/app/modules/products/services/product.service.ts` y `product.service.spec.ts`: añadir `image_url_base` a `ProductDraft`/`ProductForm` y a los payloads de actualización; `toDraft` (l.~416) guarda la imagen cargada como base; `updateProduct` (l.~247) y `toProductPayload` (l.~556) envían **siempre** `image_url_base` (`null` explícito si el producto no tenía imagen) y `image_url` **solo si cambió** respecto de la base (no reenviar la imagen sin tocar); `createProduct` (l.~225) sigue igual y sin base. Actualizar el comentario de `uploadProductImage` (l.~567). Pruebas: el payload trae `*_base` y **no** trae la imagen si no cambió; con imagen nueva trae ambas
- [X] T021 [P] [US1] `../pos-heladeria/src/app/modules/products/pages/product-form.component.ts` (+ `product-form.component.spec.ts`): asegurar que el borrador conserva la imagen base al cargar el producto y que subir una imagen nueva (l.~1283) solo modifica `image_url`, de modo que el servicio pueda decidir si cambió; cubrir en el spec "sin tocar la imagen no se envía `image_url`"
- [X] T022 [P] [US1] `../pos-heladeria/src/app/core/tenant/tenant-info.service.ts`: `uploadLogo` (l.~137) envía `{ logo_url: presign.public_url, logo_url_base: this.info()?.logo_url ?? null }`; verificar que `update()` y cualquier otro `PATCH /tenant` nunca reenvíen `logo_url`. Crear `tenant-info.service.spec.ts` (no existe) cubriendo el payload con base
- [X] T023 [P] [US1] `../pos-heladeria/src/app/modules/sales/services/payment-method.service.ts` y `payment-method.service.spec.ts`: `PaymentMethodUpdatePayload` acepta `payment_info_base?: Record<string, string> | null`; `update()` lo transmite tal cual. Prueba: el payload incluye `payment_info_base` cuando se pasa
- [X] T024 [US1] `../pos-heladeria/src/app/modules/sales/pages/payment-methods-page.component.ts`: `openFieldsForm` (l.~312) guarda una copia de `method.payment_info` como base y `submitFields` (l.~360) la envía como `payment_info_base` junto con `payment_info`. El QR sin tocar **se sigue enviando** dentro de `payment_info` (omitirlo lo eliminaría); solo se añade `payment_info_base`. Depende de T023

**Checkpoint**: desde `../pos-backend`, correr los tests de T008–T013 y `test_products_service` y `test_*_key`; desde `../pos-heladeria`, `ng test --watch=false --include='**/{product,tenant-info,payment-method}.service.spec.ts'` y `**/product-form.component.spec.ts`. US1 verificable de forma independiente (quickstart Historia 1).

---

## Phase 4: User Story 2 — No se puede guardar una imagen que no existe (Priority: P1)

**Goal**: antes de guardar una key de imagen se comprueba que el archivo existe en R2; si no existe → 422, si el almacenamiento no responde → 503 (falla cerrado), y el registro queda intacto (FR-003).

**Independent Test**: quickstart Historia 2 — edición legítima (base = vigente) con una key bien formada cuyo archivo no se subió → 422 con mensaje claro y el producto conserva su imagen; repetir con logo y QR.

### Tests

- [X] T025 [P] [US2] Extender `../pos-backend/app/characterization_tests/test_products_image_integrity.py`: edición legítima + `object_exists → False` → 422 y ningún campo cambia (nombre/precio incluidos); `object_exists` lanza `StorageUnavailable` → 503, sin cambios y sin `delete_object`; valor vacío (`None` o `""`) → `KEEP` sin llamar a R2; URL de otro origen → sin llamar a R2 (FR-005); imagen recién subida (existe) → se guarda; **desactualizado + archivo inexistente → se ignora sin error y sin llamar a R2** (resumen (a) de FR-002); `create_product` con imagen inexistente → 422 y no se crea el producto; con R2 caído → 503
- [X] T026 [P] [US2] Extender `../pos-backend/app/characterization_tests/test_tenant_logo_integrity.py` con los mismos casos para `update_tenant`
- [X] T027 [P] [US2] Extender `../pos-backend/app/characterization_tests/test_payment_methods_qr_integrity.py`: edición legítima con QR nuevo inexistente → 422; R2 caído → 503; `create_payment_method` con QR inexistente → 422 y sin fila nueva; URL de otro origen → no se consulta R2

### Implementation

- [X] T028 [US2] `../pos-backend/app/api/v1/products/service.py`: en `update_product`, cuando la decisión sea `APPLY` y la key sea gestionada, llamar `ensure_image_exists(key)` **antes** de asignar ningún campo; en `create_product` (l.61–74) llamar `ensure_image_exists` para toda `image_url` gestionada (sin base, FR-001). URL de otro origen y vacío no se verifican. Depende de T017
- [X] T029 [P] [US2] `../pos-backend/app/api/v1/tenant/router.py::update_tenant`: `ensure_image_exists(logo_key)` cuando la decisión sea `APPLY` y la key sea gestionada, antes de tocar `row`. Depende de T018
- [X] T030 [P] [US2] `../pos-backend/app/api/v1/sales/service.py`: `ensure_image_exists` para cada clave de imagen con edición legítima y valor nuevo en `update_payment_method` (paso 6 de research D7) y para cada clave de imagen gestionada en `create_payment_method` (l.72). Depende de T019

**Checkpoint**: correr T025–T027. US1 y US2 funcionan juntas: ninguna respuesta 422/503 deja el registro modificado (SC-002).

---

## Phase 5: User Story 3 — Un negocio no puede referenciar ni borrar los archivos de otro (Priority: P1)

**Goal**: solo se aceptan keys del propio negocio y de la carpeta que corresponde al campo (`{esquema}/{carpeta}/{nombre}`), y el sistema jamás borra un archivo que no le pertenece (FR-004, FR-005).

**Independent Test**: quickstart Historia 3 — con el negocio `acme`, guardar como imagen de producto `globex/products/zzz.jpg` → 422 (con o sin base, con formulario desactualizado); el archivo de `globex` sigue abriéndose.

### Tests

- [X] T031 [P] [US3] Extender `../pos-backend/app/characterization_tests/test_products_image_integrity.py`: 422 sin `delete_object` para `globex/products/zzz.jpg`, `acme/logo/x.png`, `acme/products/../logo/x.png`, `acme//products/x.jpg`, `acme\products\x.jpg`, `ACME/products/x.jpg`, `acme/products/x%2e%2e.jpg`, cada una con base = vigente, sin base y con base obsoleta; `create_product` con key ajena → 422; fila histórica con key fuera de convención: se reemplaza normalmente pero **no** se llama a `delete_object` (queda huérfana); fila con URL de otro origen: se reemplaza y **nunca** se intenta borrar el recurso antiguo; una key histórica igual a la vigente que el formulario reenvía no da 422
- [X] T032 [P] [US3] Extender `../pos-backend/app/characterization_tests/test_tenant_logo_integrity.py` con las mismas reglas (carpeta `logo`)
- [X] T033 [P] [US3] Extender `../pos-backend/app/characterization_tests/test_payment_methods_qr_integrity.py`: key de otro negocio / otra carpeta / segmentos → 422 en `create_payment_method` y `update_payment_method` (carpeta `payment-methods`)

### Implementation

- [X] T034 [US3] `../pos-backend/app/api/v1/products/service.py`: (a) `create_product` valida la key gestionada con `validate_asset_key(..., FOLDER_PRODUCTS)` **antes** de `ensure_image_exists` (`AssetKeyError` → 422 "La imagen no es válida para este negocio."); (b) en `update_product` reemplazar `object_key_for_deletion(product.image_url)` por `deletable_key(product.image_url, tenant.schema, FOLDER_PRODUCTS)`, de modo que solo se borre una key en convención del propio negocio. Depende de T028
- [X] T035 [P] [US3] `../pos-backend/app/api/v1/tenant/router.py::update_tenant`: reemplazar `object_key_for_deletion(row.logo_url)` por `deletable_key(row.logo_url, row.schema, FOLDER_LOGO)`. Depende de T029
- [X] T036 [P] [US3] `../pos-backend/app/api/v1/sales/service.py::create_payment_method`: validar cada clave de imagen gestionada con `validate_asset_key(..., FOLDER_PAYMENT_METHODS)` antes de verificar existencia (en `update_payment_method` la validación ya ocurre dentro de `decide_image_change`, T019). Depende de T030

**Checkpoint**: correr T031–T033. US1–US3 (todas las P1) completas: SC-001, SC-002 y SC-003.

---

## Phase 6: User Story 4 — El archivo anterior solo se borra si nadie más lo usa (Priority: P2)

**Goal**: al reemplazar una imagen, el archivo anterior se borra **después** del commit y solo si ninguna de las cuatro fuentes de referencias lo usa; con candado consultivo por key para cerrar la carrera guardar↔borrar (FR-006, FR-007).

**Independent Test**: quickstart Historia 4 — dos productos con la misma key K; reemplazar el primero deja K en pie; reemplazar el segundo la borra; con un único uso el reemplazo borra como siempre.

### Tests

- [X] T037 [P] [US4] Extender `../pos-backend/app/characterization_tests/test_asset_refs.py` con `is_key_referenced` (sobre el fixture de T006): una prueba por cada fuente real (`products.image_url`, valor de `payment_methods.payment_info`, `order_payment_attempts.receipt_file_url`, `shared.tenants.logo_url` de **cualquier** negocio), key referenciada por URL absoluta gestionada de fila histórica (candidatos `{key} ∪ {prefijo + key}` de `storage._managed_bucket_prefixes()`), key sin referencias → `False`; y `asset_key_lock` es no-op en SQLite (no lanza)
- [X] T038 [P] [US4] Extender `../pos-backend/app/characterization_tests/test_products_image_integrity.py`: dos productos con la misma key → reemplazar uno **no** borra, reemplazar el segundo borra con esa key; un solo uso → borra (sin regresión, SC-004); referencia por logo, por QR o por comprobante también impide el borrado; fallo de `delete_object` → cambio persistido y sin error (A-44); dos actualizaciones concurrentes del mismo producto → cada una borra solo la key que ella misma reemplazó, verificando `is_key_referenced` después de confirmar; el borrado ocurre después del commit. Actualizar también los tres tests A-44 de `test_products_service.py` con parche de `is_key_referenced` (→ `False`), citando spec 088 + A-92, en el mismo commit que T041
- [X] T039 [P] [US4] Extender `../pos-backend/app/characterization_tests/test_tenant_logo_integrity.py`: un logo cuya key también usa un producto no se borra al reemplazarlo; `update_tenant` consulta las referencias con una sesión del negocio (`with_db(tenant.schema)`), no con la sesión `shared`

### Implementation

- [X] T040 [US4] `../pos-backend/app/core/asset_refs.py`: añadir `is_key_referenced(db, key) -> bool` (cuatro fuentes de research D9: `Product.image_url IN candidatos LIMIT 1`, recorrido en Python de los valores de `PaymentMethod.payment_info`, `OrderPaymentAttempt.receipt_file_url IN candidatos LIMIT 1` y `Tenant.logo_url IN candidatos LIMIT 1` sin filtrar por negocio; solo lectura) y `asset_key_lock(db, key)` (`SELECT pg_advisory_xact_lock(hashtextextended(:key, 0))` solo si `db.bind.dialect.name == "postgresql"`; no-op en otro dialecto; research D13). Si T006 tuvo que usar la opción (b), exponer el parámetro `shared_check` inyectable
- [X] T041 [US4] `../pos-backend/app/api/v1/products/service.py::update_product`: (a) al aplicar una key gestionada nueva, tomar `asset_key_lock(db, nueva)` **antes** de `ensure_image_exists` y conservarlo hasta el `commit`; (b) borrado post-commit: en una transacción corta tomar `asset_key_lock(db, old_key)`, comprobar `not is_key_referenced(db, old_key)`, llamar `delete_object(old_key)` y terminar la transacción (mejor esfuerzo, cualquier fallo se registra en el log y no se propaga — comportamiento A-44 intacto). Cada actualización borra solo la `old_key` que ella misma reemplazó. Depende de T034, T040
- [X] T042 [P] [US4] `../pos-backend/app/api/v1/tenant/router.py::update_tenant`: mismo patrón que T041; para `is_key_referenced` y el candado de la key anterior abrir `with with_db(tenant.schema) as tdb:` (la sesión `shared` de `update_tenant` no alcanza las tablas del negocio; research D9). Depende de T035, T040
- [X] T043 [P] [US4] `../pos-backend/app/api/v1/sales/service.py`: tomar `asset_key_lock(db, nueva)` antes de verificar la existencia de un QR nuevo en `create_payment_method` y `update_payment_method` (el QR anterior **no** se borra, research D8; `deletable_key` + `is_key_referenced` ya lo soportarían si el negocio lo pide después). Depende de T036, T040

**Checkpoint**: correr T037–T039 y `test_products_service` (A-44 debe seguir en verde). SC-004 y SC-005 verificables.

---

## Phase 7: User Story 5 — El cajero solo ve comprobantes válidos del propio negocio (Priority: P2)

**Goal**: un comprobante nuevo es siempre un archivo existente, de la carpeta `comprobantes` y del propio negocio; en base de datos se guarda la key y las tres respuestas devuelven una URL lista para renderizar; las filas históricas se leen igual y se migran con un script idempotente (FR-008).

**Independent Test**: quickstart Historia 5 — el comensal adjunta un comprobante real → aparece en "Pagos por confirmar" y en base de datos queda la key; key inventada, de otro negocio, de otra carpeta y URL ajena → 422.

### Tests

- [X] T044 [P] [US5] Extender `../pos-backend/app/characterization_tests/test_asset_refs.py` con `resolve_receipt_key` (tabla de contracts/asset-guard.md): key válida existente → key; URL del dominio público anterior y del dominio de assets → la key extraída; key mal formada/ajena/otra carpeta → 422 "El comprobante no pertenece a este negocio o no es válido."; URL de otro origen → 422 "El comprobante debe ser un archivo subido desde la aplicación."; key válida inexistente → 422 con el texto de `ensure_image_exists`; R2 no responde → 503
- [X] T045 [P] [US5] Crear `../pos-backend/app/characterization_tests/test_receipts_integrity.py` (con `cart_fixtures.py`): `submit_cart` y `attach_receipt` guardan **solo la key** con key directa, URL vieja y URL de assets; inventada/ajena/otra carpeta/otro origen → 422 sin orden ni intento nuevo y sin borrar el carrito; R2 caído → 503; `409` "ya tiene comprobante" se evalúa **antes** que la validación (US5-5); efectivo con comprobante y método sin comprobante siguen con sus 422 actuales; `PaymentAttemptResponse`, `CurrentPaymentAttemptSummary` y `DinerPaymentAttempt` devuelven `{ASSETS_BASE_URL}/{key}`; fila histórica con URL absoluta del dominio anterior se reescribe al dominio de assets al leer; URL de otro origen sale intacta
- [X] T046 [P] [US5] Crear `../pos-backend/app/characterization_tests/test_migrate_receipt_keys.py` (patrón de `test_migrate_image_keys_relative.py`): simulación por defecto no modifica nada; `--apply` reescribe URL gestionada con key en convención y archivo existente; segunda corrida = 0 filas modificadas (SC-009); URL de otro origen, key fuera de convención, archivo inexistente y R2 no responde → fila intacta, reportada y sin hacer fallar el script; `--revert --apply` reconstruye `{R2_PUBLIC_BASE_URL}/{key}` solo para keys `{esquema}/comprobantes/…` del propio esquema; `--tenant` acota; no se llama nunca a `delete_object`/`put_object`
- [X] T047 [P] [US5] Actualizar `../pos-backend/app/characterization_tests/test_cart_payment_attempts.py`: comprobantes con URL fuera de convención pasan a una key `tenant_test/comprobantes/<hex>.jpg` con `object_exists` parcheado; citar spec 088 + A-93
- [X] T048 [P] [US5] Actualizar `../pos-backend/app/characterization_tests/test_orders_payment_gate.py` (l.~505–520): `attach_receipt` con key válida y `object_exists` parcheado; citar A-93
- [X] T049 [P] [US5] Actualizar `../pos-backend/app/characterization_tests/test_order_audit_log.py`: `COMPROBANTE` (l.57, hoy `"https://example.invalid/comprobantes/abc123.jpg"`) pasa a una key válida; el test FR-012 de la spec 074 (~l.681, "el mismo comprobante produce el mismo hash en sus tres eventos") calcula el hash esperado sobre la **key** persistida — la igualdad entre eventos del mismo intento se preserva (research D11); citar A-93

### Implementation

- [X] T050 [US5] `../pos-backend/app/core/asset_refs.py`: añadir `resolve_receipt_key(value, tenant_schema) -> str` (`normalize_asset_ref` → URL de otro origen (`://`) → 422; `validate_asset_key(..., FOLDER_RECEIPTS)` → 422; `object_exists` → 422/503 vía la misma conversión de `ensure_image_exists`; mensajes de contracts/asset-guard.md; el detalle de la key no se refleja en el mensaje, solo en el log). Depende de T007
- [X] T051 [P] [US5] `../pos-backend/app/api/v1/orders/schemas.py`: `receipt_file_url: AssetUrl = None` en `CurrentPaymentAttemptSummary` (l.199) y en `PaymentAttemptResponse` (l.256) — esta última alimenta "Pagos por confirmar" del cajero (tolerancia de lectura para filas antiguas)
- [X] T052 [P] [US5] `../pos-backend/app/api/v1/cart/schemas.py`: `receipt_file_url` de `DinerPaymentAttempt` (l.155) pasa a `AssetUrl`; los campos de **entrada** `SubmitCartIn.receipt_file_url` (l.181, máx. 500) y `ReceiptAttachIn.file_url` conservan nombre y límite (compatibilidad con el frontend actual)
- [X] T053 [US5] `../pos-backend/app/api/v1/cart/service.py`: añadir el parámetro `tenant_schema` a `submit_cart` (l.510) y `attach_receipt` (l.913). En `submit_cart`, tras los 422 actuales de efectivo/obligatorio (l.602–607) y **antes** de `check_availability` y de crear nada, `receipt_file_url = resolve_receipt_key(receipt_file_url, tenant_schema)` para métodos no efectivo; persistir la key (l.676) y calcular el hash de auditoría (l.734) sobre la key. En `attach_receipt`, tras el 409 existente (l.925), `resolve_receipt_key`, guardar la key (l.929) y hashear la key. Revisar el hash de l.860 (eventos de aprobación/rechazo) para que use el valor persistido. Depende de T050, T052
- [X] T054 [US5] `../pos-backend/app/api/v1/cart/router.py`: pasar `ctx.tenant.schema` a `submit_cart` (l.155) y `attach_receipt` (l.288). Depende de T053
- [X] T055 [P] [US5] Crear `../pos-backend/app/scripts/migrate_receipt_keys.py` (patrón de `migrate_image_keys_relative.py`; contrato en contracts/scripts.md y research D15): `python -m app.scripts.migrate_receipt_keys [--apply] [--revert] [--tenant SCHEMA]`; sin argumentos = simulación; por esquema y por fila de `order_payment_attempts` con `receipt_file_url` no nulo aplicar la tabla de reglas (ya key → sin cambio; URL gestionada con key en convención y archivo existente → reescribir; fuera de convención / inexistente / otro origen / R2 no responde → intacta y reportada, sin fallar); `commit` por esquema; `--revert` reconstruye `{R2_PUBLIC_BASE_URL}/{key}` solo para keys `{esquema}/comprobantes/…` del propio esquema; imprimir el resumen por negocio del ejemplo de contracts/scripts.md; no toca objetos de R2. Depende de T004

**Checkpoint**: correr T044–T049. Quickstart Historia 5 y "Migración de comprobantes". SC-006 y SC-009.

---

## Phase 8: User Story 6 — Reporte de reconciliación para el operador (Priority: P3)

**Goal**: un script de solo lectura compara la base de datos con el almacenamiento y lista, por negocio, referencias sin archivo, archivos huérfanos (con ventana de gracia) y referencias fuera de convención (FR-009).

**Independent Test**: quickstart Historia 6 — borrar a mano un objeto referenciado y subir otro sin referencia; ejecutar el script: aparecen como referencia sin archivo y huérfano, y después la base de datos y el bucket están idénticos.

- [X] T056 [P] [US6] Crear `../pos-backend/app/characterization_tests/test_reconcile_r2_references.py` con R2 simulado (`get_r2_client` parcheado con un doble que **falla** ante cualquier método de escritura y solo admite `list_objects_v2`/`head_object`): referencia sin archivo; huérfano viejo listado; huérfano más reciente que `--grace-hours` **no** listado (24 h por defecto); key histórica fuera de convención marcada; URL de otro origen contada aparte y no verificada; `--tenant` acota referencias y objetos; `--output` escribe CSV (`negocio,tipo,campo,key,detalle`, `tipo` ∈ `referencia_sin_archivo` | `archivo_sin_referencia` | `fuera_de_convencion`) y JSON e imprime la ruta; la sesión nunca hace `commit` y la base de datos queda idéntica
- [X] T057 [US6] Crear `../pos-backend/app/scripts/reconcile_r2_references.py` (contrato en contracts/scripts.md, research D14): `python -m app.scripts.reconcile_r2_references [--tenant SCHEMA] [--grace-hours 24] [--output RUTA] [--format csv|json]`; recorre `shared.tenants` con `with_db(None)` y cada esquema con `with_db(schema)` en modo lectura (`SET TRANSACTION READ ONLY`, `rollback()` al final, nunca `commit`); reúne las cuatro fuentes de referencias (`products.image_url`, claves `format:"image"` de `payment_methods.payment_info` según `catalog.fields`, `order_payment_attempts.receipt_file_url`, `shared.tenants.logo_url`); **un solo** listado paginado de `list_objects_v2` (con `Prefix` solo si hay `--tenant`); clasifica por negocio; imprime el resumen legible del ejemplo del contrato; código de salida `0` siempre que el reporte se genere. El módulo no importa `delete_object` ni `put_object`. Depende de T004

**Checkpoint**: correr T056; quickstart Historia 6. SC-007.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [X] T058 Correr la suite completa de `../pos-backend`: `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` — todo en verde, incluidos los tests `CONGELA` (SC-008, Principio X). Si algún test ajeno a los listados en T009/T010/T047–T049 falla, es una regresión: corregir el código, no el test
- [X] T059 [P] Correr en `../pos-heladeria` `ng test --watch=false` completo y `ng build` sin errores. **Resultado 2026-09-29**: `ng build` correcto; `ng test` 1033 pruebas, las 20 nuevas de esta spec en verde. Quedan 19 fallos **previos y ajenos** (idénticos en la rama base con `git stash`, 19 fallidos de 1013): `super-admin/tenant.service`, `core/auth.service`, `app.spec`, `pos-order-panel`, `pos-checkout-panel` (T032) y `menu.service`. No se tocaron
- [ ] T060 **[PARCIAL — quedan recorridos manuales, ver "Notas de implementación"]** Recorrer [quickstart.md](./quickstart.md) en un entorno con bucket de pruebas: Historias 1–6, "Migración de comprobantes" y "Regresión" (imágenes de producto, logos, QR y comprobantes existentes se siguen viendo tras el despliegue, SC-008). Los pasos que exigen el navegador o el panel de Cloudflare quedan como recorridos manuales si el entorno no los permite, anotándolo
- [X] T061 Verificar el orden de despliegue de research D10 y dejarlo por escrito en las notas de implementación: (1) `pos-heladeria`; (2) `pos-backend`; (3) `migrate_receipt_keys` en simulación → revisar → `--apply`; (4) `reconcile_r2_references` y revisar el primer informe (huérfanos y referencias rotas históricas: acción manual del operador, fuera de alcance); (5) opcional, T062
- [ ] T062 [P] **PENDIENTE A PROPÓSITO (no hacer antes del despliegue del backend)** **OPCIONAL y último — solo después de desplegar el backend** (un frontend que envíe `key` a un backend anterior mostraría una imagen rota al cajero, research D10): `../pos-heladeria/src/app/modules/tables/pages/checkout/transfer-details-step.component.ts` (l.~302) envía `presign.key` en vez de `presign.public_url`; actualizar `transfer-details-step.component.spec.ts`
- [X] T063 Marcar las tareas completadas en este archivo y registrar en `specs/088-integridad-referencias-r2/` las notas de implementación (opción de fixture de T006, desviaciones respecto de `research.md`, y las decisiones D7/D8 por confirmar con el negocio) para que las specs y el registro de anomalías queden trazables (Principio XII)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias. T001–T002 deben terminar **antes de implementar** (plan.md, fila II); T003 antes de tocar código.
- **Foundational (Phase 2)**: depende de Setup — **bloquea** todas las historias.
- **US1 (P1)**, **US2 (P1)**, **US3 (P1)**: dependen de Foundational. Comparten los mismos tres servicios (`products/service.py`, `tenant/router.py`, `sales/service.py`), por lo que **se implementan en orden US1 → US2 → US3** (cada una añade una capa a la misma función); las pruebas y el frontend de cada una sí son independientes.
- **US4 (P2)**: depende de US3 (reutiliza `deletable_key` y la ruta de guardado) y de T006 (fixture de `shared.tenants`).
- **US5 (P2)**: depende de Foundational (usa `object_exists` y `validate_asset_key`; no necesita `is_key_referenced`, pero comparte `asset_refs.py` con US4, por lo que T050 se añade después de T040 si se trabajan en paralelo); sus tareas de esquemas/migración son independientes de US1–US4.
- **US6 (P3)**: depende solo de Foundational (T004) — puede hacerse en paralelo con US4/US5.
- **Polish (Phase 9)**: depende de las historias deseadas. T062 depende del **despliegue** del backend.

### User Story Dependencies

- **US1**: sin dependencias de otras historias. Es el MVP mínimo (con su frontend).
- **US2** y **US3**: extienden las mismas funciones que US1; cada una es verificable por separado con sus pruebas.
- **US4**: necesita US3 para que `deletable_key` ya esté en la ruta de borrado.
- **US5** y **US6**: independientes del resto salvo por las primitivas.

### Within Each User Story

- Tests (que deben fallar contra el código actual) → esquemas → servicios → frontend / checkpoint.
- En backend: `schemas.py` antes de `service.py`/`router.py`; `asset_refs.py` antes de los servicios que lo usan.
- Los cambios del frontend (T020–T024) no dependen del backend pero **se despliegan antes** que él.

### Parallel Opportunities

- **Foundational**: T005, T006 y T008 en paralelo (archivos distintos) una vez que T004/T007 definan la API; T007 depende de T004.
- **US1**: T009–T013 (tests) en paralelo; T014–T016 (esquemas) en paralelo; T017–T019 (tres servicios distintos) en paralelo tras sus esquemas; T020–T023 (frontend) en paralelo con todo el backend.
- **US2/US3/US4**: las tres pruebas por historia (producto, logo, QR) en paralelo; los servicios de `tenant/router.py` y `sales/service.py` en paralelo con el de productos.
- **US5**: T044–T049 (tests) y T051, T052, T055 en paralelo; US6 (T056–T057) en paralelo con US4/US5.

---

## Parallel Example: User Story 1

```bash
# Tests de US1 (archivos distintos):
Task: "T009 Actualizar los 3 tests A-44 en test_products_service.py"
Task: "T010 Añadir la base a test_products_image_key.py, test_tenant_logo_key.py, test_payment_methods_qr_key.py"
Task: "T011 Crear test_products_image_integrity.py"
Task: "T012 Crear test_tenant_logo_integrity.py"
Task: "T013 Crear test_payment_methods_qr_integrity.py"

# Esquemas y frontend en paralelo:
Task: "T014 ProductUpdate.image_url_base en products/schemas.py"
Task: "T015 TenantUpdate.logo_url_base en tenant/schemas.py"
Task: "T016 PaymentMethodUpdate.payment_info_base en sales/schemas.py"
Task: "T020 product.service.ts + spec (image_url_base, no reenviar la imagen sin tocar)"
Task: "T022 tenant-info.service.ts + spec nuevo"
Task: "T023 payment-method.service.ts + spec"
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1: Setup (A-92, A-93, ramas).
2. Phase 2: Foundational (primitivas y su prueba).
3. Phase 3: US1 — backend + frontend.
4. **STOP y VALIDAR**: quickstart Historia 1; desplegar `pos-heladeria` primero y luego `pos-backend`.

Recomendado: entregar **US1 + US2 + US3 juntas** (las tres P1 comparten las mismas tres funciones de servicio y forman la barrera completa de entrada: SC-001, SC-002, SC-003) antes de pasar a las P2.

### Incremental Delivery

1. Setup + Foundational → primitivas listas.
2. US1 → US2 → US3 → checkpoint P1 completo (formulario desactualizado, existencia y aislamiento entre negocios).
3. US4 → el borrado solo ocurre si nadie usa el archivo.
4. US5 → comprobantes por key + migración; luego ejecutar la migración en simulación y aplicarla.
5. US6 → reporte de reconciliación; primer informe tras el despliegue.
6. Polish → suite completa, quickstart, notas de despliegue; T062 al final y solo tras el backend.

### Notas

- Cada historia deja el sistema en un estado desplegable y no rompe las anteriores; sin migración de Alembic, sin dependencias ni variables de entorno nuevas.
- Los commits los pide el usuario (Principio XV): commits pequeños por unidad (una primitiva, un servicio, un script, un formulario), en inglés, Conventional Commits, sin marcas de IA; citar A-92/A-93 y la spec 088 en los que toquen tests existentes.

---

## Notas de implementación (2026-09-29)

**Estado**: T001–T059, T061 y T063 hechas. Quedan **T060** (recorrido del quickstart en un entorno con bucket de pruebas; solo se verificó lo que no exige R2 ni navegador) y **T062** (opcional, a propósito no hecha: solo después de desplegar el backend). Sin commits ni push (Principio XV: los pide el usuario). Ramas: `feat/088-r2-reference-integrity` en `pos-backend` y `pos-heladeria`; `pos-specs` sigue en `main`.

### Verificación

- `pos-backend`: `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` → **1127 pruebas OK** (946 antes de empezar + 181 nuevas), incluidos todos los tests `CONGELA`. Los únicos tests existentes que hubo que tocar son los previstos (T009/T010/T047–T049): `test_products_service` (3 A-44), `test_products_image_key`, `test_tenant_logo_key`, `test_payment_methods_qr_key`, `test_cart_payment_attempts`, `test_orders_payment_gate`, `test_order_audit_log`; en todos se conserva lo que verifican.
- `pos-heladeria`: 20 pruebas nuevas (`product.service`, `product-form.component`, `tenant-info.service` —nuevo—, `payment-method.service`, `payment-methods-page.component` —nuevo—) y `ng build` correcto. Ver T059 para los 19 fallos previos ajenos.
- **PostgreSQL local** (solo lectura, sin tocar R2), lo único que SQLite no podía comprobar: `pg_advisory_xact_lock(hashtextextended(...))` se toma y **se libera al terminar la transacción**; `is_key_referenced` corre sobre un esquema real (`heladeria`) con `shared.tenants` y JSONB; `SET TRANSACTION READ ONLY` **rechaza** una escritura.

### Opción de fixture (T006)

Opción **(a)**: `fixtures.attach_shared_tenants(conn)` hace `ATTACH DATABASE ':memory:' AS shared` y crea `shared.tenants`, igual que `payment_catalog_fixtures` con el catálogo; lo usan `fixtures.new_session` y `cart_fixtures.new_session`. Además `fixtures._TABLE_NAMES` incorpora `payment_methods` y `order_payment_attempts` (con el shim de JSONB), `make_tenant_stub` acepta `schema` (por defecto `heladeria3`) y hay un `make_shared_tenant`. No hizo falta la opción (b) (`shared_check` inyectable).

### Desviaciones respecto de `research.md` / `contracts/`

1. **Intentos de R2**: se usa `retries={"total_max_attempts": 2, "mode": "standard"}` y no `max_attempts: 2`. En botocore `max_attempts` cuenta *reintentos* (serían 3 intentos, peor caso ~24 s); el diseño hablaba de 2 intentos y ~16 s.
2. **Helpers extra en `asset_refs.py`** (no estaban en el contrato, son los que comparten producto, logo y QR): `resolve_image_change` (decidir + 422 + candado + verificar existencia, con `rollback()` si falla para soltar el candado), `delete_if_unreferenced` (candado + `is_key_referenced` + borrado, mejor esfuerzo, `rollback()` al final), `image_key_error_to_http` e `is_managed_ref`.
3. **`tenant_schema` opcional** (palabra clave) en `create_payment_method`, `update_payment_method`, `submit_cart` y `attach_receipt`: los routers siempre lo pasan; si falta cuando hay una imagen/comprobante que validar se lanza `ValueError` (nunca se acepta en silencio). Evita reescribir ~40 llamadas de tests que no manejan imágenes.
4. **Misma imagen en otra representación (`KEEP`)**: si el formulario reenvía la imagen vigente como URL absoluta y la fila la guarda igual pero como URL histórica, se normaliza a key al volver a guardar (conserva spec 080 FR-004, que `test_tenant_logo_key` congela). No verifica ni borra nada.
5. **`is_key_referenced` corre real** en los tres A-44 (sus tablas existen en el fixture) en lugar de parcharse como sugería T038: ejercita más código y el resultado es el mismo (`old.jpg` no la usa nadie).
6. **Borrado post-commit** (`update_product`): `commit` → `delete_if_unreferenced` → `refresh`. El orden `["commit", "delete"]` que congela A-44 se conserva.
7. **QR**: quitar el QR con una base que no coincide (o sin `payment_info_base`) se ignora y se conserva el vigente (D7, paso 4 aplicado también a "vacío"); el frontend envía `payment_info_base: {}` si no llegó a registrar base.
8. **Reporte de reconciliación**: con `--tenant` se leen las tablas solo de ese negocio pero **todos los logos** (baratos); un archivo del prefijo acotado que solo lo referenciara una fila de *otro* negocio se listaría como huérfano. Documentado en el módulo.
9. **Frontend**: `update()` de `tenant-info.service` queda tipado como `Partial<Omit<TenantInfo, 'logo_url'>>` (el tipo impide reenviar el logo); `ProductForm.image_url_base` es opcional (los métodos planos `createProduct`/`updateProduct` no se usan fuera de sus specs).

### Decisiones de negocio

- **D7 — confirmada** (opción A, 2026-09-29): el QR se puede quitar por edición legítima; el archivo no se borra. Registrada en A-92.
- **D8 — POR CONFIRMAR con el negocio**: no se borra el QR anterior al reemplazarlo o quitarlo (comportamiento actual). Registrada como pendiente en A-92; añadirlo después no requiere cambio de diseño.

### Orden de despliegue (T061, research D10)

1. `pos-heladeria` (base + no reenviar la imagen sin tocar): compatible con el backend actual y con el nuevo.
2. `pos-backend` (validación, base, borrado protegido, comprobantes por key, scripts).
3. `python -m app.scripts.migrate_receipt_keys` (simulación) → revisar el informe → `--apply` (idempotente; una segunda corrida no modifica nada).
4. `python -m app.scripts.reconcile_r2_references` y revisar el primer informe (huérfanos y referencias rotas históricas: acción manual del operador, fuera de alcance).
5. Opcional y último (T062): que el comprobante envíe `presign.key`. **Nunca antes del paso 2.**

Riesgos a vigilar al desplegar: pestañas abiertas con el bundle anterior tras el paso 2 verían ignorada una subida de imagen (forzar recarga); y si hubiera que retirar el backend, ejecutar antes `migrate_receipt_keys --revert --apply`.

### Pendiente (T060) — recorridos que exigen bucket de pruebas y/o navegador

Historias 1–6 del `quickstart.md` contra un R2 real (subida real por presign, dos pestañas con formularios desactualizados, borrar un objeto a mano en Cloudflare, simular R2 caído para el 503, revocar el permiso de borrado, comprobante real como comensal en "Pagos por confirmar"), la migración de comprobantes sobre datos reales y la sección "Regresión". El `.env` local apunta a credenciales de R2 que no se identificaron como de pruebas, por lo que no se ejecutó nada contra R2.
