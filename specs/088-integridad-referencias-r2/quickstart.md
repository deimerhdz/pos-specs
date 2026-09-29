# Quickstart: Integridad de las referencias a archivos en Cloudflare R2

**Spec**: [spec.md](./spec.md) | **Contratos**: [contracts/](./contracts/) | **Datos**: [data-model.md](./data-model.md) | **Decisiones**: [research.md](./research.md)

Guía de validación de extremo a extremo por historia de usuario. No sustituye los tests automatizados (`tasks.md`); sirve para confirmar en un entorno real que el comportamiento observable coincide con los criterios de aceptación de `spec.md`.

## Prerrequisitos

1. `pos-backend` en marcha (`docker compose up -d`) con un **bucket de pruebas de R2** (o el de desarrollo) y sus variables `R2_*` / `ASSETS_BASE_URL` ya configuradas. **No hay variables nuevas.** No hay migración de Alembic.
2. `pos-heladeria` en marcha (`npm start`) apuntando a ese backend.
3. Un negocio de prueba `acme` y otro `globex` (dos esquemas), un administrador de cada uno, un cajero de `acme`, un producto "Cono doble" con imagen, un método de pago de transferencia con QR y una mesa con sesión de comensal por QR.
4. Ramas creadas según el Principio XIV (`feat/088-r2-reference-integrity`) en ambos repos antes de tocar código.
5. Un token de administrador de cada negocio (`$TOKEN_ACME`, `$TOKEN_GLOBEX`) y sus cabeceras de tenant. Ayuda para subir un archivo real por API:

```bash
# 1) pedir presign (imagen de producto de acme) y subir un archivo
curl -s -X POST "$API/uploads/presign" -H "Authorization: Bearer $TOKEN_ACME" -H "x-tenant-host: acme.$DOMAIN" \
  -H 'Content-Type: application/json' -d '{"filename":"a.png","content_type":"image/png","folder":"products"}' > /tmp/presign.json
KEY=$(jq -r .key /tmp/presign.json); curl -s -X PUT "$(jq -r .upload_url /tmp/presign.json)" -H 'Content-Type: image/png' --data-binary @a.png
echo "$KEY"   # acme/products/<uuid>.png
```

## Verificación automatizada (backend)

```bash
cd pos-backend
python -m unittest app.characterization_tests.test_storage_asset_guard -v
python -m unittest app.characterization_tests.test_asset_refs -v
python -m unittest app.characterization_tests.test_products_image_integrity -v
python -m unittest app.characterization_tests.test_tenant_logo_integrity -v
python -m unittest app.characterization_tests.test_payment_methods_qr_integrity -v
python -m unittest app.characterization_tests.test_receipts_integrity -v
python -m unittest app.characterization_tests.test_reconcile_r2_references -v
python -m unittest app.characterization_tests.test_migrate_receipt_keys -v
# Regresión completa (SC-008): toda la suite, incluidos los tests CONGELA
python -m unittest discover -s app.characterization_tests -p 'test_*.py'
```

```bash
cd pos-heladeria && ng test --watch=false --include='**/{product,tenant-info,payment-method}.service.spec.ts'
```

---

## Historia 1 — Un formulario desactualizado no deshace ni borra la imagen vigente (P1)

1. Abre "Cono doble" en **dos pestañas** (A y B) del panel de administración.
2. En A sube una imagen nueva `Y` y guarda. Anota la key `Y` (visible en la respuesta o con `GET /products/{id}`).
3. En B (formulario viejo) cambia **solo el nombre** y guarda.
   **Esperado**: el nombre se guardó; el producto sigue mostrando `Y` en la pantalla de productos y en el menú del comensal; `Y` abre en `{ASSETS_BASE_URL}/{Y}`; no hubo ningún aviso ni error.
4. Repite por API con un cliente viejo (reenvía la imagen anterior `X`):

```bash
curl -s -X PATCH "$API/products/$ID" -H "Authorization: Bearer $TOKEN_ACME" -H 'Content-Type: application/json' \
  -d "{\"name\":\"Cono doble XL\",\"image_url\":\"$KEY_X\",\"image_url_base\":\"$KEY_X\"}"
```

   **Esperado**: 200; `image_url` de la respuesta sigue siendo `Y`; `X` no se restaura.
5. Sin base (cliente antiguo/manual con una key válida distinta): mismo resultado, imagen vigente intacta, 200.
6. Repite el paso 3 con el **logo** (`PATCH /tenant`, con `logo_url_base` obsoleto) y con el **QR** de un método de pago (`payment_info_base` obsoleto): se guardan los demás campos y se conserva el valor vigente.
7. Formulario al día: sube una imagen nueva desde una sola pestaña → cambia y el archivo anterior se borra (Historia 4).
8. Formulario al día sin tocar la imagen → la petición **no** trae `image_url` (revisar en la pestaña Red del navegador) y no se borra nada.

## Historia 2 — No se puede guardar una imagen que no existe (P1)

```bash
# key con la forma correcta pero que nunca se subió
curl -s -o /dev/null -w '%{http_code}\n' -X PATCH "$API/products/$ID" -H "Authorization: Bearer $TOKEN_ACME" \
  -H 'Content-Type: application/json' \
  -d "{\"image_url\":\"acme/products/00000000000000000000000000000000.png\",\"image_url_base\":\"$CURRENT\"}"
```

**Esperado**: `422` con "La imagen no se encontró en el almacenamiento. Sube el archivo de nuevo."; el producto conserva su imagen y ningún otro campo cambió. Repite con logo y QR.

- Valor vacío (`"image_url": ""` o `null`) → 200 sin verificar nada (no toca la imagen).
- URL de otro origen (`https://xyz.supabase.co/…`) → se conserva tal cual, nunca se consulta R2.
- **R2 no responde** (simular con `R2_ENDPOINT_URL` inválido en el entorno de pruebas o cortando la red del contenedor): guardar una imagen nueva → `503` "El almacenamiento de archivos no responde…"; nada se guarda ni se borra.

## Historia 3 — Un negocio no toca los archivos de otro (P1)

```bash
# acme intenta referenciar un archivo de globex (edición legítima: base = vigente)
curl -s -o /dev/null -w '%{http_code}\n' -X PATCH "$API/products/$ID" -H "Authorization: Bearer $TOKEN_ACME" \
  -H 'Content-Type: application/json' -d "{\"image_url\":\"globex/products/zzz.jpg\",\"image_url_base\":\"$CURRENT\"}"
```

**Esperado**: `422`; el registro no cambió y `{ASSETS_BASE_URL}/globex/products/zzz.jpg` sigue abriéndose. Repite (todas → `422`, con y sin base): `acme/logo/x.png` como imagen de producto, `acme/products/../logo/x.png`, `acme//products/x.jpg`, `acme\products\x.jpg`, `ACME/products/x.jpg`, `acme/products/x%2e%2e.jpg`.

- Fila histórica con key fuera de convención: reemplazar su imagen funciona y el archivo antiguo **no** se borra (aparece "fuera de convención" en el reporte, Historia 6).

## Historia 4 — El archivo anterior solo se borra si nadie más lo usa (P2)

1. Asigna la **misma key `K`** a dos productos P1 y P2 (por API, con la base vigente de cada uno; `K` debe existir y ser de `acme/products/`).
2. Reemplaza la imagen de P1 → el registro cambia y `K` **sigue existiendo** (`curl -I {ASSETS_BASE_URL}/{K}` → 200).
3. Reemplaza la imagen de P2 → `K` se borra (404). Verifica también que con un único uso el reemplazo borra como siempre.
4. Referencia `K` desde el logo, desde un QR o desde un comprobante y reemplaza el producto → tampoco se borra.
5. Fallo de borrado (revocar temporalmente el permiso de borrado del token de R2 en pruebas): el reemplazo se confirma, no hay error visible, el archivo queda huérfano.

## Historia 5 — El cajero solo ve comprobantes válidos del propio negocio (P2)

1. Como comensal (menú QR), elige transferencia, adjunta una foto y envía. **Esperado**: en base de datos `receipt_file_url` es una **key** (`acme/comprobantes/<uuid>.jpg`) y el cajero la ve en "Pagos por confirmar".

```sql
SELECT receipt_file_url FROM acme.order_payment_attempts ORDER BY created_at DESC LIMIT 1;
```

2. Por API, con la sesión del comensal, adjunta: (a) una key inventada, (b) `globex/comprobantes/x.jpg`, (c) `acme/products/x.jpg`, (d) `https://otro.com/x.jpg` → todas `422`, sin cambios en el intento.
3. Adjunta la URL absoluta del dominio público anterior y la del dominio de assets de un comprobante real → se acepta y se guarda **solo la key**.
4. Intenta adjuntar un segundo comprobante al mismo intento → sigue `409`.
5. Filas antiguas con URL absoluta siguen viéndose en "Pagos por confirmar" (tolerancia de lectura).

### Migración de comprobantes (SC-009)

```bash
python -m app.scripts.migrate_receipt_keys                    # simulación: no modifica nada
python -m app.scripts.migrate_receipt_keys --apply            # aplica
python -m app.scripts.migrate_receipt_keys --apply            # segunda vez: 0 filas modificadas
python -m app.scripts.migrate_receipt_keys --revert --apply   # (prueba de reversión, en un entorno de pruebas)
```

**Esperado**: la simulación deja la base idéntica; la segunda aplicación no modifica ninguna fila; una fila con URL de otro origen o con archivo inexistente queda intacta y aparece en el informe sin hacer fallar el script.

## Historia 6 — Reporte de reconciliación (P3)

1. En el entorno de pruebas, **borra a mano** en el panel de Cloudflare el objeto de un producto, y **sube** otro objeto sin referencia (con más de 24 h, o ejecuta con `--grace-hours 0`).
2. Ejecuta y guarda una copia de estado (conteo de filas y `aws s3 ls` del bucket) antes y después:

```bash
python -m app.scripts.reconcile_r2_references --output /tmp/r2.csv
python -m app.scripts.reconcile_r2_references --tenant acme --grace-hours 48
```

**Esperado**: el objeto borrado aparece como "referencia sin archivo"; el sobrante como "archivo sin referencia"; una key histórica fuera de convención aparece marcada; un archivo recién subido (< ventana) **no** aparece como huérfano; `--tenant acme` cubre solo `acme`; la ruta del CSV se imprime; **la base de datos y el bucket quedan idénticos** (SC-007).

---

## Regresión (SC-008)

Sin intervención manual tras el despliegue: las imágenes de producto, los logos, los QR y los comprobantes ya existentes siguen mostrándose (panel de productos, menú del comensal, checkout del comensal, recibo con logo, "Pagos por confirmar"). Confirma además: subir una imagen de producto nueva por el flujo normal sigue funcionando sin pasos adicionales (SC-004) y la suite completa de `pos-backend` está en verde.

## Orden de despliegue (research D10)

1. `pos-heladeria` (base + no reenviar la imagen sin tocar): compatible con el backend actual y con el nuevo.
2. `pos-backend` (validación, base, borrado protegido, comprobantes por key, scripts).
3. Ejecutar `migrate_receipt_keys` en simulación → revisar → `--apply`.
4. Ejecutar `reconcile_r2_references` y revisar el primer informe (huérfanos y referencias rotas históricas: acción manual del operador, fuera de alcance).
5. Opcional, al final: que el comprobante envíe `presign.key` (nunca antes del paso 2).

## Criterios de éxito → dónde se comprueban

| Criterio | Historia / sección |
|---|---|
| SC-001 (formulario desactualizado) | Historia 1 |
| SC-002 (archivo inexistente → error y registro intacto) | Historia 2 |
| SC-003 (otro negocio / otra carpeta / segmentos) | Historia 3 |
| SC-004 (flujo normal sin pasos extra) | Historia 1 pasos 7–8 y Regresión |
| SC-005 (archivo compartido no deja imagen rota) | Historia 4 |
| SC-006 (cajero ve comprobantes; 100 % de falsos rechazados) | Historia 5 |
| SC-007 (reporte de solo lectura) | Historia 6 |
| SC-008 (sin regresión respecto a la spec 080) | Regresión |
| SC-009 (migración idempotente y simulación inocua) | Historia 5, "Migración de comprobantes" |
