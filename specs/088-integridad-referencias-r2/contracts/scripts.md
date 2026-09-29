# Contrato: scripts de línea de comandos

Ambos corren en el contenedor de `pos-backend` como módulos de Python, igual que `migrate_image_keys_relative`. Ninguno tiene endpoint. Recorren `shared.tenants` una vez (`with_db(None)`) y luego cada esquema (`with_db(schema)`).

## 1. `python -m app.scripts.reconcile_r2_references` — reporte de reconciliación (FR-009)

**Solo lectura**: no ejecuta `UPDATE`/`DELETE`/`INSERT`, no hace `commit`, y solo llama `list_objects_v2` y `head_object` sobre R2 (nunca `delete_object` ni `put_object`).

```
python -m app.scripts.reconcile_r2_references
python -m app.scripts.reconcile_r2_references --tenant acme
python -m app.scripts.reconcile_r2_references --grace-hours 48 --output /tmp/r2.csv --format csv
```

| Argumento | Por defecto | Efecto |
|---|---|---|
| `--tenant SCHEMA` | todos | Acota referencias y objetos a ese negocio (prefijo `SCHEMA/`). |
| `--grace-hours N` | `24` | Un archivo sin referencia con `LastModified` más reciente que `ahora − N h` **no** se lista como huérfano (subida aún no guardada). |
| `--output RUTA` | (solo pantalla) | Además escribe el resultado a `RUTA`; la salida imprime la ruta escrita. |
| `--format csv\|json` | `csv` | Formato de `--output`. |

### Qué revisa

Referencias: `products.image_url`, claves `format:"image"` de `payment_methods.payment_info`, `order_payment_attempts.receipt_file_url`, `shared.tenants.logo_url` (research D9, mismas cuatro fuentes que el borrado). Un listado único del bucket (`list_objects_v2` paginado). URLs de otro origen se cuentan aparte y no se verifican.

### Salida en pantalla (ejemplo)

```
Reconciliación R2 — 2026-09-29 14:02 (ventana de gracia: 24 h)

Negocio: acme
  Referencias sin archivo (1)
    products.image_url        acme/products/9c1e….png   (producto 4f2a…)
  Archivos sin referencia (2)
    acme/products/aa01….jpg   2026-09-01 10:11
    acme/comprobantes/b7c2….jpg   2026-09-20 18:40
  Fuera de convención (1)
    products.image_url        legacy/img/foto.jpg

Negocio: globex
  Sin hallazgos.

Sin negocio (archivos fuera de un prefijo conocido): 0
Referencias de otro origen (no verificadas): 3

Resumen: 1 sin archivo · 2 huérfanos · 1 fuera de convención
Archivo escrito: /tmp/r2.csv
```

### Formato CSV (`--output`)

`negocio,tipo,campo,key,detalle` con `tipo` ∈ `referencia_sin_archivo` | `archivo_sin_referencia` | `fuera_de_convencion`. JSON: lista de objetos con las mismas claves.

### Código de salida

`0` siempre que el reporte se genere (haya o no hallazgos: es un informe, no una validación). `≠0` solo por error operativo (sin conexión a la base de datos o a R2).

## 2. `python -m app.scripts.migrate_receipt_keys` — migración de comprobantes (FR-008)

```
python -m app.scripts.migrate_receipt_keys                 # SIMULACIÓN (por defecto): informa, no modifica
python -m app.scripts.migrate_receipt_keys --apply         # aplica
python -m app.scripts.migrate_receipt_keys --apply --tenant acme
python -m app.scripts.migrate_receipt_keys --revert --apply   # reconstruye {R2_PUBLIC_BASE_URL}/{key}
```

| Argumento | Efecto |
|---|---|
| (ninguno) | Simulación: cuenta y lista qué filas se reescribirían; no ejecuta ningún `UPDATE`. |
| `--apply` | Ejecuta los `UPDATE`; `commit` por esquema. |
| `--revert` | Invierte la transformación (con `--apply` para ejecutarla; sin él, simulación de la reversión). |
| `--tenant SCHEMA` | Acota a un negocio. |

### Reglas por fila de `order_payment_attempts` (research D15)

| `receipt_file_url` | Acción |
|---|---|
| `NULL` | omitir |
| ya es key (sin `://`, sin prefijo gestionado) | **sin cambio** (idempotencia) |
| URL gestionada cuya key cumple `{esquema}/comprobantes/…` y **existe** en R2 | reescribir a la key |
| URL gestionada cuya key **no** cumple la convención | **intacta**, reportada "fuera de convención" |
| URL gestionada cuya key **no existe** en R2 | **intacta**, reportada "archivo inexistente" |
| URL de otro origen | **intacta**, reportada "otro origen" |
| R2 no responde al verificar | **intacta**, reportada "no verificable"; el script sigue |

Ninguna de las filas intactas hace fallar el script. No modifica ni borra ningún objeto de R2.

### Salida (ejemplo)

```
Migración de comprobantes — MODO SIMULACIÓN (usa --apply para escribir)

acme:       12 a reescribir · 40 ya key · 1 archivo inexistente · 0 otro origen · 0 fuera de convención
globex:      0 a reescribir ·  3 ya key

Total: 12 a reescribir · 43 ya key · 1 archivo inexistente
```

### Propiedades

- **Idempotente** (SC-009): una segunda ejecución con `--apply` no modifica ninguna fila; la simulación nunca modifica ninguna.
- **Re-ejecutable con la app en marcha**, sin ventana de mantenimiento; no requiere tabla de control.
- **Reversible**: `--revert` solo toca valores que son key `{esquema}/comprobantes/…` del propio esquema.
- Recomendado: desplegar el backend nuevo → ejecutar en simulación → revisar el informe → `--apply`. Como el código nuevo lee ambas formas, el orden backend/migración no es obligatorio.
