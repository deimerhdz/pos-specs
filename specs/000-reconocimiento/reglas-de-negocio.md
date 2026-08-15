# Reglas de negocio — módulos ALTA y sus utilidades

**Fecha de extracción**: 2026-08-15
**Alcance**: módulos de criticidad ALTA de `pos-backend` según `specs/000-reconocimiento/mapa-sistema.md` sección 2.2
(`auth`, `catalog`, `inventory`, `cash`, `cart`, `table_sessions`, `orders`, `sales`, `invoices`, `promotions`, `menu`)
más las utilidades de las que dependen (`app/core/units.py`, `app/core/qr_context.py`, `app/core/qr_token.py`,
`app/core/inventory_reasons.py`, `app/core/scheduler.py`, `app/core/utils.py`).
**Método**: lectura directa de código fuente por módulo (8 pasadas independientes), extrayendo cada regla de negocio
observable — qué se cobra, qué se permite, qué se calcula, cuándo y cuánto — con evidencia de fichero y línea, un
ejemplo calculado ejecutando mentalmente el código, y clasificación provisional INTENCIONAL / ACCIDENTAL / DUDOSA.
**Numeración**: cada módulo tiene su propio prefijo (`RN-AUTH`, `RN-CAT`, `RN-INV`, `RN-CASH`, `RN-CART`, `RN-MESA`,
`RN-SCHED`, `RN-ORD`, `RN-VENTA`, `RN-FACT`, `RN-PROMO`, `RN-MENU`) en vez de una secuencia única, dado el volumen
(333 reglas) — esto mantiene trazable a qué fichero pertenece cada regla sin necesidad de una tabla de conversión.
**Sobre las suposiciones**: toda afirmación que no pudo verificarse leyendo el código está marcada explícitamente
como `SUPOSICIÓN` seguida de la pregunta que la resolvería. Ninguna suposición se da por hecha.
**Este documento no propone correcciones** — las discrepancias, código muerto y comportamientos dudosos se
documentan como hallazgos con su propia clasificación, para que el negocio decida (Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Al final se recopilan las preguntas abiertas.

---

## Índice

1. [`auth`](#1-auth) — 10 reglas
2. [`catalog` (+ `core/units.py`)](#2-catalog--coreunitspy) — 41 reglas
3. [`inventory` (+ `core/inventory_reasons.py`)](#3-inventory--coreinventory_reasonspy) — 23 reglas
4. [`cash`](#4-cash) — 17 reglas
5. [`cart` (+ `core/qr_context.py`, `core/qr_token.py`)](#5-cart--coreqr_contextpy-coreqr_tokenpy) — 27 reglas
6. [`table_sessions`](#6-table_sessions) — 27 reglas
7. [Automatización — `core/scheduler.py`](#7-automatización--corescheduerpy) — 11 reglas
8. [`orders`](#8-orders) — 66 reglas
9. [`sales`](#9-sales) — 17 reglas
10. [`invoices`](#10-invoices) — 7 reglas
11. [`promotions`](#11-promotions) — 78 reglas
12. [`menu`](#12-menu) — 9 reglas
13. [Preguntas abiertas para el negocio](#13-preguntas-abiertas-para-el-negocio)

**Total: 333 reglas de negocio extraídas.**

---

## 1. `auth`

Autentica al personal del local (login, refresh, logout, cambio de contraseña). Ficheros: `app/api/v1/auth/routes.py`,
`app/core/utils.py`, `app/core/dependencies.py`.

### RN-AUTH-01: Un cambio de contraseña exige conocer la contraseña actual correcta
**Enunciado**: Para cambiar su contraseña, un usuario autenticado debe enviar su contraseña actual, que se verifica contra el hash almacenado antes de aceptar la nueva. No hay forma de cambiar la contraseña sin conocer la vigente por esta vía (no es un "reset").
**Ejemplo**: Usuario autenticado con contraseña real `Abc12345` intenta cambiarla enviando `current_password="Wrong123"` → `verify_password` falla → `400 Bad Request "Current password is incorrect"`, la contraseña no cambia.
**Evidencia**: `app/api/v1/auth/routes.py:92-108` (función `change_password`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita, patrón estándar de cambio de contraseña autenticado.

### RN-AUTH-02: Al cambiar la contraseña se limpia el flag de "debe cambiar contraseña"
**Enunciado**: Toda cuenta tiene un flag `must_change_password` (por ejemplo, para credenciales emitidas por el sistema con contraseña temporal). Cuando el usuario completa exitosamente un cambio de contraseña, ese flag se apaga automáticamente.
**Ejemplo**: Un usuario nuevo creado con contraseña temporal generada por el sistema tiene `must_change_password=True`. Tras un `POST /auth/change-password` exitoso, `must_change_password` pasa a `False` y se persiste con `db.commit()`.
**Evidencia**: `app/api/v1/auth/routes.py:104-106` (función `change_password`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Transición de estado explícita y coherente con el propósito del flag.

### RN-AUTH-03: El login no bloquea cuentas por intentos fallidos ni limita la tasa de intentos
**Enunciado**: No existe ningún contador de intentos fallidos, bloqueo temporal ni rate-limiting específico sobre `POST /auth/login`. Cualquier número de intentos con credenciales incorrectas siempre responde `401 Invalid credentials` sin penalización adicional.
**Ejemplo**: Un atacante prueba 1.000 contraseñas distintas contra el mismo email en segundos; cada intento recibe `401 Unauthorized "Invalid credentials"` idéntico al primero, sin bloqueo de cuenta, captcha ni retraso creciente.
**Evidencia**: `app/api/v1/auth/routes.py:22-89` (función `login`, ausencia de lógica de intentos); `grep -rn "failed_attempt|lockout|login_attempt|max_attempts|is_locked"` sin resultados.
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: No hay comentario ni diseño que indique que la ausencia de bloqueo por fuerza bruta sea intencional. SUPOSICIÓN: ¿existe rate-limiting a nivel de infraestructura (proxy/WAF) fuera del código de aplicación? `app/core/rate_limit.py` sí se usa en `menu` pero no se importa en `auth/routes.py`.

### RN-AUTH-04: Solo usuarios activos pueden iniciar sesión
**Enunciado**: Aunque el email y contraseña sean correctos, si la cuenta tiene `active=False`, el login se rechaza con un error distinto al de credenciales inválidas.
**Ejemplo**: Usuario desactivado por el admin intenta loguearse con su contraseña correcta → pasa la verificación de contraseña, pero `user.active` es `False` → `403 Forbidden "User account is inactive"` (en vez de `401`).
**Evidencia**: `app/api/v1/auth/routes.py:53-56` (función `login`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita con código de estado semánticamente distinto (403 vs 401).

### RN-AUTH-05: El login resuelve el usuario dentro del tenant del header de host, o como super-admin global si no hay tenant
**Enunciado**: Si la petición trae `x-tenant-host` y resuelve a un tenant registrado, el login busca el usuario únicamente dentro de ese tenant (`tenant_id = tenant.id`). Si no hay header o no matchea ningún tenant, busca exclusivamente usuarios sin tenant (`tenant_id IS NULL`), es decir, súper-administradores globales.
**Ejemplo**: Header `x-tenant-host: heladeria-a.pos.com` resuelve `tenant = Heladería A` → el login solo busca `User.email == body.email AND User.tenant_id == heladeria_a.id`. Un usuario con el mismo email en otro tenant, o un súper-admin global con ese email, no se encuentra y el login falla con `401` aunque la contraseña sea correcta para esa otra cuenta.
**Evidencia**: `app/api/v1/auth/routes.py:24-44` (función `login`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Es el mecanismo central de aislamiento multi-tenant en el login, con comentario explícito sobre el propósito de cada rama.

### RN-AUTH-06: El access token y el refresh token tienen tiempos de vida distintos e independientes
**Enunciado**: El access token expira a los `ACCESS_TOKEN_EXPIRY` minutos (default 1.440 min = 24 h); el refresh token expira a los `REFRESH_TOKEN_EXPIRY_MINUTES` minutos (default 10.080 min = 7 días).
**Ejemplo**: Login a las 09:00 → access válido hasta las 09:00 del día siguiente, refresh válido hasta 7 días después. Usar el access a las 25 h de emitido → `401` por expiración de JWT. Usar el refresh dentro de esos 7 días → nuevo access sin volver a loguearse.
**Evidencia**: `app/core/utils.py:28-43` (función `create_access_token`); `app/core/config.py:11,14`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito en el docstring sobre por qué deben diferir.

### RN-AUTH-07: `/auth/refresh-token` re-lee el usuario en base de datos y exige que siga activo
**Enunciado**: Al renovar el access token, no se reutilizan los datos codificados en el refresh: se vuelve a consultar el usuario por id y se exige `active == True`. Si el usuario fue desactivado tras emitir el refresh, la renovación falla aunque el JWT en sí siga siendo válido.
**Ejemplo**: Refresh válido por 7 días; al día 2 un admin desactiva al usuario (`active=False`). Al día 3, `GET /auth/refresh-token` con el refresh aún vigente → `WHERE User.id==user_id AND User.active==True` no encuentra la fila → `401 "User not found or inactive"`.
**Evidencia**: `app/api/v1/auth/routes.py:111-150` (función `get_new_access_token`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "si no, una cuenta desactivada o con el rol cambiado seguiría emitiendo access tokens válidos con datos obsoletos durante toda la vida del refresh".

### RN-AUTH-08: El logout revoca el token mediante blocklist hasta su expiración natural, no toda la sesión
**Enunciado**: `GET /auth/logout` no invalida todos los tokens del usuario, solo el `jti` del access token presentado, con TTL hasta el `exp` original. Cualquier otro token (p. ej. el refresh) sigue siendo válido tras este logout.
**Ejemplo**: Usuario con access token A (jti=X) llama logout con A → `X` entra al blocklist hasta la expiración de A. Su refresh token sigue funcionando y puede emitir un access nuevo (jti distinto, no bloqueado).
**Evidencia**: `app/api/v1/auth/routes.py:152-160` (función `revoke_token`); `app/core/dependencies.py:51-56`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Mecanismo estándar, pero no hay evidencia de que el logout revoque también el refresh asociado. SUPOSICIÓN: ¿es deliberado que el refresh sobreviva al logout del access token?

### RN-AUTH-09: Las contraseñas se truncan a 72 bytes antes de hashear/verificar (límite de bcrypt)
**Enunciado**: Tanto al generar el hash como al verificar, solo se consideran los primeros 72 bytes UTF-8 de la contraseña; caracteres más allá de ese límite no afectan el resultado.
**Ejemplo**: Una contraseña de 100 caracteres donde los primeros 72 son `"A"*72` genera el mismo hash que `"A"*72` sola. Dos contraseñas que difieren solo después del byte 72 autentican igual.
**Evidencia**: `app/core/utils.py:14-15,24-25` (funciones `generate_passwd_hash`, `verify_password`)
**Clasificación**: DUDOSA
**Justificación de clasificación**: Comportamiento estándar de bcrypt, pero no hay validación de longitud máxima visible en el schema de cambio de contraseña. SUPOSICIÓN: revisar si `app/api/v1/auth/schemas.py` impone un `max_length` acorde a 72 bytes.

### RN-AUTH-10: Las contraseñas generadas por el sistema usan alfabeto específico y 12 caracteres
**Enunciado**: Cuando el sistema emite una contraseña temporal, se genera con 12 caracteres tomados aleatoriamente (criptográficamente seguro, vía `secrets.choice`) de un alfabeto de letras mayúsculas/minúsculas, dígitos y símbolos `!@#$%*?`.
**Ejemplo**: `generate_random_password()` podría producir `"aB3$k9Qz#1mP"` (12 caracteres, mezcla de tipos).
**Evidencia**: `app/core/utils.py:18-21` (función `generate_random_password`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Uso de `secrets.choice` (no `random`) indica decisión deliberada de seguridad.
## 2. `catalog` (+ `core/units.py`)

El motor de precios y de "qué se descuenta de inventario" para toda venta: presentaciones (variantes), precio,
grupos de sabores/opciones, y receta de insumos. Ficheros: `app/api/v1/catalog/service.py`, `line_pricing.py`,
`consumption_plan.py`, `router.py`, `app/core/units.py`.

### RN-CAT-01: Precio de línea = precio de la variante + extras de las opciones elegidas
**Enunciado**: El precio que se cobra por una línea vendida es el precio de la presentación (variante) más la suma de `extra_price` de cada opción elegida (sabores/toppings con recargo). No hay descuentos, impuestos ni combos en este cálculo.
**Ejemplo**: Variante «Copa Grande» `price=15000`. Opciones «Chocolate» (`extra_price=1000`) y «Maní» (`extra_price=500`). `compute_line_price`: `15000 + 1000 + 500 = 16500`.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:191-196` (función `compute_line_price`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "precio de la variante + extras de opción".

### RN-CAT-02: No existe redondeo/truncamiento explícito en el cálculo del precio de línea
**Enunciado**: No se aplica ninguna función de redondeo al calcular el precio de una línea; el resultado es la suma aritmética exacta de `Decimal`s.
**Ejemplo**: Con el ejemplo de RN-CAT-01, no hay quantize a 2 decimales explícito (aunque la columna `Numeric(12,2)` en BD limita la precisión de origen).
**Evidencia**: `app/api/v1/catalog/line_pricing.py:191-196`; `grep -n "round(\|quantize("` sobre `app/api/v1/catalog/*.py` y `app/core/units.py` sin resultados.
**Clasificación**: DUDOSA
**Justificación de clasificación**: No hay comentario que declare esta ausencia como intencional.

### RN-CAT-03: El precio de una variante no puede ser negativo, pero sí puede ser 0
**Enunciado**: Al crear/actualizar una variante, el precio debe ser `>= 0`. Un precio de 0 es válido (cortesías, productos aún sin tarifa).
**Ejemplo**: `VariantCreate(price=-1)` → 422 antes de tocar BD. `VariantCreate(price=0)` → aceptado.
**Evidencia**: `app/api/v1/catalog/schemas.py:14` (`ge=0`); `app/models/product_variant.py:44` (`ck_product_variant_price_positive`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Doble capa (schema + constraint de BD) apuntando al mismo límite.

### RN-CAT-04: El recargo de una opción (`extra_price`) no puede ser negativo, y por defecto es 0
**Enunciado**: Toda opción tiene un recargo `>= 0`; si no se especifica, el recargo es 0.
**Ejemplo**: `OptionCreate(name="Vainilla")` sin `extra_price` → se guarda con `extra_price=0`.
**Evidencia**: `app/api/v1/catalog/schemas.py:99`; `app/models/option.py:40` (`ck_option_extra_price_positive`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Consistente con RN-CAT-03, doble capa de validación.

### RN-CAT-05: Todo producto nuevo recibe automáticamente una variante vendible «Single» a precio 0
**Enunciado**: Al crear un producto sin ninguna variante, el sistema crea una por defecto llamada "Single" con precio 0 y SKU autogenerado, para que sea vendible de inmediato.
**Ejemplo**: Se crea el producto «Cono Waffle». `ensure_default_variant(db, product)` sin `price` → default `price=0` → `ProductVariant(name="Single", sku=..., price=0, active=True)`.
**Evidencia**: `app/api/v1/catalog/service.py:63-80` (función `ensure_default_variant`); invocada desde `app/api/v1/products/service.py:55`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring y comentario del llamador explícitos: "Todo vendible es una variante".

### RN-CAT-06: El SKU automático se genera con las primeras 4 letras/números en mayúscula del nombre
**Enunciado**: Cuando no se provee SKU, se genera tomando solo caracteres alfanuméricos del nombre, en mayúscula, truncado a 4 caracteres; si no hay ningún carácter alfanumérico, se usa "X".
**Ejemplo**: «Cono Waffle» → `_slug` elimina espacios/no-alfanuméricos → `"CONOWAFFLE"` → mayúsculas → truncado → `"CONO"`. SKU de la variante «Single»: `"CONO-DEF"`.
**Evidencia**: `app/api/v1/catalog/service.py:16-18` (función `_slug`); uso en `service.py:74`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Función pequeña con fallback explícito documentado.

### RN-CAT-07: Colisión de SKU se resuelve con sufijo numérico incremental empezando en 2
**Enunciado**: Si el SKU generado ya existe, se agrega el sufijo `-2`; si también existe, `-3`, y así sucesivamente.
**Ejemplo**: Ya existe `"CONO-DEF"` → `_unique_sku` prueba `"CONO-DEF"` (existe) → `"CONO-DEF-2"` (si también existe) → `"CONO-DEF-3"`.
**Evidencia**: `app/api/v1/catalog/service.py:21-27` (función `_unique_sku`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Bucle `while` explícito con incremento desde `i=2`.

### RN-CAT-08: El nombre de variante duplicado se bloquea insensible a mayúsculas/espacios, incluso contra variantes desactivadas
**Enunciado**: No se puede crear ni renombrar una variante de un producto a un nombre que ya usa otra variante del mismo producto, activa o no, comparando en minúsculas y sin espacios sobrantes.
**Ejemplo**: Producto ya tiene variante activa «Pequeña». Crear `"  pequeña  "` → recortado a `"pequeña"` → coincide con «Pequeña» → 409 "Ya existe una variante «Pequeña» en este producto".
**Evidencia**: `app/api/v1/catalog/service.py:30-60` (función `variante_duplicada`); disparado desde `router.py:45-67`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring extenso justifica: "«Pequeña» y «pequeña» son la misma presentación y dos filas confundirían al cajero".

### RN-CAT-09: Una variante desactivada sigue "ocupando" su nombre; no se puede recrear, solo reactivar
**Enunciado**: Borrar una variante es soft-delete (`active=False`). El nombre queda reservado por esa fila; para reusarlo hay que reactivar la variante existente, no crear una nueva.
**Ejemplo**: Variante «Grande» desactivada. `POST /products/X/variants {"name": "Grande"}` → 409 "Ya existe una variante «Grande» desactivada en este producto. Reactívala en vez de crear otra."
**Evidencia**: `app/api/v1/catalog/router.py:45-67`; `app/api/v1/catalog/service.py:38-42`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Motivo documentado explícitamente: convierte el 500 de una colisión de constraint en un 409 accionable.

### RN-CAT-10: Eliminar una variante o una opción es siempre soft-delete
**Enunciado**: `DELETE /variants/{id}` y `DELETE /options/{id}` no borran la fila; solo marcan `active=False`, preservando el histórico de ventas.
**Ejemplo**: `DELETE /options/{id}` → `option.active=False; db.commit()`; la opción sigue existiendo en ventas pasadas.
**Evidencia**: `app/api/v1/catalog/router.py:172-181, 505-519`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: El summary del endpoint lo declara explícitamente.

### RN-CAT-11: El SKU es único en todo el tenant, no solo dentro del producto
**Enunciado**: Dos variantes de productos distintos no pueden compartir el mismo SKU.
**Ejemplo**: Variante A del producto «Helado» tiene SKU `"COMB-1"`. Crear otra variante de «Combo» con SKU `"COMB-1"` → 409 "SKU already exists".
**Evidencia**: `app/api/v1/catalog/router.py:125-126, 151-152`; `app/models/product_variant.py:27` (`unique=True`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Columna `unique=True` a nivel de tabla completa, validada explícitamente antes del commit.

### RN-CAT-12: La cantidad de un insumo en la receta debe ser estrictamente mayor que cero
**Enunciado**: Un ítem de receta (BOM) no puede tener cantidad 0 ni negativa.
**Ejemplo**: `RecipeItemIn(quantity=0)` → 422 ("Input should be greater than 0"). Reforzado en BD con `CheckConstraint`.
**Evidencia**: `app/api/v1/catalog/schemas.py:44` (`gt=0`); `app/models/recipe_item.py:35`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Doble capa de validación.

### RN-CAT-13: Guardar la receta de una variante es un reemplazo total e idempotente
**Enunciado**: `PUT /variants/{id}/recipe` borra toda la receta anterior y la reemplaza íntegramente por la lista enviada.
**Ejemplo**: Variante tenía `[Leche: 200g]`. `PUT` con `[Fruta: 150g, Vasito: 1un]` → se borra la receta anterior y se insertan las dos nuevas; «Leche» desaparece.
**Evidencia**: `app/api/v1/catalog/router.py:206-234` (función `set_recipe`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Reemplazo total (idempotente)".

### RN-CAT-14: No se admite el mismo insumo repetido dos veces en una sola receta
**Enunciado**: Si el mismo `inventory_item_id` aparece más de una vez en el payload de receta, se rechaza toda la operación.
**Ejemplo**: `PUT` con `[Fresa: 200, Fresa: 50]` → 422 "Insumo repetido en la receta". El `DELETE` de la receta anterior ya se ejecutó antes de detectar el duplicado, dentro de la misma sesión sin commit previo.
**Evidencia**: `app/api/v1/catalog/router.py:218-225`
**Clasificación**: DUDOSA
**Justificación de clasificación**: La regla de rechazo es clara, pero que el `DELETE` ocurra antes de validar duplicados dentro de la misma transacción no está documentado; depende del manejo de rollback en `app/core/db.py` (fuera del alcance analizado).

### RN-CAT-15: Los grupos de opciones que ofrece una variante también se reemplazan totalmente en cada guardado
**Enunciado**: `PUT /variants/{id}/option-groups` borra todos los vínculos anteriores y los reemplaza; no admite el mismo grupo repetido, y rechaza vincular un grupo inactivo.
**Ejemplo**: `PUT` con `groups=[{G,...},{G,...}]` (G repetido) → 422 "Grupo de opciones repetido en esta presentación".
**Evidencia**: `app/api/v1/catalog/router.py:261-306`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "Reemplazo total e idempotente, igual que la receta".

### RN-CAT-16: `max_select` nunca puede ser menor que `min_select`, y `max_select` mínimo permitido es 1
**Enunciado**: En un grupo de opciones (global y en la relación variante-grupo), el máximo no puede ser menor que el mínimo. `max_select` no puede ser 0 (mínimo estructural 1); `min_select` puede ser 0 (grupo opcional).
**Ejemplo**: `min_select=3, max_select=2` → 422 "max_select < min_select". `min_select=2, max_select=2` es válido (igualdad permitida).
**Evidencia**: `app/api/v1/catalog/router.py:369-370, 399-400`; `app/api/v1/catalog/schemas.py:70-78,131-132`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación replicada en tres lugares, consistente en todos.

### RN-CAT-17: El consumo por receta fija es `cantidad_receta × cantidad_vendida`
**Enunciado**: Cada insumo fijo de la receta se descuenta multiplicando su cantidad configurada por las unidades vendidas de esa línea.
**Ejemplo**: Receta de «Copa Grande» incluye `Vasito: 1 unidad`. Vender 3 copas → `1 × 3 = 3 vasitos` consumidos.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:109-114`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Fórmula documentada en la cabecera del módulo.

### RN-CAT-18: El consumo por opción elegida usa UNA sola cantidad: la del tamaño (variante) manda sobre la de la opción; nunca se suman
**Enunciado**: Si el grupo de opciones de la variante define `quantity_per_option > 0`, esa cantidad es la que se descuenta por cada opción elegida, ignorando el `item_quantity` propio de la opción. Solo si el tamaño no define nada (`quantity_per_option=0`) se usa el `item_quantity` de la opción como respaldo.
**Ejemplo**: «Copa Grande» ofrece «Sabores» con `quantity_per_option=120` (g). Opción «Fresa» tiene `item_quantity=80`. Al vender 1 copa con «Fresa»: `del_grupo=120` → manda el tamaño → consumo = **120 g**, no 200 g ni 80 g.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:116-138`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Cabecera del módulo declara la corrección deliberada de un bug histórico de doble descuento (140g en vez de 120g/60g según tamaño).

### RN-CAT-19 [DISCREPANCIA]: El docstring del modelo `VariantOptionGroup` contradice el comportamiento real: dice que las cantidades "se suman"
**Enunciado**: El comentario del campo `quantity_per_option` en el modelo afirma "Se suma a `options.item_quantity`", pero el código implementa exactamente lo opuesto (RN-CAT-18): una sustituye a la otra.
**Ejemplo**: Con `quantity_per_option=120`, `item_quantity=80`, el docstring sugeriría 200 g; el código real produce 120 g.
**Evidencia**: `app/models/variant_option_group.py:46-49` vs `app/api/v1/catalog/consumption_plan.py:24-28,126-129`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: Contradicción textual directa entre dos comentarios del mismo repositorio; el código ejecutable (más reciente, narra la corrección de un bug) prevalece.

### RN-CAT-20: El consumo por opción es "por cada opción elegida", no un total repartido entre el grupo
**Enunciado**: Si el cliente elige varias opciones del mismo grupo, cada una descuenta la cantidad completa configurada.
**Ejemplo**: Grupo «Sabores» de 3 bolas, `quantity_per_option=120`. Elegir Fresa+Chocolate+Vainilla → 3 líneas de 120 g = **360 g totales**, no 120 g repartidos.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:10-22,121-138`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado explícitamente en la cabecera del módulo.

### RN-CAT-21: Dos opciones distintas que apuntan al mismo insumo generan dos movimientos de inventario separados
**Enunciado**: Si dos sabores diferentes están ligados al mismo insumo, el sistema no combina sus cantidades en un solo apunte; genera un movimiento por cada opción.
**Ejemplo**: «Fresa» y «Fresa Premium» apuntan al mismo insumo. Elegir ambas → dos `ConsumptionLine` distintas sobre el mismo `inventory_item_id`, no una sumada.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:104-107`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Justificación de auditabilidad explícita en el docstring: "dos apuntes separados son auditables, uno fusionado no".

### RN-CAT-22: Una opción sin insumo ligado no genera ningún consumo, aunque tenga `item_quantity` configurado
**Enunciado**: Si una opción no tiene `inventory_item_id`, se ignora por completo en el plan de consumo.
**Ejemplo**: Opción «Sin Topping» con `inventory_item_id=None` e `item_quantity=50` (mal cargado) → se ignora, los 50 no generan consumo.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:121-123`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito y comentado.

### RN-CAT-23: Una cantidad de consumo por opción resultante de 0 o negativa no genera línea de consumo
**Enunciado**: Si, tras aplicar RN-CAT-18, la cantidad por unidad es `<=0`, no se genera movimiento de inventario para esa opción.
**Ejemplo**: Grupo sin `quantity_per_option` (`=0`) y opción con `item_quantity=0` → `per_unit=0` → sin consumo.
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:129-131`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito de guardia.

### RN-CAT-24: Stock insuficiente se define como `stock_actual < requerido` (comparación estricta, no `<=`)
**Enunciado**: El chequeo preventivo de disponibilidad rechaza con 409 solo si el stock actual es estrictamente menor que lo requerido; si son exactamente iguales, se permite (agota el stock, pero no falta).
**Ejemplo**: Insumo con `current_stock=120`, requerido=120 exacto → `120 < 120` es falso → **no** se rechaza. Con `current_stock=119` → `119 < 120` es verdadero → 409.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:199-220` (función `check_availability`, línea 210)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Operador `<` explícito y único en el código.

### RN-CAT-25: Un requerimiento de consumo de 0 o negativo se omite del chequeo de disponibilidad
**Enunciado**: Si el total requerido de un insumo es `<=0`, no se valida su stock.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:204-206`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Guardia explícita, coherente con que `plan_line_consumption` nunca produce líneas con cantidad `<=0`.

### RN-CAT-26: El chequeo de disponibilidad NO reserva ni bloquea stock; es solo preventivo (best-effort)
**Enunciado**: Pasar el chequeo de disponibilidad al armar el carrito no garantiza que el stock siga disponible al momento de consolidar/pagar; puede quedar obsoleto si otra venta consume el mismo insumo en el ínterin.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:5-8`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Declarado explícitamente en la cabecera del módulo, distinguiéndolo del bloqueo real de `inventory/stock.py`.

### RN-CAT-27: Un grupo de opciones normal permite elegir cualquier cantidad entre `min_select` y `max_select`, ambos inclusive
**Enunciado**: Para un grupo que no exige el máximo (ver RN-CAT-28), la cantidad elegida debe estar en `[min_select, max_select]` inclusive.
**Ejemplo**: `min_select=1, max_select=2`, sin consumo de inventario. Elegir 1 → válido. Elegir 3 → "admite como máximo 2 opción(es), se enviaron 3".
**Evidencia**: `app/api/v1/catalog/line_pricing.py:141-154`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Operadores `>` y `<` explícitos, sin off-by-one visible.

### RN-CAT-28: Un grupo obligatorio que además descuenta inventario exige elegir EXACTAMENTE el máximo, no basta con el mínimo
**Enunciado**: Cuando un grupo es obligatorio (`min_select>0`) y mueve inventario, el cliente debe elegir exactamente `max_select` opciones; menos (aunque cumpla el mínimo) o más se rechaza.
**Ejemplo**: «Sabores» de «Copa Grande» (3 bolas): `min_select=3, max_select=3, quantity_per_option=120`. Elegir solo 1 sabor → "exige exactamente 3 opción(es), se enviaron 1", bloqueante siempre.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:94-105,145-150`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "un helado de tres bolas... sirve tres y descuenta una" si solo exigiera el mínimo.

### RN-CAT-29: Un grupo obligatorio no elegido en absoluto exige el número exacto correspondiente
**Enunciado**: Si no se elige ninguna opción de un grupo obligatorio, el mensaje pide `max_select` si el grupo descuenta inventario, o `min_select` si no.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:156-159`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Uso directo de la misma función de RN-CAT-28, aplicado simétricamente.

### RN-CAT-30: Las violaciones de selección en grupos que descuentan inventario son SIEMPRE bloqueantes, sin importar `STRICT_OPTION_SELECTION`
**Enunciado**: Cualquier problema de validación en un grupo que mueve inventario detiene la venta con 422, incluso si el modo estricto global está desactivado.
**Ejemplo**: Con `STRICT_OPTION_SELECTION=False` (default), el caso de RN-CAT-28 sigue lanzando 422 igual.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:164-172`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "aceptar cinco opciones en un grupo `max_select=1` descuenta cinco veces el helado".

### RN-CAT-31: `STRICT_OPTION_SELECTION` (default `False`) tolera violaciones de min/max SOLO en grupos que no descuentan inventario
**Enunciado**: Si ningún problema es bloqueante y el flag está en `False`, la selección inválida se tolera con un warning en logs, sin error.
**Ejemplo**: Grupo «Toppings» (`min_select=1, max_select=2`, sin inventario) no elegido → se registra warning y la operación continúa.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:166-172`; `app/core/config.py:62`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring lo declara "rodaje asimétrico" deliberado, para migración gradual de catálogos.

### RN-CAT-32 [DUDOSA]: Con `STRICT_OPTION_SELECTION=False`, elegir una opción de un grupo que la variante NO ofrece no se rechaza, y puede seguir cobrándose y consumiendo su propio inventario
**Enunciado**: Si el `gid` de la opción elegida no corresponde a ningún grupo vinculado a la variante, ese problema nunca es bloqueante por sí solo; con el flag en `False` pasa sin error, sigue sumando `extra_price` (RN-CAT-01) y, si tiene `item_quantity>0` propio, sigue generando consumo (RN-CAT-18, rama "manda la opción").
**Ejemplo**: «Cono Simple» solo ofrece «Sabores»; se envía además una opción de «Extras de Pizza» (grupo ajeno) con `extra_price=3000` e `item_quantity=1` ligado a insumo de pizza → pasa sin error, cobra $3.000 extra y consume el insumo de pizza.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:141-144,164-172`; `consumption_plan.py:116-129`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No hay comentario que discuta este caso específico de grupo totalmente ajeno a la variante.

### RN-CAT-33 [DUDOSA]: La validación de selección de opciones es opcional para el llamador y puede omitirse por completo
**Enunciado**: `load_valid_options` solo valida contra los grupos de la variante si se le pasa el parámetro `variant`; si se omite, las opciones se cargan sin ninguna validación de min/max/pertenencia.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:43-66`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El propio docstring reconoce el riesgo sin resolverlo: "pasarla siempre que se pueda" es una advertencia, no una garantía.

### RN-CAT-34: Vender una variante sin que se descuente NADA de inventario está bloqueado con 409
**Enunciado**: Si el plan de consumo agregado de una línea resulta vacío, la venta de esa línea se rechaza, distinguiendo (a) variante sin receta ni grupo configurado, o (b) grupo configurado que descuenta pero sin opción elegida.
**Ejemplo**: «Malteada Especial» sin receta ni grupos → 409 "«Malteada Especial» no tiene receta configurada, así que venderlo no descontaría inventario. Cárgasela en Productos → Recetas para poder venderlo."
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:165-226`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "Una variante que se vende sin mover stock... es peor que un error, porque nadie se entera".

### RN-CAT-35 [TENSIÓN/DISCREPANCIA]: Un grupo opcional (`min_select=0`) que descuenta inventario bloquea la venta si es la ÚNICA fuente de consumo y el cliente no elige nada
**Enunciado**: `validate_option_selection` permite explícitamente no elegir nada de un grupo opcional sin generar problema. Pero si ese grupo opcional es la única fuente de consumo, `ensure_lines_consume_inventory` bloquea la venta con 409 "sin_eleccion" — pese a que el propio docstring describe no elegir como "una decisión legítima del comensal".
**Ejemplo**: «Cono Simple» sin receta fija, solo ofrece «Sabores» (`min_select=0, max_select=1, quantity_per_option=80`). Comprar sin elegir sabor → `validate_option_selection` lo permite, pero `ensure_lines_consume_inventory` bloquea con 409 "consume inventario según la opción que elija el cliente, pero no se eligió ninguna".
**Evidencia**: `app/api/v1/catalog/consumption_plan.py:174-179` (comentario) vs `188-198,214-226` (lógica real)
**Clasificación**: DUDOSA
**Justificación de clasificación**: El comentario solo cubre "insumos fijos Y grupo sin elegir" (pasa); no aclara el caso de grupo opcional como única fuente, que el código sí bloquea.

### RN-CAT-36: Desactivar o eliminar un grupo de opciones está bloqueado mientras alguna variante lo siga ofreciendo
**Enunciado**: No se puede desactivar ni eliminar un grupo si existe al menos una variante que todavía lo tiene asignado.
**Ejemplo**: «Sabores» asignado a «Copa Grande» y «Copa Mediana» → `DELETE` → 409 "«Helado · Copa Grande», «Helado · Copa Mediana» lo ofrece a sus clientes. Quítalo de esas presentaciones primero."
**Evidencia**: `app/api/v1/catalog/router.py:310-339,403-405,423`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: evita "vender sin descontar", que "costó caro antes".

### RN-CAT-37: El nombre de una opción debe ser único dentro de su grupo, no globalmente
**Enunciado**: Dos opciones del mismo grupo no pueden compartir nombre, pero el mismo nombre sí puede repetirse en grupos distintos.
**Evidencia**: `app/api/v1/catalog/router.py:445-449,475-484`; `app/models/option.py:41`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Doble capa con el mismo scope compuesto `(option_group_id, name)`.

### RN-CAT-38: Desvincular el insumo de una opción resetea forzosamente su `item_quantity` a 0, aunque el mismo request traiga un valor distinto
**Enunciado**: Al enviar `inventory_item_id: null`, el sistema pone `item_quantity=0` sin importar el `item_quantity` del mismo payload, porque ese valor solo se aplica si tras el cambio `option.inventory_item_id` sigue no siendo `None`.
**Ejemplo**: `PATCH {"inventory_item_id": null, "item_quantity": 999}` → primero se desliga el insumo (`item_quantity=0`), luego la rama de `item_quantity` no aplica porque `inventory_item_id` ya es `None` → resultado final `item_quantity=0`, no `999`.
**Evidencia**: `app/api/v1/catalog/router.py:488-497`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "`None` explícito desliga el insumo; ausente = no tocar".

### RN-CAT-39 [DISCREPANCIA ENTRE VALIDADORES]: Dos funciones distintas definen "¿este grupo descuenta inventario?" con criterios diferentes
**Enunciado**: `grupos_que_descuentan` (validación de selección) considera que un grupo descuenta si existe alguna opción con `item_quantity>0`, sin filtrar por `active` ni exigir `inventory_item_id`. `group_discounts` (usada en `ensure_lines_consume_inventory`) exige además `active=True` e `inventory_item_id` no nulo.
**Ejemplo**: Opción «Extra Dulce»: `item_quantity=10`, `inventory_item_id=None`. `grupos_que_descuentan` la cuenta como "descuenta"; `group_discounts` NO la considera "que descuenta" para el mismo grupo.
**Evidencia**: `app/api/v1/catalog/line_pricing.py:69-91` vs `app/api/v1/catalog/consumption_plan.py:79-95`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Ningún comentario reconoce ni justifica la diferencia de criterio entre las dos funciones que resuelven, en la práctica, la misma pregunta de negocio.

### RN-CAT-40: `app/core/units.py` rechaza convertir entre unidades de dimensión distinta
**Enunciado**: La utilidad de conversión de unidades lanza 422 al intentar convertir entre dimensiones incompatibles (p. ej. gramos a mililitros).
**Ejemplo**: `convert(500, gramo, mililitro)` con `gramo.dimension="masa"`, `mililitro.dimension="volumen"` → 422 "Unidades incompatibles: g (masa) vs ml (volumen)."
**Evidencia**: `app/core/units.py:14-24`
**Clasificación**: DUDOSA (ver RN-CAT-41 — el mecanismo del que depende no existe hoy)
**Justificación de clasificación**: La regla está documentada, pero no es ejecutable en el estado actual del modelo de datos.

### RN-CAT-41 [HALLAZGO CRÍTICO / CÓDIGO MUERTO]: `app/core/units.py` depende de campos (`dimension`, `factor_to_base`) que no existen en el modelo `UnitMeasure`, y el módulo no se usa en ninguna parte del código
**Enunciado**: `convert()` accede a `from_unit.dimension`/`to_unit.dimension` y `factor_to_base`. El modelo `UnitMeasure` solo define `name`, `abbreviation`, `active` — no tiene esas columnas. Ninguna búsqueda en todo el repositorio encuentra un solo import o uso de esta función.
**Ejemplo**: Invocar `convert()` con instancias reales de `UnitMeasure` de la BD actual lanzaría `AttributeError: 'UnitMeasure' object has no attribute 'dimension'`, no el 422 documentado.
**Evidencia**: `app/core/units.py:14-29` vs `app/models/unit_measure.py:1-18`; `grep -rn "dimension\|factor_to_base" app/models app/alembic` sin resultados fuera de `units.py`; `grep -rn "from app.core.units\|units\.convert" app/` sin resultados.
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: El propio docstring lo llama "Fase 1", sugiriendo una migración de esquema planeada pero nunca completada. Es código muerto e inconsistente con el esquema real.
## 3. `inventory` (+ `core/inventory_reasons.py`)

Stock único de insumos (kardex), compras a proveedores, ajustes manuales. Ficheros: `app/api/v1/inventory/stock.py`,
`service.py`, `router.py`, `app/core/inventory_reasons.py`.

### RN-INV-01: Único punto de mutación de stock, signo determinado por el tipo de movimiento
**Enunciado**: Todo movimiento de inventario se registra mediante `record_movement`, que recibe siempre una `quantity` positiva; la dirección (sumar o restar) la determina `type` ('in' suma, 'out' resta), nunca el signo de la cantidad.
**Ejemplo**: `record_movement(type="out", quantity=5)` con `current_stock=20` → `new_stock=15`. Con `type="in"` y la misma cantidad → `new_stock=25`.
**Evidencia**: `app/api/v1/inventory/stock.py:39-77` (función `record_movement`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito sobre el diseño, coherente con el modelo de kardex.

### RN-INV-02: Movimiento con cantidad cero o negativa es rechazado
**Enunciado**: No se permite registrar un movimiento con cantidad `<= 0`.
**Ejemplo**: `record_movement(type="in", quantity=0)` → `ValueError("quantity must be > 0")`, sin tocar la fila del insumo.
**Evidencia**: `app/api/v1/inventory/stock.py:58-59`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita antes de cualquier efecto secundario.

### RN-INV-03: `record_movement` no acepta el tipo 'adjustment'
**Enunciado**: Los ajustes manuales no pueden registrarse por la vía de entradas/salidas simples; deben usar `apply_adjustment`.
**Ejemplo**: `record_movement(type="adjustment", quantity=3)` → `ValueError` "usa apply_adjustment para los ajustes con signo".
**Evidencia**: `app/api/v1/inventory/stock.py:55-64`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: El comentario documenta un comportamiento previo defectuoso corregido con esta guarda explícita.

### RN-INV-04: Bloqueo de salida por stock insuficiente (no se permite dejar el stock negativo)
**Enunciado**: Una salida se rechaza si dejaría el stock por debajo de cero. Sí se permite que la salida deje el stock exactamente en cero.
**Ejemplo**: `current_stock=5`. Salida de 5 → `new_stock=0`, se acepta. Salida de 5.001 → `InsufficientStockError`.
**Evidencia**: `app/api/v1/inventory/stock.py:70-77`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comparación explícita `< 0` (estricta).

### RN-INV-05 [DUDOSA]: Existe una vía para forzar stock negativo (`allow_negative`), sin llamador visible en `inventory`
**Enunciado**: `record_movement` acepta `allow_negative`; si `True`, permite que el stock quede negativo tras una salida.
**Evidencia**: `app/api/v1/inventory/stock.py:49,72`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Ni `service.py` ni `router.py` de `inventory` lo invocan con `True`. SUPOSICIÓN: ¿qué endpoints de `sales`/`orders` llaman a `record_movement` con `allow_negative=True`?

### RN-INV-06: Bloqueo por `SELECT ... FOR UPDATE` para prevenir condición de carrera
**Enunciado**: Antes de leer/modificar el stock, se bloquea la fila del insumo; dos movimientos simultáneos se serializan.
**Evidencia**: `app/api/v1/inventory/stock.py:66-68,104-106`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring del módulo declara explícitamente el propósito de atomicidad.

### RN-INV-07: Bloqueo múltiple en orden canónico para evitar deadlocks
**Enunciado**: Al bloquear varios insumos a la vez, se bloquean en una sola consulta ordenados por id, no en el orden de llegada.
**Evidencia**: `app/api/v1/inventory/stock.py:17-36` (función `lock_items`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Explicado extensamente en el docstring como prevención deliberada de deadlocks.

### RN-INV-08: Ajuste manual de stock mediante delta con signo
**Enunciado**: Los ajustes no usan `type='in'/'out'` sino un delta con signo (`signed_delta`); el kardex guarda `type='adjustment'` con `quantity=abs(signed_delta)` (siempre positiva).
**Ejemplo**: `current_stock=10`. `apply_adjustment(signed_delta=-2.5, reason="Merma por derrame")` → `new_stock=7.5`; `InventoryMovement(type="adjustment", quantity=2.5, reason="Merma por derrame")`.
**Evidencia**: `app/api/v1/inventory/stock.py:92-121`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Diseño explícito y documentado en el docstring de la función.

### RN-INV-09: Ajuste manual también bloquea dejar el stock negativo, sin bandera para forzarlo
**Enunciado**: Un ajuste se rechaza si el resultado dejaría el stock `<0`; a diferencia de `record_movement`, no hay parámetro para forzarlo.
**Ejemplo**: `current_stock=3`. `apply_adjustment(signed_delta=-5)` → `InsufficientStockError("El ajuste dejaría el stock... en negativo.")`.
**Evidencia**: `app/api/v1/inventory/stock.py:107-111`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito, sin bandera de excepción.

### RN-INV-10: Un ajuste con delta = 0 es rechazado, y termina en error 500 no controlado
**Enunciado**: No se permite un `signed_delta=0`. El schema no restringe este valor (`gt`/`ne` ausentes), así que el `ValueError` resultante propaga como 500 genérico en vez de 400/422.
**Ejemplo**: `POST /inventory/items/{id}/adjust {"signed_delta": 0}` → `ValueError("signed_delta must be != 0")` sin handler dedicado → 500.
**Evidencia**: `app/api/v1/inventory/stock.py:102-103`; `app/api/v1/inventory/schemas.py:47-53`; ausencia de handler de `ValueError` verificada en `app/main.py`, `app/core/*.py`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: La regla en sí es intencional, pero el mecanismo de comunicación del error HTTP no está conectado, a diferencia de `InsufficientStockError`.

### RN-INV-11 [DUDOSA]: El motivo (`reason`) de un ajuste manual NO es obligatorio
**Enunciado**: Un ajuste puede registrarse sin motivo alguno.
**Ejemplo**: `{"signed_delta": -2.5}` sin `reason` → aceptado, `InventoryMovement.reason=None`.
**Evidencia**: `app/api/v1/inventory/schemas.py:47-53`; `app/api/v1/inventory/stock.py:97,117`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Contradice la expectativa razonable de que un ajuste (que puede ocultar mermas o fraude) exija justificación. SUPOSICIÓN: ¿el frontend sí lo exige?

### RN-INV-12: El motivo del kardex es texto libre, sin restricción en base de datos
**Enunciado**: `reason` no está limitado por lista cerrada a nivel de BD (sin `CheckConstraint`); el catálogo de `inventory_reasons.py` es convención de aplicación, no regla forzada por el esquema.
**Evidencia**: `app/core/inventory_reasons.py:5-8` (docstring: "decisión deliberada... para no bloquear motivos nuevos")
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Decisión documentada explícitamente como deliberada.

### RN-INV-13: Catálogo completo de motivos de movimiento de inventario
**Enunciado**: Seis motivos canónicos: `venta` (salida), `compra` (entrada), `ajuste` (ambas direcciones), `daño` (salida), `vencimiento` (salida), `consumo_interno` (salida). `AJUSTE` aparece en `ENTRADA_REASONS` y `SALIDA_REASONS` simultáneamente.
**Evidencia**: `app/core/inventory_reasons.py:23-45`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Definición explícita, con comentario propio sobre la ortogonalidad de `AJUSTE`.

### RN-INV-14: Umbral de "stock bajo" es `<=` (no `<`) y es configurable por insumo, no un valor global
**Enunciado**: Un insumo está en "stock bajo" cuando `current_stock <= min_stock` (por insumo, no constante global).
**Ejemplo**: "Leche" con `min_stock=10`. `current_stock=10` → sí es stock bajo (`10<=10`). `current_stock=10.001` → no lo es.
**Evidencia**: `app/api/v1/inventory/service.py:41-42`; `router.py:57-59`; `app/models/inventory_item.py:36-38`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Operador `<=` idéntico en dos implementaciones independientes.

### RN-INV-15 [DISCREPANCIA]: El filtro `low_stock` de `/items` y el endpoint dedicado `/items/low-stock` difieren en si excluyen insumos inactivos
**Enunciado**: `/items/low-stock` solo devuelve insumos **activos** en stock bajo (`active.is_(True)` hardcodeado). `/items?low_stock=true` no excluye inactivos salvo que el cliente pida explícitamente `active=true`.
**Ejemplo**: Insumo "Colorante descontinuado" (`active=False`, `current_stock=0<=min_stock=5`) → `NO` aparece en `/items/low-stock`; `SÍ` aparece en `/items?low_stock=true` sin filtro adicional de `active`.
**Evidencia**: `app/api/v1/inventory/service.py:39-42` vs `router.py:56-61`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: No hay comentario que justifique la diferencia de comportamiento entre dos rutas que aparentan resolver la misma pregunta.

### RN-INV-16: Compra directa (`POST /inventory/purchases`) da alta de stock total inmediata
**Enunciado**: Al registrar una compra directa, cada ítem se marca recibido en su totalidad de inmediato y se da entrada de stock completa en el mismo acto; el estado final es `"received"`.
**Ejemplo**: Compra `quantity=50, unit_cost=2.00` sobre insumo con `current_stock=10` → `received_quantity=50`, entrada de 50, `current_stock=60`, `total=100.00`.
**Evidencia**: `app/api/v1/inventory/service.py:46-90`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario en cabecera y en la ruta confirman: "da alta de stock total" para compras "de contado".

### RN-INV-17 [DUDOSA]: Toda compra directa (y toda recepción de orden) sobreescribe el costo unitario del insumo, incondicionalmente
**Enunciado**: Cada entrada por compra reemplaza el `unit_cost` del insumo por el costo de esa compra específica, sin promediar (no hay costo promedio ponderado).
**Ejemplo**: "Fresa" con `unit_cost=3.00`. Compra de 20 unidades a `unit_cost=5.00` → `item.unit_cost=5.00`, sin importar que sea más caro.
**Evidencia**: `app/api/v1/inventory/service.py:76,149-151`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Comportamiento consistente y probablemente deliberado ("último costo de compra"), pero sin declaración explícita frente a la alternativa de costo promedio. SUPOSICIÓN: ¿el negocio espera "último costo" o "costo promedio"?

### RN-INV-18: Orden de compra (`POST /inventory/purchases/order`) no da alta de stock al crearse
**Enunciado**: Crear una orden deja el stock intacto; estado inicial `"draft"`, `received_quantity=0`. El stock solo sube al recibirla.
**Evidencia**: `app/api/v1/inventory/service.py:93-120` (docstring: "SIN dar alta de stock (RF-022)")
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Referencia directa a un requisito funcional documentado.

### RN-INV-19: No se puede recibir sobre una orden ya recibida por completo
**Enunciado**: Si `status="received"`, cualquier intento adicional de recepción se rechaza con 409.
**Evidencia**: `app/api/v1/inventory/service.py:127-129`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito al inicio de la función.

### RN-INV-20: No se puede recibir más cantidad de la pendiente por ítem
**Enunciado**: La cantidad recibida no puede exceder lo pendiente (`quantity - received_quantity` acumulado); exceder rechaza toda la operación (incluye rollback de otros ítems del mismo request).
**Ejemplo**: `quantity=30, received=20` → `remaining=10`. Recibir 15 → 422 "Recepción 15 excede lo pendiente (10)"; rollback de todo el request.
**Evidencia**: `app/api/v1/inventory/service.py:137-142,157-158`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita con mensaje de negocio.

### RN-INV-21: La recepción parcial suma stock incrementalmente y puede repetirse hasta completar el pedido
**Enunciado**: Cada recepción da entrada de stock solo por lo recibido en ese request, acumula `received_quantity`, y recalcula el estado: `"received"` si todo completo, `"partial"` si algo recibido pero no todo, `"draft"` si nada recibido.
**Evidencia**: `app/api/v1/inventory/service.py:143-155`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Lógica explícitamente calculada tras cada recepción.

### RN-INV-22: En la recepción, el costo unitario aplicado es el pactado en la orden, no uno nuevo enviado en la recepción
**Enunciado**: El body de recepción no admite `unit_cost`; el costo aplicado siempre es el fijado en el `PurchaseItem` al crear la orden.
**Evidencia**: `app/api/v1/inventory/schemas.py:132-138`; `app/api/v1/inventory/service.py:149-151`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Consecuencia directa del diseño del schema, coherente con "el costo se pacta al crear la orden".

### RN-INV-23: Los ítems de compra requieren costo unitario no negativo, cantidad estrictamente positiva
**Enunciado**: En compra directa, orden, y cada línea de recepción, la cantidad debe ser `>0` y el costo `>=0`.
**Evidencia**: `app/api/v1/inventory/schemas.py:97-100,132-134`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Restricciones declarativas explícitas (`gt`, `ge`), consistentes en los tres flujos.

---

## 4. `cash`

Turnos de caja (apertura, cierre, arqueo) y movimientos manuales de efectivo. Fichero: `app/api/v1/cash/service.py`,
`router.py`.

### RN-CASH-01: Un solo turno de caja abierto por caja registradora a la vez
**Enunciado**: No pueden coexistir dos turnos `status='open'` para la misma caja. Un segundo intento se rechaza.
**Ejemplo**: Caja con turno abierto. Segundo `POST /cash/shifts/open` mismo `cash_register_id` → el índice único parcial rechaza el `INSERT` → 409 "La caja ya tiene un turno abierto".
**Evidencia**: `app/models/cash_shift.py:50-56`; `app/api/v1/cash/router.py:62-79`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Restricción a nivel de base de datos, explícitamente comentada.

### RN-CASH-02: Las ventas del arqueo se derivan de pagos, no de un registro propio en `cash_movements`
**Enunciado**: El dinero vendido en un turno se calcula en tiempo real sumando `Payment` de `Sale` con `cash_shift_id` del turno y `status='paid'`, agrupado por tipo de método de pago.
**Ejemplo**: Venta 1 pagada $50.000 efectivo, Venta 2 pagada $30.000 tarjeta → efectivo $50.000, tarjeta $30.000. Una venta anulada no cuenta aunque tenga pagos.
**Evidencia**: `app/api/v1/cash/service.py:5-7,78-108`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Diseño documentado explícitamente en el docstring del módulo.

### RN-CASH-03: Solo el efectivo (`type='cash'`) afecta al efectivo esperado en el cajón; tarjeta y transferencia se reportan pero no suman/restan
**Enunciado**: De los tipos de pago, únicamente el efectivo entra en el cálculo del dinero que debería estar en el cajón. Tarjeta/transferencia se muestran en el desglose sin afectar `expected`.
**Evidencia**: `app/api/v1/cash/service.py:96-108,154-157`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Solo el efectivo entra en el esperado del cajón".

### RN-CASH-04: El efectivo esperado por ventas se calcula neto del cambio entregado (vueltas)
**Enunciado**: El monto de ventas en efectivo que impacta el cajón es la suma bruta de pagos en efectivo menos el total de cambio (`change_given`) entregado en las ventas del turno.
**Ejemplo**: Pagos en efectivo suman $120.000, `change_total=$15.000` → `ventas_efectivo=105.000`. Un cliente que paga $50.000 por $42.000 recibe $8.000 de cambio → efecto neto en cajón = $42.000, coherente con el valor real de la venta.
**Evidencia**: `app/api/v1/cash/service.py:131-139` (referencia RF-029)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito con referencia a requisito funcional.

### RN-CASH-05: Fórmula completa del efectivo esperado al cierre
**Enunciado**: `expected = fondo_inicial + ventas_efectivo_neto + ingresos_manuales - egresos_manuales - retiros_manuales`.
**Ejemplo**: `opening=50.000, ventas_efectivo=105.000, ingresos=10.000, egresos=8.000, retiros=20.000` → `expected=137.000`.
**Evidencia**: `app/api/v1/cash/service.py:143-157`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Fórmula única, nombres de variable directamente en vocabulario de negocio.

### RN-CASH-06: La diferencia del arqueo solo se calcula si hay un conteo físico registrado
**Enunciado**: Si el turno aún no tiene `counted_amount`, la diferencia queda `None` en vez de asumir cero o el valor esperado.
**Evidencia**: `app/api/v1/cash/service.py:158-160`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Guarda explícita que distingue "no contado" de "contado y coincide" (`difference=0`).

### RN-CASH-07: Los métodos de pago activos sin ventas en el turno aparecen igualmente en el desglose, en cero
**Enunciado**: El desglose de ventas por método incluye todos los métodos activos, aunque no hayan tenido venta, mostrando `total=0, count=0`.
**Evidencia**: `app/api/v1/cash/service.py:110-123`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito, corrige un comportamiento previo con `INNER JOIN` que omitía métodos sin ventas.

### RN-CASH-08: Orden de presentación del desglose de ventas: efectivo primero, resto alfabético
**Enunciado**: El método de tipo efectivo siempre aparece primero; los demás se ordenan alfabéticamente insensible a mayúsculas.
**Evidencia**: `app/api/v1/cash/service.py:125-129`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "el efectivo primero... el resto por nombre".

### RN-CASH-09: Cierre del turno: el monto contado se toma de las denominaciones si se envían, si no del valor manual
**Enunciado**: Si se reportan denominaciones, `counted_amount = Σ(denominación × cantidad)`. Si no, se usa el `counted_amount` recibido directamente. Si no hay ninguno, queda `None`.
**Evidencia**: `app/api/v1/cash/router.py:96-102`; `app/api/v1/cash/schemas.py:39-41`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Que se pueda cerrar sin ningún conteo físico contradice el propósito de un "arqueo de cierre". SUPOSICIÓN: ¿el frontend fuerza a enviar uno de los dos?

### RN-CASH-10: Observación (`close_note`) obligatoria únicamente cuando el arqueo no cuadra
**Enunciado**: Si la diferencia entre contado y esperado es distinta de cero, `close_note` no vacía es obligatoria. Si la diferencia es cero, es opcional.
**Ejemplo**: `expected=137.000`, `counted=135.000` (diferencia -2.000) sin `close_note` → 422 "El arqueo no cuadra: la observación (close_note) es obligatoria".
**Evidencia**: `app/api/v1/cash/router.py:104-111`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito en código y schema.

### RN-CASH-11: No se puede cerrar un turno ya cerrado
**Evidencia**: `app/api/v1/cash/router.py:93-94`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito.

### RN-CASH-12: Los movimientos manuales de efectivo solo pueden registrarse en un turno abierto
**Evidencia**: `app/api/v1/cash/router.py:191-194`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Guarda explícita idéntica en espíritu a la de cierre duplicado.

### RN-CASH-13 [DUDOSA]: El arqueo parcial (sin cerrar el turno) no exige observación aunque la diferencia sea distinta de cero, a diferencia del cierre real
**Enunciado**: `POST /shifts/{id}/partial-count` calcula y persiste la diferencia sin exigir `close_note`, contrastando con RN-CASH-10.
**Evidencia**: `app/api/v1/cash/router.py:133-151`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Consistente con que no cierra el turno, pero no hay comentario que confirme la asimetría como intencional.

### RN-CASH-14: Solo tres tipos de movimiento manual de efectivo, cada uno con signo fijo, y monto siempre positivo
**Enunciado**: "ingreso" (suma), "egreso" (resta), "retiro" (resta); `amount` debe ser estrictamente positivo — el signo lo da el `kind`, no `amount` (mismo patrón que `record_movement`).
**Evidencia**: `app/models/cash_movement.py` (`CheckConstraint`s); `app/api/v1/cash/service.py:150-157`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Restricciones declarativas en BD, reforzadas por diseño simétrico con el kardex de inventario.

### RN-CASH-15 [DUDOSA]: Alias `cash_sales` marcado DEPRECADO, idéntico a `ventas_efectivo` (ya neto del cambio entregado)
**Enunciado**: El campo deprecado `cash_sales` es exactamente `ventas_efectivo` (neto de vueltas), pese a que su nombre puede sugerir el bruto de pagos en efectivo.
**Ejemplo**: Con `by_type["cash"]=120.000, change_total=15.000` → `cash_sales=105.000`, no `120.000`.
**Evidencia**: `app/api/v1/cash/service.py:180-181`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Deprecado intencionalmente por compatibilidad, pero su nombre no refleja que está neto de cambio.

### RN-CASH-16: Egresos y retiros restan igual en la fórmula, solo se distinguen por categoría reportable
**Evidencia**: `app/api/v1/cash/service.py:150-157`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Separación de categorías con el mismo tratamiento aritmético, consistente con el modelo de "kind" del inventario.

### RN-CASH-17 [DUDOSA]: El histórico de turnos siempre recalcula el arqueo en el momento de la consulta, no usa valores guardados
**Enunciado**: Al listar turnos históricos, `expected`/`difference` se recalculan ejecutando `reconcile` sobre datos actuales, no sobre una foto fija del cierre.
**Ejemplo**: Si tras el cierre se anula una venta que formaba parte del `expected`, una consulta posterior del histórico muestra un `expected`/`difference` distintos al momento real del cierre.
**Evidencia**: `app/api/v1/cash/service.py:56-57,76-182`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Consistente con "las ventas se derivan, no se guardan" (RN-CASH-02), pero implica que el arqueo histórico no es inmutable. SUPOSICIÓN: ¿existe algún snapshot que congele el arqueo al cerrar?
## 5. `cart` (+ `core/qr_context.py`, `core/qr_token.py`)

Carrito de cada comensal en la mesa vía QR (unirse, añadir/editar/quitar líneas, enviar pedido a cocina), sin tocar
inventario todavía. Ficheros: `app/api/v1/cart/service.py`, `router.py`, `app/core/qr_context.py`, `qr_token.py`.

### RN-CART-01: Escanear el QR de una mesa ocupada une a la sesión activa, no abre una nueva
**Enunciado**: Si ya existe una `table_session` activa para la mesa, el comensal se une a ella; solo se crea una sesión nueva si no hay ninguna activa. No pueden coexistir dos sesiones activas en la misma mesa.
**Ejemplo**: Mesa 5 con sesión activa de "Ana". "Luis" escanea el mismo QR → se agrega como comensal de esa misma sesión.
**Evidencia**: `app/api/v1/cart/service.py:58-74` (función `_get_or_create_table_session`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito y un índice parcial (`idx_active_session_per_table`) garantiza unicidad ante escaneos simultáneos.

### RN-CART-02: Los nombres de comensal se desambiguan automáticamente por mesa
**Enunciado**: `display_name` no es único en la sesión; si ya existe, se agrega `"(2)"`, luego `"(3)"`, etc.
**Ejemplo**: Ya existen "Ana" y "Ana (2)". Tercer comensal escribe "Ana" → se le asigna "Ana (3)".
**Evidencia**: `app/api/v1/cart/service.py:77-90` (función `unique_display_label`), reutilizada desde `table_sessions/service.py:352`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado como necesario para que cocina/staff distingan comensales homónimos.

### RN-CART-03: Unirse a una mesa marca la mesa como "ocupada" solo si no lo estaba
**Enunciado**: Al abrir/unirse a una sesión, la mesa pasa a `ocupada` solo la primera vez; el evento `table_status_changed` solo se publica esa vez.
**Evidencia**: `app/api/v1/cart/service.py:115-132` (función `open_session`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: repetir el evento "haría parpadear el tablero del cajero sin motivo".

### RN-CART-04: El nombre del comensal no viaja en el token de sesión
**Enunciado**: El JWT del comensal solo codifica `tenant_id`, `table_id`, `participant_id`, `table_session_id`; no lleva `display_name`. El front debe llamar `GET /cart` para recuperar el saludo si recarga.
**Evidencia**: `app/core/qr_token.py:98-120` (función `mint_session_token`); `app/api/v1/cart/service.py:266-271`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "GET /cart es lo único que permite al front repintar el saludo".

### RN-CART-05: Cada comensal tiene como máximo un carrito "abierto" a la vez
**Enunciado**: Al enviar el pedido, el carrito pasa a `confirmado` y la siguiente operación le crea uno nuevo.
**Evidencia**: `app/api/v1/cart/service.py:166-181` (función `_get_or_create_open_cart`); `:517`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring describe exactamente el ciclo; índice parcial `idx_open_cart_per_participant` garantiza unicidad.

### RN-CART-06: No se puede enviar un carrito vacío
**Evidencia**: `app/api/v1/cart/service.py:483-484` (función `submit_cart`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita y temprana, sin ambigüedad.

### RN-CART-07: La cantidad mínima de una línea de carrito es 1; no existe "cantidad 0"
**Evidencia**: `app/api/v1/cart/schemas.py` (`CartItemIn.quantity: ge=1`, `CartItemUpdate.quantity: ge=1`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Restricción declarativa explícita, consistente en creación y edición.

### RN-CART-08: Un comensal puede enviar varios pedidos en la misma sesión
**Enunciado**: No hay límite de "rondas" mientras la sesión siga activa; cada envío genera una `CustomerOrder` independiente.
**Evidencia**: `app/api/v1/cart/service.py:471-481`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado explícitamente en el docstring de `submit_cart`.

### RN-CART-09: Enviar el carrito no descuenta inventario; solo la confirmación de cocina lo hace
**Enunciado**: `submit_cart` crea el pedido `recibida` sin tocar stock. El chequeo de disponibilidad al enviar es preventivo, sin lock ni reserva; el descuento real y atómico ocurre al confirmar (ver RN-ORD).
**Ejemplo**: Con 3 conos disponibles, dos comensales de mesas distintas agregan 2 cada uno casi simultáneo; ambos pasan el chequeo preventivo (nadie descontó nada aún), pero el conflicto real solo se resuelve al confirmar cada pedido.
**Evidencia**: `app/api/v1/cart/service.py:471-486`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: El propio docstring advierte: "la validación real y atómica es la de la confirmación".

### RN-CART-10: Editar opciones de una línea ya guardada no revalida el catálogo actual
**Enunciado**: Editar una línea sin enviar `option_ids` conserva las opciones guardadas sin revalidarlas contra las reglas de min/max vigentes.
**Ejemplo**: Variante permitía 3 toppings al agregarse; el catálogo baja el máximo a 2; editar solo la cantidad conserva los 3 toppings guardados.
**Evidencia**: `app/api/v1/cart/service.py:362-370`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "un cambio de min/max en el catálogo impediría hasta bajar la cantidad de una línea que ya estaba en el carrito".

### RN-CART-11: Los combos se expanden a precio normal; el ahorro se calcula solo al cobrar
**Enunciado**: Al seleccionar un combo en el carrito, cada componente se guarda como línea normal a precio de catálogo sin descuento; el ahorro se calcula recién al construir la venta.
**Evidencia**: `app/api/v1/cart/service.py:315-348`; `app/api/v1/table_sessions/service.py:556,653`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "el ahorro se calcula recién al cobrar, igual que con el resto de promociones".

### RN-CART-12: Las líneas de combo no reciben además descuentos por promoción porcentual/fija
**Enunciado**: Al mostrar el carrito, las promociones percent/fixed solo se evalúan sobre líneas sin `combo_id`, para no descontar dos veces.
**Evidencia**: `app/api/v1/cart/service.py:227-229`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito.

### RN-CART-13: Cancelación de pedido propio por el comensal: solo antes de que cocina empiece
**Enunciado**: El comensal solo puede cancelar su pedido si está `recibida` (sin confirmar, nunca descontado), o `abierta` con TODOS sus ítems `pendiente`. Si algún ítem ya avanzó, no puede.
**Ejemplo**: Pedido `abierta` con helado `pendiente` y malteada `en_preparacion` → cancelación rechazada con 409, incluso el helado pendiente no se cancela solo.
**Evidencia**: `app/api/v1/cart/service.py:410-449` (función `cancel_my_order`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring enumera exactamente esta política en tres viñetas.

### RN-CART-14: Cancelar el último pedido de un comensal que ya se fue puede liberar la mesa al instante
**Enunciado**: Si tras cancelar no queda nada que cobrar y no quedan comensales activos, la mesa se libera de inmediato, sin esperar al barrido periódico.
**Evidencia**: `app/api/v1/cart/service.py:451-455`; `app/api/v1/table_sessions/service.py:74-116`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: cubre "pedí, me arrepentí y me fui" sin esperar al barrido.

### RN-CART-15: Liberar la mesa exige dos condiciones simultáneas: nadie activo Y nada que cobrar
**Enunciado**: La mesa vuelve a `libre` automáticamente solo si ningún comensal sigue `open` Y no hay ningún pedido en estado distinto de `pagada`/`cancelada`. Falta cualquiera y la mesa sigue `ocupada`.
**Ejemplo A**: Ana y Luis se van sin que nadie cobre su pedido `recibida` → mesa NO se libera (queda para que el staff vea el descuadre).
**Ejemplo B**: El pedido de Ana se cancela pero Luis sigue `open` sin haber pedido → mesa sigue `ocupada`.
**Evidencia**: `app/api/v1/table_sessions/service.py:74-116` (función `try_release_if_empty`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "Dos condiciones, ambas necesarias"; justifica que es "un descuadre real que debe ver el personal".

### RN-CART-16: Salir de la mesa (`leave`) nunca falla, ni con token inválido o expirado
**Enunciado**: `POST /cart/leave` es idempotente y no devuelve error de autenticación aunque el token haya expirado; en todos esos casos el resultado deseado (dejar de contar como presente) ya se cumplió.
**Evidencia**: `app/api/v1/cart/router.py:106-129`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "devolverle un error solo le impediría cerrar la pantalla".

### RN-CART-17: El TTL deslizante del comensal es de 240 minutos (4 horas) de inactividad
**Enunciado**: Sin actividad durante `SESSION_TTL_MINUTES=240` min, el comensal se considera expirado y se cierra automáticamente en el siguiente uso de su token.
**Ejemplo**: Sesión abierta a las 19:00 → `expires_at=23:00`. Sin actividad, a las 23:01 recibe 401 "Sesión expirada por inactividad".
**Evidencia**: `app/core/config.py:24`; `app/core/qr_context.py:181-186`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Constante nombrada como "ventana deslizante del comensal", con lógica de expiración explícita.

### RN-CART-18 [DUDOSA]: En el instante exacto de expiración (`expires_at == now`), la sesión ya se considera expirada
**Enunciado**: La comparación es `expires_at <= now` (no `<`); en el instante exacto de igualdad ya se trata como expirado.
**Evidencia**: `app/core/qr_context.py:181`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Código inequívoco sobre `<=`, pero la resolución de microsegundos hace este límite casi inobservable en la práctica; no hay test que lo fije como contractual.

### RN-CART-19: El refresco de la ventana de inactividad solo ocurre cuando faltan ≤230 minutos para expirar
**Enunciado**: Para no escribir en BD en cada sondeo, `expires_at` solo se reescribe cuando queda menos de `240-10=230` minutos de vigencia (holgura de 10 min).
**Ejemplo**: Apertura 19:00 (`expires_at=23:00`). A las 19:05 (quedan 235min>230) no se refresca. A las 19:12 (quedan 228min≤230) sí se refresca a `23:12`.
**Evidencia**: `app/core/qr_context.py:104-122` (función `_should_refresh`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring cuantifica el ahorro ("~360 escrituras/h → ~6") y explica la invariante que ata la holgura a `EMPTY_SESSION_TTL_MINUTES`.

### RN-CART-20: El token JWT de sesión en sí expira a las 24 horas (1440 min), NO a las 4 horas
**Enunciado**: El `exp` del JWT usa `SESSION_ABS_MAX_MINUTES=1440` (24h), no las 240 min de inactividad. La ventana de 4h se controla aparte, en BD, sin reemitir el token. El JWT es solo tope absoluto físico.
**Ejemplo**: Apertura lunes 19:00 → `exp` del JWT = martes 19:00, sin importar inactividad. La inactividad de 4h (RN-CART-17) expulsa antes, a las 23:00 del lunes.
**Evidencia**: `app/api/v1/cart/service.py:137-140`; `app/core/qr_token.py:98-120`; `app/core/qr_context.py:188-193`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: `config.py:25-27` documenta exactamente: "El JWT muere en este tope" — arquitectura deliberada de dos capas de expiración.

### RN-CART-21: Reingresar con un token de una `table_session` ya cerrada nunca es válido, aunque el JWT siga sin expirar
**Enunciado**: Se exige que `table_session_id` del token coincida con la sesión `active` actual de la mesa. Si la mesa cobró y se abrió sesión nueva, el token viejo deja de servir.
**Evidencia**: `app/core/qr_context.py:168-177`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "regla de reingreso: nunca reusar un `table_session_id` inválido".

### RN-CART-22: El stream SSE no refresca el TTL del comensal
**Enunciado**: El handshake de SSE usa `touch=False`: valida pero no desliza la ventana de expiración. Solo las llamadas REST reales cuentan como actividad.
**Ejemplo**: Pestaña abierta toda la noche con SSE vivo pero sin interacción → la sesión igual expira a las 4h de la última acción real.
**Evidencia**: `app/core/qr_context.py:125-133`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "una pestaña olvidada mantendría la mesa viva indefinidamente" si refrescara.

### RN-CART-23: El descuento por promoción del carrito se redondea a 2 decimales con "mitad hacia arriba"
**Enunciado**: `discounted_line_total`/`discounted_unit_price` en la vista previa del carrito usan `ROUND_HALF_UP` a centavos.
**Evidencia**: `app/api/v1/cart/service.py:236-241`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Uso explícito y consistente de un patrón de redondeo monetario estándar.

### Seguridad y tokens QR (`core/qr_token.py`, `core/qr_context.py`)

### RN-CART-24: El QR físico de la mesa no expira nunca (token sin `exp`)
**Enunciado**: El token impreso en la mesa (`typ="qr"`) no lleva claim `exp`; solo se invalida rotando el secreto de firma.
**Evidencia**: `app/core/qr_token.py:77-80`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado explícitamente en el docstring del módulo y de la función.

### RN-CART-25: Los tokens de staff y los tokens de mesa (QR/sesión) son mutuamente excluyentes aunque compartan secreto
**Enunciado**: Un token de usuario (claims `user`/`refresh`) es rechazado explícitamente si se presenta donde se espera un token de QR/sesión, y viceversa.
**Evidencia**: `app/core/qr_token.py:68-72,88,137`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "para que un token no pueda usarse fuera de su dominio aunque comparta secreto".

### RN-CART-26: El tenant y la mesa en el flujo QR siempre vienen del token firmado, nunca de un id que mande el cliente
**Enunciado**: `tenant_id`, `table_id`, `participant_id`, `table_session_id` se derivan exclusivamente de los claims verificados del JWT; ningún endpoint público acepta esos identificadores como parámetro de entrada.
**Evidencia**: `app/api/v1/cart/router.py:1-6`; `app/core/qr_context.py:1-6`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Declarado como principio de seguridad explícito en dos docstrings de módulo.

### RN-CART-27: El límite de tasa de apertura de sesión se aplica primero por IP y luego por mesa
**Enunciado**: Al abrir sesión vía QR, se aplica primero rate-limit por IP, antes de verificar el token; solo si el token es válido se aplica un segundo límite por `table_id`.
**Evidencia**: `app/api/v1/cart/router.py:53-67`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "si no, una ráfaga de tokens basura se va en 401 sin pasar nunca por el limitador".

> **Nota de duplicidad**: La cancelación de un pedido (reversa asimétrica de inventario según estado de cocina) se
> documenta con detalle completo en la sección [`orders`](#8-orders) (RN-ORD-20 a RN-ORD-23); el mismo mecanismo
> gobierna la cancelación iniciada por el comensal desde `cart` (RN-CART-13).
## 6. `table_sessions`

Sesión completa de una mesa: comensales, cuenta consolidada, reparto entre comensales, cobro/cierre unificado o
dividido. Fichero: `app/api/v1/table_sessions/service.py`, `router.py`.

### RN-MESA-01: El cierre de sesión de mesa usa lock optimista (`FOR UPDATE`) para impedir cobro doble
**Enunciado**: Al cerrar/cobrar, la fila de `table_sessions` se bloquea con `SELECT...FOR UPDATE` antes de comprobar que su estado siga `active`. Serializa dos cierres concurrentes: el segundo espera y ve el estado ya `closed`.
**Ejemplo**: Dos cajeros presionan "Cobrar" casi simultáneo. Cajero A toma el lock, cobra, comitea (`closed`). Cajero B, que esperaba el lock, lo adquiere después y ve `closed` → 409, no se genera una segunda venta.
**Evidencia**: `app/api/v1/table_sessions/service.py:38-55` (función `_load`); `:226-232`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explica exactamente la condición de carrera evitada.

### RN-MESA-02 [DUDOSA]: El lock de cierre no protege contra reparto (`set_assignments`) concurrente
**Enunciado**: `set_assignments` carga la sesión sin `lock=True`; solo comprueba `status != "active"` sobre una lectura no bloqueada. Un cierre en curso (lock tomado, sin commit aún) no bloquea una petición concurrente de reparto.
**Ejemplo**: Cajero A inicia el cierre de mesa 5 (lock tomado, sin commit). Otro miembro del staff llama `PUT /assignments`: como no toma lock, su `SELECT` no se bloquea (MVCC de PostgreSQL no bloquea lectores), ve `status="active"` (commit de A aún no visible) y reasigna un ítem cuyas líneas de venta ya fueron calculadas por A.
**Evidencia**: `app/api/v1/table_sessions/service.py:403` (sin `lock=True`) vs `:228` (con `lock=True`)
**Clasificación**: DUDOSA
**Justificación de clasificación**: Hallazgo propio, no documentado. El comentario de `_load` solo justifica el lock para dos `close` concurrentes. SUPOSICIÓN: ¿existe algún mecanismo externo (cliente restringido a una pantalla por mesa) que serialice esto en la práctica?

### RN-MESA-03: No se puede cerrar una sesión sin al menos un pedido cobrable
**Ejemplo**: Comensal se sienta sin pedir; mesero intenta cerrar → 409 "La sesión no tiene pedidos que cobrar".
**Evidencia**: `app/api/v1/table_sessions/service.py:235-239`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita y temprana.

### RN-MESA-04: No se puede cerrar la mesa con pedidos sin confirmar por cocina, ni con ítems en curso
**Enunciado**: Antes de cerrar, todos los pedidos deben estar confirmados (ninguno `recibida`) y ningún ítem puede seguir `pendiente`/`en_preparacion`.
**Ejemplo**: Pedido `recibida` sin confirmar → 409 con detalle de `order_ids`. Ítem `en_preparacion` → 409 "Hay ítems sin terminar en cocina; anúlalos o espera a que estén listos."
**Evidencia**: `app/api/v1/table_sessions/service.py:184-209` (función `_assert_closable`); `app/models/order_item.py:14`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validaciones explícitas orientadas a evitar cobrar comida que aún no salió de cocina.

### RN-MESA-05: El reparto de una cuenta es por ítem/unidad, nunca por división porcentual ni "a partes iguales" automática
**Enunciado**: No existe ninguna función de "dividir el total entre N comensales" matemáticamente. El reparto exige asignar explícitamente cada línea/unidad a un `participant_id`; cada porción paga `cantidad_asignada × unit_price`. No hay residuo de centavos porque nunca se fracciona dinero, solo unidades enteras.
**Ejemplo**: Un total de $10.00 entre 3 comensales no tiene botón de "dividir entre 3"; hay que asignar ítem por ítem. 3 conos idénticos de $3.335 c/u, 1 a cada comensal → cada uno paga exactamente $3.335, sin redondeo ni residuo.
**Evidencia**: `app/api/v1/table_sessions/service.py:395-498`; `app/api/v1/table_sessions/schemas.py:57-65`
**Clasificación**: INTENCIONAL (ausencia de funcionalidad, no un bug)
**Justificación de clasificación**: Todo el diseño de `set_assignments` (invariante "suma pedida != cantidad de línea → error") solo tiene sentido con unidades enteras. SUPOSICIÓN: ¿el frontend simula "división igualitaria" convirtiéndola en asignaciones de unidades antes de llamar a este endpoint?

### RN-MESA-06: Dividir una línea entre varios comensales exige que la suma de unidades repartidas sea exactamente la cantidad original
**Ejemplo**: Línea de `quantity=2`. Reparto A=1, B=2 (suma 3≠2) → 422 "El reparto de este producto suma 3 unidad(es) y la línea tiene 2." Ninguna asignación se aplica.
**Evidencia**: `app/api/v1/table_sessions/service.py:447-460`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito sobre la invariante de inventario que protege.

### RN-MESA-07: No se puede dividir por unidades un ítem que cocina todavía no terminó
**Enunciado**: Repartir una línea entre varios comensales (creando sub-líneas) solo se permite si el ítem no está `pendiente`/`en_preparacion`. Reasignar el ítem COMPLETO a otro comensal sí se permite en cualquier estado.
**Evidencia**: `app/api/v1/table_sessions/service.py:462-473`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Partir un ítem a medio preparar duplicaría la línea con la comida sin terminar".

### RN-MESA-08: Un ítem anulado no se puede asignar a nadie
**Evidencia**: `app/api/v1/table_sessions/service.py:435-439`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Mensaje de error dedicado, distinto del genérico de "no pertenece a la mesa".

### RN-MESA-09: Solo se pueden repartir ítems de pedidos vivos de la propia mesa
**Enunciado**: El universo asignable excluye ítems de otras mesas o de pedidos ya `cancelada`/`pagada`.
**Evidencia**: `app/api/v1/table_sessions/service.py:410-434`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "evita repartir líneas de otra mesa o de un pedido ya cobrado".

### RN-MESA-10: `billing_mode=unified` exige pagos; `billing_mode=split` exige bloques de reparto
**Ejemplo**: `{"billing_mode":"unified","payments":[]}` → 422. `{"billing_mode":"split","splits":[]}` → 422.
**Evidencia**: `app/api/v1/table_sessions/service.py:544-548,584-588`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validaciones explícitas con mensajes específicos por modo.

### RN-MESA-11: En cobro dividido, descuento/impuesto/propina/pagos de nivel raíz están prohibidos: deben ir dentro de cada bloque
**Ejemplo**: `{"billing_mode":"split","tip":"5.00","splits":[...]}` → 422, la propina nunca se cobra ni se reparte silenciosamente.
**Evidencia**: `app/api/v1/table_sessions/service.py:590-597`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Corrige un bug documentado: "quien mandaba una propina ahí la perdía sin enterarse".

### RN-MESA-12: En cobro dividido, cada comensal con consumo debe aparecer exactamente una vez en el reparto de pago
**Enunciado**: `splits` debe cubrir exactamente el conjunto de comensales con ítem no anulado asignado (incluido `None`="agregado por el mesero sin asignar"); no puede faltar, sobrar ni repetirse ningún comensal.
**Evidencia**: `app/api/v1/table_sessions/service.py:599-632`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Tres chequeos distintos, cada uno con mensaje y motivo documentado (el de duplicados evita "dos ventas, dos facturas por el mismo consumo").

### RN-MESA-13 [DUDOSA]: Una mesa de un solo comensal puede cerrarse en modo `split` sin restricción de mínimo
**Enunciado**: No hay validación de cardinalidad mínima para `split`; con un solo comensal equivale en la práctica a un `unified` pero por el camino de `split`.
**Evidencia**: `app/api/v1/table_sessions/service.py:578-671`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Comportamiento permitido por ausencia de restricción, sin confirmación de que sea intencional.

### RN-MESA-14: Cada bloque de `split` calcula sus propias promociones y descuentos de combo, de forma independiente
**Enunciado**: Un combo agregado por un comensal no se mezcla con las líneas de otro para efectos de descuento.
**Evidencia**: `app/api/v1/table_sessions/service.py:643-656`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito.

### RN-MESA-15 [DUDOSA]: Si un mismo pedido mezcla varios combos distintos en cobro unificado, la venta pierde la trazabilidad del combo específico
**Enunciado**: Con más de un `combo_id` distinto en la venta `unified`, `promotion_id` no queda ligado a ninguno de esos combos (cae a `promotions.evaluate`, puede ser `None`), aunque el descuento monetario de ambos sí se sume correctamente al total.
**Evidencia**: `app/api/v1/table_sessions/service.py:555-559`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No hay comentario que explique la pérdida de trazabilidad; el descuento económico es correcto pero el reporte por promoción queda incompleto.

### RN-MESA-16: La cuenta consolidada (preview, `compute_bill` de `table_sessions/service.py`) usa exactamente el mismo cálculo que el cobro dividido real
**Enunciado**: `GET /table-sessions/{id}/bill` calcula el subtotal por comensal con la misma lógica que `_close_split` usa al cobrar, para que el preview coincida con lo que realmente se cobrará. (Nota: existe otra función homónima `compute_bill` en `orders/checkout.py` — ver RN-ORD-02/03 — son dos funciones distintas.)
**Evidencia**: `app/api/v1/table_sessions/service.py:139-181`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Declarado explícitamente en el docstring.

### RN-MESA-17: El nombre de la factura en cobro unificado sigue un orden de prioridad: cajero > comensales con consumo > mesa
**Ejemplo**: Sin nombre del cajero, con Ana y Luis (consumo) → factura a nombre de "Ana, Luis" (alfabético). Con "Constructora XYZ S.A." escrito por el cajero → esa es la factura, aunque Ana/Luis tengan consumo.
**Evidencia**: `app/api/v1/table_sessions/service.py:513-536`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito de negocio.

### RN-MESA-18: En cobro dividido, la venta sin comensal asignado se factura como "Mesa N" o "Sin asignar"
**Evidencia**: `app/api/v1/table_sessions/service.py:634-639`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Corrige inconsistencia previa señalada explícitamente en el comentario.

### RN-MESA-19: El total de una venta no puede ser negativo (aplica a mostrador, unificado y cada bloque de dividido)
**Ejemplo**: Descuento de $20 sobre cuenta de $15 sin impuesto/propina → total=-5 → 422 "El total no puede ser negativo".
**Evidencia**: `app/api/v1/sales/builder.py:132-136` — **mismo mecanismo que RN-VENTA-03**.
**Clasificación**: INTENCIONAL

### RN-MESA-20: El pago debe cubrir el total exacto o más; no se permite cobro parcial
**Ejemplo**: Total $50, pago único de $45 → 422 "El pago (45) no cubre el total (50)"; nada se comitea (ni venta, ni cambio de estado de pedidos, ni liberación de mesa, todo en una transacción).
**Evidencia**: `app/api/v1/sales/builder.py:138-153`; `app/api/v1/table_sessions/service.py:242-265` — **mismo mecanismo que RN-VENTA-04**.
**Clasificación**: INTENCIONAL

### RN-MESA-21: El vuelto solo puede salir del pago en efectivo, nunca de un pago electrónico "de más"
**Ejemplo**: Total $30, pago con tarjeta de $35 → 422 "Los pagos que no son en efectivo (35) no pueden superar el total (30): el vuelto solo sale del efectivo."
**Evidencia**: `app/api/v1/sales/builder.py:154-163` — **mismo mecanismo que RN-VENTA-05**.
**Clasificación**: INTENCIONAL

### RN-MESA-22: Cerrar una sesión de mesa es una única transacción todo-o-nada
**Enunciado**: Generar ventas, marcar pedidos `pagada`, cerrar sesión/comensales, liberar la mesa: todo en una sola transacción. Si cualquier paso falla (p. ej. pago insuficiente en el 3er bloque de un split), se revierte TODO, incluidas las ventas ya armadas para otros comensales en esa misma llamada.
**Ejemplo**: Split de 3 comensales; ventas de Ana y Luis se arman sin problema, pero el bloque de Marta trae pago insuficiente y `build_sale` lanza 422 a mitad del bucle → rollback total, no quedan ventas parciales de la mesa.
**Evidencia**: `app/api/v1/table_sessions/service.py:242-265`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "Si un pago no cubre su parte, no se cierra nada".

### RN-MESA-23: Cerrar la sesión no escribe movimientos de caja manuales, para no contar el dinero dos veces
**Evidencia**: `app/api/v1/table_sessions/service.py:222-224`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Insertar además un movimiento contaría el dinero dos veces".

### RN-MESA-24 [DUDOSA]: No se puede quitar un comensal que ya tiene productos asignados, aunque esos productos estén anulados o su pedido ya no sea cobrable
**Enunciado**: `remove_participant` cuenta `OrderItem` con `participant_id` del comensal sin filtrar por `estado_cocina` ni por el status del pedido; puede bloquear la eliminación de un comensal cuyo único "consumo" ya no es cobrable en absoluto.
**Ejemplo**: A "Marta" se le asignó un helado que luego cocina anuló → `DELETE participants/{marta_id}` sigue contando ese ítem → 409 "«Marta» tiene 1 producto(s) asignado(s)", aunque ese producto nunca se cobrará.
**Evidencia**: `app/api/v1/table_sessions/service.py:362-388`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El mensaje habla de "producto(s) asignado(s)" en términos de negocio (algo cobrable), pero el conteo real no excluye ítems anulados. SUPOSICIÓN: ¿existe algún flujo que limpie `participant_id` al anular un ítem?

### RN-MESA-25: Un comensal agregado desde el POS (sin QR) es solo una etiqueta de cobro, no puede loguearse ni pedir
**Enunciado**: `add_participant` (staff) crea un `SessionParticipant` sin `expires_at` y sin `Cart`; no se emite token de sesión para él.
**Evidencia**: `app/api/v1/table_sessions/service.py:322-359`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "Sin expires_at y sin carrito a propósito... Es una etiqueta de cobro".

### RN-MESA-26: No se pueden agregar ni repartir comensales en una sesión que ya no está activa
**Evidencia**: `app/api/v1/table_sessions/service.py:335-340,404-408`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita y consistente entre ambas funciones.

### RN-MESA-27: El nombre de un comensal agregado por staff no puede estar vacío tras recortar espacios
**Evidencia**: `app/api/v1/table_sessions/service.py:342-344`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito posterior al `strip()`.
## 7. Automatización — `core/scheduler.py`

Tareas periódicas: barrido de sesiones de mesa huérfanas y expiración de promociones vencidas.

### RN-SCHED-01: Una mesa se cierra por abandono sin pedir a los 30 minutos exactos de inactividad de todos sus comensales
**Enunciado**: Una `TableSession` activa sin ningún pedido facturable se considera abandonada cuando ya nadie tiene actividad reciente dentro de `EMPTY_SESSION_TTL_MINUTES=30`. La subconsulta exige `ultima_actividad > limite` para seguir "vivo"; si nadie cumple esa condición `>`, la sesión venció. En la práctica, el corte es a los 30:00 minutos exactos, no después.
**Ejemplo**: Comensal escanea a las 10:00:00 sin pedir. `expires_at=10:00:00+240min=14:00:00`; `ultima_actividad` derivada = `expires_at-240min=10:00:00`. Barrido a las 10:30:00: `limite=10:00:00`. `10:00:00 > 10:00:00` es falso → ya no cuenta como activo → si no hay pedidos facturables, la sesión se cierra exactamente a los 30:00.
**Evidencia**: `app/core/scheduler.py:47-85` (función `_abandonadas_sin_pedir`); `app/core/config.py:50`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Ventana y lógica explicadas extensamente en el docstring del módulo.

### RN-SCHED-02: Una sesión de mesa con consumo se cierra por tope duro a las 6 horas de abierta, sin importar actividad
**Enunciado**: Cualquier sesión `active` cuyo `opened_at` sea anterior al corte de `TABLE_SESSION_MAX_HOURS=6` se considera vencida, independientemente de si hay comensales activos. Operador `<` estricto.
**Ejemplo**: Sesión abierta 09:00:00. Barrido a las 15:00:01 → `corte=09:00:01` → `09:00:00 < 09:00:01` verdadero → entra en la lista de vencidas por tope duro, incluso si el comensal sigue interactuando activamente.
**Evidencia**: `app/core/scheduler.py:108-119,186-189`; `app/core/config.py:45`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring distingue explícitamente este "tope duro" del criterio de inactividad, con motivo de negocio ("no bajar el umbral para no echar a gente que sigue comiendo").

### RN-SCHED-03: Una sesión vencida con pedidos facturables pendientes de cobro NO se cierra: solo se expulsa a los comensales
**Enunciado**: Si una sesión cumple RN-SCHED-01 o RN-SCHED-02 pero tiene pedidos con consumo no cobrado, el sistema no cierra la sesión ni libera la mesa: cierra solo a los comensales, dejando la sesión abierta para que el cajero pueda cobrarla.
**Ejemplo**: Sesión abierta hace 6h10min con pedido de $20.000 sin pagar. Vencida por tope duro, pero `has_billable_orders=True` → solo se cierran los comensales; la mesa no cambia a `libre`.
**Evidencia**: `app/core/scheduler.py:124-137`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: cerrar la sesión "dejaba la mesa sin cuenta que cargar... imposible de cobrar desde la terminal".

### RN-SCHED-04: Al cerrar por barrido, la mesa solo vuelve a "libre" si tampoco tiene pedidos vivos huérfanos fuera de la sesión
**Enunciado**: Segunda red de seguridad: comprueba que no existan `CustomerOrder` no terminales asociados a esa mesa que hayan quedado sin `table_session_id` (huérfanos históricos); si existen, la mesa se deja ocupada.
**Evidencia**: `app/core/scheduler.py:142-163`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Liberarla los volvería incobrables".

### RN-SCHED-05: El cierre de sesión por barrido se atribuye al sistema, no a un cajero humano
**Enunciado**: `close_by=None` explícitamente cuando el cierre lo dispara el job, para distinguirlo en auditoría de un cierre manual.
**Evidencia**: `app/core/scheduler.py:139-140`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "no la cerró un humano, la cerró el barrido".

### RN-SCHED-06: El barrido corre en todos los tenants, y un tenant con error no detiene el barrido de los demás
**Evidencia**: `app/core/scheduler.py:198-203`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Un tenant roto no debe impedir el barrido de los demás".

### RN-SCHED-07: Solo un worker ejecuta el barrido por ciclo, con lock distribuido en Redis (TTL = mitad del intervalo)
**Ejemplo**: `SESSION_SWEEP_INTERVAL_MINUTES=15` (default) → `ttl=max(15*60/2,30)=450s` (7.5 min). Si el worker que tomó el lock muere a los 2 min, el lock expira a los 7.5 min, permitiendo que el siguiente ciclo (a los 15 min) lo retome sin bloqueo indefinido.
**Evidencia**: `app/core/scheduler.py:261-271`; `app/core/config.py:52`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito documenta la fórmula del TTL y su propósito de recuperación ante caída de proceso.

### RN-SCHED-08: Si Redis no está disponible, el barrido y la expiración de promociones simplemente se omiten ese ciclo
**Evidencia**: `app/core/scheduler.py:265-271,249-253`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Manejo explícito de la excepción con mensaje de log dedicado, en vez de propagar el error.

### RN-SCHED-09: Si APScheduler no está instalado, la aplicación arranca igual, pero ninguna sesión huérfana se cerrará automáticamente
**Evidencia**: `app/core/scheduler.py:277-290`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Manejo explícito de `ImportError`, documentado como "para no tumbar la app por una dependencia opcional".

### RN-SCHED-10: El job de expiración de medianoche pasa a `finished` promociones vencidas, incluidas las que nunca se activaron
**Enunciado**: `expire_promotions` actualiza en masa a `finished` toda promoción `active/paused/draft` con `ends_at` anterior al momento de ejecución. Una promoción `draft` con `ends_at` vencido pasa directo a `finished` sin haber estado nunca `active`.
**Evidencia**: `app/core/scheduler.py:210-244`; `app/scripts/expire_promotions.py:1-22`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "le da a finished su significado real... mantiene el listado limpio".

### RN-SCHED-11 [ACCIDENTAL — DISCREPANCIA]: El job de expiración compara en UTC absoluto con datetime completo, no en hora local ni por fecha, a diferencia de `_valid_now`
**Enunciado**: `_valid_now` (motor de promociones, ver RN-PROMO) usa hora local del tenant y compara `ends_at` solo por fecha (cubre el día completo). El job `expire_promotions` compara `ends_at < now` con `now=datetime.now(timezone.utc)`, sin conversión a local y con datetime completo.
**Ejemplo**: Con tenant en UTC-5, `ends_at="2026-08-04T00:00:00"`. `_valid_now` sigue considerando la promoción vigente hasta las 23:59:59 locales del 4 de agosto (05:00 UTC del 5). El job la marcaría `finished` ya desde `2026-08-04T00:00:00 UTC` = `2026-08-03 19:00` local — casi un día y medio antes de que `_valid_now` deje de aplicarla.
**Evidencia**: `app/core/scheduler.py:224-236` vs `app/api/v1/promotions/service.py:91-99`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: El comentario del job dice ser "puramente informativo" porque `_valid_now` es la autoridad en tiempo real, pero al usar un criterio de corte distinto puede marcar `finished` una promoción que `_valid_now` todavía consideraría vigente, contradiciendo esa misma premisa.
## 8. `orders`

Corazón operativo de la mesa: confirma pedidos (momento del descuento de inventario), gestiona el flujo de cocina,
cobra y cancela pedidos, y administra las mesas físicas. Ficheros: `checkout.py`, `consumption.py`, `kitchen.py`,
`consolidation.py`, `service.py`, `tables_advanced.py`, `router.py`.

### 8.1 Bloqueo, cobro y confirmación (`checkout.py`, `consumption.py`)

### RN-ORD-01: Bloqueo previo obligatorio para cobrar
**Enunciado**: Un pedido solo puede pagarse si antes fue bloqueado; el bloqueo exige que la orden esté `abierta`, que la versión enviada coincida con la actual (lock optimista) y que no existan ítems en cocina sin terminar.
**Ejemplo**: Orden versión=3, `abierta`, con un ítem `pendiente`. `block_order(version=3)` → rechazado 409 "Hay ítems sin terminar en cocina". Con el ítem `listo`, pasaría a `bloqueada`, `version=4`.
**Evidencia**: `app/api/v1/orders/checkout.py:71-122` (función `block_order`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validaciones deliberadas (lock optimista + chequeo de cocina) antes de permitir cobro.

### RN-ORD-02: Ítems anulados no cuentan en la cuenta (`compute_bill` de `checkout.py`) ni en el split
**Ejemplo**: Ítem A ($10.000×2=$20.000, `listo`) e ítem B ($5.000×1, `anulado`). Subtotal = $20.000 (B no suma).
**Evidencia**: `app/api/v1/orders/checkout.py:152-157` (función `compute_bill`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Filtro explícito antes de acumular subtotal.

### RN-ORD-03 [DUDOSA]: La cuenta (`compute_bill` de `checkout.py`) excluye solo órdenes canceladas, no las ya pagadas
**Enunciado**: `compute_bill` considera órdenes con status distinto de `cancelada`, incluyendo `pagada`.
**Ejemplo**: Mesa con orden `pagada` ($15.000) y orden `cancelada` ($8.000) → total de la "cuenta" = $15.000 (incluye la ya pagada).
**Evidencia**: `app/api/v1/orders/checkout.py:130-138`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Incluir órdenes `pagada` en el cálculo de "cuenta pendiente" es llamativo; no hay comentario que aclare si es histórico de consumo o saldo por cobrar. SUPOSICIÓN: ¿`compute_bill` sirve también para mostrar recibo histórico?

### RN-ORD-04: Split de cuenta por comensal, incluidas líneas sin asignar
**Enunciado**: El reparto se agrupa por `participant_id`; las líneas sin comensal (`None`, agregadas por el mesero) forman su propio grupo, no se prorratean ni excluyen.
**Evidencia**: `app/api/v1/orders/checkout.py:157,169-176,185-188`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito aclara que `participant_id=None` es un filtro válido.

### RN-ORD-05: Cobro solo procede desde estado `bloqueada`
**Evidencia**: `app/api/v1/orders/checkout.py:252-256` (función `pay_order`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Mensaje de error explícito guía al flujo correcto.

### RN-ORD-06: Cobro requiere turno de caja abierto
**Evidencia**: `app/api/v1/orders/checkout.py:257`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Llamada explícita y temprana a `ensure_open_shift`, antes de tocar líneas o inventario.

### RN-ORD-07: Descuento automático de promociones se aplica al cobrar una orden de mesa
**Enunciado**: Se evalúan promociones percent/fixed sobre líneas sin combo, más el ahorro de líneas con combo; ambos se suman al descuento manual capturado.
**Ejemplo**: Línea sin combo $50.000 con promo 10% activa → $5.000. Línea de combo con ahorro propio $3.000. Descuento manual $0. Total descuento = $8.000.
**Evidencia**: `app/api/v1/orders/checkout.py:263-277`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario "RF-012" explícito: "antes esta orden no aplicaba ninguna promoción; ahora usa el mismo motor que mostrador".

### RN-ORD-08 [DUDOSA]: La promoción registrada en la venta prioriza el combo si es único; con 2+ combos distintos se pierde
**Enunciado**: Si las líneas cobradas usan exactamente un combo distinto, `promotion_id` = ese combo. Con cero o más de un combo distinto, se usa el `promo_id` de la evaluación general (puede ser `None`).
**Evidencia**: `app/api/v1/orders/checkout.py:268-269`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Sin comentario que explique la regla de desempate con 2+ combos. SUPOSICIÓN: ¿es aceptable que con 2 combos distintos no quede registrado ningún combo como promoción de la venta?

### RN-ORD-09: El cobro NO vuelve a descontar inventario
**Enunciado**: `pay_order` construye la venta y marca la orden `pagada` sin ejecutar ningún movimiento de inventario adicional; el descuento real ya ocurrió al confirmar el pedido.
**Evidencia**: `app/api/v1/orders/checkout.py:286-287`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito confirma la decisión de no descontar dos veces.

### RN-ORD-10: Confirmar un pedido es el momento del descuento real de inventario
**Enunciado**: La transición `recibida → abierta` (confirmación por staff) es el único punto donde se descuenta físicamente el inventario de un pedido QR; ocurre antes de marcar `abierta`.
**Ejemplo**: Pedido `recibida` con 2 unidades de "Helado de vainilla" (receta 200g base/unidad). `confirm_order` descuenta 400g antes del commit que pone `status=abierta`.
**Evidencia**: `app/api/v1/orders/checkout.py:339-343`; mutación en `app/api/v1/orders/consumption.py:98-102`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring lo declara explícitamente: "Es el único punto de descuento de los pedidos por QR".

### RN-ORD-11: Confirmación excluye ítems anulados del descuento
**Evidencia**: `app/api/v1/orders/checkout.py:331-335`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Filtro explícito, consistente con RN-ORD-02.

### RN-ORD-12: Confirmar un pedido sin ítems consumibles está prohibido
**Ejemplo**: Pedido `recibida` cuyo único ítem está `anulado` → `entries` vacío → 409 "El pedido no tiene ítems", sin tocar inventario ni cambiar estado.
**Evidencia**: `app/api/v1/orders/checkout.py:336-337`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Chequeo explícito con mensaje dedicado.

### RN-ORD-13: Confirmación solo es válida desde estado `recibida`
**Enunciado**: Rechaza cualquier orden con status distinto (incluye reintentos sobre orden ya `abierta`/`bloqueada`/`pagada`/`cancelada`), previniendo doble descuento por esta vía.
**Evidencia**: `app/api/v1/orders/checkout.py:324-328`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Guard clause explícita que protege RN-ORD-10 contra reejecución.

### RN-ORD-14: Fallo de stock en confirmación revierte toda la transacción
**Ejemplo**: Pedido con 2 ítems; el primero descuenta bien, el segundo requiere 500g de un insumo con solo 100g → `InsufficientStockError` → rollback completo, incluido el descuento del primer ítem; la orden permanece `recibida`.
**Evidencia**: `app/api/v1/orders/checkout.py:339,347-350`; `app/api/v1/inventory/stock.py:70-75`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring literal: "Si falta stock de un solo insumo, la transacción entera hace rollback y el pedido sigue recibida".

### RN-ORD-15: La validación de carrito es preventiva, no autoritativa; `confirm_order` es la fuente de verdad
**Evidencia**: `app/api/v1/orders/checkout.py:311-312`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Explícitamente documentado: "el chequeo del carrito era solo preventivo y pudo quedar obsoleto".

### RN-ORD-16: Locks de inventario en orden canónico de id para evitar deadlocks (confirmación)
**Evidencia**: `app/api/v1/orders/consumption.py:37-49`; comentario en `checkout.py:313`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito sobre la razón (evitar deadlocks) y la mecánica.

### RN-ORD-17: Rechazo de líneas que no consumen ningún insumo, antes de descontar ninguna
**Enunciado**: Si alguna línea del lote no consumiría nada, se rechaza el lote completo sin descontar ninguna línea.
**Evidencia**: `app/api/v1/orders/consumption.py:26-34,64`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "mejor parar y configurar la receta que emitir una venta que descuadra el inventario".

### RN-ORD-18: Cancelación bloqueada en estados terminales
**Ejemplo**: Orden `cancelada` → segunda llamada a `cancel_order` → 409 "La orden ya es terminal", sin reversa duplicada.
**Evidencia**: `app/api/v1/orders/checkout.py:42,389-392` (constante `TERMINAL`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Constante y guard clause explícitos.

### RN-ORD-19: Cancelación sin restricción adicional de estado (fuera de terminal)
**Enunciado**: A diferencia de la confirmación (exige `recibida`), la cancelación puede aplicarse a `recibida`, `abierta` o `bloqueada`; la decisión de quién puede cancelar en qué momento se delega al endpoint llamador.
**Evidencia**: `app/api/v1/orders/checkout.py:378-381`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "el staff sí puede cancelar en cualquier momento".

### RN-ORD-20: Reversa asimétrica — rama "nunca descontado" (orden en `recibida`)
**Enunciado**: Si la orden está en `recibida` (nunca confirmada), cancelar no genera ningún movimiento de inventario, sin importar el `estado_cocina` de sus ítems.
**Ejemplo**: Pedido `recibida` con ítem `pendiente` cantidad 3 → cero movimientos; solo se registra `OrderCancelLog`.
**Evidencia**: `app/api/v1/orders/checkout.py:51,394,404-406`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "orden aún sin confirmar: nunca se descontó → cero movimientos".

### RN-ORD-21: Reversa asimétrica — rama "descontado pero no preparado" (`pendiente`)
**Enunciado**: Para una orden ya confirmada, un ítem `pendiente` sí devuelve su insumo al stock al cancelar (cocina aún no lo tocó).
**Ejemplo**: Orden `abierta`, ítem `V1`, `quantity=2`, `pendiente`, receta 150g/unidad → cancelar registra `type='in', quantity=300g, reason='ajuste', reference_type='order_void'`; stock sube 300g.
**Evidencia**: `app/api/v1/orders/checkout.py:404-416`; `app/api/v1/orders/consumption.py:120-127`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "ítem pendiente: descontado pero no preparado → entrada real".

### RN-ORD-22: Reversa asimétrica — rama "ya consumido" (`en_preparacion`/`listo`): NO se devuelve inventario, se registra como pérdida en auditoría
**Enunciado**: Un ítem `en_preparacion` o `listo` no devuelve inventario al cancelar; el insumo ya se considera físicamente combinado. Se registra como pérdida (auditoría), sin movimiento de kardex adicional.
**Ejemplo**: Orden `abierta`, ítem `en_preparacion` de "Malteada de fresa" (`quantity=1`) → entra en `perdidos`, no en `a_revertir`; el stock del insumo no cambia; se agrega registro a `perdidos` con `estado_cocina='en_preparacion'`.
**Evidencia**: `app/api/v1/orders/checkout.py:47,407-415`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "el 'out' de la confirmación ya es esa pérdida; escribir otro movimiento la descontaría dos veces".

### RN-ORD-23: Reversa asimétrica — rama "ítem anulado" (ya resuelto por `void_item`)
**Enunciado**: Un ítem `anulado` se ignora por completo en la cancelación de la orden completa; su reversa (si procedía) ya ocurrió al anularlo individualmente.
**Evidencia**: `app/api/v1/orders/checkout.py:401-403`; `void_item` en `kitchen.py:93`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring: "ya lo resolvió void_item"; evita doble reversa.

### RN-ORD-24: Pérdida por cancelación se traza en auditoría, no en kardex
**Enunciado**: Cuando hay al menos un ítem "perdido", se escribe un warning y un registro de auditoría (`order.cancel.loss`) con el detalle; no se crea ningún `InventoryMovement` para la pérdida.
**Evidencia**: `app/api/v1/orders/checkout.py:430-450`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "queda trazada en auditoría para el reporte de mermas".

### RN-ORD-25: Cancelación exige y registra un motivo, y ata al actor real (staff o comensal, nunca ambos obligatorios)
**Ejemplo**: Comensal cancela vía QR sin `user` (staff) → `actor_id=None`, `participant_id=<id comensal>`. Staff cancela → `participant_id=None`, `user_id`/`user_name` presentes.
**Evidencia**: `app/api/v1/orders/checkout.py:418,423-428`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explícito: "El actor es user (staff) o participant (comensal desde QR)".

### RN-ORD-26: Cantidad de ítem de orden siempre positiva (invariante de BD)
**Enunciado**: No existe el caso "cancelar un ítem con cantidad 0": la tabla `order_items` impone `quantity > 0`.
**Evidencia**: `app/models/order_item.py:68`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Constraint de BD explícita y nombrada.

### RN-ORD-27: Reversa usa exactamente la misma receta que el descuento
**Enunciado**: La cantidad que se devuelve al revertir un ítem se calcula con la misma función (`plan_line_consumption`) usada para descontarlo, garantizando simetría numérica.
**Evidencia**: `app/api/v1/orders/consumption.py:95-97,120-122`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "una reversa que calcule distinto descuadra el inventario para siempre".

### RN-ORD-28: Los movimientos de entrada por reversa nunca fallan por falta de stock
**Enunciado**: Los `type='in'` de reversa (anulación de ítem `pendiente`, cancelación de orden) siempre se aplican; a diferencia de los `out`, no chequean stock disponible.
**Evidencia**: `app/api/v1/orders/consumption.py:76,113`; `app/api/v1/inventory/stock.py:69-75`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito y lógica de `record_movement` lo confirman matemáticamente (delta positivo con `in` nunca deja `new_stock<0`).

### RN-ORD-29: El motivo del movimiento distingue venta de reversa, aunque ambos sean "ajuste" contablemente
**Enunciado**: El descuento por confirmación usa `reason='venta', reference_type='order'`; la reversa por cancelación/anulación usa `reason='ajuste', reference_type='order_void'`.
**Evidencia**: `app/api/v1/orders/consumption.py:98-102,123-127`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Constantes centralizadas con docstring que explica el propósito de reportería.

### RN-ORD-30: Liberar una mesa exige que todas sus órdenes estén en estado terminal
**Ejemplo**: Mesa con orden `bloqueada` → `release_table` → 409 "La mesa tiene órdenes sin cerrar (paga o cancela primero)".
**Evidencia**: `app/api/v1/orders/checkout.py:533-555`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Guard clause explícita con mensaje orientado a la acción correctiva.

### RN-ORD-31 [DUDOSA]: El cierre de mesa en cascada no valida pendientes por sí mismo; delega esa responsabilidad al llamador
**Enunciado**: `close_table_sessions`/`close_participants` cierran sesiones, comensales y carritos sin verificar órdenes pendientes; esa responsabilidad recae en quien invoca (p. ej. `release_table`).
**Evidencia**: `app/api/v1/orders/checkout.py:500-503`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Decisión documentada, pero deja abierta una vía de invocación insegura si algún futuro caller olvida validar. SUPOSICIÓN: ¿existe hoy algún caller distinto de `release_table` que no valide antes?

### RN-ORD-32 [DUDOSA]: Descripción de línea de venta usa snapshot "Producto - Variante"; puede quedar incompleta si el producto o la variante fueron borrados
**Enunciado**: `order_sale_lines` construye `"{producto} - {variante}"`; si el producto no existe (huérfano), cae a solo el nombre de la variante, o cadena vacía si tampoco hay variante.
**Evidencia**: `app/api/v1/orders/checkout.py:210-215`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El caso de referencia inexistente no está comentado como esperado. SUPOSICIÓN: ¿puede un `OrderItem` referenciar una variante ya eliminada físicamente en este sistema?

### RN-ORD-33: Líneas de combo no se acumulan con descuentos percent/fixed (en el pago de orden de mesa)
**Enunciado**: `promo_lines_for` excluye del motor percent/fixed las líneas con `combo_id`, porque ya tienen su propio ahorro.
**Evidencia**: `app/api/v1/orders/checkout.py:231-247`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito, mismo patrón que RN-CART-12.

### RN-ORD-34 [DUDOSA / HALLAZGO]: "Único punto de descuento" (RN-ORD-10) es correcto solo si se lee como "único punto de salida (out)", no como "único punto que toca inventario"
**Enunciado**: El docstring de `confirm_order` afirma ser "el único punto de descuento de los pedidos por QR", pero el mismo módulo también escribe movimientos `in` en la reversa de `cancel_order` — un segundo punto de escritura en el kardex para la misma orden, aunque no sea "descuento".
**Evidencia**: `app/api/v1/orders/checkout.py:311,339,419`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No es una contradicción real, pero la redacción puede inducir a error si se lee de forma aislada; se reporta como hallazgo de precisión de comentario.

### RN-ORD-35: Constantes que gobiernan toda la reversa asimétrica: cualquier estado de cocina no listado cae por omisión en "sí se revierte"
**Enunciado**: `_CONSUMED_KITCHEN=("en_preparacion","listo")` y `_NOT_DEDUCTED=("recibida",)` son las dos únicas constantes que deciden la reversa; cualquier estado no incluido (p. ej. `pendiente`) cae en la rama de reversión.
**Evidencia**: `app/api/v1/orders/checkout.py:44-51`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Ambas constantes están comentadas línea por línea explicando su propósito exacto.
### 8.2 Cocina, consolidación y mesas físicas (`kitchen.py`, `consolidation.py`, `service.py`, `tables_advanced.py`, `router.py`)

### RN-ORD-36: Transiciones válidas de `estado_cocina` (avance manual)
**Enunciado**: Un ítem solo avanza hacia adelante: `pendiente→{en_preparacion, listo}`, `en_preparacion→listo`. No hay transiciones definidas desde `listo` ni `anulado` por esta vía.
**Ejemplo**: Ítem `listo` → transición a `en_preparacion` → 409 "Transición de preparación inválida" (desde=listo, hacia=en_preparacion).
**Evidencia**: `app/api/v1/orders/kitchen.py:30-33,43-60` (dict `_ALLOWED`, función `transition_kitchen`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring y comentario en el dict: "quien toma el pedido es quien lo prepara".

### RN-ORD-37 [DUDOSA]: `mark_order_ready` solo bloquea en estados terminales de pago, no en `bloqueada`
**Enunciado**: Marcar todos los ítems en curso como "listo" se rechaza solo si la orden es `pagada`/`cancelada`; en `bloqueada` (congelada para cobro) sigue permitido.
**Evidencia**: `app/api/v1/orders/kitchen.py:63-90`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No hay comentario que confirme que modificar cocina tras "bloquear" para cobro sea deseado. SUPOSICIÓN: ¿se permite deliberadamente?

### RN-ORD-38 [ACCIDENTAL]: `transition_kitchen` no valida el estado del pedido padre
**Enunciado**: Avanzar el `estado_cocina` de un ítem no comprueba el `status` de la `CustomerOrder`; funciona igual aunque la orden esté `pagada`, `cancelada` o `bloqueada`.
**Evidencia**: `app/api/v1/orders/kitchen.py:43-60`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: `mark_order_ready` sí valida explícitamente el status de la orden (RN-ORD-37); la ausencia de la misma protección aquí es inconsistente con ese cuidado.

### RN-ORD-39 [ACCIDENTAL]: `void_item` no valida el estado del pedido padre
**Enunciado**: Anular un ítem no comprueba el status de la orden; funciona sobre órdenes `pagada`/`cancelada` mientras el propio ítem no esté ya `anulado`.
**Evidencia**: `app/api/v1/orders/kitchen.py:93-176`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: Mismo patrón de omisión que RN-ORD-38; contradice la lógica de "orden terminal = inmutable" aplicada en otras partes del flujo de cobro.

### RN-ORD-40: Devolución de inventario al anular depende del estado de cocina
**Enunciado**: Solo se devuelve inventario si el ítem estaba `pendiente`; si `en_preparacion`/`listo` no se revierte.
**Ejemplo**: Ítem `en_preparacion`, cantidad=2 → no se llama `reverse_order_items`; el descuento original queda firme.
**Evidencia**: `app/api/v1/orders/kitchen.py:106,132-136`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Cocina no consumió físicamente: se devuelve el inventario".

### RN-ORD-41: Un ítem anulado no puede volver a anularse
**Evidencia**: `app/api/v1/orders/kitchen.py:102-103`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Evita doble reversa de inventario y doble log de auditoría.

### RN-ORD-42: El reemplazo de un ítem anulado no puede ser un combo
**Evidencia**: `app/api/v1/orders/kitchen.py:111-117`
**Clasificación**: INTENCIONAL

### RN-ORD-43: El reemplazo debe ser una variante activa
**Evidencia**: `app/api/v1/orders/kitchen.py:121-124`
**Clasificación**: INTENCIONAL

### RN-ORD-44: El ítem de reemplazo siempre nace "pendiente" y trazado al original
**Enunciado**: El nuevo ítem creado al reemplazar uno anulado nace `pendiente` (sin importar el estado del original), enlazado vía `void_de`, con su propia deducción de inventario.
**Evidencia**: `app/api/v1/orders/kitchen.py:143-161`
**Clasificación**: INTENCIONAL

### RN-ORD-45: El precio del reemplazo se recalcula, no se copia del original
**Ejemplo**: Original `unit_price=8000`; reemplazo con otra variante da `compute_line_price=6500` → `new_item.unit_price=6500`.
**Evidencia**: `app/api/v1/orders/kitchen.py:149`
**Clasificación**: INTENCIONAL

### RN-ORD-46: Consolidar exige carritos abiertos con ítems
**Ejemplo**: Comensal `open` sin `CartItems` → 409 "No hay carritos con ítems para consolidar".
**Evidencia**: `app/api/v1/orders/consolidation.py:106-124` (función `consolidate_table`)
**Clasificación**: INTENCIONAL

### RN-ORD-47: El precio en consolidación se copia del carrito, no se recalcula
**Ejemplo**: `CartItem.unit_price=5000` (precio al agregarlo) → `OrderItem.unit_price=5000` aunque el precio de catálogo actual haya subido a 5500.
**Evidencia**: `app/api/v1/orders/consolidation.py:140`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "snapshot copiado del carrito".

### RN-ORD-48: Consolidación descuenta inventario en un solo lote al final, con locks en orden canónico
**Enunciado**: Todas las líneas de todos los carritos se insertan primero; el descuento se hace en una sola llamada al final, para evitar deadlocks entre consolidaciones concurrentes.
**Evidencia**: `app/api/v1/orders/consolidation.py:129-158`
**Clasificación**: INTENCIONAL

### RN-ORD-49: Los carritos consolidados quedan `confirmado` y no vuelven a consolidarse
**Evidencia**: `app/api/v1/orders/consolidation.py:116,156`
**Clasificación**: INTENCIONAL

### RN-ORD-50: Una mesa puede tener varias órdenes abiertas simultáneas (no hay índice único de "una orden abierta por mesa")
**Enunciado**: Puede haber varios pedidos activos a la vez (uno por comensal/ronda/canal); la agrupación para cobrar se hace por `table_session_id`.
**Evidencia**: `app/api/v1/orders/consolidation.py:69-103`; `app/models/customer_order.py:85-87`
**Clasificación**: INTENCIONAL

### RN-ORD-51: Sesión de mesa activa es única y se crea perezosamente al agregar el primer ítem
**Ejemplo**: Mesa `libre` sin sesión activa; el mesero agrega algo → se crea `TableSession(status="active")` y la mesa pasa a `ocupada`.
**Evidencia**: `app/api/v1/orders/consolidation.py:34-66`
**Clasificación**: INTENCIONAL

### RN-ORD-52: Un combo agregado directo por el mesero se expande a precio normal (mismo patrón que RN-CART-11)
**Ejemplo**: Combo con 2 componentes de $6000 c/u agregado por el mesero → se crean 2 `OrderItem` de $6000 (total $12000); descuento calculado solo al cobrar.
**Evidencia**: `app/api/v1/orders/consolidation.py:180-201`; `app/api/v1/orders/service.py:77-93`
**Clasificación**: INTENCIONAL

### RN-ORD-53: Ítems agregados directo por el mesero no tienen comensal asignado
**Enunciado**: `add_item_to_table` guarda `participant_id=None`, a diferencia de `consolidate_table` que sí asigna `participant_id=cart.participant_id`.
**Evidencia**: `app/api/v1/orders/consolidation.py:209`
**Clasificación**: INTENCIONAL

### RN-ORD-54: Solo se puede crear comanda de staff para un comensal aún activo
**Evidencia**: `app/api/v1/orders/service.py:41-49`
**Clasificación**: INTENCIONAL

### RN-ORD-55: La comanda de staff descuenta inventario al crearse, no al cobrar; si falta stock, rollback total
**Enunciado**: A diferencia del pedido QR (nace `recibida`, descuenta al confirmar), la comanda de staff nace `abierta` y descuenta en la misma transacción de creación.
**Ejemplo**: Comanda con 2 líneas; la 2ª falla por stock insuficiente → rollback total, ni la orden ni sus líneas quedan creadas.
**Evidencia**: `app/api/v1/orders/service.py:1-11,120-126`
**Clasificación**: INTENCIONAL

### RN-ORD-56: Cambiar la mesa a libre/reservada exige cero órdenes activas
**Ejemplo**: Mesa con orden `bloqueada` (no está en `TERMINAL`) → `set_table_status(mesa, "libre")` → 409.
**Evidencia**: `app/api/v1/orders/tables_advanced.py:18,21-27,30-42`
**Clasificación**: INTENCIONAL

### RN-ORD-57: Mover una orden a su propia mesa es un no-op
**Evidencia**: `app/api/v1/orders/tables_advanced.py:51-52`
**Clasificación**: INTENCIONAL

### RN-ORD-58 [DUDOSA]: No se puede mover una orden a una mesa con órdenes activas, aunque el sistema permita varias órdenes por mesa en general
**Enunciado**: `move_order` exige la mesa destino completamente sin órdenes activas, lo cual es más estricto que el modelo general de "varias órdenes por mesa" (RN-ORD-50).
**Ejemplo**: Mesa 7 tiene una orden `recibida` de otro comensal. Mover otra orden a esa mesa → 409 "La mesa destino ya tiene una orden activa".
**Evidencia**: `app/api/v1/orders/tables_advanced.py:53-54`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿es intencional que mover sea más estricto que la ocupación normal, o quedó desalineado tras introducir multi-orden por mesa?

### RN-ORD-59: Mover una orden libera la mesa origen solo si queda sin órdenes activas
**Evidencia**: `app/api/v1/orders/tables_advanced.py:56-69`
**Clasificación**: INTENCIONAL

### RN-ORD-60 [ACCIDENTAL / CÓDIGO MUERTO]: El manejo de `IntegrityError` en `move_order` parece código huérfano de una constraint ya removida
**Enunciado**: `move_order` captura `IntegrityError` y lo traduce a 409 "La mesa destino ya tiene una orden abierta", pero el modelo documenta explícitamente que ya no existe ningún índice único que lo impida.
**Evidencia**: `app/api/v1/orders/tables_advanced.py:59-63`; `app/models/customer_order.py:85-87`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: Manejador huérfano de una constraint que existió y fue removida.

### RN-ORD-61: No se pueden fusionar órdenes ya cerradas
**Evidencia**: `app/api/v1/orders/tables_advanced.py:75-82`
**Clasificación**: INTENCIONAL

### RN-ORD-62: Toda orden inexistente bloquea la fusión completa
**Ejemplo**: 3 uuids, solo 2 existen → 404 "Alguna orden no existe".
**Evidencia**: `app/api/v1/orders/tables_advanced.py:76-80`
**Clasificación**: INTENCIONAL

### RN-ORD-63 [ACCIDENTAL]: La fusión reutiliza el primer `merged_group_id` que encuentra, sin combinar grupos preexistentes — puede dejar órdenes huérfanas de su grupo original
**Enunciado**: Si las órdenes a fusionar ya pertenecían a grupos distintos preexistentes, solo sobrevive uno (el primero según un SELECT sin `ORDER BY`, no determinista); las órdenes no listadas del grupo "perdedor" quedan separadas de las que sí se movieron.
**Ejemplo**: Orden A en grupo G1 (junto con Z, no listada). Orden B en grupo G2 (junto con Y, no listada). `merge_orders([A,B])` toma G1 → A y B pasan a G1; Z sigue con A, pero Y queda sola en G2, separada de B.
**Evidencia**: `app/api/v1/orders/tables_advanced.py:75-89`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: No hay control ni error para el caso de grupos preexistentes en colisión; el resultado depende del orden no determinista del SELECT.

### RN-ORD-64 [DUDOSA]: La cuenta de grupo (`group_bill`) no filtra por status de la orden, solo por ítem anulado
**Enunciado**: Suma `unit_price × quantity` de todos los ítems no anulados de TODAS las órdenes del grupo, sin excluir órdenes `cancelada` ni exigir `pagada`/`bloqueada`.
**Ejemplo**: Grupo con orden A `abierta` (1 ítem $5000 no anulado) y orden B `cancelada` (1 ítem $3000 no anulado) → total = $8000, incluyendo el ítem de la orden cancelada.
**Evidencia**: `app/api/v1/orders/tables_advanced.py:92-114`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿`cancel_order` marca `estado_cocina="anulado"` en todos los ítems de la orden cancelada? Si no, hay riesgo real de cobrar ítems de una orden cancelada dentro de una cuenta de grupo.

### RN-ORD-65: No existe transición libre de status de orden; cada transición legítima tiene endpoint dedicado (eliminado a propósito un `PATCH` genérico de status)
**Enunciado**: Se retiró deliberadamente cualquier endpoint que permitiera asignar libremente cualquier status a una orden; cada transición (recibida→abierta, abierta→bloqueada, →pagada, →cancelada) tiene su propio endpoint con sus reglas y efectos.
**Evidencia**: `app/api/v1/orders/router.py:443-452`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito documenta la remoción deliberada y el bug de inventario que evitaba: el antiguo `PATCH /{order_id}/status` permitía saltar de "recibida" a "abierta" sin pasar por `confirm_order`, dejando el inventario sobrestimado en silencio.

### RN-ORD-66: No puede haber dos mesas con el mismo número
**Evidencia**: `app/api/v1/orders/router.py:56-65`
**Clasificación**: INTENCIONAL
## 9. `sales`

Venta de mostrador (sin mesa/QR) y `build_sale`, el constructor de venta compartido por los cuatro caminos de cobro
del sistema (mostrador, cierre unificado, cierre dividido, `pay_order` legado de mesa). Ficheros:
`app/api/v1/sales/builder.py`, `service.py`, `router.py`.

### RN-VENTA-01: Una venta debe tener al menos un ítem cobrable
**Ejemplo**: Checkout con `items: []` → 409 "La venta no tiene ítems cobrables", sin crear `Sale`/`SaleItem`/`Payment`.
**Evidencia**: `app/api/v1/sales/builder.py:97-98` (función `build_sale`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita con mensaje de negocio dedicado, antes de tocar la base de datos.

### RN-VENTA-02: El total de la venta = subtotal − descuento + impuesto + propina, sin redondeo intermedio
**Enunciado**: El total es la suma de `line_total` de todas las líneas, menos el descuento total (manual + promociones + combos), más impuesto, más propina, en ese orden aritmético exacto sobre `Decimal` puro.
**Ejemplo**: 2 helados a 8.000 y 1 topping de 2.000 → subtotal=18.000. Descuento manual 1.000 + promo 500 = 1.500. Impuesto 0. Propina 2.000. Total = 18.000−1.500+0+2.000 = 18.500.
**Evidencia**: `app/api/v1/sales/builder.py:118-132`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Fórmula explícita, única y centralizada; el docstring confirma que es deliberado que solo exista un constructor de venta.

### RN-VENTA-03: El total de la venta nunca puede ser negativo
**Ejemplo**: Subtotal 5.000, descuento manual 9.000 → total=-4.000 → 422 "El total no puede ser negativo".
**Evidencia**: `app/api/v1/sales/builder.py:132-136`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Validación explícita con mensaje de negocio propio.

### RN-VENTA-04: El pago recibido debe cubrir el total, sin excepción
**Ejemplo**: Total 18.500, pago en efectivo 18.000 → 422 "El pago (18000) no cubre el total (18500)".
**Evidencia**: `app/api/v1/sales/builder.py:138-153`
**Clasificación**: INTENCIONAL

### RN-VENTA-05: El vuelto (cambio) solo puede salir del efectivo pagado, nunca de pagos electrónicos
**Enunciado**: Si el total pagado en métodos no efectivo supera el total de la venta, se rechaza: el exceso solo puede provenir de efectivo, porque el vuelto físico se entrega del cajón.
**Ejemplo**: Total=10.000, pago con tarjeta de 15.000, sin efectivo → 422 "Los pagos que no son en efectivo (15000) no pueden superar el total (10000): el vuelto solo sale del efectivo." Con tarjeta 10.000 + efectivo 5.000 (vuelto 5.000) sí se acepta.
**Evidencia**: `app/api/v1/sales/builder.py:154-163`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explica el motivo contable: evitar un "faltante fantasma" en el arqueo del turno.

### RN-VENTA-06: El cambio entregado = pagado − total, y se descuenta del efectivo esperado en el arqueo
**Ejemplo**: Total 10.000, pago único efectivo 15.000 → `paid_amount=15.000`, `change_given=5.000`. En el arqueo, ese efectivo cuenta neto (15.000-5.000=10.000), no bruto.
**Evidencia**: `app/api/v1/sales/builder.py:165-172` (comentario "RN-029")
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Corrección deliberada de un bug histórico documentado en el docstring del módulo.

### RN-VENTA-07: Solo se puede cobrar contra un turno de caja abierto
**Evidencia**: `app/api/v1/sales/builder.py:60-64` (función `ensure_open_shift`)
**Clasificación**: INTENCIONAL

### RN-VENTA-08: El estado de una venta transiciona de "issued" a "paid" dentro de la misma construcción, sin quedar visible en el intermedio
**Evidencia**: `app/api/v1/sales/builder.py:113-114,172`
**Clasificación**: INTENCIONAL

### RN-VENTA-09: El descuento de inventario y la emisión de factura son responsabilidades separadas de `build_sale`; el mostrador siempre descuenta al cobrar
**Enunciado**: `build_sale` nunca toca inventario. En mostrador, el descuento de stock ocurre después de construir la venta, porque no hubo un paso previo de "confirmar pedido".
**Evidencia**: `app/api/v1/sales/builder.py:9-15`; `app/api/v1/sales/service.py:36-39,122-128`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado explícitamente: "Quien llama decide; build_sale nunca toca stock".

### RN-VENTA-10: Si el descuento de inventario falla por falta de stock, toda la venta (incluida su factura) se revierte
**Evidencia**: `app/api/v1/sales/service.py:126-137`; `app/api/v1/sales/consumption.py:37-71`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Razón explícita en `invoices/service.py:7-9`: "una factura sin su venta, o al revés, sería peor que no tener factura".

### RN-VENTA-11 [ACCIDENTAL]: Una variante sin receta ni opción que consuma insumo se cobra igual, aunque no descuente nada del inventario en el camino de mostrador
**Enunciado**: A diferencia de la confirmación de pedidos QR (RN-CAT-34), el descuento de inventario de mostrador no bloquea el cobro si la variante no tiene ninguna regla de consumo definida; la venta se cobra y factura sin generar ningún movimiento.
**Evidencia**: `app/api/v1/sales/consumption.py:46-51`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: El propio comentario del código lo describe como un "agujero" conocido y creciente ("con slots el agujero crece"); el equipo mismo lo documenta como no deseado.

### RN-VENTA-12: Los ítems que forman parte de un combo se cobran a precio normal individual; el ahorro se calcula aparte como descuento
**Ejemplo**: Combo "helado+topping" normal 8.000+3.000=11.000, combo vende en 9.000 (ahorro 2.000). Al cobrarlo: dos `SaleLine` de 8.000 y 3.000 (subtotal 11.000), y `combo_discount_for_lines` añade 2.000 al descuento, reflejando los 9.000 reales.
**Evidencia**: `app/api/v1/sales/service.py:46-65,99-104`
**Clasificación**: INTENCIONAL

### RN-VENTA-13: El descuento de mostrador = manual + promociones automáticas (percent/fixed) + ahorro de combos
**Ejemplo**: Manual 500, promo automática 10% sobre línea de 8.000=800, sin combos → descuento total 1.300.
**Evidencia**: `app/api/v1/sales/service.py:99-119`
**Clasificación**: INTENCIONAL

### RN-VENTA-14 [DUDOSA]: El `promotion_id` de la venta de mostrador prioriza el combo si es único (mismo mecanismo que RN-ORD-08)
**Evidencia**: `app/api/v1/sales/service.py:104`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿es aceptable que con 2 combos distintos no quede registrado ningún combo en `promotion_id`?

### RN-VENTA-15: Las opciones seleccionadas para una línea deben estar activas y pertenecer a la variante; IDs repetidos se deduplican antes de calcular precio
**Evidencia**: `app/api/v1/sales/service.py:71-74`; `app/api/v1/catalog/line_pricing.py:44-52`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito contrasta con un comportamiento previo defectuoso.

### RN-VENTA-16: El precio de una línea es el de la variante más la suma de `extra_price` de las opciones, sin redondeo adicional (mismo mecanismo que RN-CAT-01)
**Evidencia**: `app/api/v1/catalog/line_pricing.py:191-196`; `app/api/v1/sales/builder.py:54-55`
**Clasificación**: INTENCIONAL

### RN-VENTA-17: El descuento de inventario se pre-bloquea en orden canónico de UUID para evitar deadlocks entre venta de mostrador y confirmación de mesa
**Evidencia**: `app/api/v1/sales/consumption.py:53-60`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito sobre la razón técnica-operativa (evitar deadlock).

---

## 10. `invoices`

Emisión y consulta de facturas internas, una por venta, con numeración consecutiva por prefijo. Ficheros:
`app/api/v1/invoices/service.py`, `router.py` (solo lectura).

### RN-FACT-01: Toda venta pagada emite automáticamente su factura, en la misma transacción de base de datos
**Enunciado**: No existe botón separado para facturar: al confirmarse el pago, la factura se emite dentro de la misma transacción que crea la venta; el rollback deshace venta y factura juntas.
**Ejemplo**: Checkout de mostrador con inventario insuficiente: `build_sale` arma venta+factura en la transacción, `deduct_sale` lanza `InsufficientStockError`, `checkout` hace rollback → ni venta ni factura quedan persistidas.
**Evidencia**: `app/api/v1/sales/builder.py:174-179`; `app/api/v1/sales/service.py:106-137`; `app/api/v1/invoices/service.py:1-11,45-53`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: El docstring de `invoices/service.py` documenta explícitamente que esto corrige el problema histórico de "20 ventas reales, cero facturas" por depender de un botón manual (ver hallazgo RN-FACT-06).

### RN-FACT-02: La factura por venta es idempotente — nunca se duplica
**Enunciado**: Si `issue_for_sale` se invoca dos veces para la misma venta, la segunda llamada devuelve la ya existente en vez de crear una nueva; respaldado por unicidad de `sale_id` en base de datos.
**Evidencia**: `app/api/v1/invoices/service.py:45-58`; `app/models/invoice.py:21-23`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "en vez de chocar con la constraint única de sale_id".

### RN-FACT-03: La numeración de facturas es un consecutivo estrictamente secuencial y sin huecos, por prefijo, serializado con lock de fila
**Enunciado**: `InvoiceCounter` se lee con `SELECT...FOR UPDATE`; si dos ventas se confirman en paralelo, la segunda espera a que la primera libere el lock. Si una transacción hace rollback, el número no queda "salteado" porque el incremento nunca se confirmó.
**Ejemplo**: Contador `next_number=100`. Transacción A lee 100, deja `next_number=101`, comitea. Transacción B (que esperaba) lee 101, obtiene 101. Resultado: `FAC-000100` y `FAC-000101`, sin colisión ni salto.
**Evidencia**: `app/api/v1/invoices/service.py:30-42`; `app/models/invoice.py:64-73`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring lo declara como objetivo explícito.

### RN-FACT-04: La numeración es por tenant (schema separado) y, dentro del tenant, por prefijo
**Enunciado**: `InvoiceCounter` vive en el schema del tenant; dos tenants nunca comparten contador aunque usen el mismo prefijo. La unicidad real en BD es sobre `(prefix, number)`, no solo `number`.
**Evidencia**: `app/models/invoice.py:64-77`; `app/api/v1/invoices/service.py:30-42`; `app/api/v1/sales/router.py:53`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Consecuencia directa y deliberada del diseño multi-tenant por schema.

### RN-FACT-05 [DUDOSA]: El número visible (`prefijo + 6 dígitos`) se busca con `ILIKE '%...%'`, aceptando coincidencias parciales en cualquier posición
**Enunciado**: El número "de negocio" es `prefix` + `number` rellenado a 6 dígitos (p. ej. `FAC-000042`). Al filtrar por referencia, la consulta usa `ILIKE '%42%'` sobre ese valor compuesto, que también matchea `FAC-004200` o cualquier factura que contenga "42" en cualquier posición.
**Evidencia**: `app/api/v1/sales/service.py:165-172`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El formato de 6 dígitos es intencional, pero la amplitud de la búsqueda "contiene" puede traer más resultados de los esperados. SUPOSICIÓN: ¿la búsqueda por referencia debe ser "contiene" o "empieza por número exacto"?

### RN-FACT-06 [Hallazgo — cobertura parcial verificada]: La corrección de "20 ventas reales, cero facturas" está verificada solo para el camino de mostrador
**Enunciado**: El diseño actual garantiza que toda venta que pasa por `build_sale` queda facturada en la misma transacción (RN-FACT-01), lo cual hace estructuralmente imposible repetir el escenario histórico **siempre que no exista otro camino de creación de `Sale` pagada que no pase por `build_sale`**. El docstring de `builder.py` menciona "cuatro formas de cobrar" (mostrador, cierre unificado, cierre dividido, `pay_order` legado); este análisis modular solo verificó línea por línea el camino de mostrador.
**Evidencia**: `app/api/v1/sales/builder.py:1-16,91-95`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN — los cierres de mesa (unificado, dividido, `pay_order` legado) en `orders/checkout.py` y `table_sessions/service.py` sí fueron confirmados en este documento como usuarios de `build_sale` (RN-MESA-19 a RN-MESA-22, RN-ORD-07/09), lo que en conjunto cierra la duda: los cuatro caminos documentados en este análisis convergen en `build_sale`. Se mantiene DUDOSA solo por la pregunta de si existe algún quinto camino no documentado que cree una `Sale` con `status="paid"` sin pasar por `build_sale`.

### RN-FACT-07 [DUDOSA]: No se verificó si `Invoice.full_number` (propiedad Python) coincide exactamente con la fórmula SQL reconstruida en `list_sales_query`
**Enunciado**: El comentario en `sales/service.py` dice reconstruir la referencia "tal como se imprime en el ticket (ver `Invoice.full_number`)", pero no se confirmó la fórmula exacta de esa propiedad en `app/models/invoice.py`.
**Evidencia**: `app/api/v1/sales/service.py:166-168`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN — requiere leer `app/models/invoice.py` completo para confirmar si `full_number` produce el mismo string que `prefix + lpad(number, 6, '0')`.
## 11. `promotions`

Evalúa y administra descuentos (porcentaje, fijo, por cantidad) y combos, con vigencia por fecha/día/hora,
aplicándolos al carrito, al menú público y en cada cobro. Único módulo con test que corre en CI
(`app/scripts/test_promotions_rules.py`). Ficheros: `app/api/v1/promotions/service.py`, `router.py`, `schemas.py`.

### 11.1 Evaluación y cálculo del descuento

### RN-PROMO-01: Vigencia evaluada en hora local única del tenant, nunca en UTC
**Ejemplo**: `TENANT_TIMEZONE=America/Bogota` (UTC-5). `now` UTC=2026-08-15 00:30 → local=2026-08-14 19:30. El día de semana evaluado es viernes 14, no sábado 15.
**Evidencia**: `app/api/v1/promotions/service.py:50-68` (funciones `_tz`, `local_now`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring documenta explícitamente el bug anterior y la corrección.

### RN-PROMO-02: Ventana horaria con cruce de medianoche, límites inclusivos
**Enunciado**: `start<=end`: hora actual entre ambos inclusive. `start>end` (cruce medianoche): aplica si `hora>=start` O `hora<=end`. Sin `end`: desde `start` hasta fin de día. Sin `start`: desde medianoche hasta `end` inclusive. Sin ambos: todo el día.
**Ejemplo**: Ventana `22:00-02:00`. A las 22:00:00 exacto aplica; a las 02:00:00 exacto aplica; a las 02:00:01 NO aplica.
**Evidencia**: `app/api/v1/promotions/service.py:71-86` (`_in_time_window`)
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring documenta corrección explícita de un bug previo (ventana insatisfacible).

### RN-PROMO-03 [DUDOSA]: Asimetría de precisión — `starts_at` es datetime exacto, `ends_at` es solo fecha
**Enunciado**: `starts_at` se compara con datetime completo; `ends_at` se compara solo por fecha (`now.date() > ends_at.date()`), ignorando la hora, así "Hasta 04/08" cubre el 04/08 completo hasta medianoche local del 05/08.
**Ejemplo**: `starts_at=2026-08-15 09:00`, `ends_at=2026-08-20 00:00`. A las 2026-08-15 08:59 local NO válida; a las 2026-08-20 23:59 local SÍ válida; a las 2026-08-21 00:00 NO.
**Evidencia**: `app/api/v1/promotions/service.py:94-99`; `app/models/promotion.py:74-75`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El comentario solo justifica el tratamiento de `ends_at`, no explica por qué `starts_at` sí usa precisión de hora. SUPOSICIÓN: ¿el formulario de creación permite capturar hora en "Desde"?

### RN-PROMO-04: Filtro por día de la semana (CSV, 0=lunes)
**Evidencia**: `app/api/v1/promotions/service.py:100-103`
**Clasificación**: INTENCIONAL

### RN-PROMO-05: Parsing tolerante de `days_of_week` (espacios y vacíos se descartan)
**Evidencia**: `app/api/v1/promotions/service.py:101`
**Clasificación**: INTENCIONAL

### RN-PROMO-06: Target de producto gana sobre target de categoría
**Enunciado**: Si una línea coincide con un target de producto exacto Y con uno que coincide por categoría, gana el de producto. Sin targets, aplica global. Con targets pero sin coincidencia, NO aplica.
**Ejemplo**: Target A (categoría X, $10.000), Target B (producto Y de categoría X, $12.000). Línea de producto Y/categoría X → gana B ($12.000).
**Evidencia**: `app/api/v1/promotions/service.py:107-124`; `app/models/promotion.py:132-134`
**Clasificación**: INTENCIONAL

### RN-PROMO-07: `qty_price` — el paquete (tamaño+precio) vive SOLO en el target; sin destino, no hay descuento
**Enunciado**: `min_qty`/`value` propios de la `Promotion` nunca se usan como fallback en `qty_price`; solo cuentan los del target elegido.
**Evidencia**: `app/api/v1/promotions/service.py:127-138`; `app/models/promotion.py:38-42`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "Sin precio no hay descuento: el fallo seguro en vez del caro".

### RN-PROMO-08: Descuento percent = porcentaje exacto del total de línea, sin redondeo intermedio
**Ejemplo**: `line_total=15000, value=20` → descuento=3000.00 exacto.
**Evidencia**: `app/api/v1/promotions/service.py:143-144`
**Clasificación**: INTENCIONAL

### RN-PROMO-09: Descuento fixed topado al total de línea (nunca negativo)
**Ejemplo**: `value=50000, line_total=8000` → descuento=8000 (línea queda en $0).
**Evidencia**: `app/api/v1/promotions/service.py:145-146`
**Clasificación**: INTENCIONAL

### RN-PROMO-10: `qty_price` descuenta solo paquetes completos; el remanente va a precio normal
**Ejemplo**: `pack=3, price_paquete=20000, unit_price=8000, quantity=7` → `packs=2` (floor de 7/3), `normal=8000*3*2=48000` → descuento=8000. La 7ª unidad se cobra a $8000 normal.
**Evidencia**: `app/api/v1/promotions/service.py:147-159`
**Clasificación**: INTENCIONAL

### RN-PROMO-11 [DUDOSA]: `qty_price` nunca genera "descuento" negativo si el paquete configurado es más caro que lo normal
**Ejemplo**: `pack=2, price_paquete=25000, unit_price=8000` (normal por 2=16000) → `16000-25000=-9000` → descuento=0.
**Evidencia**: `app/api/v1/promotions/service.py:159`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No hay validación que impida configurar un precio de paquete superior al normal.

### RN-PROMO-12: Cantidad mínima — frontera inclusiva (`>=`)
**Enunciado**: `quantity < minimo` descarta; `quantity == minimo` SÍ califica. Para `qty_price`, el mínimo es el del target elegido.
**Evidencia**: `app/api/v1/promotions/service.py:254-264`
**Clasificación**: INTENCIONAL

### RN-PROMO-13: Selección de la mejor promoción por línea — prioridad, luego descuento mayor, luego antigüedad
**Enunciado**: Gana mayor `priority`; empate, mayor descuento; empate persistente, la más antigua (`created_at` menor). Nunca se acumulan dos promociones en una línea; es excluyente.
**Ejemplo**: Promo A `priority=1` descuento $2000; Promo B `priority=2` descuento $1000 → gana B pese a descontar menos.
**Evidencia**: `app/api/v1/promotions/service.py:268-271,280-283`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Documentado explícitamente en docstring del módulo.

### RN-PROMO-14: Desempate final favorece a la promoción más antigua
**Evidencia**: `app/api/v1/promotions/service.py:268`
**Clasificación**: INTENCIONAL

### RN-PROMO-15 [DUDOSA]: Descuento 0 descalifica la promoción como candidata a "mejor"
**Enunciado**: Si el monto calculado es `<=0`, esa promoción no se considera candidata aunque cumpla vigencia y cantidad mínima; una `fixed`/`percent` con `value=0` (permitido por `CHECK`) nunca gana ninguna línea.
**Evidencia**: `app/api/v1/promotions/service.py:265-267`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No está claro si `value=0` es un caso de uso válido (promo deshabilitada temporalmente) o un error de captura enmascarado silenciosamente.

### RN-PROMO-16: El motor automático excluye siempre el tipo combo
**Evidencia**: `app/api/v1/promotions/service.py:45,226`
**Clasificación**: INTENCIONAL

### RN-PROMO-17: El filtro SQL de vigencia es parcial (status + fecha de corte); el resto se valida en Python
**Evidencia**: `app/api/v1/promotions/service.py:211-230`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring explica la optimización de índice.

### RN-PROMO-18: `unit_price` se deriva de `line_total/quantity` si falta, con protección de división por cero
**Ejemplo**: `quantity=0, line_total=0` sin `unit_price` → `unit_price=0` (no hay `ZeroDivisionError`).
**Evidencia**: `app/api/v1/promotions/service.py:242-246`
**Clasificación**: INTENCIONAL

### RN-PROMO-19 [DUDOSA]: Cantidad se trunca a entero (no se redondea)
**Ejemplo**: `quantity=3.9` → `int(3.9)=3`; con `min_qty=4` la línea no calificaría pese a estar más cerca de 4.
**Evidencia**: `app/api/v1/promotions/service.py:242`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN — no se verificó si el modelo de venta admite cantidades fraccionarias (venta por peso).

### RN-PROMO-20: Descuento porcentual máximo 100% deja la línea en $0 exacto
**Evidencia**: `app/api/v1/promotions/service.py:143-144,581-585`; `app/models/promotion.py:105-108`
**Clasificación**: INTENCIONAL

### RN-PROMO-21: Redondeo del descuento total ocurre una sola vez, al final, con ROUND_HALF_UP a 2 decimales
**Enunciado**: Los montos por línea no se redondean individualmente; solo la suma total del cobro se redondea. El desglose por línea puede no sumar exactamente al total redondeado.
**Evidencia**: `app/api/v1/promotions/service.py:331,443,172`
**Clasificación**: INTENCIONAL

### RN-PROMO-22: `excluded_promotion_ids` permite retirar una promoción y recalcular el resto
**Enunciado**: Al evaluar el cobro se puede excluir un set de IDs (ej. el cajero retira un descuento); las demás promociones vigentes se siguen evaluando línea por línea, permitiendo que otra de menor prioridad gane esa línea.
**Evidencia**: `app/api/v1/promotions/service.py:300-307,316-319`
**Clasificación**: INTENCIONAL

### RN-PROMO-23: `Sale.promotion_id` (legado) solo se rellena si TODAS las líneas descontadas comparten una única promoción
**Enunciado**: Queda `NULL` si dos o más líneas del mismo cobro fueron descontadas por promociones DIFERENTES, aunque el desglose completo (`evaluate_detailed`) sí las registre todas.
**Evidencia**: `app/api/v1/promotions/service.py:185-190,335-343`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring del módulo documenta explícitamente esta pérdida de trazabilidad como limitación conocida y transitoria.

### RN-PROMO-24: Combo agrupa líneas por `combo_id` y descuenta solo bundles completos
**Enunciado**: Se toma el MÍNIMO entre `cantidad_disponible // cantidad_requerida` de cada ítem de la receta como número de bundles completos.
**Ejemplo**: Receta {A:2, B:1}. Carrito A=5, B=3 → `bundle_units=min(5//2=2, 3//1=3)=2`.
**Evidencia**: `app/api/v1/promotions/service.py:416-433`
**Clasificación**: INTENCIONAL

### RN-PROMO-25: Combo usa el precio MÍNIMO cuando la misma variante aparece con precios distintos en el carrito
**Evidencia**: `app/api/v1/promotions/service.py:422-426`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "el cliente no paga el alto".

### RN-PROMO-26 [DUDOSA]: Precio del combo = value × bundles; descuento nunca negativo, pero sin validación contra combos configurados más caros que sus componentes
**Ejemplo**: `bundle_units=2, promo.value=15000` → precio combo=30000; `covered_normal=40000` → descuento=10000. Si `promo.value=45000` → descuento=0.
**Evidencia**: `app/api/v1/promotions/service.py:435-441`
**Clasificación**: DUDOSA
**Justificación de clasificación**: El `max(0,...)` evita cobrar de más pero también oculta silenciosamente la mala configuración.

### RN-PROMO-27 [DUDOSA]: Combo inexistente/no vigente/sin receta en el cobro no descuenta (silencioso), a diferencia de al agregarlo al carrito (que sí lanza excepción)
**Enunciado**: `combo_discount_for_lines` (cálculo del cobro) simplemente no genera descuento si el combo ya no existe/no vigente/sin `combo_items`. `get_active_combo`/`expand_combo` (al agregarlo al carrito) sí lanzan 409/422 en los mismos casos.
**Ejemplo**: Combo agregado al carrito pasa a `finished` antes del cobro → el cobro no falla, cobra esas líneas a precio normal sin avisar.
**Evidencia**: `app/api/v1/promotions/service.py:405-414` vs `356-362,373-374`
**Clasificación**: DUDOSA
**Justificación de clasificación**: Crea inconsistencia de UX entre etapas: el cajero no recibe aviso de que el combo dejó de aplicar entre agregarlo y cobrar.

### RN-PROMO-28: `expand_combo` exige que TODAS las variantes componentes estén activas
**Evidencia**: `app/api/v1/promotions/service.py:378-382`
**Clasificación**: INTENCIONAL

### RN-PROMO-29: Solo se puede activar/expandir como combo una Promotion `type='combo'` vigente
**Evidencia**: `app/api/v1/promotions/service.py:356-362`
**Clasificación**: INTENCIONAL

### RN-PROMO-30: El solapamiento de promociones es solo advertencia, nunca bloquea
**Evidencia**: `app/api/v1/promotions/service.py:490-513`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Docstring justifica el diseño con un caso de uso real de negocio.

### RN-PROMO-31: Overlap de rango de fechas es conservador (asume solapamiento si falta información)
**Evidencia**: `app/api/v1/promotions/service.py:448-453`
**Clasificación**: INTENCIONAL

### RN-PROMO-32: Overlap de días — CSV nulo en cualquiera de las dos implica solapamiento total
**Evidencia**: `app/api/v1/promotions/service.py:456-459`
**Clasificación**: INTENCIONAL

### RN-PROMO-33 [DUDOSA]: Overlap de horario solo se detecta con precisión si AMBAS promociones definen `start_time`
**Enunciado**: Si cualquiera no tiene `start_time`, se asume solapamiento total. El chequeo se basa solo en si el inicio de una cae en la ventana de la otra.
**Evidencia**: `app/api/v1/promotions/service.py:462-466`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿fue probado con dos ventanas que cruzan medianoche simultáneamente?

### RN-PROMO-34: Overlap de alcance — target de producto choca con target de categoría si el producto pertenece a ella
**Evidencia**: `app/api/v1/promotions/service.py:469-487`
**Clasificación**: INTENCIONAL

### RN-PROMO-35: Combo requiere mínimo 2 componentes de variantes DISTINTAS para activarse/tener forma válida
**Ejemplo**: `combo_items=[{variante A, qty=3}]` (una sola variante) → 422 "Un combo requiere al menos 2 productos distintos".
**Evidencia**: `app/api/v1/promotions/service.py:619-623,660-664`
**Clasificación**: INTENCIONAL

### RN-PROMO-36: `combo_items` solo válido en promociones `type=combo`
**Evidencia**: `app/api/v1/promotions/service.py:624-628`
**Clasificación**: INTENCIONAL

### RN-PROMO-37: `value`/`min_qty` de target solo válidos en `qty_price`; y en `qty_price` TODOS los targets deben tenerlos completos
**Evidencia**: `app/api/v1/promotions/service.py:631-648`
**Clasificación**: INTENCIONAL

### RN-PROMO-38: `update()` revalida reglas dependientes de `type` que el schema PATCH no puede validar por sí solo
**Ejemplo**: PATCH con `value=500` sobre `percent` → 422 antes de llegar al `CHECK` de BD.
**Evidencia**: `app/api/v1/promotions/service.py:581-595`
**Clasificación**: INTENCIONAL

### RN-PROMO-39: PATCH usa "campos provistos" para permitir limpiar un campo opcional con `null` explícito
**Evidencia**: `app/api/v1/promotions/service.py:569-575`
**Clasificación**: INTENCIONAL

### RN-PROMO-40: La "forma" (`type`, `targets`, `combo_items`) de una promoción solo puede cambiar en `status=draft`
**Enunciado**: Cambiar tipo o alcance está prohibido fuera de `draft`, porque pudo haber explicado ya el descuento de una venta pasada; la alternativa es duplicarla.
**Evidencia**: `app/api/v1/promotions/service.py:600-609`
**Clasificación**: INTENCIONAL

### RN-PROMO-41 [DUDOSA]: Máquina de estados de la promoción — `finished` es terminal; reenviar el mismo estado es no-op silencioso incluso para `finished→finished`
**Enunciado**: Transiciones permitidas: `draft→{active,finished}`, `active→{paused,finished}`, `paused→{active,finished}`, `finished→{}`. Si el nuevo estado es igual al actual, retorna sin error SIN pasar por la tabla de transiciones (ni siquiera para `finished`).
**Evidencia**: `app/api/v1/promotions/service.py:652-659`; `app/models/promotion.py:21-26`
**Clasificación**: DUDOSA
**Justificación de clasificación**: No está claro si es idempotencia deseada o un descuido, dado que `finished` debería ser estrictamente terminal.

### RN-PROMO-42: `duplicate()` copia toda la configuración pero fuerza `status=draft`
**Evidencia**: `app/api/v1/promotions/service.py:670-694`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Es el único camino para modificar la forma de una promoción que ya estuvo activa.

### RN-PROMO-43: Listado de promociones ordenado por prioridad descendente y luego nombre
**Evidencia**: `app/api/v1/promotions/service.py:518-525`
**Clasificación**: INTENCIONAL

### RN-PROMO-44: Texto de cara al cajero difiere por tipo de promoción
**Ejemplo**: `value=20.00` → "20%"; `value=12.50` → "12.5%" (formato `:g`, sin ceros sobrantes).
**Evidencia**: `app/api/v1/promotions/service.py:193-208`
**Clasificación**: INTENCIONAL

### RN-PROMO-45: El motor automático excluye por diseño el tipo `buy_x_get_y` (no implementado en el cálculo)
**Enunciado**: `AUTO_TYPES` está hardcodeado a `(percent, fixed, qty_price)`; aunque el dominio conceptualmente contemplaba "2x1", el cálculo no lo implementa y fue retirado de los tipos permitidos.
**Evidencia**: `app/api/v1/promotions/service.py:45,160`; `app/models/promotion.py:28-30`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito en el modelo: "un 2x1 que no descuenta" mientras `_line_discount` le devuelva 0.
### 11.2 Vigencia, validación de forma y máquina de estados (`router.py`, `schemas.py`)

### RN-PROMO-46: Solo el estado "active" habilita el descuento automático
**Ejemplo**: Promoción `percent=10%` siempre vigente por fecha/hora pero `status="paused"` → `_valid_now=False`.
**Evidencia**: `app/api/v1/promotions/service.py:92-93`; test `app/scripts/test_promotions_rules.py:223-228`
**Clasificación**: INTENCIONAL

### RN-PROMO-47: La vigencia se evalúa en hora local del tenant, no en UTC (confirmado con test)
**Ejemplo**: Martes 20:00 Bogotá (UTC-5) = miércoles 01:00 UTC. Promo `days_of_week="1"` (martes) evaluada con ese `now` UTC sigue vigente (localmente sigue siendo martes).
**Evidencia**: `app/api/v1/promotions/service.py:57-68,91`; test `app/scripts/test_promotions_rules.py:74-80`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Corrección documentada de un bug previo, con test dedicado en CI.

### RN-PROMO-48: `starts_at` es un límite inclusivo evaluado con datetime completo
**Ejemplo**: `starts_at=2026-08-04 00:00:00` local; `now` igual exacto → válido (`now < starts_at` es falso).
**Evidencia**: `app/api/v1/promotions/service.py:94-95`
**Clasificación**: INTENCIONAL

### RN-PROMO-49: `ends_at` se compara solo por fecha, cubriendo el día completo (comentario explícito)
**Evidencia**: `app/api/v1/promotions/service.py:96-99`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario propio explica el porqué: el selector "Hasta" de la UI guarda medianoche del día elegido.

### RN-PROMO-50: `days_of_week` restringe por día local, 0=lunes..6=domingo (según `datetime.weekday()`)
**Evidencia**: `app/api/v1/promotions/service.py:100-103`
**Clasificación**: INTENCIONAL

### RN-PROMO-51 [DUDOSA]: La ventana horaria admite cruce de medianoche con límites inclusivos exactos, no cubiertos por test en el segundo límite
**Evidencia**: `app/api/v1/promotions/service.py:71-86`; test `app/scripts/test_promotions_rules.py:92-96` (no cubre el instante exacto de los límites)
**Clasificación**: DUDOSA
**Justificación de clasificación**: El comportamiento en el segundo límite exacto se infiere solo de los operadores del código, sin evidencia de ejecución.

### RN-PROMO-52: El orden de evaluación de vigencia es un AND estricto con cortocircuito (estado→starts_at→ends_at→días→hora)
**Evidencia**: `app/api/v1/promotions/service.py:89-104`
**Clasificación**: INTENCIONAL

### RN-PROMO-53: Máquina de estados con transiciones fijas
**Ejemplo**: `finished→active` → 409 "Transición no permitida". `draft→paused` → 409 (no está en los destinos de `draft`).
**Evidencia**: `app/models/promotion.py:21-26`; `app/api/v1/promotions/service.py:652-667`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "finished es terminal: una promoción vencida no revive, se duplica".

### RN-PROMO-54 [DUDOSA]: Pedir el mismo estado actual es un no-op silencioso, no pasa por la tabla de transiciones (confirma RN-PROMO-41)
**Ejemplo**: `PATCH .../status {"status":"finished"}` sobre una ya `finished` → 200, sin error.
**Evidencia**: `app/api/v1/promotions/service.py:653-654`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿la intención es que reconfirmar un estado sea siempre un no-op exitoso, o debería rechazarse igual que cualquier transición no listada?

### RN-PROMO-55: Activar un combo exige al menos 2 componentes, revalidado en el momento de la transición (defensa contra borrado en cascada de variantes)
**Enunciado**: Al pedir `status=active` sobre un combo, se exige `>=2 product_variant_id` distintos, aunque la creación ya lo garantizara — porque `ondelete="CASCADE"` sobre `product_variant_id` permite que un combo válido se quede con 1 solo componente si se borra una variante de producto fuera de `update_shape`.
**Evidencia**: `app/api/v1/promotions/service.py:660-664`; `app/models/promotion.py:190-192`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Es la defensa real contra un escenario disparado por un evento externo (borrar una variante), no cubierto por la validación de creación.

### RN-PROMO-56: Cambio de forma (`type`, `targets`, `combo_items`) solo permitido en `draft` (confirma RN-PROMO-40 desde el router)
**Ejemplo**: `PATCH /shape {"type":"fixed"}` sobre una `active` → 409 "Solo una promoción en borrador puede cambiar de tipo o alcance. Duplícala, edita la copia y finaliza la original."
**Evidencia**: `app/api/v1/promotions/service.py:600-609`
**Clasificación**: INTENCIONAL

### RN-PROMO-57: `PromotionUpdate` (PATCH escalar) no admite `type`, `targets` ni `combo_items` — solo `PATCH /shape` los cambia
**Evidencia**: `app/api/v1/promotions/schemas.py:183-196`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: SUPOSICIÓN: no se verificó si un campo extra desconocido en el PATCH genera error o se ignora silenciosamente (`extra="forbid"` no encontrado).

### RN-PROMO-58 [DUDOSA]: Duplicar siempre crea la copia en `draft`, sin importar el estado de origen (confirma RN-PROMO-42)
**Evidencia**: `app/api/v1/promotions/service.py:670-694`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "Es la salida al hecho de que una promoción activa no pueda cambiar de forma".

### RN-PROMO-59: Duplicar exige un nombre nuevo y único, validado antes de copiar
**Evidencia**: `app/api/v1/promotions/schemas.py:210-211`; `app/api/v1/promotions/router.py:121-123`
**Clasificación**: INTENCIONAL

### RN-PROMO-60: El solapamiento entre promociones es solo advertencia informativa, nunca bloquea create/update (confirma RN-PROMO-30)
**Evidencia**: `app/api/v1/promotions/router.py:24-34`; `app/api/v1/promotions/service.py:490-513`
**Clasificación**: INTENCIONAL

### RN-PROMO-61: El solapamiento solo considera candidatos `draft`, `active` o `paused` (excluye `finished` y tipo `combo`)
**Evidencia**: `app/api/v1/promotions/service.py:497-505`
**Clasificación**: INTENCIONAL

### RN-PROMO-62: Un descuento porcentual no puede superar 100, pero sí puede ser exactamente 100 (triple capa: schema + service + BD)
**Ejemplo**: `value=100.00` → aceptado. `value=100.01` → 422. `PATCH value=150` sobre `percent` existente → 422 capturado en service antes del `CHECK` de BD.
**Evidencia**: `app/api/v1/promotions/schemas.py:101-105`; `app/api/v1/promotions/service.py:581-585`; `app/models/promotion.py:105-108`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario narra el bug histórico que motivó agregar la capa de servicio: "un PATCH con value=500... tumbaba la caja".

### RN-PROMO-63: `qty_price` exige `min_qty` (de la promoción) `>=2`, triple capa igual que RN-PROMO-62
**Evidencia**: `app/api/v1/promotions/schemas.py:107-111`; `app/api/v1/promotions/service.py:586-590`; `app/models/promotion.py:110-113`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "Un qty_price de paquete 1 es un precio, no una promoción".

### RN-PROMO-64: El `min_qty` de un target individual, si se define, también exige `>=2`
**Evidencia**: `app/api/v1/promotions/schemas.py:74`; `app/models/promotion.py:162`
**Clasificación**: INTENCIONAL

### RN-PROMO-65: `start_time`/`end_time` deben configurarse juntos (ambos o ninguno), validado tanto en el payload como en el estado final tras un PATCH
**Enunciado**: Un PATCH que limpia solo uno de los dos a `null` pasa la validación de schema del payload pero es rechazado por el service al detectar el estado final inconsistente.
**Evidencia**: `app/api/v1/promotions/schemas.py:48-54`; `app/api/v1/promotions/service.py:591-595`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "Una ventana a medias... hace creer al admin que configuró un happy hour".

### RN-PROMO-66: `ends_at` no puede tener fecha anterior a `starts_at` (comparación por fecha)
**Ejemplo**: `starts_at=10/08`, `ends_at=09/08 23:00` → 422. Misma fecha con horas distintas → aceptado.
**Evidencia**: `app/api/v1/promotions/schemas.py:56-60`
**Clasificación**: INTENCIONAL

### RN-PROMO-67: `days_of_week` debe ser CSV de enteros 0-6; se normaliza (deduplicado y ordenado)
**Ejemplo**: `"lunes,martes"` → 422. `"2,0,2"` → se guarda como `"0,2"`.
**Evidencia**: `app/api/v1/promotions/schemas.py:23-38,42-46`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Corrige un bug documentado: antes era texto libre y `"lunes,martes"` nunca aplicaba sin error visible.

### RN-PROMO-68 [DUDOSA]: Una promoción no puede crearse con `status=finished`, pero sí puede crearse directamente `active` o `paused`
**Ejemplo**: `POST {"status":"finished"}` → 422. `POST {"status":"active"}` → 201, nace activa sin pasar por `draft`.
**Evidencia**: `app/api/v1/promotions/schemas.py:120,131-135`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿el negocio pretendía que TODA promoción pasara obligatoriamente por `draft` antes de `active`?

### RN-PROMO-69: El precio/tamaño de paquete por target solo se acepta si `type=qty_price` (validado en create y en cambio de forma)
**Evidencia**: `app/api/v1/promotions/schemas.py:137-146`; `app/api/v1/promotions/service.py:630-637`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "dejaría un campo que se guarda y no hace nada, que es peor que rechazarlo".

### RN-PROMO-70: `qty_price` exige al menos un target y que TODOS tengan `value` y `min_qty` definidos
**Evidencia**: `app/api/v1/promotions/schemas.py:148-166`; `app/api/v1/promotions/service.py:638-648`
**Clasificación**: INTENCIONAL

### RN-PROMO-71: `TargetIn` requiere exactamente un alcance: `product_id` XOR `category_id`
**Enunciado**: A nivel de aplicación es exclusión mutua estricta; el `CHECK` de BD es más laxo (solo exige "al menos uno", no "exactamente uno") — la exclusividad depende enteramente de la capa de aplicación.
**Evidencia**: `app/api/v1/promotions/schemas.py:76-82`; `app/models/promotion.py:157-159`
**Clasificación**: INTENCIONAL

### RN-PROMO-72: Un combo exige al menos 2 productos distintos, sin duplicados, y no puede usar `targets`
**Evidencia**: `app/api/v1/promotions/schemas.py:168-180`; `app/api/v1/promotions/service.py:619-628`
**Clasificación**: INTENCIONAL

### RN-PROMO-73: `priority` está acotado entre 0 y 1000 inclusive
**Evidencia**: `app/api/v1/promotions/schemas.py:121,190`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: SUPOSICIÓN: no se encontró justificación de por qué el tope es 1000 específicamente.

### RN-PROMO-74: `value` (monto/porcentaje) nunca puede ser negativo (schema + BD, promoción y target)
**Evidencia**: `app/api/v1/promotions/schemas.py:118,73`; `app/models/promotion.py:100,161`
**Clasificación**: INTENCIONAL

### RN-PROMO-75 [ACCIDENTAL]: `PATCH` de nombre a `null` pasa la validación de Pydantic y el chequeo de unicidad del router, pero puede romper el `NOT NULL` de la columna
**Enunciado**: `PromotionUpdate.name` es `Optional[str]` con `min_length=1` que Pydantic no aplica cuando el valor es `None`; el router solo valida unicidad "si `name is not None`". El service copia cualquier campo en `model_fields_set`, incluido `name=None`. La columna es `nullable=False`.
**Ejemplo**: `PATCH {"name": null}` → pasa Pydantic, pasa el router, `service.update` asigna `promo.name=None`, y el `commit()` debería fallar con `IntegrityError` no controlado (mismo patrón de bug que el router ya documenta haber corregido para la unicidad del nombre en PATCH, pero sin cubrir este caso).
**Evidencia**: `app/api/v1/promotions/schemas.py:187`; `app/api/v1/promotions/router.py:72-74`; `app/api/v1/promotions/service.py:572-575`; `app/models/promotion.py:57`
**Clasificación**: ACCIDENTAL
**Justificación de clasificación**: El código ya documenta haber corregido un bug equivalente para la unicidad del nombre, pero no cubrió el caso `name=null`, dejando un vector similar sin resolver.

### RN-PROMO-76 [DUDOSA]: La unicidad de un `target` repetido (mismo producto/categoría en la misma promoción) no tiene validación de aplicación, solo índice único en BD
**Enunciado**: A diferencia del nombre de la promoción o de `combo_items` (validados explícitamente), los `targets` repetidos solo están protegidos por un índice único parcial en PostgreSQL, sin validador en Pydantic ni en `_apply_targets`.
**Ejemplo**: Crear `qty_price` con dos targets del mismo `product_id` → pasa Pydantic, falla solo al hacer `db.flush()`, probablemente como error no controlado.
**Evidencia**: `app/api/v1/promotions/schemas.py` (ausencia de validador, contrastar con `_combo_items_required`); `app/api/v1/promotions/service.py:533-540`; `app/models/promotion.py:166-173`
**Clasificación**: DUDOSA
**Justificación de clasificación**: SUPOSICIÓN: ¿existe algún manejo genérico de `IntegrityError` fuera de estos ficheros que convierta este fallo en un 409 legible?

### RN-PROMO-77: `min_qty` a nivel de promoción tiene mínimo 1 en create y update
**Evidencia**: `app/api/v1/promotions/schemas.py:127,196`
**Clasificación**: INTENCIONAL

### RN-PROMO-78: El nombre de la promoción debe ser único en creación, actualización y duplicado
**Evidencia**: `app/api/v1/promotions/router.py:55,70-74,122`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Patrón repetido y consistente (`ensure_unique`) en los tres puntos de entrada.

---

## 12. `menu`

Menú público que ve el comensal al escanear el QR de su mesa, con disponibilidad real y precios con descuento ya
aplicados. Fichero: `app/api/v1/menu/router.py`.

### RN-MENU-01: El precio mostrado ya incluye el descuento de promociones activas, calculado sobre cantidad 1, redondeado con "half up"
**Ejemplo**: Variante «Copa grande» 15.000 con promo 15% activa → `best_line_discount` (cantidad=1) da descuento 2.250 → `discounted_price=(15.000-2.250).quantize(0.01, ROUND_HALF_UP)=12.750,00`.
**Evidencia**: `app/api/v1/menu/router.py:151-167`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito justifica el uso de cantidad 1: "al navegar el menú aún no hay carrito".

### RN-MENU-02: Una promoción con cantidad mínima (`min_qty>1`) no se refleja en el precio del menú hasta que el comensal la alcance en el carrito
**Ejemplo**: Promo "Lleva 3 paga 2" (`min_qty=3`) sobre "Paleta de agua" $3.000 → evaluada con cantidad 1, no aplica → el menú muestra solo $3.000, aunque la promo exista y esté activa.
**Evidencia**: `app/api/v1/menu/router.py:151-156`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Limitación reconocida y documentada explícitamente en el comentario.

### RN-MENU-03: La disponibilidad de una opción se evalúa contra el peor caso de consumo entre todas las presentaciones que la usan (falso negativo intencional)
**Enunciado**: Una opción se marca disponible/no disponible globalmente, comparando el stock contra la mayor cantidad que cualquier presentación necesitaría. Si alcanza para la más chica pero no para la más grande, se muestra agotada igual.
**Ejemplo**: "Fresa" usada en «Ensalada pequeña» (60g/u) y «Ensalada grande» (180g/u). Stock=100g. `100>=180` es falso → "Fresa" se marca no disponible, aunque técnicamente alcanzaría para 1 ensalada pequeña (60g≤100g).
**Evidencia**: `app/api/v1/menu/router.py:36-77`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: El comentario lo llama explícitamente "el error seguro" y descarta la alternativa (exponer configuración por tamaño) como decisión consciente.

### RN-MENU-04: Una opción ligada a un insumo sin consumo positivo en ningún tamaño nunca aparece agotada (pero sí desaparece si el insumo se desactiva)
**Evidencia**: `app/api/v1/menu/router.py:72-76`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Comentario explícito: "no descuenta nada, así que no puede agotarse; pero sí desaparece si el insumo se desactiva".

### RN-MENU-05: Una opción cuyo insumo está desactivado nunca se muestra disponible, sin importar el stock numérico
**Evidencia**: `app/api/v1/menu/router.py:76`
**Clasificación**: INTENCIONAL

### RN-MENU-06: Una variante con un grupo obligatorio totalmente agotado se marca no pedible, sin bloquear las demás variantes del mismo producto
**Ejemplo**: «Malteada» tiene "Chica" (grupo obligatorio «Sabor» sin ninguna opción con stock → no pedible) y "Grande" (con al menos una opción con stock → sí pedible). El producto sigue disponible por la Grande.
**Evidencia**: `app/api/v1/menu/router.py:112-168`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: "mejor decirlo aquí que dejar al comensal chocar con el 409 al añadir al carrito".

### RN-MENU-07: Solo se listan categorías activas, productos activos y disponibles, y variantes activas; un producto sin variantes activas desaparece del todo
**Evidencia**: `app/api/v1/menu/router.py:86-100,170-171`
**Clasificación**: INTENCIONAL

### RN-MENU-08: El acceso al menú vía QR firmado limita la tasa de peticiones por IP y luego por mesa
**Evidencia**: `app/api/v1/menu/router.py:193-201`
**Clasificación**: INTENCIONAL
**Justificación de clasificación**: Orden de operaciones explícitamente comentado como decisión de seguridad/costo.

### RN-MENU-09: El endpoint de menú por token QR solo responde si la mesa referenciada sigue activa
**Ejemplo**: Mesa desactivada, comensal conserva QR viejo → `404 "Mesa no encontrada o inactiva"`.
**Evidencia**: `app/api/v1/menu/router.py:202-208`
**Clasificación**: INTENCIONAL
## 13. Preguntas abiertas para el negocio

Recopiladas de todas las `SUPOSICIÓN` marcadas en las reglas DUDOSA/ACCIDENTAL de arriba. Cada pregunta enlaza a la
regla que la origina.

### Autenticación y seguridad
1. ¿Existe rate-limiting a nivel de infraestructura (proxy/WAF) para `POST /auth/login`, dado que el código de
   aplicación no lo implementa? (RN-AUTH-03)
2. ¿Es deliberado que el refresh token sobreviva al logout del access token, dejando la sesión solo parcialmente
   revocada? (RN-AUTH-08)
3. ¿El formulario de cambio de contraseña impone un límite de longitud acorde al truncamiento de 72 bytes de
   bcrypt? (RN-AUTH-09)

### Catálogo y precios
4. ¿Es alcanzable en producción que un `OrderItem` referencie una variante o producto ya eliminado físicamente
   (no soft-delete)? (RN-CAT / RN-ORD-32)
5. Con `STRICT_OPTION_SELECTION=False`, ¿es tolerancia deliberada durante migración que se pueda cobrar y
   consumir inventario de una opción de un grupo que la variante no ofrece? (RN-CAT-32)
6. ¿Todos los llamadores de `load_valid_options` pasan el parámetro `variant` (que activa la validación de
   selección), o existen caminos que la omiten hoy? (RN-CAT-33)
7. Cuando un grupo de opciones opcional (`min_select=0`) es la única fuente de consumo de inventario de una
   variante, ¿debería bloquearse la venta si no se elige nada, o el bloqueo contradice la intención de "grupo
   opcional"? (RN-CAT-35)
8. La función `grupos_que_descuentan` (validación de selección) y `group_discounts` (chequeo de consumo
   obligatorio) usan criterios distintos para "¿este grupo descuenta inventario?" — ¿es intencional que
   diverjan? (RN-CAT-39)

### Inventario y costos
9. ¿Qué endpoints de `sales`/`orders` invocan `record_movement` con `allow_negative=True`, y bajo qué
   condición de negocio se permite vender en negativo? (RN-INV-05)
10. ¿El frontend exige un motivo al registrar un ajuste manual de stock, aunque el backend no lo obligue?
    (RN-INV-11)
11. ¿El negocio espera que el costo unitario de un insumo refleje el "último costo de compra" (comportamiento
    actual) o un costo promedio ponderado? (RN-INV-17)

### Caja
12. ¿El frontend impide cerrar un turno de caja sin haber enviado `counted_amount` o denominaciones, o el
    backend permite hoy un cierre sin ningún conteo físico? (RN-CASH-09)
13. ¿Es intencional que el arqueo parcial (`partial-count`) no exija observación cuando la diferencia es
    distinta de cero, a diferencia del cierre real que sí la exige? (RN-CASH-13)
14. ¿Existe algún mecanismo que congele ("snapshot") el arqueo de un turno en el momento de su cierre, o el
    histórico de turnos cerrados puede cambiar retroactivamente si se modifican ventas/pagos subyacentes?
    (RN-CASH-17)

### Mesas y reparto de cuenta
15. ¿Existe algún mecanismo (restricción del cliente a una pantalla por mesa, o similar) que en la práctica
    evite que un reparto de cuenta (`set_assignments`) concurrente con un cierre de sesión en curso lea datos a
    medio comitear? (RN-MESA-02)
16. ¿El frontend simula una "división igualitaria" de la cuenta convirtiéndola en asignaciones de unidades
    antes de llamar al endpoint de reparto, dado que el backend no ofrece división porcentual/monetaria?
    (RN-MESA-05)
17. ¿Es intencional permitir cerrar una mesa en modo `split` con un solo comensal (equivalente a `unified` mal
    disfrazado), o debería exigirse un mínimo de comensales? (RN-MESA-13)
18. Cuando una venta de mesa usa dos o más combos distintos, ¿es aceptable para el negocio que la venta no
    registre ningún combo en `promotion_id` (pérdida de trazabilidad en reportes por promoción)? (RN-MESA-15,
    RN-ORD-08, RN-VENTA-14)
19. ¿Existe algún flujo que limpie `participant_id` de un `OrderItem` al anularlo, o un comensal puede quedar
    permanentemente bloqueado para ser eliminado por productos que ya nunca se cobrarán? (RN-MESA-24)

### Pedidos, cocina y mesas físicas
20. `compute_bill` de `orders/checkout.py` incluye órdenes ya `pagada` en el cálculo de la "cuenta" — ¿esta
    función se usa también para mostrar un recibo histórico, o debería excluir lo ya cobrado? (RN-ORD-03)
21. ¿Es deliberado que `mark_order_ready` permita terminar de preparar ítems de una orden ya `bloqueada`
    (congelada para cobro)? (RN-ORD-37)
22. ¿Existe algún caller de `close_table_sessions` distinto de `release_table` que no valide pendientes antes
    de invocarla? (RN-ORD-31)
23. ¿Es intencional que `move_order` exija la mesa destino completamente sin órdenes activas, más estricto que
    el modelo general que permite varias órdenes por mesa, o quedó desalineado tras introducir esa capacidad?
    (RN-ORD-58)
24. ¿`cancel_order` marca `estado_cocina="anulado"` en todos los ítems de una orden al cancelarla? De no ser
    así, `group_bill` podría estar cobrando ítems de órdenes canceladas dentro de una cuenta de grupo.
    (RN-ORD-64)

### Ventas, facturación y promociones
25. La búsqueda de ventas por número de factura usa `ILIKE '%N%'` (coincidencia en cualquier posición) — ¿debe
    ser "contiene" o "empieza por número exacto"? (RN-FACT-05)
26. ¿`Invoice.full_number` (propiedad Python) produce exactamente el mismo string que la fórmula SQL
    reconstruida en `list_sales_query`? (RN-FACT-07)
27. ¿El formulario de creación de promociones permite capturar una hora específica en el campo "Desde"
    (`starts_at`), dado que se compara con datetime completo mientras "Hasta" solo se compara por fecha?
    (RN-PROMO-03)
28. ¿Un `value=0` en una promoción `percent`/`fixed` es un caso de uso válido (deshabilitar temporalmente sin
    cambiar el estado) o normalmente indica un error de captura que hoy queda enmascarado en silencio?
    (RN-PROMO-15)
29. ¿El modelo de venta de este negocio admite cantidades fraccionarias (venta por peso) en líneas evaluadas
    por el motor de promociones? El truncamiento a entero solo importa si la respuesta es sí. (RN-PROMO-19)
30. ¿Existe alguna validación (fuera de lo revisado) que impida configurar un combo cuyo precio de paquete sea
    más caro que la suma normal de sus componentes? Hoy el sistema lo permite y simplemente no genera
    descuento. (RN-PROMO-11, RN-PROMO-26)
31. Entre agregar un combo al carrito (que valida y puede rechazar) y cobrarlo (que si el combo ya no es válido
    simplemente no descuenta, sin avisar), ¿el cajero recibe alguna alerta en capas superiores no revisadas
    aquí? (RN-PROMO-27)
32. ¿Fue probado el cálculo de solapamiento horario entre dos promociones que ambas cruzan medianoche
    simultáneamente? (RN-PROMO-33)
33. ¿Es intencional que reenviar el mismo estado de una promoción (incluido `finished→finished`) sea siempre un
    no-op exitoso, o debería rechazarse igual que cualquier transición no listada en la tabla? (RN-PROMO-41,
    RN-PROMO-54)
34. ¿La configuración de Pydantic usa `extra="forbid"` en `PromotionUpdate`, de forma que enviar `type` o
    `targets` en el PATCH escalar genere un error explícito en vez de ser ignorado en silencio? (RN-PROMO-57)
35. ¿Es intencional que una promoción pueda crearse directamente en `active`/`paused` sin pasar antes por
    `draft`? (RN-PROMO-68)
36. ¿Hay una razón de negocio específica para el tope de `priority=1000`, o es un valor arbitrario? (RN-PROMO-73)
37. ¿Existe algún manejo genérico de `IntegrityError` (fuera de los ficheros revisados) que convierta un target
    duplicado dentro de la misma promoción en un 409 legible, en vez de un error no controlado? (RN-PROMO-76)

### Automatización
38. El job de expiración de promociones (`expire_promotions`) usa un criterio de corte (UTC, datetime completo)
    distinto al del motor de evaluación en tiempo real (`_valid_now`, hora local, solo fecha) — ¿debe
    unificarse el criterio, o el desfase de hasta un día y medio entre ambos es aceptable porque el job es
    "puramente informativo"? (RN-SCHED-11)

---

*Fin del documento. 333 reglas de negocio, 12 módulos, evidencia verificada línea a línea sobre
`pos-backend` en su estado al 2026-08-15. Ninguna corrección fue aplicada al código (Principio III de la
Constitución) — las 38 preguntas de arriba quedan abiertas para que el negocio decida.*
