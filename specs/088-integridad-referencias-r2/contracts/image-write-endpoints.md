# Contrato: escritura de imágenes de producto, logo y método de pago

Aplica a `PATCH /products/{id}`, `POST /products`, `PATCH /tenant`, `POST /sales/payment-methods` y `PATCH /sales/payment-methods/{id}`. **Las respuestas no cambian** (siguen devolviendo la URL de visualización ensamblada, spec 080). `POST /uploads/presign` no cambia.

## Campos nuevos de petición (todos opcionales)

| Endpoint | Campo nuevo | Tipo | Significado |
|---|---|---|---|
| `PATCH /products/{id}` | `image_url_base` | `string \| null` | Imagen que el formulario mostraba al abrirse. |
| `PATCH /tenant` | `logo_url_base` | `string \| null` | Logo que el formulario mostraba al abrirse. |
| `PATCH /sales/payment-methods/{id}` | `payment_info_base` | `object<string,string>` | `payment_info` que el formulario mostraba al abrirse (basta con las claves de imagen). |

- Aceptan key, URL absoluta del bucket gestionado (se normaliza a key) o `null`. **No** existen en `POST` (en una creación no hay imagen vigente, FR-001).
- **"No enviado" ≠ `null`**: `null` explícito = "el formulario no vio ninguna imagen" (producto sin imagen); campo ausente = cliente que no puede declarar un cambio legítimo (research D5).
- Un backend anterior ignora estos campos (Pydantic descarta los extra), lo que permite desplegar el frontend primero (research D10).

## `PATCH /products/{id}`

```json
{ "name": "Cono doble", "image_url": "acme/products/9c1e….png", "image_url_base": "https://assets.skeilopos.com/acme/products/46f1….png" }
```

Resolución de `image_url` (antes de tocar ningún otro campo, research D6):

| Petición | Resultado |
|---|---|
| sin `image_url` o `image_url: null`/`""` | no se toca la imagen (sin verificar nada) |
| `image_url` = imagen vigente | no se toca (aunque la key sea histórica/fuera de convención) |
| key de otro negocio / otra carpeta / mal formada | **422**, sin base o con base de cualquier valor |
| key válida distinta + **sin** `image_url_base` | otros campos se guardan, imagen vigente intacta, **200 sin aviso** |
| key válida distinta + base **≠** vigente | ídem (formulario desactualizado) — sin verificar existencia, sin borrar nada |
| key válida distinta + base **=** vigente + archivo inexistente | **422** "La imagen no se encontró…" |
| key válida distinta + base = vigente + R2 no responde | **503**, nada cambia |
| key válida distinta + base = vigente + archivo existe | se guarda; **después del commit** se borra la anterior si `deletable_key` la devuelve **y** `is_key_referenced` es falso |
| URL de otro origen + base = vigente | se guarda sin verificar; la anterior se borra según la misma regla |

Errores existentes (404 producto, 403 plan de inventario, 409 variantes) sin cambios.

## `POST /products`

`image_url` se valida (carpeta `products`, FR-004) y se verifica (FR-003). Sin base. Key inválida/ajena → 422; inexistente → 422; R2 caído → 503. URL de otro origen o sin imagen → como hoy.

## `PATCH /tenant`

Mismas reglas que producto con `logo_url` / `logo_url_base` y carpeta `logo`. `receipt_message` e `invoice_prefix` se guardan siempre, también con un `logo_url` ignorado. Para decidir el borrado, `update_tenant` abre una sesión del negocio (`with_db(tenant.schema)`) porque su sesión `shared` no alcanza las tablas del negocio.

## `POST /sales/payment-methods` y `PATCH /sales/payment-methods/{id}`

Se evalúa **cada clave** de `payment_info` cuyo `catalog.fields[*].format == "image"` (research D7). Carpeta `payment-methods`.

| Caso por clave de imagen `k` | Resultado |
|---|---|
| `payment_info[k]` = valor vigente | sin cambio |
| forma inválida / de otro negocio / otra carpeta | **422** |
| `payment_info_base` ausente o `base[k]` ≠ vigente | se conserva el valor vigente de `k`; las demás claves (`celular`, `cuenta`…) se guardan |
| edición legítima, `payment_info[k]` ausente o vacío | `k` se elimina (comportamiento de hoy; el archivo **no** se borra, research D8) |
| edición legítima, valor nuevo inexistente | **422** |
| edición legítima, valor nuevo, R2 no responde | **503** |
| edición legítima, valor nuevo existente | se guarda |

`is_complete` se recalcula sobre el `payment_info` **resultante** (con la imagen conservada si se ignoró la enviada). `POST` (activar): cada clave de imagen se valida y verifica, sin base.

## Sin efecto en

Cálculo de montos, ventas, facturas, inventario, `POST /uploads/presign` y la forma de las respuestas.

## Compatibilidad con clientes

| Cliente | Efecto |
|---|---|
| Frontend nuevo → backend anterior | Los `*_base` se ignoran. En producto y logo, no reenviar la imagen sin tocar equivale a "no tocar". En el QR no aplica: se sigue enviando `payment_info` completo. Sin efecto negativo. |
| Frontend nuevo → backend nuevo | Comportamiento completo. |
| Frontend **anterior** → backend nuevo | Reenvía la imagen sin tocar ⇒ `KEEP` (igual a la vigente). Una subida legítima sin base ⇒ **ignorada en silencio** (por eso el frontend se despliega primero, research D10). |
| Llamada manual/API sin base | Un cambio legítimo exige enviar la base (Assumptions de la spec). |
