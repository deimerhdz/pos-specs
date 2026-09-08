# Research: Almacenar imágenes como key relativa y servirlas por dominio personalizado

Fase 0. El spec no dejó ningún `NEEDS CLARIFICATION` (5 aclaraciones resueltas con el negocio el 2026-09-08). Lo que sigue son las decisiones **técnicas** para implementar ese contrato reutilizando las primitivas de almacenamiento y de esquemas ya vigentes en `pos-backend`.

Contexto del código actual (verificado 2026-09-08):

- `app/core/storage.py` centraliza R2: `build_object_key(schema, folder, ext)` → `{schema}/{folder}/{uuid}.{ext}`; `public_url_for(key)` → `{R2_PUBLIC_BASE_URL}/{key}`; `key_from_public_url(url)` → key o `None` si no empieza por `{R2_PUBLIC_BASE_URL}/`; `delete_object(key)` best-effort.
- `POST /uploads/presign` (`uploads/router.py`) devuelve `{upload_url, key, public_url, expires_in}` para las carpetas `products` / `logo` / `payment-methods`. Hoy el frontend guarda `public_url`.
- Escritura de las 3 referencias: `products/service.py::create_product` / `update_product` (`image_url`), `tenant/router.py::update_tenant` (`logo_url`), `sales/service.py::create_payment_method` / `update_payment_method` (`payment_info`).
- Borrado del objeto anterior (best-effort, post-commit, A-44 / spec 021): solo en `products/service.py::update_product` y `tenant/router.py::update_tenant`, vía `key_from_public_url(old) -> delete_object(key)`. Los métodos de pago **no** borran el QR anterior hoy.
- Lectura: `image_url` / `logo_url` se sirven **tal cual** desde la columna (`ProductResponse`, `TenantInfoResponse`, `MenuProductResponse`, `MenuBusinessResponse`); `payment_info` se sirve tal cual en `DinerPaymentMethod` (checkout del comensal) y `PaymentMethodResponse` (panel admin, sin `available`).
- Comprobantes del comensal: `cart/service.py::presign_payment_receipt` / `presign_receipt` usan las **mismas** primitivas y guardan `receipt_file_url` = `public_url` completo. `test_cart_payment_attempts.py` (`"CONGELA"`) lo congela.
- El frontend (`pos-heladeria`) **no arma URLs de imagen**: usa la que recibe de la API (`<img [src]="...">`). Verificado en `product-form.component.ts`, `tenant-info.service.ts`, `payment-method.service.ts`, `transfer-details-step.component.ts`, `sidebar.component.ts`, `receipt.util.ts`, `diner.service.ts`, `public-menu.component.ts`.
- Precedente de tipo `Annotated` con serializador de Pydantic v2 en el proyecto: `app/core/timezone.py::UtcDatetime = Annotated[datetime, PlainSerializer(_serialize_utc, return_type=str)]`, usado en decenas de esquemas de respuesta.
- Precedente de migración de datos puntual: `app/scripts/migrate_payment_methods_catalog.py` (recorre `shared.tenants` + cada schema con `with_db(schema)`, `--report-only`, idempotente vía "si ya está migrada, saltar").

---

## 1. Configuración: `ASSETS_BASE_URL` nuevo; `R2_PUBLIC_BASE_URL` se conserva para comprobantes y reversión

**Decision**: se añade un setting nuevo `ASSETS_BASE_URL` (`app/core/config.py`, `Field(..., env="ASSETS_BASE_URL")`, obligatorio; `.env.example` con comentario). Valor de producción: `https://assets.skeilopos.com`. `R2_PUBLIC_BASE_URL` **no se repunta**: se queda apuntando a `https://pub-d819ae78f038476da2ef5fa0bb171aa5.r2.dev`.

- `asset_display_url(...)` y el `public_url` de `POST /uploads/presign` se arman con `ASSETS_BASE_URL`.
- `public_url_for(...)` (que usan **solo** los presign de comprobantes tras esta spec) sigue armando con `R2_PUBLIC_BASE_URL` → los comprobantes se guardan y se sirven desde `pub-…r2.dev`, sin cambios (FR-014).
- El "bucket gestionado" a efectos de normalización/migración/borrado se reconoce contra **ambos** prefijos: `_managed_bucket_prefixes() -> tuple[str, ...]` = `(f"{ASSETS_BASE_URL.rstrip('/')}/", f"{R2_PUBLIC_BASE_URL.rstrip('/')}/")` (de-duplicado por si en algún entorno coinciden).
- Reversión de la migración: reconstruir `{R2_PUBLIC_BASE_URL}/{key}` — exactamente la forma que había antes (SC-008).

**Rationale**:
- FR-014 exige que los comprobantes "se sigan guardando con la URL completa y sirviéndose desde el dominio anterior, **sin cambios**". Repuntar `R2_PUBLIC_BASE_URL` cambiaría el dominio de los comprobantes nuevos e introduciría justo la inconsistencia (unos comprobantes en un dominio, otros en otro) que el spec dice abordar en una spec aparte. Dejar `R2_PUBLIC_BASE_URL` intacto hace que la ruta de comprobantes **no cambie ni una línea**.
- FR-004 exige normalizar tanto `pub-…r2.dev/{key}` como `assets.skeilopos.com/{key}`. Reconocer dos prefijos es trivial y cubre el reenvío habitual del frontend (que manda la URL de visualización del dominio nuevo al guardar un formulario que no tocó la imagen).
- FR-008 exige que el dominio de assets sea configuración desplegable, no un literal. `ASSETS_BASE_URL` lo cumple; un cambio futuro de dominio es una variable de entorno y cero migración (SC-005).
- `Field(...)` (obligatorio, sin default) es coherente con el resto de settings de R2 y hace que un despliegue sin la variable falle al arrancar en vez de servir URLs rotas.

**Alternatives considered**:
- *Repuntar `R2_PUBLIC_BASE_URL` a `assets.skeilopos.com` y dar a los comprobantes un `R2_LEGACY_PUBLIC_BASE_URL`*: descartado — mismo número de variables, pero **toca** la ruta de comprobantes (nuevo builder de URL en `cart/service.py`), roza el veto de `test_cart_payment_attempts.py` y contradice el "sin cambios" de FR-014.
- *Un único setting y derivar el prefijo viejo de un literal en el código*: descartado — FR-008 prohíbe el literal incrustado; y el prefijo viejo hay que tenerlo en config para poder revertir la migración a la forma exacta.
- *Lista `ASSET_MANAGED_PREFIXES` configurable*: descartado por ahora — solo hay dos dominios conocidos y ambos ya son settings; una lista añade superficie de configuración sin caso de uso. Si aparece un tercer dominio, se amplía entonces (Principio VI).

---

## 2. Los tres helpers viven en `app/core/storage.py` — única fuente de la regla key ↔ URL

**Decision**: toda la lógica de "¿esto es una key, una URL del bucket gestionado, o de otro origen?" vive en `app/core/storage.py`, en tres funciones puras (sin I/O, testeables en aislamiento):

```
asset_display_url(value: str | None) -> str | None      # para RESPONDER al consumidor
normalize_asset_ref(value: str | None) -> str | None     # para PERSISTIR en base de datos
object_key_for_deletion(value: str | None) -> str | None  # para BORRAR el objeto anterior
```

Semántica completa en [`contracts/asset-reference.md`](./contracts/asset-reference.md) y tabla de verdad en [`data-model.md`](./data-model.md §3). Resumen:

| Entra | `normalize_asset_ref` (persistir) | `asset_display_url` (responder) | `object_key_for_deletion` (borrar) |
|---|---|---|---|
| `None` / `""` | `None` | `None` | `None` |
| key `heladeria3/products/x.png` | igual (key) | `https://assets.skeilopos.com/heladeria3/products/x.png` | `heladeria3/products/x.png` |
| `https://pub-…r2.dev/heladeria3/products/x.png` | `heladeria3/products/x.png` | `https://assets.skeilopos.com/heladeria3/products/x.png` | `heladeria3/products/x.png` |
| `https://assets.skeilopos.com/heladeria3/products/x.png` | `heladeria3/products/x.png` | igual (ya es dominio nuevo) | `heladeria3/products/x.png` |
| `https://otro-origen.com/foto.jpg` (otro origen) | igual (intacto, FR-010) | igual (intacto, FR-010) | `None` (no se borra, FR-013) |

`key_from_public_url` se **renombra** a `object_key_for_deletion` y se amplía (hoy solo reconoce URL del prefijo único y devuelve `None` para todo lo demás, incluida una key directa). Los dos llamadores (`products/service.py`, `tenant/router.py`) se actualizan.

**Rationale**:
- El spec repite la misma regla en FR-004, FR-005, FR-009, FR-010, FR-011, FR-013 y en varios edge cases. Una sola implementación evita que migración, escritura, lectura y borrado diverjan (es el patrón de anomalías A-01 / A-13 / A-32 del propio proyecto: "la misma pregunta con tres respuestas distintas").
- `storage.py` ya es el módulo de R2 y ya tiene `key_from_public_url`; es el lugar natural.
- Funciones puras → los tests de la tabla de verdad no necesitan base de datos ni fixtures.

**Alternatives considered**:
- *Lógica repartida en cada servicio*: descartado — cuatro copias que envejecen distinto.
- *Una clase `AssetRef` con métodos*: descartado — sobra estado; tres funciones puras + dos tipos `Annotated` es más simple y más fácil de testear.

---

## 3. Normalización en escritura: en el validador del esquema, **antes** de la comparación que dispara el borrado

**Decision**: la normalización a key en entrada se hace con un tipo `Annotated` reutilizable aplicado al campo del esquema de request:

```
AssetRefIn = Annotated[str | None, AfterValidator(normalize_asset_ref)]
```

- `ProductCreate.image_url: AssetRefIn`, `ProductUpdate.image_url: AssetRefIn`, `TenantUpdate.logo_url: AssetRefIn`.
- Para `payment_info` (dict), un validador a nivel de campo que aplica `normalize_asset_ref` a **cada valor** del dict: `{k: normalize_asset_ref(v) or v for k, v in info.items()}` (los valores que no son URL del bucket gestionado —celular, cuenta— quedan idénticos porque `normalize_asset_ref` solo toca los que empiezan por un prefijo gestionado). Se aplica en `PaymentMethodCreate.payment_info` y `PaymentMethodUpdate.payment_info`.

Consecuencia clave en `products/service.py::update_product` y `tenant/router.py::update_tenant`: cuando el código hace `if data.image_url is not None and data.image_url != product.image_url:`, **ambos lados ya son key** (`data.image_url` viene normalizada del esquema; `product.image_url` es key tras la migración). El bloque de borrado del objeto anterior solo se ejecuta cuando la key **de verdad** cambió.

**Rationale**:
- **US3 / FR-011 / FR-012 (la razón principal de esta decisión)**: hoy el frontend, al guardar un formulario de producto que no tocó la imagen, reenvía la URL de visualización que recibió en el `GET`. Tras esta spec esa URL es `https://assets.skeilopos.com/{key}` y en BD está la `key`. Si se comparara URL-entrante contra key-almacenada, **siempre** diferirían → el código creería que la imagen cambió → `delete_object(key_del_objeto_vigente)` → **imagen rota en un guardado que no tocó nada**. Normalizar en el esquema, antes de la comparación, lo evita de raíz. Escenario cubierto por `test_products_image_key.py::guardar_sin_tocar_la_imagen_no_borra`.
- Hacerlo declarativamente en el tipo del campo mantiene los servicios casi sin cambios y es imposible de "olvidar" en un endpoint nuevo que use el mismo esquema.
- `AfterValidator` corre después de la coerción de tipo de Pydantic, con el `str` ya validado — el momento correcto.

**Alternatives considered**:
- *Normalizar en el servicio, primera línea*: funciona, pero hay que acordarse en cada punto de escritura (3 servicios, 5 funciones) y en los futuros; y deja el `data.image_url` "sucio" para cualquier otra lógica del servicio.
- *Normalizar en un middleware / dependencia de FastAPI*: descartado — no sabe qué campos son referencias de imagen.
- *No normalizar y confiar en que el frontend mande siempre la key*: descartado — FR-004 exige la garantía en el servidor "sin depender del comportamiento del cliente"; y hay filas viejas y formularios que reenvían la URL.

---

## 4. Ensamblado en lectura: tipo `Annotated` en las respuestas simples; mapeo explícito para `payment_info`

**Decision**:

```
AssetUrl = Annotated[str | None, PlainSerializer(asset_display_url, return_type=str | None)]
```

- `ProductResponse.image_url: AssetUrl` (cubre `ProductListResponse`, `ProductDetailResponse`, `ProductSaveResponse` por herencia), `MenuProductResponse.image_url: AssetUrl`, `TenantInfoResponse.logo_url: AssetUrl`, `MenuBusinessResponse.logo_url: AssetUrl`.
- `payment_info` (dict, y solo algunas claves son imágenes): **no** se puede resolver con un serializador de campo ciego, porque una clave `celular` con valor `"3001234567"` no debe convertirse en `https://assets.skeilopos.com/3001234567`. Hace falta la metadata `fields[*].format == "image"` del catálogo:
  - `DinerPaymentMethod` (checkout del comensal) ya expone `fields`: se añade `@model_validator(mode="after")` que, para cada `f in self.fields` con `f["format"] == "image"`, reemplaza `self.payment_info[f["key"]]` por `asset_display_url(...)`.
  - `PaymentMethodResponse` (panel admin) **no** lleva `fields`. Se construye la respuesta explícitamente en `sales/router.py` con un helper `payment_method_response(method: PaymentMethod) -> PaymentMethodResponse` que usa `method.fields` (propiedad ya existente que lee de `method.catalog`). Los 3 endpoints que devuelven `PaymentMethodResponse` (`GET /sales/payment-methods` sin `available`, `POST`, `PATCH`) pasan por ese helper. `list_payment_methods` del servicio del comensal ya hace `selectinload(PaymentMethod.catalog)`; el `list` admin del router hace un `select(...).scalars().all()` — se le añade el mismo `selectinload` para no disparar N+1.

**Rationale**:
- FR-006/FR-007: la URL absoluta solo existe en la respuesta, nunca en BD. `PlainSerializer` actúa en la serialización de salida y no toca el modelo ORM ni la columna.
- FR-009b (tolerancia de lectura): `asset_display_url` reconoce el prefijo viejo, así que una fila que se escape de la migración **se muestra igual** (reescrita al dominio nuevo al vuelo), sin código extra.
- El `model_validator` de `DinerPaymentMethod` corre tras poblar el modelo desde el ORM (`from_attributes`), con `fields` y `payment_info` ya disponibles — momento correcto, sin tocar el servicio.
- Un helper explícito para `PaymentMethodResponse` es más honesto que "colar" `fields` en un esquema que la clarificación 2026-08-24 #1 (spec 032) dice que **no** debe exponer datos de integración crudos a caja — `PaymentMethodResponse` es del admin, no de caja, pero mantener su forma actual y ensamblar en el router evita reabrir esa discusión.

**Alternatives considered**:
- *Ensamblar en el servicio y devolver dicts ya listos*: descartado — rompe `from_attributes`, obliga a mapear a mano campos que hoy fluyen solos, y varios characterization tests (`test_sales_payment_methods_catalog.py`) comparan el `payment_info` del **retorno del servicio** (ORM) — deben seguir viendo la key, no la URL.
- *Añadir una columna calculada `image_display_url` al modelo*: descartado — es exactamente lo que FR-006 prohíbe (la URL no vive en el dato).
- *Serializador de campo en `payment_info` que solo transforme valores "con pinta de key"*: descartado — heurística frágil (¿`titular` = "Heladería La 14" tiene pinta de key?). La metadata `fields` es la fuente correcta.

---

## 5. `POST /uploads/presign`: sigue devolviendo la key; `public_url` pasa a armarse con el dominio nuevo

**Decision**: `uploads/router.py` mantiene la respuesta `{upload_url, key, public_url, expires_in}`. Único cambio: `public_url` se arma con `asset_display_url(key)` (dominio nuevo) en vez de `public_url_for(key)` (dominio viejo). El docstring del endpoint y de `PresignResponse.public_url` se actualiza: "URL de visualización lista para usar; **no** es lo que se persiste — en base de datos vive la key (spec 080)".

**Rationale**:
- Este endpoint sirve **exclusivamente** las carpetas `products` / `logo` / `payment-methods` (las tres en alcance). Devolver el `public_url` ya contra `assets.skeilopos.com` lo deja coherente con lo que el consumidor verá después.
- El frontend no cambia: si sigue guardando `public_url` y lo reenvía al PATCH, la normalización del esquema (§3) lo convierte a key igual. Si en el futuro el frontend prefiere guardar `key` directamente, ya la recibe.
- Los presign de **comprobantes** (`cart/service.py`) son funciones distintas y siguen usando `public_url_for` → dominio viejo (FR-014). No se tocan.

**Alternatives considered**:
- *Quitar `public_url` de la respuesta*: descartado — rompe el contrato del endpoint sin necesidad; el frontend lo lee hoy.
- *Dejar `public_url` con el dominio viejo*: funciona (la normalización lo arregla), pero deja la respuesta del endpoint incoherente con el dominio real de servicio.

---

## 6. Migración: script de datos idempotente y reversible, no una migración de Alembic

**Decision**: `app/scripts/migrate_image_keys_relative.py`, espejo de `migrate_payment_methods_catalog.py`:

```
python -m app.scripts.migrate_image_keys_relative --report-only   # cuenta filas a migrar por campo/tenant, no escribe
python -m app.scripts.migrate_image_keys_relative                 # migra
python -m app.scripts.migrate_image_keys_relative --revert        # reconstruye {R2_PUBLIC_BASE_URL}/{key} en las filas que hoy son key
```

- **Alcance por corrida**: `shared.tenants.logo_url` (una vez, `with_db(None)`) + por cada schema de `SELECT schema FROM shared.tenants`: `products.image_url` y `payment_methods.payment_info` (`with_db(schema)`, `commit` por schema).
- **Predicado de selección** (idéntico en las tres): procesar una fila **solo si** su valor (para `payment_info`, cada valor string del dict) empieza por alguno de los prefijos de `_managed_bucket_prefixes()`. Reescribir a `normalize_asset_ref(valor)`. Dejar intactas: las que ya son key, las `NULL`/`""`, y las URLs de otro origen (FR-009c, FR-010).
- **Idempotencia**: como el predicado exige "empieza por prefijo del bucket gestionado" y el resultado ya no lo cumple, una segunda corrida no toca ninguna fila ya migrada (SC-009). No hace falta una tabla de "ya migrado".
- **Reversión** (`--revert`): para cada fila cuyo valor es una key (sin `://`) en los tres campos, reescribir a `f"{settings.R2_PUBLIC_BASE_URL.rstrip('/')}/{key}"`. Deja intactas las de otro origen y las vacías. Aplicar migración + revert devuelve los tres campos a su estado exacto previo (SC-008). *Nota operativa*: `--revert` asume que no se subieron imágenes nuevas (que nacerían como key) entre la migración y el revert; se documenta en `contracts/data-migration.md` y en `quickstart.md`.
- **`payment_info`**: se reescribe el dict completo con las claves de imagen convertidas; las demás claves se copian sin tocar. Para el `--revert`, solo se reconstruyen valores que son key (no un `celular`).
- **Reporte**: por tenant y por campo, cuántas filas migradas / ya-key / vacías / otro-origen. Log con el mismo formato que `migrate_payment_methods_catalog`.

**Rationale**:
- **No hay cambio de esquema** (mismos tipos de columna, key más corta que URL) → una migración de Alembic no aplica; sería una `op.execute` con SQL de manipulación de datos, más difícil de hacer idempotente/reversible y de correr "con la app en marcha".
- FR-009c exige idempotente, re-ejecutable con la app arriba, sin ventana de mantenimiento. El patrón "predicado que se auto-excluye tras aplicarse" lo garantiza sin locks: una fila que la app reescribe a key mientras el script corre simplemente ya no entra en el predicado.
- `migrate_payment_methods_catalog.py` ya probó este molde (recorrido `shared` + por schema, `--report-only`, reejecutable) en este mismo proyecto.
- Correr `normalize_asset_ref` (la misma función que usa la escritura) dentro del script garantiza que "migrado por el script" y "guardado por la app" produzcan exactamente la misma key.

**Alternatives considered**:
- *Migración de Alembic con `UPDATE ... WHERE image_url LIKE 'https://pub-%'`*: descartado — `payment_info` es JSONB (SQL más enrevesado), la reversión y la idempotencia quedan como SQL a mano, y Alembic no es el vehículo natural para un backfill multi-schema que debe correr sin bloquear.
- *Sin migración, solo tolerancia de lectura permanente*: descartado — FR-009 exige la migración; la tolerancia es un cinturón temporal (FR-009b), no la estrategia.
- *Migrar también `receipt_file_url`*: descartado explícitamente por FR-014.

---

## 7. Tolerancia de lectura (FR-009b) y su retirada futura

**Decision**: la tolerancia **no es código aparte**: es una propiedad de `asset_display_url`, que reconoce el prefijo viejo (`R2_PUBLIC_BASE_URL`) además del nuevo y reescribe al dominio nuevo al vuelo. Una fila que se escape de la migración se muestra igual. Si además esa fila se vuelve a guardar, `normalize_asset_ref` (§3) la deja como key. En `quickstart.md` se documenta cómo verificar, tras la migración, que `SELECT count(*) FROM ... WHERE image_url LIKE 'https://pub-%'` es 0 en los tres campos y en todos los schemas; cuando eso se confirme en producción, una spec posterior puede reducir `_managed_bucket_prefixes()` a solo `ASSETS_BASE_URL` (y `object_key_for_deletion` a solo key + dominio nuevo).

**Rationale**: coste cero (el prefijo viejo ya hay que reconocerlo para la normalización de FR-004 y para la migración), y la retirada futura es borrar un elemento de una tupla.

---

## 8. Comprobantes del comensal: no se toca ni una línea (FR-014)

**Decision**: `cart/service.py::presign_payment_receipt`, `presign_receipt`, `attach_receipt` y el flujo de `receipt_file_url` quedan **exactamente como están**. Siguen usando `build_object_key(schema, "comprobantes", ext)` + `public_url_for(key)` → `pub-…r2.dev/comprobantes/...`. `test_cart_payment_attempts.py` no se modifica y debe seguir en verde.

**Rationale**: FR-014 es inequívoco. `R2_PUBLIC_BASE_URL` intacto (§1) es lo único que hace falta para que esto se cumpla sin esfuerzo. La carpeta `comprobantes` **no** está en la whitelist de `/uploads/presign` (`products` / `logo` / `payment-methods`), así que el cambio de `public_url` de ese endpoint (§5) no la alcanza.

**Verificación**: `contracts/asset-reference.md` incluye un caso "valor en carpeta `comprobantes`" para dejar constancia de que ninguno de los 3 helpers se aplica a ese flujo, y `quickstart.md` verifica que un comprobante subido tras el cambio sigue guardándose como URL completa contra el dominio viejo.

---

## 9. Frontend (`pos-heladeria`): cero cambios

**Decision**: no se toca ningún archivo de `pos-heladeria`.

**Rationale**:
- FR-007 + Assumptions: los consumidores siguen recibiendo una URL absoluta lista para usar por cada imagen; el ensamblado ocurre en el servidor. Los `<img [src]="...">` de producto, logo y QR reciben la URL contra `assets.skeilopos.com` y renderizan igual.
- El caso "el formulario reenvía la URL de visualización al guardar sin tocar la imagen" lo resuelve la normalización del servidor (§3), no el cliente (FR-004 lo exige así).
- El preview inmediato tras elegir un archivo (`product-form.component.ts` usa `URL.createObjectURL(file)`, un blob local) no depende de la URL persistida.
- Principio V (nada de refactors oportunistas) y VI (incremento mínimo): meter cambios de frontend "ya que estamos" (p. ej. guardar `presign.key` en vez de `presign.public_url`) no lo exige ningún FR/SC y ampliaría el blast radius a dos repos. Si más adelante se quiere limpiar ese round-trip, es una spec propia.

**Alternatives considered**:
- *Cambiar los 3 helpers de subida del frontend para devolver/guardar `presign.key`*: técnicamente trivial, pero sin FR que lo pida, con un matiz de preview a cuidar (el `<img>` del formulario mostraría la key desnuda entre el fin de la subida y la navegación), y a costa de un cambio en `pos-heladeria`. Se deja fuera; la garantía de SC-001 la da el servidor.

---

## 10. Borrado del objeto anterior: se preserva A-44 y se amplía a "acepta key directa"

**Decision**: `products/service.py::update_product` y `tenant/router.py::update_tenant` conservan su estructura (comparar → asignar → tras `commit`, `delete_object`). Cambios:
- `key_from_public_url(old_*_url)` → `object_key_for_deletion(old_*_url)`: ahora devuelve la key tanto si `old` era una URL del bucket gestionado (dominio viejo o nuevo) como si `old` ya era una key directa; devuelve `None` si `old` es de otro origen → **no se borra** (FR-013).
- La comparación "¿cambió?" es key vs key gracias a §3, así que el borrado solo corre en un cambio real (FR-011/FR-012, US3).
- Orden `commit` → `delete_object` y carácter best-effort: **intactos** (A-44 / spec 021).
- Métodos de pago: siguen **sin** borrar el QR anterior (comportamiento actual; FR-011 solo menciona producto y logo).

**Rationale**: US3 exige que la garantía de A-44 sobreviva al cambio de cómo se referencia la imagen. La ampliación de `object_key_for_deletion` es lo mínimo para que el borrado funcione cuando `old` ya es una key (caso normal tras la migración) y para que ignore otros orígenes (FR-013).

**Verificación**: los 3 tests A-44 existentes (2 sin tocar, 1 con la aserción del valor actualizada — Constitution Check fila III) + tests nuevos en `test_products_image_key.py` / `test_tenant_logo_key.py`: "reemplazo con `old` = URL vieja borra la key correcta", "reemplazo con `old` = key borra esa key", "reemplazo con `old` = otro origen no borra nada", "guardado sin cambio de imagen no borra".

---

## 11. Cambio de aserción en `test_products_service.py` (única desviación de un test protegido)

**Decision**: en `test_a44_fallo_de_delete_object_no_revierte_el_cambio_de_imagen`, cambiar:

```python
self.assertEqual(result.image_url, NEW_URL)
```

por:

```python
# spec 080 (A-73): en base de datos vive la key, no la URL. El comportamiento que este
# test congela —un fallo de delete_object no revierte el cambio ya persistido— se mantiene.
self.assertEqual(result.image_url, "tenant/products/new.jpg")  # key de NEW_URL
```

En el mismo commit: cita a `specs/080-imagenes-key-relativa-r2/` y a la entrada `A-73`, y una línea confirmando que los otros dos tests A-44 siguen intactos y en verde (evidencia de que ningún otro comportamiento protegido se degrada — Principio III punto 4).

**Rationale**: `NEW_URL = "https://example.invalid/tenant/products/new.jpg"` y en test `R2_PUBLIC_BASE_URL = "https://example.invalid"` → `NEW_URL` es una URL del bucket gestionado → tras esta spec se persiste su key. La aserción medía "el cambio persistió pese al fallo de borrado" usando la URL como proxy; el proxy correcto ahora es la key. El comportamiento congelado (best-effort, no revierte) no cambia.

**Alternatives considered**:
- *Hacer que la normalización respete `example.invalid` como "no gestionado" en test*: descartado — falsearía el test; `example.invalid` **es** el `R2_PUBLIC_BASE_URL` de test a propósito, para ejercitar el camino del bucket gestionado.
- *No normalizar en escritura y dejar la URL*: descartado — viola FR-002/FR-004.

---

## 12. Matriz de verificación

| Qué | Cómo | FR / SC |
|---|---|---|
| Los 3 helpers, todas las formas de valor | `test_storage_asset_refs.py` (unit, sin BD) — tabla de `data-model.md` §3 | FR-001..FR-005, FR-010 |
| `asset_display_url` usa `ASSETS_BASE_URL` dinámicamente: cambiar el setting reapunta la URL sin tocar datos | `test_storage_asset_refs.py::cambiar_assets_base_url_reapunta` | FR-008, SC-005 |
| `/uploads/presign` devuelve key + `public_url` con dominio nuevo | `test_uploads_presign_key.py` | FR-015, FR-016 |
| Subir imagen de producto persiste key (forma vieja y nueva de entrada) | `test_products_image_key.py` | FR-001, FR-002, FR-004, SC-001 |
| Guardar producto sin tocar la imagen **no** llama `delete_object` | `test_products_image_key.py::guardar_sin_tocar_no_borra` | FR-012, SC-004 |
| Reemplazo de imagen borra la key correcta (old = URL vieja / old = key); old = otro origen no borra | `test_products_image_key.py`, `test_tenant_logo_key.py` | FR-011, FR-013, SC-004 |
| Logo persiste key; `GET /tenant` y `GET /menu` devuelven URL absoluta contra `assets.skeilopos.com` | `test_tenant_logo_key.py` | FR-001, FR-005, FR-007 |
| `payment_info["qr"]` persiste key; `DinerPaymentMethod` y `PaymentMethodResponse` lo devuelven ensamblado; `celular`/`cuenta` intactos | `test_payment_methods_qr_key.py` | FR-001, FR-005, FR-007 |
| `ProductResponse` / `MenuProductResponse` devuelven URL absoluta; la columna sigue con key | `test_products_image_key.py` | FR-006, FR-007 |
| Migración: idempotente (2 corridas ≡ 1) | `test_migrate_image_keys_relative.py` | FR-009c, SC-009 |
| Migración: reversión exacta (migrar + revert ≡ estado previo) | `test_migrate_image_keys_relative.py` | FR-009a, SC-008 |
| Migración: "otro origen", filas ya-key, vacías y comprobantes intactos | `test_migrate_image_keys_relative.py` | FR-010, FR-014, SC-007 |
| Tolerancia de lectura: fila con URL vieja se sigue viendo | `test_tenant_logo_key.py::fila_no_migrada_se_ve_igual` | FR-009b |
| A-44 preservada | `test_products_service.py` (2 tests intactos + 1 aserción actualizada) | US3 |
| Comprobantes sin cambios | `test_cart_payment_attempts.py` (intacto, en verde) + `quickstart.md` | FR-014 |
| Sin regresiones | `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` completo | X |

---

## 13. Prerrequisito de proceso: entrada `A-73`

Antes de implementar la Historia 1, crear en `specs/000-reconocimiento/registro-de-anomalias.md` la entrada **`A-73 — [DECISIÓN DE NEGOCIO — spec 080]`** (formato de A-72): qué cambia (la subida de imagen de producto / logo / método de pago persiste la key relativa, no la URL pública; la URL de visualización se arma en el servidor contra `assets.skeilopos.com`), por qué (desacoplar el contenido del dominio que lo sirve; un cambio futuro de dominio = solo configuración), quién y cuándo (propietario del proyecto, 2026-09-08, en `spec.md` + las 5 aclaraciones de esa fecha), funcionalidades afectadas (subida en `products`, `tenant`, `sales`; el borrado del objeto anterior amplía su extractor de key; comprobantes explícitamente fuera). `/speckit-tasks` la ordena como tarea T00x y los commits de la Historia 1 la citan.

---

## 14. Dependencias nuevas

Ninguna (Principio IX). Se usa: boto3 + `app/core/storage.py` (ya en producción), Pydantic v2 `Annotated` + `AfterValidator` + `PlainSerializer` (ya en el proyecto vía `app/core/timezone.py`), `app/core/db.py::with_db` (ya usado por `migrate_payment_methods_catalog.py`), biblioteca estándar (`urllib`/manejo de strings) para el troceo de prefijos.
