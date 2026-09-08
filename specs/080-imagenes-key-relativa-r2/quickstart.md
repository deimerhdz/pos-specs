# Quickstart — Validación: imágenes como key relativa servidas por dominio personalizado

Guía para verificar que la funcionalidad cumple el spec de punta a punta. El contrato de los helpers está en [`contracts/asset-reference.md`](./contracts/asset-reference.md); el del endpoint de subida, en [`contracts/uploads-presign.md`](./contracts/uploads-presign.md); el de la migración, en [`contracts/data-migration.md`](./contracts/data-migration.md); las formas de valor y las tablas de verdad, en [`data-model.md`](./data-model.md).

## Prerrequisitos

- `pos-backend` en local con:
  - `ASSETS_BASE_URL=https://assets.skeilopos.com` en `.env` (o el dominio del entorno de pruebas).
  - `R2_PUBLIC_BASE_URL` **sin cambiar** (dominio `pub-…r2.dev`).
  - Un tenant de pruebas con: ≥ 1 producto con foto cargada **antes** del cambio (campo con `https://pub-…r2.dev/...`), el logo del negocio cargado antes del cambio, un método de pago con imagen QR (p. ej. Nequi) cargado antes del cambio, y —para el caso FR-010— **al menos un** producto cuya `image_url` sea una URL de otro origen (p. ej. `https://xyz.supabase.co/...`, insertada a mano).
- `pos-heladeria` en local apuntando a ese backend. **No se modifica**; solo se usa para comprobar que las imágenes se siguen viendo.
- Entrada **`A-73`** ya creada en `specs/000-reconocimiento/registro-de-anomalias.md` (prerrequisito de la Historia 1 — Principio II).

## Comandos de verificación automática

```bash
# Backend — desde pos-backend/
python -m unittest discover -s app/characterization_tests -p 'test_*.py'
#   Debe pasar TODO, incluido:
#   - test_cart_payment_attempts.py SIN modificar (comprobantes, FR-014)
#   - test_products_service.py con las 2 pruebas A-44 intactas y 1 aserción actualizada (spec 080 / A-73)
#   - los nuevos: test_storage_asset_refs, test_uploads_presign_key, test_products_image_key,
#     test_tenant_logo_key, test_payment_methods_qr_key, test_migrate_image_keys_relative
```

`pos-heladeria` no tiene specs nuevos (no se toca); su suite (`npm test -- --watch=false`) debe seguir en verde sin cambios.

---

## Historia 1 — Subir una imagen y verla con el nuevo esquema (P1)

Prerrequisito: `A-73` registrada.

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 1.1 | En el panel, editar un producto, subir una foto nueva, guardar. Inspeccionar `SELECT image_url FROM products WHERE id = ...` | El campo contiene **solo la key** `{tenant}/products/{uuid}.{ext}` — sin `http`, sin dominio | FR-001, FR-002, SC-001 |
| 1.2 | `GET /products/{id}` y `GET /menu` | En la respuesta, `image_url` es `https://assets.skeilopos.com/{tenant}/products/{uuid}.{ext}` | FR-005, FR-007 |
| 1.3 | Abrir el menú QR y el catálogo del POS | La foto se ve, servida desde `assets.skeilopos.com` (comprobar en la pestaña Red del navegador). Nunca desde `pub-…r2.dev` | FR-005, SC-002, SC-006 |
| 1.4 | Subir un logo nuevo en "Información del negocio", guardar. Inspeccionar `SELECT logo_url FROM shared.tenants WHERE id = ...` | El campo es `{tenant}/logo/{uuid}.{ext}` (key). `GET /tenant` lo devuelve como URL absoluta contra `assets.skeilopos.com`. El logo aparece en la barra lateral y al generar un recibo | FR-001, FR-005 |
| 1.5 | Editar el método de pago con QR, subir una imagen nueva, guardar. Inspeccionar el JSON de `payment_info` | El valor de la clave de imagen (`qr`) es `{tenant}/payment-methods/{uuid}.{ext}` (key). Los demás campos (`celular`, `cuenta`) intactos | FR-001, FR-002 |
| 1.6 | Como comensal, llegar al paso de pago por transferencia de ese método (`GET /cart/payment-methods`) | El comensal ve el QR; en la respuesta el valor viene como `https://assets.skeilopos.com/{tenant}/payment-methods/{uuid}.{ext}` | FR-005, FR-007 |
| 1.7 | Guardar el formulario de un producto/logo/método **sin tocar la imagen** (el cliente reenvía la URL `https://assets.skeilopos.com/{key}` que recibió). Inspeccionar el campo | Sigue siendo la **key**, nunca una URL absoluta | FR-004, escenario 6 |
| 1.8 | Guardar un producto/negocio/método **sin imagen** | El campo queda `NULL`/vacío; no se arma ninguna URL en la respuesta | escenario 5 |
| 1.9 | `POST /uploads/presign` con `folder: "products"` | Respuesta trae `key` y `public_url` = `https://assets.skeilopos.com/{key}` | [`contracts/uploads-presign.md`](./contracts/uploads-presign.md) |

---

## Historia 2 — Las imágenes ya cargadas siguen viéndose + migración (P2)

Prerrequisito: código de la Historia 1 desplegado.

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 2.1 | **Antes** de migrar: abrir el menú QR / POS / recibo / checkout con los datos sembrados (todos con URL `pub-…r2.dev`) | Todas las imágenes se ven — servidas desde `assets.skeilopos.com` (tolerancia de lectura: `asset_display_url` reescribe el prefijo viejo al vuelo) | FR-009b, SC-002, SC-003 |
| 2.2 | `python -m app.scripts.migrate_image_keys_relative --report-only` | Reporta, por tenant y campo, cuántas filas se migrarían; no escribe nada | [`contracts/data-migration.md`](./contracts/data-migration.md) G-reporte |
| 2.3 | `python -m app.scripts.migrate_image_keys_relative` | Las filas sembradas quedan como key. `SELECT count(*) FROM products WHERE image_url LIKE 'https://pub-%'` → `0` en todos los schemas; idem `logo_url` en `shared.tenants` y los valores de imagen de `payment_info` | FR-009, SC-007 |
| 2.4 | Repetir 2.1 tras migrar | Todas las imágenes siguen viéndose desde `assets.skeilopos.com`; ningún recurso apunta a `pub-…r2.dev` | SC-002, SC-003, SC-006 |
| 2.5 | El producto sembrado con URL **de otro origen** | Su `image_url` quedó **intacta** (misma URL de otro origen) y la imagen carga tal cual | FR-010, SC-007 |
| 2.6 | Volver a correr `python -m app.scripts.migrate_image_keys_relative` | Reporta `migradas=0` para todos los campos; ninguna fila cambia | FR-009c, SC-009 |
| 2.7 | Simular interrupción: correr la migración, cortarla a mitad (Ctrl-C entre schemas), relanzarla | Converge al mismo estado que una corrida completa; filas ya convertidas sin cambio | FR-009c, escenario 6 |
| 2.8 | `python -m app.scripts.migrate_image_keys_relative --revert` y comparar con un dump previo de los tres campos | Los tres campos quedan **exactamente** como antes de 2.3 (URLs `pub-…r2.dev` reconstruidas); "otro origen" y vacíos sin cambio | FR-009a, SC-008 |
| 2.9 | (Tras `--revert`, volver a `migrar` para dejar el entorno migrado) | — | — |

---

## Historia 3 — Reemplazar una imagen sigue limpiando la anterior (P3)

| # | Pasos | Resultado esperado | FR / SC |
|---|---|---|---|
| 3.1 | Producto con imagen previa **como key** (ya migrado): subir una imagen nueva, confirmar el guardado | El objeto anterior deja de existir en R2; el producto apunta a la nueva key | FR-011, SC-004 |
| 3.2 | Producto con imagen previa **como URL `pub-…r2.dev`** (no migrado): subir nueva, confirmar | El objeto anterior se borra (la key se extrae de la URL vieja); el producto queda con la nueva key | FR-011 |
| 3.3 | Producto con imagen previa **de otro origen**: subir nueva, confirmar | **No** se intenta borrar nada de ese otro origen; el producto queda con la nueva key | FR-013, escenario US3-4 |
| 3.4 | Forzar un fallo al guardar (p. ej. validación de otra parte del formulario) tras elegir imagen nueva | El objeto anterior **no** se borra; el producto conserva su imagen anterior intacta (A-44 / spec 021) | FR-012, SC-004 |
| 3.5 | Guardar un producto **sin cambiar** la imagen (el form reenvía `https://assets.skeilopos.com/{key}`) | `delete_object` **no** se invoca (la comparación es key vs key tras normalizar) | FR-012, [`data-model.md`](./data-model.md) §4 |
| 3.6 | Repetir 3.1–3.5 con el logo del negocio | Mismo comportamiento | FR-011, FR-012, FR-013 |

---

## Comprobantes del comensal — fuera de alcance (FR-014)

| # | Pasos | Resultado esperado |
|---|---|---|
| C.1 | Como comensal, subir un comprobante de pago por transferencia y enviar el pedido. Inspeccionar `receipt_file_url` del intento de pago | Sigue siendo una **URL completa** contra el dominio anterior (`pub-…r2.dev/{tenant}/comprobantes/...`) — sin cambios |
| C.2 | Ver el comprobante desde el panel del cajero ("Pagos por confirmar") | Se ve igual que antes del cambio |
| C.3 | `python -m app.scripts.migrate_image_keys_relative` y volver a inspeccionar `receipt_file_url` | **Sin cambios** — la migración no toca ese campo |

---

## Cierre — criterios de éxito del spec

- [ ] SC-001: 100 % de imágenes subidas tras el cambio quedan como key (verificado 1.1, 1.4, 1.5)
- [ ] SC-002 / SC-003: 100 % de imágenes previas se siguen viendo, cero rotas (2.1, 2.4)
- [ ] SC-004: reemplazo deja exactamente un objeto vigente y ninguna referencia rota; A-44 intacta (3.1–3.5)
- [ ] SC-005: cambiar `ASSETS_BASE_URL` cambia el dominio de todas las imágenes sin migración ni edición de datos (cambiar el valor, reiniciar, recargar el menú)
- [ ] SC-006: ninguna pantalla cambia su comportamiento visible más allá del dominio de origen
- [ ] SC-007: tras migrar, 100 % de filas V2/V3 → key; "otro origen" y comprobantes sin cambios (2.3, 2.5, C.3)
- [ ] SC-008: migración reversible (2.8)
- [ ] SC-009: re-ejecutar la migración no cambia nada (2.6, 2.7)
- [ ] Suite completa de `pos-backend` en verde; suite de `pos-heladeria` en verde sin cambios
