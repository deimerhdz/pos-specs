# Data Model: Integridad de las referencias a archivos en Cloudflare R2

**Sin cambios de esquema.** No hay entidades, columnas, índices ni relaciones nuevas, y no hay migración de Alembic. Cambia el **contenido** de una columna existente (`order_payment_attempts.receipt_file_url`) y se **formaliza** el significado de las otras tres referencias. Este documento cumple el Principio VIII.

## 1. Referencias a archivos (las cuatro columnas)

| # | Columna | Esquema | Tipo | Carpeta que le corresponde | Forma esperada del valor |
|---|---|---|---|---|---|
| 1 | `products.image_url` | del negocio | `String(500)`, nulo | `products` | key `{esquema}/products/{nombre}` |
| 2 | `shared.tenants.logo_url` | global (`shared`) | `String(500)`, nulo | `logo` | key `{esquema}/logo/{nombre}` |
| 3 | `payment_methods.payment_info` → clave con `format:"image"` del catálogo (p. ej. `qr`) | del negocio | `JSONB`, valor `str` | `payment-methods` | key `{esquema}/payment-methods/{nombre}` |
| 4 | `order_payment_attempts.receipt_file_url` | del negocio | `String(500)`, nulo | `comprobantes` | key `{esquema}/comprobantes/{nombre}` — **antes de esta spec: URL absoluta** |

`{esquema}` es `shared.tenants.schema` del negocio dueño. `{nombre}` cumple `[A-Za-z0-9][A-Za-z0-9._-]{0,199}` sin `..` (research D2).

Ninguna otra tabla guarda archivos (research, "Hallazgos del código"). Un valor de `1`–`3` puede ser además una **URL de otro origen** (imagen histórica no gestionada): se conserva, no se verifica ni se borra (FR-005). Un valor de `4` de otro origen se rechaza al **entrar** (FR-008) pero las filas históricas que ya lo tengan se conservan.

### Formas de un valor almacenado (compatibilidad con datos existentes)

| Forma | Origen | Lectura | Escritura nueva | Borrado del anterior |
|---|---|---|---|---|
| Key en convención | flujo actual (spec 080) | `asset_display_url` → `{ASSETS_BASE_URL}/{key}` | ✔ es lo único que se guarda | sí, si nadie más la usa |
| Key **fuera de convención** | histórico | igual | ✘ 422 si llega como valor **nuevo**; ✔ si es igual al vigente (no es nueva) | **nunca** (queda huérfana; el reporte la marca) |
| URL absoluta del bucket gestionado (`pub-…r2.dev` o `assets…`) | histórico sin migrar (`receipt_file_url` de hoy; 1–3 antes de la spec 080) | igual (se reescribe al dominio de assets, tolerancia de lectura) | se normaliza a key | se extrae la key; aplica la convención |
| URL de otro origen | histórico | intacta | 1–3: se conserva; 4: 422 | **nunca** |

## 2. Transición de `order_payment_attempts.receipt_file_url` (Principio VIII)

**Antes**: `https://pub-…r2.dev/{esquema}/comprobantes/{uuid}.jpg` (`public_url_for`, spec 025).
**Después**: `{esquema}/comprobantes/{uuid}.jpg`.

- **Compatibilidad**: el código nuevo **lee ambas formas** (`AssetUrl` en las tres respuestas) y **escribe** solo la key. Por eso el backend puede desplegarse **antes** de migrar los datos y las filas sin migrar siguen viéndose (SC-008).
- **Migración de datos**: `app/scripts/migrate_receipt_keys.py` (contrato en [contracts/scripts.md](./contracts/scripts.md)). Predicado de selección: `receipt_file_url IS NOT NULL` y empieza por un prefijo gestionado (`R2_PUBLIC_BASE_URL` o `ASSETS_BASE_URL`, sin distinguir mayúsculas en el host). Transformación: quitar el prefijo. Guardas: solo se reescribe si la key resultante pasa la convención `{esquema}/comprobantes/…` **y** el archivo existe en R2; si no, la fila queda **intacta** y se reporta (FR-008).
- **Idempotencia** (SC-009): una fila ya migrada (key) no cumple el predicado; una segunda corrida no modifica nada. Sin tabla de control. Re-ejecutable con la app en marcha; `commit` por esquema.
- **Simulación por defecto**: sin `--apply` no se ejecuta ningún `UPDATE`.
- **Reversión**: `--revert` reconstruye `{R2_PUBLIC_BASE_URL}/{key}` para valores que son key de comprobantes del propio esquema. Necesaria solo si se retira el backend nuevo (el anterior no sabe leer keys en este campo). No toca URLs de otro origen ni filas fuera de convención.
- **No toca objetos de R2**: solo cambia cómo la base de datos referencia un archivo (mismo criterio que la spec 080, FR-009a).
- **Datos históricos contables** (Principio VII): `receipt_file_url` no forma parte de ninguna factura ni de su importe. Los eventos de auditoría (spec 074) ya emitidos conservan su `receipt_hash`; no se reescriben (research D11).

## 3. Entidades conceptuales de la spec y dónde viven

| Entidad de la spec | Representación |
|---|---|
| Referencia a archivo | Cualquiera de las cuatro columnas de §1. |
| Key | Valor de §1 en convención. Función: `validate_asset_key`. |
| Imagen base del formulario | **No se persiste.** Campo de petición (`image_url_base`, `logo_url_base`, `payment_info_base`); solo existe durante el `PATCH`. |
| Archivo huérfano | Objeto en R2 sin ninguna referencia de §1; no es una fila. Lo detecta el reporte. |
| Referencia rota | Referencia de §1 cuya key no existe en R2. Lo detecta el reporte; el sistema deja de producirlas por sí mismo. |
| Reporte de reconciliación | Salida del script (pantalla + CSV/JSON opcional). No se persiste en base de datos. |

## 4. Reglas de validación (resumen; contrato en [contracts/asset-guard.md](./contracts/asset-guard.md))

- **V1** (FR-004) una key nueva cumple `{esquema}/{carpeta}/{nombre}` exacto, sensible a mayúsculas, sin `..`, `//`, `\` ni caracteres de control.
- **V2** (FR-003) una key nueva aceptada existe en R2 en el momento de guardar (`HEAD`); si R2 no responde ⇒ 503 sin efectos.
- **V3** (FR-002) una imagen enviada solo se aplica en una **creación** o si la base enviada coincide con la vigente.
- **V4** (FR-006/FR-007) el archivo anterior solo se borra si su key es borrable (en convención del propio negocio, no otro origen) **y** ninguna de las cuatro columnas la referencia, **después** del commit.
- **V5** (FR-008) un comprobante nuevo es siempre un archivo gestionado del propio negocio en `comprobantes/`; nunca una URL de otro origen.

## 5. Rollback

- **Código**: revertir el despliegue del backend restaura el comportamiento anterior (aceptar texto libre). Antes de revertir, ejecutar `migrate_receipt_keys --revert --apply` para que los comprobantes vuelvan a ser URLs que el código anterior sabe mostrar. El frontend es compatible en ambos sentidos (research D10).
- **Datos de productos/logo/QR**: sin cambio de contenido en esta spec; nada que revertir.
- **Reporte**: solo lectura, sin efectos que deshacer.
