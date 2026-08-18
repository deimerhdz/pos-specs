# Contrato: `GET /menu` y `GET /menu/qr-token/{token}`

Endpoints existentes (`app/api/v1/menu/router.py:182-209`, ambos llaman `_build_menu`),
**sin cambios de forma** en esta spec — mismo esquema de respuesta, mismos códigos de estado. Lo
único que cambia es **cuándo**, en el límite horario de una promoción, `_build_menu` la considera
vigente.

## Request

```
GET /menu
Headers: x-tenant-host (requerido, resuelve el tenant) — sin cambios

GET /menu/qr-token/{token}
Path: token (string, JWT firmado con tenant + mesa) — sin cambios
```

Sin autenticación de usuario en ninguno de los dos (`menu/router.py:1-4`, comentario de módulo).

## Response — `200 OK`

`GET /menu` → `list[MenuCategoryResponse]`. `GET /menu/qr-token/{token}` → objeto con `table`,
`business` y `menu` (este último, el mismo `list[MenuCategoryResponse]`). **Esquema sin cambios**
en ambos casos.

Cambia únicamente el campo `discounted_price`/`active_promotion` (o equivalente, según
`MenuProductResponse`/`MenuVariantResponse`) cuando una promoción con ventana horaria está cerca de
su límite: antes de esta corrección, una promoción podía aparecer vigente hasta 5 horas fuera de su
ventana real (`TENANT_TIMEZONE=America/Bogota`, UTC-5); después, coincide exactamente con la hora
local real y con lo que ya calculan los caminos de cobro real (A-07).

## Response — `404 Not Found`

Sin cambios: mesa inexistente o inactiva (`GET /menu/qr-token/{token}`).

## Consumidor (`pos-heladeria`)

El menú público ya consume ambos endpoints. Ningún cambio de contrato de tipos ni de código de
estado — el único efecto visible es que el precio con descuento mostrado deja de aparecer/
desaparecer hasta 5 horas antes o después de la ventana real de la promoción, ahora coincidiendo
con lo que el comensal terminará pagando.
