# Contrato: migración única de datos

Script: `app/scripts/migrate_image_keys_relative.py` en `pos-backend`. Reescribe a su key todas las filas de los tres campos afectados que hoy guardan una URL del bucket gestionado (FR-009). Espejo operativo de `app/scripts/migrate_payment_methods_catalog.py`.

## Invocación

```bash
# 1. Inspección (no escribe): cuántas filas se migrarían, por tenant y por campo
python -m app.scripts.migrate_image_keys_relative --report-only

# 2. Migración
python -m app.scripts.migrate_image_keys_relative

# 3. Reversión (deja los tres campos como estaban antes del paso 2)
python -m app.scripts.migrate_image_keys_relative --revert
```

- Corre en el contenedor de `pos-backend`, con la app **en marcha** (no requiere ventana de mantenimiento, FR-009c).
- Código de salida: `0` si todo ok; `!= 0` si algún tenant falló (se loguea y se continúa con los demás; el commit es por schema).

## Conjunto objetivo

| Objetivo | Recorrido | Sesión |
|---|---|---|
| `shared.tenants.logo_url` | una vez | `with_db(None)` |
| `{schema}.products.image_url` | por cada `schema` de `SELECT schema FROM shared.tenants` | `with_db(schema)` |
| `{schema}.payment_methods.payment_info` | ídem — cada valor string del dict JSONB | `with_db(schema)` |

`commit` por schema (y uno para `shared`). Una interrupción entre schemas deja los ya procesados migrados y el resto sin tocar; relanzar completa (idempotencia).

## Regla por valor (modo migración)

Para el valor `v` de `logo_url` / `image_url`, o para cada valor string `v` dentro de `payment_info`:

```
si v empieza por algún prefijo de _managed_bucket_prefixes():   # V2 (pub-…r2.dev) o V3 (assets.skeilopos.com)
    v_nuevo = normalize_asset_ref(v)                            # -> key
si no:                                                          # V0 vacío, V1 ya-key, V4 otro origen
    no tocar
```

- Se usa **la misma función** `normalize_asset_ref` que la escritura de la app ⇒ "migrado por el script" y "guardado por la app" producen la misma key (FR-003).
- `payment_info`: se reescribe el dict con las claves gestionadas convertidas; las demás (`celular`, `cuenta`, …) se copian sin cambio.

## Regla por valor (modo `--revert`)

```
si v es una key (no contiene "://") y no está vacío:
    v_nuevo = f"{settings.R2_PUBLIC_BASE_URL.rstrip('/')}/{v}"   # reconstruye V2, forma exacta anterior
si no:
    no tocar                                                    # V0, V4
```

## Reporte (`--report-only` y al final de cada modo)

Por tenant y por campo:

```
tenant=heladeria3  image_url: migradas=812 ya_key=0 vacias=140 otro_origen=3
tenant=heladeria3  payment_info.qr: migradas=2 ya_key=0 vacias=1 otro_origen=0
shared.tenants.logo_url: migradas=41 ya_key=0 vacias=6 otro_origen=1
TOTAL migradas=... otro_origen=... (revisar que 'otro_origen' es esperado)
```

## Garantías

| # | Garantía | FR / SC |
|---|---|---|
| G1 | Tras migrar, el 100 % de las filas de los tres campos que eran V2/V3 quedan como key (sin `http`) | FR-009, SC-007 |
| G2 | Ninguna imagen deja de verse durante ni después (la app tolera V2 en lectura mientras tanto; V1 se sirve ensamblado) | FR-009b, SC-002, SC-003 |
| G3 | Idempotente: 2ª corrida (o relanzar tras interrupción) no cambia ninguna fila ya migrada | FR-009c, SC-009 |
| G4 | Reversible: `migrar` + `--revert` deja los tres campos idénticos al estado previo | FR-009a, SC-008 |
| G5 | No toca ningún objeto de R2 (no mueve, renombra, recrea ni borra) | FR-009a |
| G6 | Valores V4 (otro origen) intactos en ambos modos | FR-010 |
| G7 | `receipt_file_url` / carpeta `comprobantes` fuera del recorrido | FR-014 |
| G8 | Ninguna factura ni venta emitida se altera | Principio VII |

## Nota operativa sobre `--revert`

`--revert` reconstruye `{R2_PUBLIC_BASE_URL}/{key}` para **toda** fila que sea key, sin distinguir si esa key la puso la migración o una subida nueva posterior. Si entre `migrar` y `--revert` se subieron imágenes nuevas (que nacen como key, V1), el revert las dejará como V2 apuntando al mismo objeto físico (que sí existe): la imagen se sigue viendo, solo cambia el dominio de la referencia. Por eso el revert es una salida de emergencia inmediata post-despliegue, no una operación a ejecutar días después. Se recomienda correr `--report-only` antes de `--revert` para ver el conteo.

## Prerrequisitos de despliegue

1. `ASSETS_BASE_URL` desplegada y `assets.skeilopos.com` sirviendo el bucket por su key (Assumptions del spec).
2. Código de la Historia 1 + 2 desplegado (escritura persiste key, lectura ensambla, tolerancia de lectura activa) **antes** de correr la migración — así, una fila migrada se sirve bien y una no migrada también.
3. Entrada `A-73` registrada (Principio II) — ya garantizada por el punto 2: `A-73` bloquea la Historia 1 y esta se despliega antes de la migración. El **script** de migración no cambia comportamiento observable y no cita `A-73` en su código (ver `tasks.md`, Fase 4).
