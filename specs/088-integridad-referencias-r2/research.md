# Research: Integridad de las referencias a archivos en Cloudflare R2

Fase 0 de `/speckit-plan`. La spec no dejó ningún `NEEDS CLARIFICATION`; lo que sigue son las decisiones de diseño que la spec delegaba al plan ("el plan las confirma contra el código"), verificadas leyendo `pos-backend` y `pos-heladeria` el 2026-09-29.

## Hallazgos del código (base de las decisiones)

| Tema | Hallazgo verificado |
|---|---|
| Columnas con archivos | Solo cuatro: `products.image_url` (`String(500)`, esquema del negocio), `shared.tenants.logo_url` (`String(500)`, esquema global), `payment_methods.payment_info` (`JSONB`, esquema del negocio; la clave con `format:"image"` del catálogo, p. ej. `qr`) y `order_payment_attempts.receipt_file_url` (`String(500)`, esquema del negocio). |
| Tablas sin imágenes | `presentations`, `promotions`, `combos`, `categories`, `options`, `order_items`, `sale_items`: ninguna tiene columna de imagen (`grep` de `image`/`logo`/`_url` sobre `app/models` y `app/core/models.py`). FR-006 queda confirmado con estas cuatro fuentes. El catálogo global de métodos de pago (`payment_method_catalog.fields`) solo define qué claves son de tipo imagen; no guarda archivos. |
| Borrado hoy | `products/service.py::update_product` (l.117–122, 151) y `tenant/router.py::update_tenant` (l.62–65, 79–80): "distinto del vigente ⇒ borrar el anterior", post-commit, sin ninguna otra comprobación. `sales/service.py::update_payment_method` **no borra** el QR anterior. `cart/service.py` no borra comprobantes. |
| Validación hoy | Ninguna: `AssetRefIn` solo normaliza URL→key; no comprueba prefijo, carpeta ni existencia. `PresignRequest.folder` es un `Literal["products","logo","payment-methods"]`; el comprobante se presigna aparte (`cart/service.py`, carpeta `comprobantes`). |
| Comprobante hoy | `SubmitCartIn.receipt_file_url` y `ReceiptAttachIn.file_url` son `str` libres (máx. 500). `presign_*` devuelve `public_url=public_url_for(key)` (dominio viejo `pub-…r2.dev`); el frontend reenvía ese `public_url`. Se lee sin transformación en `DinerPaymentAttempt`, `CurrentPaymentAttemptSummary` (l.199) y `PaymentAttemptResponse` (l.256) — esta última alimenta "Pagos por confirmar". |
| Frontend hoy | `product.service.ts` envía `image_url: draft.image_url || null` en **cada** guardado (reenvía la imagen sin tocar: origen del bug de la US1). `payment-methods-page.component.ts` reenvía todo `payment_info` con la URL de visualización del QR. `uploadLogo` hace `PATCH /tenant {logo_url}` inmediato. Ningún formulario conoce hoy una "imagen base". |
| Sesiones de test | `fixtures.py` usa **SQLite en memoria** y omite las tablas `tenants`/`plans` (comentario de `make_tenant_stub`). |
| Anomalías | Última entrada del registro: **A-91** (spec 087). Corresponde **A-92** y **A-93**. |

---

## D1. Dónde vive la regla

**Decisión**: lo puro y lo que toca R2 en `app/core/storage.py` (`validate_asset_key`, `object_exists`, `deletable_key`); lo que consulta la base de datos o decide un cambio en un módulo nuevo `app/core/asset_refs.py` (`is_key_referenced`, `asset_key_lock`, `decide_image_change`, `ensure_image_exists`, `resolve_receipt_key`). Los servicios existentes (`ProductService`, `update_tenant`, `sales.service`, `cart.service`) solo invocan estas funciones.

**Razón**: la spec 080 estableció `storage.py` como "única fuente de la regla key↔URL" y sus 3 helpers tienen tests propios. Mezclar allí consultas SQLAlchemy lo volvería dependiente de modelos (importación circular: los modelos importan `core`). Un módulo aparte mantiene `storage.py` libre de base de datos.

**Alternativas**: (a) todo en `storage.py` — descartada por la importación circular y porque mezcla I/O de R2 con SQL; (b) una clase `AssetGuard` — descartada por sobreingeniería: son cinco funciones sin estado.

---

## D2. Forma de una key válida (FR-004)

**Decisión**: una key **nueva** es válida si y solo si `re.fullmatch(rf"{re.escape(schema)}/{folder}/[A-Za-z0-9][A-Za-z0-9._-]{{0,199}}", key)` **y** no contiene `..`. Se compara tal cual (sensible a mayúsculas; el esquema se escapa). `folder` ∈ {`products`, `logo`, `payment-methods`, `comprobantes`} según el campo. No se hace `strip`, `lower`, `unquote` ni `normpath`: lo que no encaja se rechaza.

**Razón**: el patrón de nombre cubre lo que genera `build_object_key` (`{uuid4().hex}.{ext}`) con margen y excluye por construcción `/`, `\`, `%`, espacios y caracteres de control (el `\n` final no pasa `fullmatch`). `..` se excluye aparte porque el carácter `.` sí está permitido en el nombre (`a.b.png`); un nombre como `..hidden` empieza por `.` y ya no encaja con `[A-Za-z0-9]` inicial, y `a..b` se rechaza por la regla explícita.

**Qué NO se valida**: (1) una **URL de otro origen** (`://` tras normalizar) — se conserva intacta (FR-005) y no se verifica ni se borra; (2) un valor **igual al vigente** en la fila (ver D6): una key histórica fuera de convención que el formulario devuelve tal cual no es una key nueva.

**Alternativas**: exigir `[0-9a-f]{32}\.(jpg|png|webp|gif)` — descartada: rompería keys históricas con otro formato de nombre si alguna se reenvía y no aporta seguridad adicional sobre el patrón elegido.

---

## D3. Verificar existencia (FR-003)

**Decisión**: `object_exists(key)` hace `head_object(Bucket, Key)` con el cliente compartido. `ClientError` con código `404`/`NoSuchKey`/`NotFound` → `False`. **Cualquier otro** error (`ClientError` 5xx/403, `EndpointConnectionError`, `ConnectTimeoutError`, `ReadTimeoutError`, `BotoCoreError`) → se lanza `StorageUnavailable`, que el llamador convierte en **503** con mensaje reintentable. `get_r2_client()` pasa a `BotoConfig(signature_version="s3v4", region_name="auto", connect_timeout=3, read_timeout=5, retries={"max_attempts": 2, "mode": "standard"})`.

**Razón**: "fallar cerrado" exige distinguir "el archivo no existe" (422, culpa del cliente) de "no pude saberlo" (503, reintentable). Los timeouts por defecto de boto3 (60 s) dejarían la petición colgada; con 3 s + 5 s y 2 intentos el peor caso es ~16 s y el frontend lo recibe como error claro. El cliente es compartido, así que `delete_object` también gana timeouts (mejora inocua: ya es best-effort y nunca levanta).

**Cambio de comportamiento observable**: ninguno para el flujo feliz. Un 403 de R2 (credenciales) pasa a ser 503 en vez de "guardar sin verificar": es el comportamiento buscado.

**Alternativas**: verificar con un `GET` público contra `assets.skeilopos.com` — descartada: depende de la caché de Cloudflare (la memoria del proyecto ya documenta que esa caché produjo 504 falsos) y de que el dominio público esté bien. Verificar con `list_objects_v2(Prefix=key)` — descartada: más lento y devuelve prefijos que coinciden parcialmente.

---

## D4. Decisión de cambio de imagen (FR-001/FR-002)

**Decisión**: una función pura `decide_image_change(*, tenant_schema, folder, sent, base_provided, base, current, is_creation) -> ImageDecision` (con `KEEP`, `IGNORE`, `APPLY`) implementa la matriz de la spec. Todos los valores entran **normalizados** con `normalize_asset_ref` (vacío→`None`, URL gestionada→key, otro origen intacto). Orden fijo:

| # | Condición (sobre valores normalizados) | Resultado |
|---|---|---|
| 1 | `sent is None` (vacío = "no tocar", Edge Case "Vaciar la imagen") | `KEEP` — no valida ni verifica nada |
| 2 | `sent == current` | `KEEP` — no es una key nueva (D6) |
| 3 | `sent` gestionado y **no** pasa `validate_asset_key` | **422** — siempre, con o sin base, aunque el formulario esté desactualizado |
| 4 | `is_creation` | `APPLY` (sin comparar base, FR-001) |
| 5 | `not base_provided` **o** `base != current` | `IGNORE` — en silencio, sin verificar existencia, sin borrar |
| 6 | `sent` de otro origen | `APPLY` sin verificar (FR-005) |
| 7 | `sent` gestionado | `APPLY` ⇒ el llamador ejecuta `ensure_image_exists(sent)` (D3) antes de escribir |

**Consecuencias verificadas contra la spec**:
- Resumen (a) de FR-002 (key válida distinta + base no coincide + archivo inexistente) → fila 5 ⇒ `IGNORE`, sin llamar a R2. Resumen (b) (base coincide + archivo inexistente) → fila 7 ⇒ `ensure_image_exists` ⇒ 422. ✔
- Clarificación "key de otro negocio con formulario desactualizado" → fila 3 va **antes** que la 5 ⇒ 422. ✔
- Clarificación "sin base y key válida distinta" → fila 5 ⇒ `IGNORE`. ✔
- Creación con imagen → fila 4 ⇒ valida (fila 3) y verifica (fila 7 del llamador). Para no duplicar lógica, en creación `decide_image_change` devuelve `APPLY` y el llamador siempre llama `ensure_image_exists` cuando la key es gestionada.
- Producto/negocio sin imagen (`current=None`) con edición legítima: el frontend envía `base=null` **explícito** ⇒ `base_provided=True`, `base == current` ⇒ fila 7. ✔ (ver D5).

**Consecuencia asumida** (informada al negocio): si el usuario, con un formulario desactualizado, sube una imagen **nueva** Z, esa Z se ignora en silencio y queda huérfana (el reporte la lista tras la ventana de gracia). Es lo que fija la aclaración "Se ignora en silencio"; la alternativa (avisar) contradice FR-002.

**Alternativas**: comparar por `updated_at`/versión de la fila — descartada: la aclaración eligió explícitamente concurrencia optimista sobre la **imagen** base, no sobre la fila, y una versión de fila haría fallar ediciones legítimas de otros campos.

---

## D5. La imagen base viaja en un campo propio, y "no enviada" ≠ `null`

**Decisión**: campos nuevos, todos opcionales, en los esquemas de **actualización** (no en los de creación):

| Endpoint | Imagen | Base |
|---|---|---|
| `PATCH /products/{id}` | `image_url` | `image_url_base` |
| `PATCH /tenant` | `logo_url` | `logo_url_base` |
| `PATCH /sales/payment-methods/{id}` | `payment_info[k]` (k con `format:"image"`) | `payment_info_base` (`dict[str,str]`) |

Los campos `*_base` usan `AssetRefIn` (normalizan URL→key) y el servicio distingue "no enviada" de "enviada como `null`" con `"image_url_base" in data.model_fields_set` (patrón ya usado en `update_product` para `variants`). Un `null` explícito significa "el formulario no vio ninguna imagen".

**Razón**: la base debe poder ser `null` de forma legítima (producto que aún no tiene imagen). Sin la distinción, un producto sin imagen no podría recibir su primera imagen. Pydantic v2 ignora campos extra por defecto, así que un frontend nuevo que envíe `*_base` a un backend viejo no rompe nada (D10).

**Alternativas**: cabecera `If-Match` con la key — descartada: `PATCH /products` mezcla campos y presentaciones, y el frontend ya serializa todo en el cuerpo; `payment_info` requeriría una cabecera por clave.

---

## D6. Resolver la imagen antes de mutar nada; una key igual a la vigente no se valida

**Decisión**: en `update_product`, `update_tenant` y `update_payment_method`, la resolución (`decide_image_change` + `ensure_image_exists`) se ejecuta **primero**, antes de asignar cualquier campo del ORM. Un 422/503 deja el registro exactamente como estaba (SC-002). Si la key enviada es igual a la vigente (tras normalizar) el resultado es `KEEP` **sin** validar forma: es lo que permite que una fila histórica con key fuera de convención (FR-004, "se conserva y se lee") siga guardándose cuando un cliente antiguo la reenvíe.

**Razón**: hoy `update_product` asigna `name`, `description`, etc. antes de la imagen; si la validación de imagen fallara después, esos cambios quedarían en la sesión (un `commit` posterior o un `refresh` los arrastraría). Resolver primero elimina esa clase de error.

---

## D7. QR: reemplazo completo de `payment_info` — algoritmo por clave

**Contexto**: `update_payment_method` asigna `method.payment_info = data.payment_info` (reemplazo total). En la UI, `removeImage` pone `''` y `buildPaymentInfo` filtra vacíos, así que **hoy el QR se puede quitar** (la clave desaparece del dict). La spec dice "quitar una imagen ya cargada queda fuera de alcance" pensando en producto/logo (`None` = no tocar); para el QR el comportamiento real es distinto.

**Decisión**: preservar el comportamiento actual del QR **cuando la edición es legítima**, y aplicar la regla nueva a la clave de imagen. Para cada `k` con `format:"image"` en `catalog.fields`:

1. `sent = normalize(payment_info.get(k))`, `current = normalize(method.payment_info.get(k))`, `base = normalize(payment_info_base.get(k))`.
2. `sent == current` → sin cambio.
3. `sent` gestionado y forma inválida → 422 (D4 fila 3).
4. `payment_info_base` no enviado **o** `base != current` → `IGNORE`: se restituye `payment_info[k] = current` (o se elimina la clave si `current` es `None`); el resto de claves (`celular`, `cuenta`…) se guardan.
5. Edición legítima con `sent` vacío/ausente → se acepta la **eliminación** (comportamiento de hoy; el archivo no se borra, D8).
6. Edición legítima con `sent` nuevo → `ensure_image_exists` (422/503) y se guarda.

**Punto para confirmación del negocio** (se registra en A-92): si el negocio prefiere que el QR también sea "vacío = no tocar" como producto y logo, es un cambio de una línea en el paso 5; se deja el comportamiento actual porque quitarlo sería una regresión visible del administrador sin spec que lo pida.

**Creación** (`create_payment_method`): sin base; cada clave de imagen gestionada se valida (carpeta `payment-methods`) y se verifica.

---

## D8. No se añade borrado del QR anterior

**Decisión**: reemplazar o quitar el QR **no** borra el archivo anterior (como hoy). Solo producto y logo borran (FR-007 dice "seguir ocurriendo").

**Razón**: añadir borrado de QR sería comportamiento nuevo sin decisión de negocio (Principio II) y la propia spec acepta huérfanos como seguros. El reporte (US6) los detecta. Si se decide añadirlo después, `deletable_key` + `is_key_referenced` ya lo soportan sin cambios de diseño. Se registra en A-92 como decisión explícita.

---

## D9. "¿Otra fila lo usa?" (FR-006) — fuentes, forma de consultar y pruebas

**Decisión**: `is_key_referenced(db, key) -> bool` comprueba, con los **candidatos** `{key} ∪ {prefijo + key para cada prefijo gestionado}` (para cubrir filas históricas con URL absoluta, `storage._managed_bucket_prefixes()`):

| Fuente | Consulta | Esquema |
|---|---|---|
| Producto | `Product.image_url IN candidatos` (`LIMIT 1`) | negocio |
| Método de pago | filas de `payment_methods` (≤ decenas por negocio), y en Python: algún valor de `payment_info` ∈ candidatos | negocio |
| Intento de pago | `OrderPaymentAttempt.receipt_file_url IN candidatos` (`LIMIT 1`) | negocio |
| Logo | `Tenant.logo_url IN candidatos` (`LIMIT 1`, **sin filtrar por tenant**: cubre el "esquema global que pueda usar ese prefijo") | global (`shared`) |

Recorrer `payment_info` en Python en vez de SQL sobre JSONB evita depender de `jsonb_each_text` (SQLite de los tests no lo tiene) y el costo es despreciable.

**Sesiones**: la sesión de negocio (`with_db(schema)`) alcanza `shared.tenants` por nombre calificado (así lo hace `with_db` con `schema_translate_map={"tenant": schema}`); por eso `update_product` pasa su propia `db`. `update_tenant` usa la sesión `shared`, que **no** alcanza las tablas del negocio: abre `with with_db(tenant.schema) as tdb:` para el chequeo (post-commit, sesión corta).

**Pruebas (SQLite)**: `fixtures.py` no crea `shared.tenants`. Se resuelve en tasks con la primera opción viable: (a) `ATTACH DATABASE ':memory:' AS shared` en la conexión del fixture + `Tenant.__table__.create`; o (b) parámetro `shared_check` inyectable en `is_key_referenced` que los tests sustituyen. **Riesgo registrado**: si (a) no es viable con `schema_translate_map`, se usa (b); la lógica de las cuatro fuentes se prueba igual porque cada consulta es independiente.

**Límite conocido**: la comparación contra URLs históricas es exacta por prefijo configurado; una URL con host en mayúsculas distintas no se reconoce como referencia. Efecto: se podría borrar un archivo aún usado **por una fila histórica con esa forma inusual**. Es un caso teórico (esas URLs las generó `public_url_for` con la configuración vigente) y el reporte lo detecta como referencia sin archivo.

---

## D10. Orden de despliegue: frontend primero, backend después

**Hallazgo**: con el backend nuevo, un frontend **antiguo** (sin `*_base`) que sube una imagen legítima la vería **ignorada en silencio** (D4 fila 5). Al revés funciona: el backend antiguo ignora los campos extra `*_base`, y un frontend que **no reenvía** la imagen sin tocar equivale a `null` = "no tocar", que el backend antiguo ya respeta.

**Decisión**: (1) desplegar `pos-heladeria` con `*_base` + no reenviar imagen sin tocar (compatible con ambos backends); (2) desplegar `pos-backend`; (3) opcional, después, que el comprobante envíe `presign.key`. Riesgo residual: pestañas abiertas con el bundle viejo durante la ventana posterior al paso 2; su subida de imagen quedaría ignorada. Mitigación: refrescar el service worker / forzar recarga en el despliegue (si el proyecto ya lo hace, sin cambios) y anotarlo en el despliegue. Es exactamente el "despliegue coordinado" que la spec reservó al plan.

**Comprobantes**: el backend acepta key **o** URL gestionada (FR-008), así que el frontend actual (que reenvía `public_url`) sigue funcionando sin cambios; el paso 3 es una limpieza opcional que no bloquea nada. Importante: un frontend que enviara **key** a un backend **anterior** guardaría la key cruda y el cajero vería una imagen rota — por eso el paso 3 va **después** del backend.

---

## D11. Comprobantes: key en base, URL al responder, hash de auditoría sobre la key

**Decisión**:
- `resolve_receipt_key(value, tenant_schema)`: `normalize_asset_ref` → si queda una URL de otro origen (`://`) → 422 "El comprobante debe ser un archivo subido desde la aplicación."; `validate_asset_key(..., folder="comprobantes")` → 422; luego `object_exists` → 422/503. Devuelve la key.
- Puntos de entrada: `submit_cart(receipt_file_url)` (solo métodos no efectivo; se ejecuta **después** de los chequeos 422 actuales de efectivo/obligatorio y **antes** de `check_availability` y de crear nada) y `attach_receipt(file_url)` (después del 409 "ya tiene comprobante"; el 409 se mantiene primero, US5-5). El router pasa `ctx.tenant.schema`.
- `DinerPaymentAttempt`, `CurrentPaymentAttemptSummary` y `PaymentAttemptResponse` declaran `receipt_file_url: AssetUrl`. El cajero (`PaymentAttemptResponse`) y la vista previa "Pagos por confirmar" reciben URL absoluta ensamblada; una fila histórica con `pub-…r2.dev` se reescribe al dominio de assets al vuelo (tolerancia de lectura, igual que 080); una URL de otro origen sale intacta.
- **Auditoría (spec 074)**: los eventos `payment_attempt_created`, `transfer_approved` y `transfer_rejected` comparten `receipt_hash` (HMAC del valor guardado; ver `test_order_audit_log.py` ~681). Para conservar esa igualdad **dentro de un intento**, `submit_cart`/`attach_receipt` hashean la **key resuelta** (lo que se persiste) y no el valor que llegó. Los eventos ya emitidos con hash de URL no se reescriben (Principio VII); un intento migrado a key cuyos eventos previos hashearon la URL verá hashes distintos entre eventos viejos y nuevos — aceptable: el hash es un identificador de correlación de un mismo evento de vida corta (pendiente→resuelto), no un identificador estable entre sistemas.

**Garantía limitada** (spec, FR-008/Assumptions): el comensal no tiene sesión, así que solo se garantiza prefijo del negocio + carpeta + existencia; no se garantiza quién subió el archivo.

**Alternativa descartada**: bindear la key al `participant_id` en el presign (p. ej. `comprobantes/{participant}/…`) — cambia la convención de carpetas, que la spec deja explícitamente fuera.

---

## D12. Matriz de verificación (Principio X)

| Historia / criterio | Prueba |
|---|---|
| US1 / SC-001 | Tabla de verdad de `decide_image_change` (las 7 filas de D4 × creación/edición × base presente/ausente/`null`); `update_product`, `update_tenant`, `update_payment_method` con formulario desactualizado: otros campos se guardan, imagen vigente intacta, `delete_object` **no** llamado, respuesta sin error. |
| US2 / SC-002 | Edición legítima + `object_exists → False` ⇒ 422 y registro sin cambios; `object_exists` lanza `StorageUnavailable` ⇒ 503, sin `delete_object` ni cambios; valor vacío ⇒ `KEEP` sin llamar a R2; URL de otro origen ⇒ no se llama a R2. |
| US3 / SC-003 | `validate_asset_key`: otro negocio, otra carpeta, `..`, `//`, `\`, `\x00`, mayúsculas distintas, `%2e%2e`, sin nombre ⇒ 422 con y sin base y con formulario desactualizado; ningún `delete_object`; key histórica fuera de convención: se reemplaza pero `deletable_key` devuelve `None`; URL de otro origen: `deletable_key` devuelve `None`. |
| US4 / SC-004, SC-005 | Dos productos con la misma key: reemplazar uno no borra; reemplazar el segundo borra (`delete_object` con la key); un solo uso ⇒ borra (sin regresión); fallo de `delete_object` ⇒ cambio persistido y sin error (A-44); referencia por logo, por QR y por comprobante también impide el borrado; dos actualizaciones concurrentes: cada una borra solo su `old_key`. |
| US5 / SC-006, SC-009 | Comprobante: key válida, URL del dominio viejo, URL del dominio de assets ⇒ se guarda solo la key; inventada/ajena/otra carpeta/otro origen ⇒ 422; 409 se mantiene; `PaymentAttemptResponse`/`DinerPaymentAttempt`/`CurrentPaymentAttemptSummary` devuelven URL; fila histórica con URL absoluta se sigue leyendo. Migración: simulación no modifica, aplicación reescribe, segunda corrida = 0 cambios, filas de otro origen o con archivo inexistente intactas y reportadas sin fallar, `--revert` restituye. |
| US6 / SC-007 | Con R2 simulado: referencia sin archivo, huérfano viejo listado, huérfano reciente (< ventana) no listado, key fuera de convención marcada, `--tenant` acota; base de datos y R2 idénticos tras ejecutar (se verifica que solo se llamó a `list_objects_v2`/`head_object`, nunca a `delete_object`/`put_object`, y que la sesión no hizo `commit`). |
| SC-008 | Corrida completa de `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` en verde + recorrido manual de `quickstart.md` §Regresión (producto, logo, QR y comprobante existentes se siguen viendo). |
| Frontend | `.spec.ts` de `product.service`, `tenant-info.service`, `payment-method.service`: el payload trae `*_base` y **no** trae la imagen si no cambió; con imagen nueva trae ambas. |

---

## D13. Candado consultivo por key (cierra la carrera guardar ↔ borrar)

**Problema**: entre "nadie usa K" (chequeo del borrado) y `delete_object(K)`, otra petición puede empezar a referenciar K (tras verificar que existe) ⇒ referencia rota causada por el sistema. Es ventana estrecha pero real, y la spec exige "nunca".

**Decisión**: `asset_key_lock(db, key)` ejecuta `SELECT pg_advisory_xact_lock(hashtextextended(:key, 0))` **solo** si `db.bind.dialect.name == "postgresql"`. Se toma:
- al **guardar** una key nueva: antes de `object_exists(nueva)` y se conserva hasta el `commit` de la transacción de la petición (cubre verificar + escribir);
- al **borrar**: en una transacción corta nueva, antes de `is_key_referenced(vieja)` y hasta después de `delete_object(vieja)`.

Con esto, o el guardado de K termina primero (el borrado ve la referencia y no borra) o el borrado termina primero (el guardado verifica después, K no existe ⇒ 422). El candado es por key (`xact` lo libera solo al terminar la transacción), sin tablas nuevas y sin contención salvo sobre la misma key.

**Riesgo**: mantener el candado mientras se hace el `HEAD` de R2 (≤ ~16 s peor caso). Solo bloquea a quien toque **esa misma key**, y un guardado con key nueva es exactamente lo que el flujo normal hace una sola vez. Aceptable.

**Alternativas**: (a) aceptar la carrera ("último en escribir gana", literalmente lo que dice FR-007 para actualizaciones del **mismo producto**) — se reserva como plan B si el candado resultara problemático en pruebas; no cubre el caso de dos filas distintas que comparten la key. (b) `SELECT … FOR UPDATE` sobre la fila — no cubre filas de otras tablas ni el logo (esquema global).

---

## D14. Reporte de reconciliación (FR-009)

**Decisión**: `python -m app.scripts.reconcile_r2_references [--tenant SCHEMA] [--grace-hours 24] [--output RUTA] [--format csv|json]`.

1. Recorre `shared.tenants` (`with_db(None)`) y, por cada esquema (o solo `--tenant`), abre `with_db(schema)` **en modo lectura** (`SET TRANSACTION READ ONLY` y `rollback()` al final; nunca `commit`).
2. Reúne las referencias: `products.image_url`, valores `format:"image"` de `payment_methods.payment_info` (según `catalog.fields`), `order_payment_attempts.receipt_file_url` y `shared.tenants.logo_url`. Cada una se normaliza con `normalize_asset_ref`; las de otro origen se cuentan aparte y no se verifican (FR-005).
3. **Un solo listado** del bucket con `list_objects_v2` paginado (una vez, sin filtrar por prefijo salvo `--tenant`), guardando `Key → LastModified`. Un bucket de este tamaño se lista en pocas páginas; es más simple y barato que un `HEAD` por referencia.
4. Clasifica por negocio (primer segmento de la key = esquema; el resto va a "sin negocio"):
   - **Referencia sin archivo**: referencia gestionada cuya key no está en el listado (para las pocas keys fuera del prefijo listado por `--tenant`, se confirma con `head_object`).
   - **Archivo sin referencia**: objeto del listado que ninguna referencia de **ningún** negocio menciona **y** cuyo `LastModified` es anterior a `now − grace_hours` (FR-009, ventana de gracia).
   - **Fuera de convención**: referencia gestionada cuya key no pasa `validate_asset_key` con el esquema/carpeta esperados del campo.
5. Imprime un resumen legible agrupado por negocio; con `--output` escribe además CSV (`negocio,tipo,campo,key,detalle`) o JSON y **imprime la ruta** escrita.

**Solo lectura verificable**: el script importa únicamente `list_objects_v2`/`head_object`; no importa `delete_object`. El test lo comprueba parcheando `get_r2_client` con un doble que falla si se llama a cualquier método de escritura.

---

## D15. Migración de comprobantes (FR-008)

**Decisión**: `python -m app.scripts.migrate_receipt_keys [--apply] [--revert] [--tenant SCHEMA]`. **Sin argumentos = simulación** (informa qué reescribiría, no modifica). Por cada esquema y por cada `order_payment_attempts` con `receipt_file_url` no nulo:

- valor ya es key (no empieza por prefijo gestionado ni contiene `://`) → **sin cambio** (idempotencia);
- URL absoluta del bucket gestionado → se extrae la key; si la key **pasa** `validate_asset_key(schema, "comprobantes")` **y** `object_exists` → se reescribe a la key; si no pasa la forma o no existe → **intacta**, se lista en el reporte ("fuera de convención" / "archivo inexistente") y **no** hace fallar el script;
- URL de otro origen → intacta y reportada;
- el almacenamiento no responde al verificar → intacta, reportada como "no verificable"; el script continúa con las demás filas (una segunda corrida las retoma, es idempotente).

`commit` por esquema (mismo patrón que 080), re-ejecutable con la app en marcha. `--revert` reconstruye `{R2_PUBLIC_BASE_URL}/{key}` solo para valores que son key de **este** esquema y carpeta `comprobantes` (nunca toca `celular`/URL de otro origen); es la estrategia de reversión de datos del Principio VIII, necesaria si se retira el backend nuevo (el código anterior mostraría la key cruda como imagen rota). Como el código nuevo lee ambas formas, **no hay orden obligatorio entre desplegar el backend y ejecutar la migración**; se recomienda backend → simulación → aplicación.

**Por qué verificar existencia en la migración**: FR-008 exige que las filas que apuntan a un archivo inexistente queden intactas y reportadas — migrarlas a key las haría indistinguibles de una key válida rota.
