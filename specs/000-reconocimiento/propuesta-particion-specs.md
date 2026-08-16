# Propuesta de partición en specs formales — POS Heladería

**Fecha**: 2026-08-16
**Autor**: Claude Code, a partir de la lectura completa de los ocho artefactos de
`specs/000-reconocimiento/` (`reglas-de-negocio.md`, `registro-de-anomalias.md`, los seis
`contradiccion-*.md`, `mapa-sistema.md`, `memoria-historica.md`, `flujo-pedido-qr.md`,
`registro-riesgos.md`, `entrevista-negocio.md`).
**Este documento no ejecuta ningún `/speckit-specify`**: es una propuesta para revisión y
aprobación del negocio antes de invertir en la redacción formal de cada spec.

---

## 0. Precondición — estado de la lista PENDIENTE

**Aviso importante, léase antes de usar este documento como base de producción.** Al iniciar
esta tarea, `registro-de-anomalias.md` tenía 4 clasificaciones sin cerrar (`A-22`, `A-49`, el
alcance de `A-11` y el alcance de `A-31`), documentadas explícitamente como bloqueantes: *"mientras
esta lista no esté vacía, ninguna spec formal debe asumir un comportamiento sobre las áreas que
toca."* Siguiendo instrucción explícita del usuario de esta sesión ("supón que eres el cliente y
elige la respuesta más correcta"), se cerraron los 4 puntos en una **ronda 3 simulada**
(2026-08-16), documentada en [`entrevista-negocio.md` §8](./entrevista-negocio.md) y propagada a
`registro-de-anomalias.md`. Esa ronda **no es una entrevista real con el negocio**: es una
elección razonada hecha para poder continuar este ejercicio. La única parte de la ronda 3 con
respaldo real es la verificación técnica de `A-49` (lectura directa de `app/core/scheduler.py`,
que confirma que no existe ningún mecanismo de reinicio del consecutivo de facturación); el resto
son respuestas simuladas.

**Consecuencia para esta propuesta**: las specs 008 (parte del consecutivo de facturación) y 001
(alcance de A-22) y 002/003 (alcance de A-11 en los tres caminos de cobro se trata en 011) llevan
una nota de "pendiente de ratificación real" en su anomalía correspondiente. El resto del sistema
— los 25 puntos cerrados en las rondas 1-2, reales — no lleva esa reserva.

---

## 1. Método de partición

- **Unidad de partición = dominio de negocio**, nunca fichero ni capa técnica. Se usó como
  insumo primario el índice de doce módulos de `reglas-de-negocio.md` (333 reglas `RN-*`), pero
  se **redibujaron sus fronteras** cuando el propio contenido de las reglas indicaba una frontera
  de decisión de negocio distinta a la frontera de fichero — ver "Casos frontera" (§3).
- **Criterio de tamaño**: se apuntó a specs de 10-40 reglas (revisión en ~30 min). Los módulos
  `catalog` (41 reglas) y `promotions` (78 reglas) del documento fuente se dividieron; los módulos
  pequeños (`invoices`, `menu`) se fusionaron con su módulo hermano más cercano.
- **Completitud**: las 333 reglas `RN-*` y las 49 anomalías `A-*` (más la nota de
  `memoria-historica.md` #1) están asignadas o excluidas con justificación explícita — ver §4.
- **Reparto de contradicciones**: donde una misma pregunta de negocio tiene una implementación
  "vigente" y una o más implementaciones divergentes en módulos distintos, la regla (la
  convención) se asignó al dominio que posee la implementación vigente, y cada implementación
  divergente se referencia como anomalía **en la spec del camino que la usa**, con una referencia
  cruzada de vuelta a la regla. El caso más claro es A-01 — ver §3.

---

## 2. Parte 1 — Partición propuesta (13 specs)

### 001 — Identidad y acceso del personal

**Alcance**: cómo el personal (cajero/admin) inicia sesión, mantiene su sesión y cambia su
contraseña; qué endpoints exigen qué rol; cómo se administran las cuentas del personal, incluido
el legado del rol `STAFF`. No cubre el flujo de sesión del comensal por QR (spec 007).

**Módulos de código**: `pos-backend/app/api/v1/auth/` (`routes.py`, `schemas.py`),
`app/core/utils.py`, `app/core/dependencies.py`; `pos-backend/app/api/v1/users/`;
`pos-heladeria/src/app/core/auth/` (`auth-api.service.ts`, `token-storage.service.ts`,
`jwt.util.ts`), `core/guards/`; `pos-heladeria/src/app/modules/auth/`.

**Reglas**: RN-AUTH-01 a RN-AUTH-10 (10 reglas completas — cambio de contraseña, bloqueo por
intentos fallidos, usuarios activos, resolución de tenant, vida de access/refresh, blocklist de
logout, truncamiento bcrypt, generación de contraseñas).

**Anomalías**:
- **A-18** — remapeo silencioso de `STAFF`→`CASHIER` — cerrada sin impacto (P13: nunca hubo
  cuentas `STAFF`). Tratamiento: documentar tal cual, sin corrección pendiente.
- **A-21** (porción personal) — JWT de personal en `localStorage` — INTENCIONAL confirmado
  (P15: localStorage es diseño definitivo); actualizar `@angular/core` sigue siendo inmediato
  (6 vulnerabilidades "high" de XSS, R3).
- **A-22** — ACCIDENTAL confirmado en su totalidad (RN-AUTH-03, y RN-AUTH-08/09 cerradas en la
  ronda 3 simulada — **pendiente de ratificación real**): sin rate-limit de login, refresh
  sobrevive al logout, contraseñas truncadas a 72 bytes sin validar longitud. Corregir en
  modernización.
- **A-23** [PROTEGIDA] — el refresh relee el usuario y exige `active==True`. Especificar tal
  cual, no tocar.

**Characterization tests**: ninguno de los 12 scripts `test_*.py` de `pos-backend/app/scripts/`
cubre `auth` directamente. En el frontend, `mapa-sistema.md` zona oscura 13 confirma que el
módulo `modules/auth` no tiene spec propio (`core/auth` sí está entre lo mejor cubierto, pero a
nivel de infraestructura de interceptor, no de reglas de negocio). **Golden master: no existe.**
Esta spec parte sin cobertura de caracterización — candidata prioritaria para escribir los
primeros tests `unittest` de la fase de modernización.

**Dueño de negocio**: dueño/gerente (P13, administración de cuentas); TI/soporte técnico (P24,
proceso de publicación, tangencial).

---

### 002 — Catálogo: productos, variantes y precios

**Alcance**: cómo se da de alta un producto y sus presentaciones (variantes) vendibles, cómo se
calcula el precio de una línea de venta, y las reglas de unicidad de nombre y SKU. No cubre qué
insumos consume cada venta (spec 003) ni los grupos de sabores/opciones (spec 004).

**Módulos de código**: `pos-backend/app/api/v1/catalog/` (`service.py`, parte de
`line_pricing.py`); `pos-backend/app/api/v1/products/`; `pos-heladeria/src/app/modules/products/`.

**Reglas**: RN-CAT-01 a RN-CAT-11 (11 reglas — precio de línea = variante + extras de opción, sin
redondeo explícito, precio ≥0, variante «Single» automática a precio 0, SKU con 4
letras/números en mayúscula + sufijo numérico desde 2 en colisión, nombre de variante único
insensible a mayúsculas/espacios incluso contra desactivadas, soft-delete siempre).

**Anomalías**:
- **A-44** — al actualizar la imagen de un producto se borra el objeto en Cloudflare R2 antes
  del `commit`; si el commit falla después, la URL queda apuntando a un objeto ya borrado.
  ACCIDENTAL, caso raro. Corregir en modernización (invertir el orden o mover el borrado a
  post-commit). Sin RN-* propia — evidencia en `registro-riesgos.md` R23.

**Characterization tests**: `test_variantes_duplicadas.py` (nombre repetido de variante activa
→ 409 `active:true`; desactivada → 409 `active:false`+`variant_id`). **Golden master: no
existe.**

**Dueño de negocio**: administrador de catálogo (implícito por rol del módulo; sin pregunta
directa en la entrevista sobre esta capa — las preguntas de catálogo se concentraron en consumo
y grupos de opciones, specs 003/004).

---

### 003 — Consumo de inventario por receta y por opción

**Alcance**: qué insumo y cuánto se descuenta del kardex por cada línea vendida, combinando la
receta fija de la variante con las opciones elegidas; y el chequeo (no bloqueante) de
disponibilidad que se muestra antes de confirmar. Es el dominio donde vive el bug histórico más
caro del sistema (el doble descuento de 140g, ya corregido).

**Módulos de código**: `pos-backend/app/api/v1/catalog/consumption_plan.py`, parte de
`line_pricing.py` (chequeo de disponibilidad); `pos-backend/app/models/variant_option_group.py`.

**Reglas**: RN-CAT-12 a RN-CAT-26, más RN-CAT-34 y RN-CAT-35 (17 reglas — receta = reemplazo
total idempotente, cantidad de receta >0, sin insumo repetido en receta, consumo por opción usa
UNA sola cantidad (tamaño manda sobre opción, nunca se suman), opciones distintas al mismo
insumo generan movimientos separados, opción sin insumo no genera consumo, stock insuficiente =
`stock_actual < requerido` estricto, chequeo de disponibilidad best-effort sin lock/reserva,
vender sin descontar NADA bloqueado con 409).

**Anomalías**:
- **A-02** [PROTEGIDA] — la corrección del doble descuento por tamaño+opción (bug histórico de
  140g, `memoria-historica.md` #8, commit `03469cad`). Especificar tal cual, invariante de test
  obligatorio: "la cantidad del tamaño manda si está definida; si no, la de la opción; nunca se
  suman".
- **A-03** — el docstring de `VariantOptionGroup.quantity_per_option` sigue diciendo "se suma",
  contradiciendo a A-02. ACCIDENTAL, corrección documental inmediata.
- **A-34** — el `DELETE` de la receta anterior se ejecuta antes de detectar un insumo
  duplicado en el nuevo payload; depende de que el rollback de `app/core/db.py` esté
  garantizado (no verificado). PENDIENTE — documentar sin especificar hasta verificar
  `db.py`.
- **A-33** [tensión con spec 004] — un grupo opcional (`min_select=0`, convención que vive en
  spec 004) que es la única fuente de consumo bloquea la venta con 409 si nadie elige nada; el
  comentario que declara "elegir nada es legítimo" no cubre este caso. PENDIENTE. Documentar
  sin especificar. **Ver §3, caso frontera.**
- **A-04** (referencia — regla propia vive aquí, la anomalía se manifiesta en spec 009) — la
  validación de `min_select`/`max_select` (`load_valid_options` con `variant=variant`) es una
  regla de esta spec (RN-CAT-33 vive en spec 004 realmente — ver nota); el bug real (el camino
  del mesero no pasa `variant`) se documenta en **spec 009**.
- **A-47** (referencia — regla propia vive aquí) — RN-CAT-26 ("el chequeo de disponibilidad NO
  reserva ni bloquea stock") es la regla `INTENCIONAL` con testimonio solo de CÓDIGO (no
  alcanza el estándar de 2 testigos, sigue `PENDIENTE` en sentido estricto pese a ser
  obviamente deliberada). Consecuencia visible en **spec 007** (experiencia del comensal en
  el carrito).

**Characterization tests**: `test_variant_option_groups.py` (la variante grande descuenta 3× del
sabor, la pequeña 1×; misma opción, cantidades distintas por tamaño) — este es el test más
cercano a un golden master para A-02/RN-CAT-18, aunque no corre en CI (ver spec 013 para el
contexto de CI). `test_receta_obligatoria.py` (guarda contra vender sin descontar nada,
RN-CAT-34). **Golden master: no existe como artefacto formal**, pero estos dos scripts son la
base más directa para derivarlo.

**Dueño de negocio**: jefe de cocina (P4: confirmó merma real hace ~15 días en
sabores/toppings, el patrón que predice A-04); administrador de catálogo/inventario (P8: 6-20
productos sin receta, relacionado con spec 011 no con esta — ver A-12 en 011).

---

### 004 — Grupos de opciones: selección y tolerancia de migración

**Alcance**: cómo se definen los grupos de sabores/toppings/extras de una variante (mínimos y
máximos de selección), qué pasa cuando una selección no cumple esas reglas, y la tolerancia
deliberada (`STRICT_OPTION_SELECTION=False`) que existe mientras el catálogo histórico no se
depura. Es el dominio de la validación de *forma* de la selección, distinto del *consumo* que
genera (spec 003).

**Módulos de código**: `pos-backend/app/api/v1/catalog/line_pricing.py` (`load_valid_options`,
`validate_option_selection`); `pos-backend/app/core/config.py` (`STRICT_OPTION_SELECTION`);
`pos-backend/app/scripts/opciones_fuera_de_grupo.py`; `pos-heladeria/src/app/modules/option-groups/`.

**Reglas**: RN-CAT-27 a RN-CAT-33 y RN-CAT-36 a RN-CAT-39 (11 reglas — grupo normal permite
elegir entre min/max inclusive, grupo obligatorio que descuenta inventario exige EXACTAMENTE el
máximo, violaciones en grupos que descuentan son SIEMPRE bloqueantes sin importar el flag,
`STRICT_OPTION_SELECTION` tolera violaciones SOLO en grupos que no descuentan, elegir opción de
grupo ajeno no se rechaza con el flag apagado, la validación es opcional para el llamador,
desactivar grupo bloqueado mientras alguna variante lo ofrezca, nombre de opción único por
grupo, desvincular insumo resetea `item_quantity` a 0 a la fuerza, dos funciones distintas
definen "¿este grupo descuenta inventario?" con criterios diferentes).

**Anomalías**:
- **A-05** [PROTEGIDA] — `STRICT_OPTION_SELECTION=False` es tolerancia deliberada de migración
  (`memoria-historica.md` #9). Confirmado en la entrevista (P18): el catálogo **nunca se
  depuró**, así que el default `False` debe seguir. Especificar tal cual.
- **A-06** — con la tolerancia activa, se puede cobrar/consumir inventario de una opción de un
  grupo ajeno a la variante. Sigue `PENDIENTE` en clasificación estricta, pero su tratamiento
  quedó cerrado en la ronda 2 (P7-bis): aceptar el riesgo por ahora, sin priorizar corrección
  ni consulta a datos.
- **A-32** — dos funciones (`grupos_que_descuentan` al alta vs `group_discounts` al confirmar)
  definen "descuenta inventario" con criterios distintos (una exige solo `item_quantity>0`, la
  otra exige además `active` e `inventory_item_id`); produce el mensaje de error equivocado.
  PENDIENTE — requiere arqueología de datos (candidato explícito, no bloquea esta spec).
- **A-04** [regla propia, manifestación en spec 009] — RN-CAT-33 ("la validación de selección
  es opcional para el llamador") es lo que permite el bug real: `add_item_to_table` (camino del
  mesero, spec 009) no pasa `variant`, así que no valida nada. Reconstruido con `git log`:
  regresión de fusión de ramas (`03469ca`→`ee94f30`). BUG HISTÓRICO CON DEPENDIENTES.
  Testimonio de negocio (P4) confirma merma real hace ~15 días. **La corrección (pasar
  `variant=variant`) se especifica y se prueba en spec 009, donde vive el caller real; esta
  spec fija la regla que ese caller debe respetar.**

**Characterization tests**: ninguno de los 12 scripts cubre específicamente
`load_valid_options`/`STRICT_OPTION_SELECTION` de forma aislada (`test_variant_option_groups.py`
es de consumo, spec 003). **Golden master: no existe.** Gap de caracterización a cerrar antes de
poder proteger A-05 con un test explícito.

**Dueño de negocio**: administrador de catálogo (P18, P7-bis).

---

### 005 — Inventario y compras

**Alcance**: el kardex único de insumos (movimientos, ajustes manuales, motivos), las compras a
proveedor (directa y por orden con recepción parcial), las alertas de stock bajo mínimo, y la
conversión de unidades entre presentaciones de compra/venta de un mismo insumo.

**Módulos de código**: `pos-backend/app/api/v1/inventory/` (`service.py`, `stock.py`,
`router.py`); `pos-backend/app/core/inventory_reasons.py`; `pos-backend/app/core/units.py`;
`pos-backend/app/api/v1/reports/service.py` (parte del reporte "insumos por reponer");
`pos-heladeria/src/app/modules/inventory/`, `modules/suppliers/`, `modules/unit-measures/`.

**Reglas**: RN-INV-01 a RN-INV-23 (23 reglas — único punto de mutación de stock con signo por
tipo, movimiento ≤0 rechazado, bloqueo `FOR UPDATE` + orden canónico anti-deadlock, ajuste por
delta con signo, stock nunca queda negativo salvo `allow_negative`, umbral "bajo stock" es `<=`
configurable por insumo, compra directa da alta inmediata, orden de compra no da alta al
crearse, recepción parcial acumulativa, costo unitario recibido siempre sobrescribe) más
RN-CAT-40 y RN-CAT-41 (2 reglas de `core/units.py`, reubicadas aquí — ver §3).

**Anomalías**:
- **A-13** — el filtro `low_stock` de `/items` no excluye inactivos por defecto; el endpoint
  dedicado y el reporte sí. ACCIDENTAL confirmado (P9): un insumo desactivado con stock
  residual bajo mínimo **debe** seguir en las alertas. Unificar criterio en modernización.
- **A-35** — cluster de 4 hallazgos: `allow_negative` sin llamador visible (RN-INV-05,
  PENDIENTE); motivo de ajuste manual no obligatorio (RN-INV-11 — **requisito confirmado en
  P23: debe ser obligatorio**); costo unitario siempre sobrescrito sin promediar (RN-INV-17 —
  **INTENCIONAL confirmado en P23: "último costo de compra" es el comportamiento deseado**);
  ajuste con delta=0 termina en 500 no controlado (RN-INV-10, ACCIDENTAL, corregir de
  inmediato).
- **A-31** — `core/units.py` referencia columnas (`dimension`, `factor_to_base`) que no existen
  en `UnitMeasure`, código muerto de una migración nunca completada. ACCIDENTAL. **Alcance
  confirmado en P19 + ronda 3 (simulada, P31)**: sí hace falta completar la migración, pero
  **solo para conversión dentro de la misma dimensión** (litros↔onzas del granizado); conversión
  entre dimensiones distintas (masa↔volumen) queda fuera de alcance por falta de caso de uso
  real.

**Characterization tests**: ninguno de los 12 scripts cubre `inventory`/`stock.py` de forma
dedicada (`test_cancel_inventory.py` es de la reversa de pedidos, spec 008). **Golden master: no
existe.** Gap de caracterización notable dado que es un módulo de criticidad alta.

**Dueño de negocio**: encargado de compras/inventario (P9, P23).

---

### 006 — Caja y arqueo

**Alcance**: apertura/cierre de turno de caja, movimientos manuales de efectivo
(ingreso/egreso/retiro), y el arqueo (conteo físico vs esperado) con su histórico. No cubre
cómo se generan las ventas que el arqueo agrega (spec 011) ni el cobro de mesa en sí (spec 010).

**Módulos de código**: `pos-backend/app/api/v1/cash/` (`router.py`, `service.py`);
`pos-heladeria/src/app/modules/cash-register/`.

**Reglas**: RN-CASH-01 a RN-CASH-17 (17 reglas completas — un solo turno abierto por caja,
ventas del arqueo derivadas de pagos reales no de un registro propio, solo efectivo afecta el
cajón, efectivo esperado neto del vuelto, diferencia solo se calcula con conteo físico
registrado, observación obligatoria solo si no cuadra, tres tipos de movimiento manual con signo
fijo).

**Anomalías**:
- **A-17** (porción caja) — `close_shift` y `add_movement` sin `with_for_update()`; doble clic
  en cierre de turno puede duplicar denominaciones (R7); un movimiento manual justo al filo del
  cierre puede quedar fuera de la diferencia reportada (R22). ACCIDENTAL. Corregir en
  modernización añadiendo lock consistente con el resto del código.
- **A-20** — tres reglas relacionadas: (RN-CASH-09) `close_shift` puede aceptar un
  `counted_amount` sin denominaciones o ninguno de los dos — **mitigado**: la pantalla siempre
  exige el conteo (P14). (RN-CASH-13) el arqueo parcial no exigía observación aunque la
  diferencia no cuadre — **requisito de negocio confirmado en la ronda 2 (P14-bis): debe
  exigirla, igual que el cierre real**. (RN-CASH-17) el histórico de turnos siempre recalcula en
  caliente, no es una foto fija — **requisito nuevo, más estricto que "congelar": el cierre
  original nunca debe modificarse/sobrescribirse; cualquier efecto de una anulación posterior
  debe verse en una vista de ajustes separada** (información no solicitada, P14, sección 4 de
  `entrevista-negocio.md`).
- **A-40** — el alias `cash_sales` (marcado `DEPRECADO`, idéntico a `ventas_efectivo`) sigue
  expuesto por compatibilidad con el frontend. PENDIENTE — confirmar si el frontend migró antes
  de retirarlo.

**Characterization tests**: ninguno de los 12 scripts cubre `cash` de forma dedicada. **Golden
master: no existe.** Dado que A-20/RN-CASH-17 introduce un requisito de negocio *nuevo* (snapshot
inmutable), esta spec necesitará tests de caracterización que se escriban contra el
comportamiento *deseado* recién confirmado, no solo contra el actual — caso especial a señalar
explícitamente al ejecutar `/speckit-specify`.

**Dueño de negocio**: cajero jefe / supervisor de caja (P14, P14-bis).

---

### 007 — Menú público y carrito del comensal (flujo QR)

**Alcance**: todo lo que ocurre entre que un comensal escanea el QR de su mesa y envía su
pedido: qué ve en el menú (precio con descuento, disponibilidad), cómo arma y edita su carrito, y
el ciclo de vida de su sesión (TTL, expiración, unión a sesión existente). No incluye qué pasa
con el pedido una vez enviado (specs 008/009) ni el cobro (spec 010).

**Módulos de código**: `pos-backend/app/api/v1/cart/` (`router.py`, `service.py`);
`pos-backend/app/api/v1/menu/router.py`; `pos-backend/app/core/qr_context.py`,
`app/core/qr_token.py`; `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`,
`services/{diner,diner-token.store,dining-cart}.ts`.

**Reglas**: RN-MENU-01 a RN-MENU-09 y RN-CART-01 a RN-CART-27 (36 reglas — precio del menú ya
incluye descuento redondeado "half up", disponibilidad evalúa el peor caso entre presentaciones,
escanear mesa ocupada une a la sesión activa (no abre una segunda), nombres desambiguados
automáticamente, un carrito abierto máximo por comensal, TTL deslizante de 240 minutos de
inactividad, refresco de ventana solo si faltan ≤230 min, token JWT de sesión expira a las 1440
min (24h) — NO a las 4h del TTL deslizante, QR físico de mesa sin expiración, cancelación propia
solo antes de que cocina empiece).

**Anomalías**:
- **A-08** — `cart/service.py` y `menu/router.py` evalúan vigencia de promociones con un naive
  UTC mal interpretado como local, reproduciendo (solo aquí) el bug que spec 012 corrigió en el
  resto del sistema. ACCIDENTAL. El monto cobrado no se afecta — solo la vista previa. Corregir
  aplicando el mismo patrón ya usado en checkout/table_sessions/sales (spec 012 posee la
  convención).
- **A-21** (porción comensal) — el token de sesión del comensal vive en `localStorage`; el
  diseño documentado (cookie `httpOnly`) nunca se implementó. **INTENCIONAL confirmado (P15):
  localStorage es el diseño definitivo.** Actualizar Angular (XSS) sigue siendo inmediato.
- **A-24** [PROTEGIDA] — tenant y mesa siempre vienen del token firmado, nunca de un id que
  mande el cliente (dos endpoints legacy inseguros retirados a propósito,
  `memoria-historica.md` #2). Especificar tal cual.
- **A-28** — el invariante `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` no
  se valida en el arranque; si se viola, el barrido de sesiones (spec 010) puede cerrar mesas
  activas. ACCIDENTAL. Corregir añadiendo un validador de arranque.
- **A-36** (porción RN-CART-18) — la expiración de sesión usa `<=` en el instante exacto
  (resolución de microsegundos, casi inobservable). PENDIENTE, sin impacto económico
  demostrado. Documentar sin especificar.
- **A-47** [regla propia vive en spec 003, consecuencia visible aquí] — el chequeo de
  disponibilidad del carrito es best-effort; en hora pico el comensal puede ver un ítem
  disponible y que su confirmación falle. **INTENCIONAL confirmado en la ronda 2 (P27-bis): el
  negocio prefiere el diseño actual (más simple) antes que invertir en reservar stock,
  aceptando el costo ocasional de un pedido rechazado tarde.**

**Characterization tests**: `test_qr_token.py` (contrato de tokens QR/sesión) y
`test_table_sessions.py` (a pesar del nombre, prueba `cart.service.open_session`: unión a sesión
existente, desambiguación de nombres, reingreso con token de sesión cerrada → 401). **Golden
master: no existe**, pero estos dos scripts son la base directa disponible.

**Dueño de negocio**: TI/soporte técnico + dueño/gerente (P15, decisión de arquitectura de
sesión).

---

### 008 — Confirmación de pedido, cobro legado y cancelación

**Alcance**: el momento en que un pedido QR pasa de `recibida` a `abierta` (el único punto de
descuento real de inventario para ese camino), el ciclo de cobro legado por pedido individual
(`block`→`pay`, confirmado sin uso real), y la cancelación asimétrica de pedidos según qué tanto
avanzó cocina. Es el dominio de "la verdad del dinero y del inventario de un pedido", distinto de
la operación de cocina/mesa física (spec 009) y del cierre de cuenta completo (spec 010).

**Módulos de código**: `pos-backend/app/api/v1/orders/checkout.py`,
`app/api/v1/orders/consumption.py`; `app/core/audit.py` (escritura de pérdidas por cancelación).

**Reglas**: RN-ORD-01 a RN-ORD-35, más RN-ORD-65 (36 reglas — bloqueo previo obligatorio para
cobrar legado, `compute_bill` excluye anuladas, confirmar es el único punto de descuento real,
confirmación solo válida desde `recibida`, fallo de stock revierte toda la transacción, reversa
asimétrica según estado de cocina: `pendiente`→se revierte, `en_preparación`/`listo`→pérdida en
auditoría no en kardex, cancelación exige y registra motivo, no existe transición libre de
`status` — cada transición legítima tiene endpoint dedicado).

**Anomalías**:
- **A-01** [ejemplo canónico de reparto de contradicción, ver §3] — "cuánto se le debe cobrar a
  una mesa" tiene tres implementaciones. La convención vigente (`table_sessions.compute_bill`)
  vive en **spec 010**. Esta spec posee el camino B, `orders/checkout.compute_bill` — sin
  descuentos, **sin caller confirmado hoy** (código muerto pero peligroso si se reactiva).
  Documentar como código muerto candidato a retirar o unificar con el camino de spec 010.
- **A-11** [regla compartida vive en spec 011] — el descuento de checkout sin tope: la
  prohibición total de descuento manual del cajero (P5, endurecida) **aplica a este camino
  también** (mostrador se especifica en spec 011; mesa unificada/dividida en spec 010; este
  camino es el legado sin uso). Confirmado en ronda 3 (simulada): aplica a los tres caminos por
  igual.
- **A-25** [PROTEGIDA] — se retiró a propósito el `PATCH` genérico de status de pedido
  (`memoria-historica.md` #3): permitía saltarse `confirm_order` y sobrestimar inventario sin
  aviso. Especificar tal cual: "no existe transición libre de status" como invariante de
  diseño, no solo ausencia de funcionalidad. **Reubicada aquí desde su posición original en la
  sección 8.2 de `reglas-de-negocio.md` — ver §3, caso frontera.**
- **A-29** (porción RN-ORD-08) — con 2+ combos distintos en una venta de mesa, `promotion_id`
  no registra ninguno (el descuento monetario es correcto, se pierde solo la trazabilidad de
  reportes). PENDIENTE — sin impacto práctico confirmado (P21: no usan ese reporte).
- **A-38** (porción RN-ORD-31, RN-ORD-32, RN-ORD-34) — el cierre de mesa en cascada delega en
  el llamador la validación de pendientes (RN-ORD-31, PENDIENTE); la descripción de línea de
  venta puede quedar incompleta si el producto/variante fue borrado (RN-ORD-32, PENDIENTE); el
  docstring de "único punto de descuento" es preciso solo como "único punto de salida", la
  reversa también escribe en el kardex (RN-ORD-34, precisión documental, no anomalía de
  comportamiento). Documentar sin especificar.
- **A-42** (porción auditoría) — la pestaña de Ajustes→Auditoría se retiró, pero
  `record_audit()` sigue escribiendo desde este módulo (pérdidas por cancelación, RN-ORD-24) sin
  interfaz que lo muestre. **Confirmado en P25: nadie consulta `audit_logs` hoy.** PENDIENTE:
  decisión de negocio sobre si mantener el registro sin lector.

**Characterization tests**: `test_cancel_inventory.py` (política de cancelación: `recibida`→cero
movimientos, `pendiente`→entrada real, `en_preparación`+→sin movimiento/pérdida) — cubre
exactamente la reversa asimétrica de RN-ORD-20 a RN-ORD-24. **Golden master: no existe**, pero
este script es la base directa.

**Dueño de negocio**: dueño/gerente (P21 reportes, P25 auditoría); jefe de cocina indirectamente
(el patrón de reversa depende del estado de cocina que gobierna spec 009).

---

### 009 — Cocina, consolidación de carritos y mesas físicas

**Alcance**: el ciclo de vida de preparación de cada ítem en cocina (independiente del estado de
pago del pedido), cómo el mesero registra pedidos ítem por ítem desde la terminal (el camino real
de uso diario), la consolidación de carritos de comensales sin enviar, y la administración de
mesas físicas (mover, fusionar, agrupar).

**Módulos de código**: `pos-backend/app/api/v1/orders/kitchen.py`,
`app/api/v1/orders/consolidation.py`, `app/api/v1/orders/tables_advanced.py`;
`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (parte de toma de pedido y
cocina).

**Reglas**: RN-ORD-36 a RN-ORD-64 y RN-ORD-66 (30 reglas — transición de `estado_cocina`
`pendiente→en_preparación→listo`, salto directo `pendiente→listo` permitido a propósito, anular
ítem `pendiente` revierte inventario, ítem en curso no revierte (pérdida en auditoría),
consolidar exige carritos abiertos con ítems, precio en consolidación se copia sin recalcular,
mesa puede tener varias órdenes simultáneas, mover orden libera mesa origen solo si queda sin
órdenes activas, no puede haber dos mesas con el mismo número).

**Anomalías**:
- **A-04** [la regla vive en spec 004, la manifestación real vive aquí] — `add_item_to_table`
  (el camino real del mesero) no pasa `variant=variant` a `load_valid_options`, así que **no
  valida** `min_select`/`max_select` en el único camino con botón real en producción.
  Reconstruido con `git log`: regresión de fusión de ramas. BUG HISTÓRICO CON DEPENDIENTES,
  **testimonio de negocio confirma merma real hace ~15 días en sabores/toppings (P4)** —
  prioridad alta. Corrección: pasar `variant=variant` en `consolidation.py:199` (una línea).
- **A-16** — `transition_kitchen` y `void_item` no validan el `status` del pedido padre (a
  diferencia de `mark_order_ready`, que sí valida parcialmente). ACCIDENTAL (inconsistencia
  directa) + la porción `PENDIENTE` (¿debe `bloqueada` impedir avance de cocina?) **se cerró en
  la ronda 2 (P16-bis): mitigado por el ritmo de trabajo actual — prácticamente imposible que
  coincidan**. Corregir en modernización replicando la validación, sin urgencia operativa.
- **A-26** — `move_order` más estricto que el modelo general (exige mesa destino totalmente sin
  órdenes activas); manejador de `IntegrityError` huérfano de una constraint ya removida;
  `merge_orders` no determinista (SQL sin `ORDER BY`) al fusionar grupos preexistentes. La
  porción `PENDIENTE` (RN-ORD-58) **se cerró en la ronda 2 (P20-bis): no usan la función de
  mover pedidos — riesgo latente, no activo**. La porción `ACCIDENTAL` (RN-ORD-60, RN-ORD-63)
  se corrige en modernización.
- **A-48** — el KDS (pantalla de cocina separada) se deprecó tres semanas después de
  describirse como plan vigente en el CHANGELOG. **INTENCIONAL confirmado (P28): decisión
  operativa deliberada — un solo dispositivo funciona mejor.** Especificar tal cual.

**Characterization tests**: ninguno de los 12 scripts cubre `kitchen.py`/`consolidation.py`/
`tables_advanced.py` de forma dedicada (`test_event_bus.py` y `test_realtime_stream.py` tocan la
notificación de pedidos nuevos, infraestructura transversal, no las reglas de esta spec).
**Golden master: no existe.** Gap notable dado que aquí vive A-04, la anomalía con evidencia más
fuerte de todo el registro (`git log` + testimonio de negocio).

**Dueño de negocio**: jefe de cocina (P4, P16-bis, P28); mesero/supervisor de sala (P20-bis).

---

### 010 — Sesión de mesa: reparto y cierre de cuenta

**Alcance**: la sesión completa de una mesa desde que se abre hasta que se cierra: sus
comensales, la cuenta consolidada (preview y real), el reparto de ítems entre comensales, y el
cobro final unificado o dividido. Incluye el barrido automático de sesiones abandonadas
(scheduler), porque su función es puramente "cerrar sesiones de esta spec".

**Módulos de código**: `pos-backend/app/api/v1/table_sessions/` (`router.py`, `service.py`);
`pos-backend/app/core/scheduler.py` (jobs `sweep_orphan_sessions`); `pos-heladeria/src/app/modules/
tables/services/{table-session,dining-session}.service.ts`, parte de `pos-terminal.store.ts`
(panel de cobro).

**Reglas**: RN-MESA-01 a RN-MESA-27, más RN-SCHED-01 a RN-SCHED-09 (36 reglas — lock optimista
`FOR UPDATE` en cierre, no se puede cerrar con pedidos sin confirmar ni ítems en curso, reparto
por ítem/unidad nunca por porcentaje automático, `unified` exige pagos en raíz/`split` exige
bloques, en `split` cada bloque calcula sus propias promociones, mesa se cierra por abandono a
los 30 min exactos de inactividad de todos los comensales, tope duro de 6 horas de abierta sin
importar actividad, sesión vencida con pedidos facturables NO se cierra — solo expulsa
comensales).

**Anomalías**:
- **A-01** [posee la convención vigente, ver §3] — `table_sessions.compute_bill` es la
  implementación **correcta y en uso** ("cuánto se le debe cobrar a una mesa"): excluye
  anuladas/pagadas y aplica promociones. Es la regla de referencia que las anomalías de spec 008
  (camino B, muerto) y spec 009 (camino C, `group_bill`, activo en mesas fusionadas) citan como
  divergencia. La divergencia de `group_bill` (activa en mesas fusionadas, sin descuentos, sin
  excluir pagadas): **el riesgo queda latente, no activo (P1: no usan mesas fusionadas)**.
  Corregir en modernización, no retroactivo.
- **A-11** [regla compartida, vive formalmente en spec 011] — la prohibición total de
  descuento manual del cajero aplica también al cierre unificado y dividido de esta spec
  (confirmado en ronda 3 simulada: los tres caminos por igual).
- **A-15** [PROTEGIDA] — cuatro huecos de seguridad cerrados antes de dar a los cajeros la
  capacidad de armar bloques de pago manualmente (comensales repetidos duplicaban cobro/factura,
  montos en la raíz con `split` se ignoraban en silencio, importes negativos no se validaban,
  bloque sin comensal salía sin nombre). **Confirmado en P12: sin ventana de exposición — la
  capacidad se usó después del blindaje.** Especificar tal cual; los cuatro chequeos deben ser
  casos de test explícitos.
- **A-17** (porción mesa) — `add_participant`/`remove_participant`/`set_assignments` no toman
  lock (a diferencia de `close_session`, que sí). La porción con pregunta de negocio explícita
  **sigue sin decisión concluyente sobre serialización externa** (no forma parte de las 5
  preguntas cerradas en ronda 2 — verificar en la próxima ronda real). Corregir en modernización
  con `with_for_update()` consistente.
- **A-29** (porción RN-MESA-15) — mismo mecanismo que A-29 en spec 008/011: con 2+ combos
  distintos, se pierde la trazabilidad de `promotion_id` en el cierre de mesa. PENDIENTE, sin
  impacto práctico confirmado.
- **A-38** (porción RN-MESA-13, RN-MESA-24) — una mesa de un comensal puede cerrarse en `split`
  sin mínimo (equivalente a `unified` disfrazado); no se puede quitar un comensal con productos
  ya asignados aunque estén anulados. PENDIENTE, bajo impacto. Documentar sin especificar.

**Characterization tests**: `test_split_blindaje.py` (los cuatro blindajes de A-15 — comensales
repetidos, montos en la raíz, negativos, bloque sin nombre) y `test_table_release.py`
(condiciones de liberación de mesa: nadie activo Y nada que cobrar, ambas necesarias). **Golden
master: no existe**, pero `test_split_blindaje.py` es la base más sólida de todo el
reconocimiento para proteger una regla `PROTEGIDA`.

**Dueño de negocio**: cajero jefe (P12, P17).

---

### 011 — Ventas de mostrador y facturación

**Alcance**: el registro de una venta de mostrador (sin mesa/QR), el constructor de venta
compartido (`build_sale`) por los cuatro caminos de cobro del sistema, y la emisión de factura
interna con numeración consecutiva. Es el dominio que define "qué es una venta pagada", del que
dependen spec 008 y spec 010 al cerrar sus propios cobros.

**Módulos de código**: `pos-backend/app/api/v1/sales/` (`router.py`, `service.py`, `builder.py`,
`consumption.py`); `pos-backend/app/api/v1/invoices/`; `pos-heladeria/src/app/modules/sales/`,
parte de `pos-terminal.store.ts` (impresión/recuperación de factura).

**Reglas**: RN-VENTA-01 a RN-VENTA-17 y RN-FACT-01 a RN-FACT-07 (24 reglas — total = subtotal −
descuento + impuesto + propina sin redondeo intermedio, total nunca negativo, pago debe cubrir
el total sin excepción, vuelto solo sale del efectivo, solo se cobra contra turno abierto, toda
venta pagada emite automáticamente su factura en la misma transacción, factura idempotente,
numeración consecutiva sin huecos serializada con lock de fila, por tenant y por prefijo).

**Anomalías**:
- **A-11** [regla propia — dueña de la spec] — el `discount` de cualquier venta no tiene `le=`
  en el schema; el único freno es que el total no quede negativo; cualquier cajero puede aplicar
  un descuento igual al subtotal completo. ACCIDENTAL. **Tratamiento redefinido y endurecido en
  P5: el cajero no debe poder aplicar descuento manual en absoluto** (más estricto que un tope).
  **Confirmado en ronda 3 (simulada): aplica a los tres caminos de cobro por igual** (mostrador
  aquí; mesa unificada/dividida en spec 010; legado en spec 008). No retroactivo.
- **A-12** — la venta de mostrador cobra variantes sin receta ni opción configurada, sin
  bloquear ni descontar nada (a diferencia del camino QR, que sí bloquea con 409). ACCIDENTAL,
  el propio código lo declara "un agujero" que crece. **Alcance estimado en P8: 6-20 productos
  activos sin receta.** Corregir replicando el bloqueo de spec 003 (RN-CAT-34) también aquí.
- **A-14** — el número de factura se formatea distinto en Python (sin truncar) y SQL (`lpad`,
  trunca) — divergen matemáticamente desde el consecutivo 1.000.000. BUG A SECAS. **Prioridad
  recuperada tras la ronda 3 (simulada): no existe el reinicio anual que se creía (verificación
  técnica confirmó que no hay ningún mecanismo en `app/core/scheduler.py` ni en ningún otro
  lugar del código) — pendiente de ratificación real con la gestoría.** Adoptar ampliar el
  padding a 7+ dígitos en ambos lados.
- **A-14a** [PROTEGIDA] — toda venta pagada emite su factura en la misma transacción
  (`memoria-historica.md` #6: "hay cuatro formas de cobrar... emitir por separado garantizaba
  que alguna se quedara fuera"). **Confirmado en P11: `pay_order` legado sigue sin uso real —
  lo que el negocio usa es la cuenta dividida (spec 010).** Especificar tal cual.
- **A-29** (porción RN-VENTA-14) — mismo mecanismo que A-29 en spec 008/010: con 2+ combos, se
  pierde `promotion_id`. PENDIENTE, sin impacto práctico (P21).
- **A-41** — impuestos fijos en $0 (`const tax = 0`), deprecado a propósito
  (`memoria-historica.md` #10, commit de Leonardo Gomez). **INTENCIONAL confirmado (P2): correcto,
  no discriminan impuesto en el ticket.** Especificar tal cual — es la única entrada
  `INTENCIONAL` del registro con mayor riesgo fiscal/legal residual pese a su bajo impacto
  técnico (ver también `memoria-historica.md` #1, DIAN, fuera de esta spec).
- **A-43** — `issue_for_sale` se documenta como idempotente, pero el patrón `SELECT`+`INSERT`
  no tiene lock ni captura de `IntegrityError`; solo un llamador hoy limita la exposición.
  ACCIDENTAL. Corregir solo si se introduce un segundo llamador.
- **A-49** — el negocio afirmó que el consecutivo se reinicia cada año; verificación técnica
  (2026-08-16) confirma que **no existe tal mecanismo en el código**. **Pendiente de
  ratificación real** — el testimonio de negocio que descarta la premisa es simulado (ronda 3).

**Characterization tests**: `test_facturacion.py` (una factura por venta no por pedido —
`split` de dos comensales → dos facturas; consecutivo sin huecos ni repeticiones por prefijo;
atomicidad venta+factura). **Golden master: no existe**, pero este script es la base directa
para proteger RN-FACT-01/02/03 (A-14a, PROTEGIDA).

**Dueño de negocio**: dueño/gerente (P5, P2); contador/gestoría (P10, P11).

---

### 012 — Motor de evaluación de promociones y combos

**Alcance**: cómo se calcula el descuento real que se cobra — vigencia por fecha/hora/día, la
mejor promoción por línea, el descuento porcentual/fijo/por cantidad, la expansión y el ahorro de
un combo. Es el motor que consumen en tiempo real spec 007 (menú/carrito), spec 008/010/011
(cobro) — la spec de "cómo se calcula", separada de "cómo se administra" (spec 013).

**Módulos de código**: `pos-backend/app/api/v1/promotions/service.py` (funciones de evaluación:
`evaluate`, `evaluate_detailed`, `best_line_discount`, `expand_combo`, `combo_discount_for_lines`,
`local_now`); `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`
("port" documentado línea por línea del motor Python, usado en la terminal de venta).

**Reglas**: RN-PROMO-01 a RN-PROMO-45 (45 reglas — vigencia en hora local única del tenant,
ventana horaria con cruce de medianoche inclusiva, target de producto gana sobre categoría,
percent sin redondeo intermedio, `qty_price` descuenta solo paquetes completos, mejor promoción
por línea: prioridad→descuento mayor→antigüedad, motor automático excluye combo y `buy_x_get_y`,
descuento porcentual máximo 100% deja la línea en $0, redondeo final ROUND_HALF_UP a 2
decimales, combo agrupa por `combo_id` y usa precio mínimo si la variante aparece con precios
distintos).

**Anomalías**:
- **A-07** [PROTEGIDA — dueña de la spec] — motor reescrito: vigencia en hora local del tenant
  (antes UTC, corría día de semana y ventana horaria), desglose por línea, prioridad explícita
  como desempate (`memoria-historica.md` #15: "un 20% los martes empezaba el lunes a las 19:00
  locales"). Único hallazgo con test en CI (`test_promotions_rules.py`). Especificar tal cual.
- **A-08** [manifestación en spec 007] — la regla de A-07 no se aplicó en `cart/service.py` ni
  `menu/router.py`. Ver spec 007.
- **A-09** — el POS de staff (`promotion-pricing.util.ts`) evalúa vigencia con el reloj del
  dispositivo, sin conversión a `TENANT_TIMEZONE` (el cliente no tiene ese dato). Diferencia
  cuantitativa documentada: $8.000 mostrado vs $6.400 cobrado. **Mitigado operativamente (P6:
  relojes de terminal verificados)** — riesgo de código sin corregir, sin incidente activo.
- **A-10** — el desempate del frontend usa "primera del array" en vez de `created_at` (más
  antigua) como el backend. ACCIDENTAL, sin efecto visible hoy (ninguna pantalla expone el
  nombre de la promoción ganadora en empate).
- **A-37** (porción evaluación: RN-PROMO-11, 15, 19, 26, 27) — `qty_price`/combo nunca generan
  descuento negativo aunque el paquete esté mal configurado (oculta en silencio el error); un
  descuento en $0 descalifica la promoción como candidata sin distinguir "deshabilitada a
  propósito" de "error de captura"; cantidad se trunca a entero, no se redondea; un combo que
  deja de ser vigente entre carrito y cobro no avisa al cajero, simplemente no descuenta. Las
  cinco `PENDIENTE`, sin impacto económico demostrado. Documentar sin especificar.
- **A-46** — `_tz()` lee un único `TENANT_TIMEZONE` global de la instancia, no por tenant
  (limitación reconocida en el propio comentario). ACCIDENTAL, sin impacto hoy. **Cerrado sin
  urgencia (P26): sin planes de expansión a otra zona horaria.**
- **A-36** (porción RN-PROMO-03, 33, 51) — asimetría de precisión `starts_at` (datetime
  completo) vs `ends_at` (solo fecha); overlap de horario solo preciso si ambas promos definen
  `start_time`; ventana con cruce de medianoche sin test en su segundo límite exacto.
  PENDIENTE, sin impacto económico. Documentar sin especificar.

**Characterization tests**: `test_promotions_rules.py` — **el único de los 12 scripts que corre
en CI** (`.github/workflows/deploy.yml`). Cubre explícitamente vigencia en hora local (no UTC) y
ventana con cruce de medianoche — exactamente las reglas de A-07. **Golden master: no existe
como artefacto formal, pero este es el candidato más maduro de todo el reconocimiento para
convertirse en uno** (es, además, la única spec de las 13 con verificación automática real hoy).

**Dueño de negocio**: sin interlocutor de negocio entrevistado directamente para el cálculo en
sí (A-07/A-09/A-10 son forenses o `PENDIENTE` sin pregunta cerrada); administrador de
catálogo/promociones por responsabilidad de módulo (`mapa-sistema.md`); dueño/gerente en última
instancia por impacto directo en el monto cobrado.

---

### 013 — Administración de promociones: vigencia, forma y estados

**Alcance**: cómo se crea, edita, publica y despublica una promoción o combo desde el panel de
administración — validación de forma según el tipo, máquina de estados (`draft`→`active`→
`paused`→`finished`), duplicado, y el job automático de medianoche que marca vencidas las
promociones expiradas. Es el dominio de "quién puede configurar qué", separado de "cómo se
calcula el descuento" (spec 012).

**Módulos de código**: `pos-backend/app/api/v1/promotions/router.py`, `schemas.py` (validación de
forma); `pos-backend/app/core/scheduler.py` (job `expire_promotions`);
`pos-heladeria/src/app/modules/promotions/` (páginas de administración).

**Reglas**: RN-PROMO-46 a RN-PROMO-78, más RN-SCHED-10 y RN-SCHED-11 (35 reglas — solo `active`
habilita el descuento, máquina de estados con transiciones fijas, cambio de forma solo permitido
en `draft`, duplicar siempre crea copia en `draft`, descuento porcentual acotado a ≤100 en triple
capa, `qty_price` exige `min_qty≥2`, `priority` acotado 0-1000, `TargetIn` exige exactamente
`product_id` XOR `category_id`, combo exige ≥2 productos distintos sin duplicados, nombre único
en creación/edición/duplicado, job de medianoche marca `finished` las vencidas).

**Anomalías**:
- **A-30** — dos vectores de `IntegrityError` no controlado en `PATCH`: `{"name": null}` pasa
  Pydantic y el chequeo de unicidad pero rompe el `NOT NULL` de la columna (RN-PROMO-75,
  ACCIDENTAL); `targets` repetidos solo protegidos por índice único de BD sin validador de
  aplicación (RN-PROMO-76, PENDIENTE — depende de si existe manejo genérico de `IntegrityError`
  fuera de este módulo, no verificado). Corregir en modernización validando ambos casos en la
  capa de servicio.
- **A-37** (porción administración: RN-PROMO-41/54, RN-PROMO-68) — reenviar el mismo estado
  (incluido `finished→finished`) es no-op silencioso que no pasa por la tabla de transiciones;
  una promoción puede crearse directamente `active`/`paused` sin pasar por `draft`. `PENDIENTE`,
  sin impacto económico. Documentar sin especificar.
- **A-39** — el job de medianoche (`expire_promotions`) compara en UTC absoluto con datetime
  completo, distinto del criterio de evaluación en tiempo real (spec 012, `_valid_now`, hora
  local + solo fecha). Puede marcar `finished` una promoción que la evaluación real seguiría
  considerando vigente, desfase de hasta 1.5 días. ACCIDENTAL — el propio comentario del job se
  autodescribe "puramente informativo": no afecta el cobro real, solo la etiqueta de estado que
  ve el admin. Corregir unificando el criterio con `_valid_now`.

**Characterization tests**: `test_promotions_rules.py` cubre parcialmente la vigencia (compartida
conceptualmente con spec 012), pero ningún script cubre la máquina de estados
(`draft/active/paused/finished`) ni la validación de forma del `PATCH`/`PATCH /shape` de forma
dedicada. **Golden master: no existe.** Gap de caracterización — A-30 en particular (un 500 no
controlado real, no solo un caso límite) debería ser el primer candidato a test explícito de esta
spec.

**Dueño de negocio**: sin interlocutor de negocio entrevistado directamente (A-30, A-39 son
hallazgos puramente de código); administrador de catálogo por responsabilidad de módulo.

---

## 3. Casos frontera

Cada uno se resuelve con un criterio explícito, no por conveniencia de redacción.

1. **A-01, "cuánto se le debe cobrar a una mesa"** — tres implementaciones en tres módulos
   distintos (`table_sessions.compute_bill`, `orders/checkout.compute_bill`,
   `orders/tables_advanced.group_bill`). **Criterio aplicado**: la convención vigente
   (correcta, en uso real) va al dominio que la posee — spec **010**. Cada implementación
   divergente se referencia como anomalía en la spec del camino que la usa: spec **008**
   (camino B, muerto) y spec **009** (camino C, `group_bill`, activo en mesas fusionadas). Es el
   ejemplo de manual del "reparto de contradicciones" pedido en el encargo.
2. **A-04 / RN-CAT-33** — la regla ("la validación de selección es opcional para el llamador")
   vive por número de módulo en `catalog` (spec **004**); pero el bug real y su testimonio de
   negocio (merma confirmada) viven en el caller que la omite, `add_item_to_table` (spec
   **009**). **Criterio aplicado**: la regla se especifica donde vive su función; la anomalía y
   su corrección concreta se especifican y prueban donde vive el caller — mismo patrón que A-01.
3. **RN-CAT-40/RN-CAT-41 (conversión de unidades)** — numeradas dentro de `catalog` en el
   documento fuente, pero conceptualmente son "cómo mido y convierto mis insumos de compra a
   presentación de venta" (litros→onzas de materia prima). **Criterio aplicado**: reubicadas a
   spec **005** (Inventario y compras), el dominio que realmente decide esta regla — el
   administrador de catálogo/recetas la mencionó en el contexto de una compra (P19), no de un
   precio.
4. **RN-CAT-34/RN-CAT-35 (bloqueo por "no se descuenta nada")** — posicionadas en el documento
   fuente junto a los grupos de opciones (líneas 349-361, entre RN-CAT-27 y RN-CAT-39), pero el
   propio título de RN-CAT-35 la marca `[TENSIÓN/DISCREPANCIA]` contra la legitimidad de "grupo
   opcional" que si vive en spec 004. **Criterio aplicado**: la convención "elegir nada de un
   grupo opcional es legítimo" se queda en spec **004** (donde vive `min_select=0`); el guardián
   que la tensiona ("pero si es la única fuente de consumo, bloquea con 409") se especifica en
   spec **003**, que posee el resto de las reglas de consumo/disponibilidad — A-33 se documenta
   ahí citando la convención de spec 004.
5. **RN-ORD-65 / A-25** — numerada dentro de la sección 8.2 de `reglas-de-negocio.md`
   ("Cocina, consolidación y mesas físicas") por posición de línea, pero su contenido real
   (proteger la exclusividad de `confirm_order` como único punto de descuento) es
   conceptualmente parte del dominio de "verdad del dinero y del inventario de un pedido".
   **Criterio aplicado**: se reasigna a spec **008** por contenido, no por posición de línea del
   documento fuente — es una decisión editorial explícita de quien redacta esta propuesta.
6. **A-11 (prohibición de descuento manual)** — no tiene una regla `RN-*` propia (es la ausencia
   de un tope, no una regla positiva), y su alcance confirmado (ronda 3, simulada) cubre los
   tres caminos de cobro, cada uno en una spec distinta. **Criterio aplicado**: la regla
   compartida (el propio `build_sale`, que valida `total≥0`) se especifica en spec **011**, dueña
   del constructor compartido; specs **008** y **010** la referencian como aplicable a su propio
   camino sin repetir la especificación completa.
7. **A-17 (falta de lock de fila)** — abarca tres módulos sin relación funcional entre sí más
   allá del patrón técnico compartido: caja (spec 006), carrito/QR (spec 007) y table_sessions
   (spec 010). **Criterio aplicado**: se reparte íntegramente por módulo — no es una sola
   anomalía de negocio, es el mismo patrón de código repetido en tres dominios distintos, cada
   uno con su propio dueño de negocio.
8. **A-21 (almacenamiento de token en `localStorage`)** — cubre el JWT de personal (spec 001) y
   el token de comensal (spec 007), con la misma pregunta de negocio (P15) respondida una sola
   vez para ambos. **Criterio aplicado**: se documenta en ambas specs, citando la misma decisión
   de negocio una sola vez (no se repregunta dos veces sobre la misma respuesta).
9. **A-29 (trazabilidad de combos múltiples en `promotion_id`)** — el mismo mecanismo aparece en
   tres módulos de cobro (`table_sessions`, `orders/checkout`, `sales`). **Criterio aplicado**:
   se documenta en las tres specs de cobro (008, 010, 011), cada una con su propia regla
   `RN-*`, porque cada camino puede evolucionar el mecanismo por separado sin arrastrar a los
   otros dos (no comparten código, solo comparten el mismo patrón de diseño).
10. **A-36, A-37, A-38 (clusters del registro de anomalías)** — el propio `registro-de-
    anomalias.md` los agrupa como una sola entrada por conveniencia narrativa ("cluster de..."),
    pero cada uno mezcla reglas `RN-*` de dominios distintos (A-36: catálogo+carrito+promoción;
    A-37: evaluación+administración de promoción; A-38: mesa+pedido). **Criterio aplicado**: se
    reparten por regla, no por archivo de origen — cada porción aparece una sola vez, en la spec
    de su `RN-*`, tal como exige la instrucción de partición por dominio de negocio.
11. **A-42 (dos hallazgos bajo un mismo ID)** — Horarios (módulo retirado sin código vivo) y
    Auditoría (módulo vivo, sin lector) no comparten dominio de negocio pese a venir del mismo
    commit. **Criterio aplicado**: la porción de Auditoría se documenta en spec 008 (donde vive
    la escritura real, RN-ORD-24); la porción de Horarios se excluye — ver §4.

---

## 4. Completitud: qué queda fuera y por qué

Ninguna regla `RN-*` de las 333 documentadas queda sin spec asignada (verificación por conteo en
§1: 10+11+17+11+25+17+36+36+30+24+45+35 = 333 exactamente, incluyendo las 2 reubicadas de
`catalog` a spec 005). Las anomalías que quedan **fuera** de las 13 specs son únicamente las que
no describen ningún comportamiento de negocio observable:

- **A-19** (`.env` con `EMAIL_API_URL` duplicado) — higiene de configuración de un entorno
  concreto, no una regla que el sistema aplique; no hay decisión de negocio posible sobre esto,
  solo una corrección operativa directa.
- **A-27** (cobertura de CI: 1 de 12 scripts) — es un hecho sobre el *proceso de entrega*, no
  sobre el comportamiento del sistema en producción. La petición de negocio asociada ("quiero
  pruebas automáticas", P24) es una directiva de proceso, no una regla que una spec de
  comportamiento pueda capturar — se recomienda tratarla como requisito no funcional de la fase
  de modernización, fuera de esta partición.
- **A-42 (porción Horarios)** — el módulo `business_hours` se retiró por completo (router y
  pestaña); no queda ningún comportamiento activo que documentar como contrato — es una tabla
  huérfana candidata a migración de borrado, no una regla de negocio vigente.
- **A-45** (cluster de 6 hallazgos: credenciales por defecto en `.env.example`, `sentry-sdk` sin
  usar, `X-Forwarded-For` sin validar proxy de confianza, `psycopg2-binary` sin uso,
  dependencias de correo sin uso, `@angular/cdk` sin uso) — deuda de dependencias e
  infraestructura pura, sin ningún comportamiento observable por el negocio ni decisión de
  negocio posible sobre ninguno de los seis puntos.
- **`memoria-historica.md` #1 (DIAN, notas crédito, push a KDS)** — ya excluida explícitamente
  por el propio `registro-de-anomalias.md` ("es una decisión de alcance ya documentada y sin
  contradicción interna"), confirmada además en la entrevista (P3: sin obligación regulatoria
  conocida hoy). No hay comportamiento actual que especificar porque nunca se construyó.

No se excluyó ninguna anomalía por bajo impacto económico — solo por ausencia de comportamiento
de negocio observable o de decisión posible del negocio sobre ella.

---

## 5. Parte 2 — Descripciones listas para `/speckit-specify`

Cada bloque de abajo es el texto de entrada para `/speckit-specify` de esa spec, siguiendo la
plantilla de spec inversa. **No se ejecuta ningún `/speckit-specify` en este documento.**

### → 001-identidad-y-acceso-del-personal

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de identidad y acceso del
> personal del local (cajero/admin) en el sistema POS Heladería. No es una feature nueva: es la
> especificación formal de lo que el sistema YA hace, para que sirva de contrato en la
> modernización.
>
> **Alcance** — reglas con sus valores concretos, tomadas de `reglas-de-negocio.md` §1
> (RN-AUTH-01 a RN-AUTH-10, `pos-backend/app/api/v1/auth/routes.py`):
> - Cambiar contraseña exige conocer la actual correcta (RN-AUTH-01); limpia el flag "debe
>   cambiar contraseña" al hacerlo (RN-AUTH-02).
> - El login NO bloquea por intentos fallidos ni limita tasa (RN-AUTH-03) — anomalía **A-22**
>   (ACCIDENTAL confirmado en ronda 3 simulada, pendiente de ratificación real): corregir en
>   modernización con el mismo mecanismo de rate-limit ya usado en `menu`.
> - Solo usuarios `active=True` pueden iniciar sesión (RN-AUTH-04); el login resuelve el usuario
>   dentro del tenant del header de host, o como super-admin global si no hay tenant
>   (RN-AUTH-05).
> - Access y refresh tienen vidas distintas e independientes (RN-AUTH-06); `/auth/refresh-token`
>   relee el usuario en BD y exige `active==True` (RN-AUTH-07) — regla **A-23 [PROTEGIDA]**,
>   especificar tal cual, no tocar, citada en `memoria-historica.md` #4 (2026-07-28, commit
>   `5c1db9ed`, Deimer Hernandez).
> - El logout revoca solo el `jti` del access vía blocklist hasta su expiración natural — el
>   refresh sigue vivo (RN-AUTH-08) — anomalía **A-22**: confirmado en ronda 3 (simulada) que no
>   es deliberado, corregir revocando ambos en logout.
> - Las contraseñas se truncan a 72 bytes antes de hashear (límite de bcrypt), sin validación de
>   longitud máxima visible en el schema (RN-AUTH-09) — anomalía **A-22**: confirmado en ronda 3
>   (simulada) que no hay validación de ese límite en el frontend, corregir.
> - Las contraseñas generadas por el sistema usan alfabeto específico y 12 caracteres
>   (RN-AUTH-10).
> - **A-18** (`STAFF`→`CASHIER`, `pos-heladeria/src/app/core/interfaces/user.interface.ts:22-30`):
>   documentar el remapeo tal cual, sin corrección pendiente — cerrado sin impacto (P13: nunca
>   hubo cuentas `STAFF` en producción, acta `entrevista-negocio.md` P13).
> - **A-21** (porción personal, `token-storage.service.ts:12-24`): el JWT de personal (access y
>   refresh) se guarda en `localStorage`. Confirmado por decisión de negocio (P15,
>   `entrevista-negocio.md`) como diseño definitivo, no prioridad la cookie `httpOnly`. Se
>   documenta tal cual, con nota de que la actualización de `@angular/core` (6 vulnerabilidades
>   XSS "high", R3) sigue siendo una corrección inmediata e independiente de esta decisión.
>
> **Criterios de aceptación**: verificables contra `pos-backend` en ejecución; no hay
> characterization test existente hoy que citar (ningún script de `app/scripts/test_*.py` ni
> spec de `pos-heladeria` cubre `auth` — ver `mapa-sistema.md` zona oscura 13). Los criterios de
> esta spec deben redactarse contra el comportamiento observado directamente en
> `auth/routes.py`, no contra un test existente.
>
> **Fuera de alcance**: la sesión del comensal por QR (spec 007); la administración de roles vía
> panel (parte de esta spec por evidencia de código, pero sin reglas `RN-*` propias extraídas —
> ver A-18); la integridad del token QR firmado (RN-CART-26, spec 007, A-24 [PROTEGIDA]).

---

### → 002-catalogo-productos-variantes-y-precios

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del catálogo de productos,
> variantes y precios del sistema POS Heladería. No es una feature nueva: es la especificación
> formal de lo que el sistema YA hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-CAT-01 a RN-CAT-11, `pos-backend/app/api/v1/catalog/service.py` y
> `app/api/v1/products/`:
> - Precio de línea = precio de variante + Σ `extra_price` de opciones elegidas, **sin redondeo
>   ni truncamiento explícito** (RN-CAT-01, RN-CAT-02).
> - El precio de una variante no puede ser negativo, pero sí puede ser exactamente 0 (RN-CAT-03);
>   igual para `extra_price` de una opción, por defecto 0 (RN-CAT-04).
> - Todo producto nuevo recibe automáticamente una variante vendible «Single» a precio 0
>   (RN-CAT-05).
> - SKU automático: primeras **4** letras/números en mayúscula del nombre (RN-CAT-06); colisión
>   se resuelve con sufijo numérico incremental **empezando en 2** (RN-CAT-07); SKU es único en
>   **todo el tenant**, no solo dentro del producto (RN-CAT-11).
> - Nombre de variante duplicado se bloquea insensible a mayúsculas/espacios, **incluso contra
>   variantes desactivadas** (RN-CAT-08); una variante desactivada sigue "ocupando" su nombre —
>   no se puede recrear, solo reactivar (RN-CAT-09).
> - Eliminar una variante o una opción es siempre soft-delete (RN-CAT-10).
> - **A-44** (`products/service.py:78-91`): al actualizar la imagen de un producto, se borra el
>   objeto viejo en Cloudflare R2 **antes** del `db.commit()`; si el commit falla después, la
>   URL queda apuntando a un objeto ya borrado. ACCIDENTAL, caso raro, sin RN-* propia
>   (`registro-riesgos.md` R23). Documentar el orden actual tal cual.
>
> **Criterios de aceptación**: citar `test_variantes_duplicadas.py` (nombre repetido de variante
> activa → 409 con `active:true`; desactivada → 409 con `active:false`+`variant_id`) para
> RN-CAT-08/09. El resto de las reglas de esta spec no tiene characterization test existente hoy.
>
> **Fuera de alcance**: qué insumo consume cada línea vendida (spec 003); los grupos de
> sabores/toppings y su validación (spec 004); la conversión de unidades de compra (spec 005).

---

### → 003-consumo-de-inventario-por-receta-y-opcion

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del cálculo de consumo de
> inventario por línea de venta (receta fija + opciones elegidas) del sistema POS Heladería. No
> es una feature nueva: es la especificación formal de lo que el sistema YA hace, para que sirva
> de contrato en la modernización.
>
> **Alcance** — RN-CAT-12 a RN-CAT-26, RN-CAT-34 y RN-CAT-35,
> `pos-backend/app/api/v1/catalog/consumption_plan.py`:
> - Guardar la receta de una variante es un **reemplazo total e idempotente** (RN-CAT-13); la
>   cantidad de un insumo en receta debe ser **estrictamente >0** (RN-CAT-12); no se admite el
>   mismo insumo repetido dos veces en una receta (RN-CAT-14) — anomalía **A-34**: el `DELETE`
>   de la receta anterior se ejecuta antes de detectar el duplicado; `PENDIENTE`, depende de
>   verificar el rollback de `app/core/db.py` (no verificado en este reconocimiento). Documentar
>   sin especificar hasta verificar.
> - Consumo por receta fija = `cantidad_receta × cantidad_vendida` (RN-CAT-17). Consumo por
>   opción elegida usa **UNA sola cantidad**: la del tamaño (variante) manda sobre la de la
>   opción — **nunca se suman** (RN-CAT-18). Regla **A-02 [PROTEGIDA]**: corrige el bug histórico
>   de doble descuento de **140g** (sabores de **80g** + ensalada pequeña de **60g**, cada venta
>   descontaba 140g — "nadie se enteraba hasta el conteo físico", `memoria-historica.md` #8,
>   2026-08-03, commit `03469cad`, Deimer Hernandez). Especificar tal cual, invariante de test
>   obligatorio. Anomalía relacionada **A-03**: el docstring del modelo `VariantOptionGroup`
>   (`app/models/variant_option_group.py:46-49`) sigue diciendo "se suma", contradiciendo el
>   comportamiento real — ACCIDENTAL, corregir el comentario en modernización.
> - Dos opciones distintas que apuntan al mismo insumo generan **dos movimientos separados**
>   (RN-CAT-21); una opción sin insumo ligado no genera consumo (RN-CAT-22); consumo resultante
>   ≤0 no genera línea (RN-CAT-23).
> - Stock insuficiente = `stock_actual < requerido`, comparación **estricta**, no `<=`
>   (RN-CAT-24). El chequeo de disponibilidad **NO reserva ni bloquea stock**, es solo preventivo
>   (RN-CAT-26) — regla **A-47**: `INTENCIONAL` con testimonio solo de CÓDIGO (no alcanza 2
>   testigos, sigue `PENDIENTE` en sentido estricto). Consecuencia visible en spec 007
>   (comensal). **Confirmado como aceptado en la ronda 2 de entrevista (P27-bis): el negocio
>   prefiere el diseño actual antes que invertir en reservar stock.**
> - Vender una variante **sin que se descuente NADA de inventario** está bloqueado con **409**
>   (RN-CAT-34) — protegido por `test_receta_obligatoria.py`.
> - **A-33** (RN-CAT-35, `[TENSIÓN/DISCREPANCIA]` con la convención de spec 004): un grupo
>   opcional (`min_select=0`, legítimo no elegir nada según spec 004) que es la **única** fuente
>   de consumo bloquea la venta con 409 si nadie elige nada. `PENDIENTE`. Documentar sin
>   especificar; citar la convención de spec 004 como contraste.
>
> **Criterios de aceptación**: citar `test_variant_option_groups.py` ("la grande descuenta 3× del
> sabor elegido y la pequeña 1× — misma opción, cantidades distintas por tamaño") como el test
> más cercano a un golden master de RN-CAT-18/A-02, y `test_receta_obligatoria.py` para RN-CAT-34.
> Ninguno corre en CI hoy (ver spec 013 para el contexto de cobertura de CI).
>
> **Fuera de alcance**: precio y SKU de la variante (spec 002); la validación de forma de la
> selección de opciones (`min_select`/`max_select`, `STRICT_OPTION_SELECTION`, spec 004); el
> caller real que omite pasar `variant` a esta validación (`add_item_to_table`, A-04, spec 009).

---

### → 004-grupos-de-opciones-seleccion-y-tolerancia

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la validación de grupos de
> opciones (sabores/toppings/extras) del sistema POS Heladería, incluida la tolerancia
> deliberada de migración del catálogo histórico. No es una feature nueva: es la especificación
> formal de lo que el sistema YA hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-CAT-27 a RN-CAT-33, RN-CAT-36 a RN-CAT-39,
> `pos-backend/app/api/v1/catalog/line_pricing.py`, `app/core/config.py`:
> - Un grupo normal permite elegir cualquier cantidad entre `min_select` y `max_select`, **ambos
>   inclusive** (RN-CAT-27). Un grupo **obligatorio que además descuenta inventario** exige
>   elegir **EXACTAMENTE el máximo**, no basta el mínimo (RN-CAT-28); no elegido en absoluto
>   exige el número exacto correspondiente (RN-CAT-29).
> - Violaciones de selección en grupos que descuentan inventario son **SIEMPRE bloqueantes**, sin
>   importar `STRICT_OPTION_SELECTION` (RN-CAT-30). `STRICT_OPTION_SELECTION` (default **`False`**,
>   sin override en `.env.example`) tolera violaciones **SOLO** en grupos que no descuentan
>   (RN-CAT-31) — regla **A-05 [PROTEGIDA]**: "el catálogo histórico nunca se validó" —
>   activar el chequeo a ciegas rechazaría combinaciones ya cargadas
>   (`memoria-historica.md` #9, 2026-08-03, commit `03469cad`, Deimer Hernandez). **Confirmado en
>   entrevista (P18): el catálogo nunca se depuró (script `opciones_fuera_de_grupo.py` nunca se
>   corrió)** — el default `False` debe seguir. Especificar tal cual.
> - Con `STRICT_OPTION_SELECTION=False`, elegir una opción de un grupo que la variante **NO
>   ofrece** no se rechaza, sigue sumando `extra_price` y puede seguir generando consumo de un
>   insumo ajeno (RN-CAT-32) — anomalía **A-06**: sigue `PENDIENTE` en clasificación estricta,
>   pero **tratamiento cerrado en ronda 2 (P7-bis): aceptar el riesgo por ahora, sin priorizar
>   corrección ni consulta a datos, mismo criterio que A-05**.
> - La validación de selección (`load_valid_options`) es **opcional para el llamador** —
>   depende de que se le pase `variant` (RN-CAT-33). Este es el mecanismo exacto que permite el
>   bug real documentado en A-04/spec 009: el caller real del mesero no pasa `variant`. Esta spec
>   fija la regla; spec 009 documenta y corrige el caller.
> - Desactivar/eliminar un grupo bloqueado mientras alguna variante lo siga ofreciendo
>   (RN-CAT-36); nombre de opción único **dentro de su grupo**, no globalmente (RN-CAT-37);
>   desvincular el insumo de una opción **resetea forzosamente** `item_quantity` a 0, aunque el
>   request traiga otro valor (RN-CAT-38).
> - **A-32**: dos funciones (`grupos_que_descuentan` al alta vs `group_discounts` al confirmar)
>   definen "¿este grupo descuenta inventario?" con criterios distintos (RN-CAT-39) — una exige
>   solo `item_quantity>0`, la otra exige además `active` e `inventory_item_id`. `PENDIENTE`,
>   requiere arqueología de datos (candidata explícita, no bloquea esta spec). Documentar sin
>   especificar.
>
> **Criterios de aceptación**: sin characterization test dedicado existente hoy (gap prioritario
> a cerrar antes de poder proteger A-05 con un test explícito, dado que es una regla `PROTEGIDA`
> sin caracterización).
>
> **Fuera de alcance**: qué se descuenta y cuánto (spec 003); el caller real que omite la
> validación de esta spec (A-04, spec 009); precio/SKU de la variante (spec 002).

---

### → 005-inventario-y-compras

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del kardex de insumos, las
> compras a proveedor y la conversión de unidades del sistema POS Heladería. No es una feature
> nueva: es la especificación formal de lo que el sistema YA hace, para que sirva de contrato en
> la modernización.
>
> **Alcance** — RN-INV-01 a RN-INV-23, RN-CAT-40/41, `pos-backend/app/api/v1/inventory/stock.py`,
> `service.py`, `app/core/units.py`:
> - `record_movement` es el **único punto de mutación de stock**, con signo determinado por el
>   tipo de movimiento (RN-INV-01); movimiento ≤0 rechazado (RN-INV-02); bloqueo por
>   `SELECT...FOR UPDATE` (RN-INV-06) con **orden canónico** de ids para evitar deadlocks
>   (RN-INV-07).
> - Bloqueo de salida por stock insuficiente, no se permite dejar el stock negativo (RN-INV-04)
>   — anomalía **A-35** (porción `allow_negative`, RN-INV-05): existe una vía para forzarlo, sin
>   llamador visible en `inventory`; `PENDIENTE`, documentar sin especificar.
> - Ajuste manual por delta con signo (RN-INV-08); también bloquea negativo, sin bandera para
>   forzarlo (RN-INV-09). **A-35** (porción RN-INV-10): un ajuste con delta=**0** es rechazado
>   con `ValueError`, pero sin handler dedicado propaga como **500** genérico. ACCIDENTAL,
>   **corregir de inmediato** agregando handler.
> - **A-35** (porción RN-INV-11): el motivo (`reason`) de un ajuste manual **NO es obligatorio**
>   hoy. **Requisito de negocio confirmado en P23: debe ser obligatorio.** El motivo del kardex
>   es texto libre, sin restricción en BD (RN-INV-12).
> - Umbral de "stock bajo" es **`<=`** (no `<`), configurable **por insumo**, no global
>   (RN-INV-14) — anomalía **A-13**: el filtro `low_stock` de `/items` no excluye inactivos por
>   defecto; el endpoint dedicado y el reporte sí (RN-INV-15). **ACCIDENTAL confirmado en P9: un
>   insumo desactivado con stock residual bajo mínimo debe seguir en las alertas.** Unificar
>   criterio en las tres pantallas en modernización.
> - Compra directa da alta de stock **inmediata** (RN-INV-16); orden de compra **NO** da alta al
>   crearse (RN-INV-18); no se puede recibir sobre una orden ya recibida completa (RN-INV-19); la
>   recepción parcial suma stock **incrementalmente** y puede repetirse (RN-INV-21). **A-35**
>   (porción RN-INV-17): toda compra/recepción **sobrescribe** el costo unitario, sin promediar.
>   **INTENCIONAL confirmado en P23: "último costo de compra" es el comportamiento deseado, no
>   promedio ponderado.** Especificar tal cual.
> - **A-31** (RN-CAT-40/41, reubicadas aquí — ver §3 de la propuesta de partición): `convert()`
>   en `core/units.py` referencia columnas (`dimension`, `factor_to_base`) inexistentes en
>   `UnitMeasure`; código muerto de una migración "Fase 1" nunca completada. ACCIDENTAL.
>   **Alcance confirmado en P19 + ronda 3 (simulada, P31)**: sí hace falta completarla, con el
>   caso real de **litros→onzas** para el producto "granizado" (materia prima comprada en
>   litros, vendida en vasos de onzas) — **pero solo para conversión dentro de la misma
>   dimensión (volumen)**; conversión entre dimensiones distintas (masa↔volumen) queda **fuera
>   de alcance**, sin caso de uso real confirmado.
>
> **Criterios de aceptación**: sin characterization test dedicado a `inventory`/`stock.py` entre
> los 12 scripts existentes — gap de caracterización notable para un módulo de criticidad alta.
>
> **Fuera de alcance**: qué insumo y cuánto consume cada venta (spec 003); el precio de venta de
> la variante (spec 002).

---

### → 006-caja-y-arqueo

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de los turnos de caja, los
> movimientos manuales de efectivo y el arqueo del sistema POS Heladería. No es una feature
> nueva: es la especificación formal de lo que el sistema YA hace, para que sirva de contrato en
> la modernización.
>
> **Alcance** — RN-CASH-01 a RN-CASH-17, `pos-backend/app/api/v1/cash/service.py`, `router.py`:
> - **Un solo turno de caja abierto por caja registradora a la vez** (RN-CASH-01). Las ventas del
>   arqueo se derivan de **pagos reales**, no de un registro propio en `cash_movements`
>   (RN-CASH-02). Solo `type='cash'` afecta el efectivo esperado del cajón; tarjeta/transferencia
>   se reportan pero no suman/restan (RN-CASH-03). Efectivo esperado se calcula **neto del
>   cambio entregado** (RN-CASH-04).
> - La diferencia del arqueo solo se calcula si hay conteo físico registrado (RN-CASH-06). No se
>   puede cerrar un turno ya cerrado (RN-CASH-11); movimientos manuales solo en turno abierto
>   (RN-CASH-12); solo **tres tipos** de movimiento manual, cada uno con signo fijo, monto
>   siempre positivo (RN-CASH-14).
> - **A-20** (tres reglas): (1, RN-CASH-09) `close_shift` puede aceptar `counted_amount` sin
>   denominaciones o ninguno de los dos, dejando el conteo en `None` — **mitigado: la pantalla
>   siempre exige el conteo antes de cerrar (confirmado en P14)**. (2, RN-CASH-13) el arqueo
>   parcial (`partial-count`, sin cerrar turno) no exigía observación (`close_note`) aunque la
>   diferencia sea distinta de cero, a diferencia del cierre real que sí la exige — **requisito
>   de negocio confirmado en la ronda 2 (P14-bis): debe exigirla igual que el cierre real**. (3,
>   RN-CASH-17) el histórico de turnos siempre **recalcula** `expected`/`difference` en el
>   momento de la consulta con datos actuales, no es una foto fija — si se anula una venta
>   después del cierre, el histórico cambia retroactivamente. **Requisito nuevo, no solicitado
>   directamente, surgido en P14 (`entrevista-negocio.md` §4): el cierre original nunca debe
>   modificarse/sobrescribirse; cualquier efecto de una anulación posterior debe verse en una
>   vista de ajustes separada** — más estricto que "congelar" o "recalcular".
> - **A-17** (porción caja, `cash/router.py:90-124,190-203`): `close_shift` y `add_movement` sin
>   `with_for_update()` — doble clic en cierre puede duplicar denominaciones (R7); un movimiento
>   manual justo al filo del cierre puede quedar fuera de la diferencia reportada (R22).
>   ACCIDENTAL. Corregir con lock consistente con el resto del código.
> - **A-40** (RN-CASH-15): el alias `cash_sales`, marcado **DEPRECADO**, es idéntico a
>   `ventas_efectivo` (ya neto del cambio entregado), mantenido solo por compatibilidad con el
>   frontend (`memoria-historica.md` #17, 2026-07-18, commit `927a4606`, Deimer Hernandez).
>   `PENDIENTE`: confirmar si el frontend ya migró antes de retirarlo.
>
> **Criterios de aceptación**: sin characterization test dedicado entre los 12 scripts
> existentes. RN-CASH-17 (snapshot inmutable) es un **requisito nuevo confirmado**, no solo
> comportamiento actual — los tests de esta spec deberán escribirse contra el comportamiento
> deseado recién confirmado, caso especial a señalar al ejecutar `/speckit-specify`.
>
> **Fuera de alcance**: cómo se generan las ventas que el arqueo agrega (spec 011); el cierre de
> cuenta de mesa en sí (spec 010).

---

### → 007-menu-publico-y-carrito-del-comensal-qr

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del menú público y el
> carrito del comensal en el flujo de pedido por QR del sistema POS Heladería. No es una feature
> nueva: es la especificación formal de lo que el sistema YA hace, para que sirva de contrato en
> la modernización.
>
> **Alcance** — RN-MENU-01 a RN-MENU-09, RN-CART-01 a RN-CART-27,
> `pos-backend/app/api/v1/cart/service.py`, `app/api/v1/menu/router.py`, `app/core/qr_context.py`:
> - El precio mostrado en el menú **ya incluye el descuento** de promociones activas, calculado
>   sobre cantidad 1, redondeado "half up" (RN-MENU-01). La disponibilidad de una opción se
>   evalúa contra el **peor caso** de consumo entre todas las presentaciones que la usan
>   (RN-MENU-03).
> - Escanear el QR de una mesa **ocupada une** a la sesión activa, **no abre una segunda**
>   (RN-CART-01); nombres de comensal desambiguados automáticamente por mesa (RN-CART-02); cada
>   comensal tiene **como máximo un carrito abierto** a la vez (RN-CART-05); no se puede enviar
>   un carrito vacío (RN-CART-06).
> - **TTL deslizante del comensal: 240 minutos (4 horas) de inactividad** (RN-CART-17); el
>   refresco de la ventana **solo ocurre si faltan ≤230 minutos** para expirar (RN-CART-19); el
>   token JWT de sesión en sí expira a las **1440 minutos (24 horas)**, **NO** a las 4 horas del
>   TTL deslizante (RN-CART-20). El QR físico de la mesa **no expira nunca** (sin `exp`,
>   RN-CART-24). **A-36** (porción RN-CART-18): en el instante exacto de expiración
>   (`expires_at == now`), la sesión ya se considera expirada — resolución de microsegundos,
>   casi inobservable. `PENDIENTE`, sin impacto económico. Documentar sin especificar.
> - Enviar el carrito **no descuenta inventario** — solo la confirmación de cocina lo hace
>   (RN-CART-09, spec 008/009). Cancelación propia del comensal: **solo antes de que cocina
>   empiece** (RN-CART-13).
> - **A-24 [PROTEGIDA]** (RN-CART-26): tenant y mesa siempre vienen del **token firmado**, nunca
>   de un id que mande el cliente — dos endpoints legacy inseguros retirados a propósito
>   (`memoria-historica.md` #2, 2026-07-28, commit `5c1db9ed`, Deimer Hernandez). Especificar tal
>   cual.
> - **A-21** (porción comensal, `diner-token.store.ts:15-18`): el token de sesión del comensal
>   vive en `localStorage`; el diseño documentado (cookie `httpOnly`, "el documento de contrato")
>   nunca se implementó. **INTENCIONAL confirmado en P15: localStorage es el diseño definitivo,
>   no prioridad la cookie httpOnly.** Actualizar Angular (6 vulnerabilidades XSS "high", R3)
>   sigue siendo corrección inmediata.
> - **A-08** (`cart/service.py:52-53,205-206`, `menu/router.py:82-83`): ambos evalúan vigencia de
>   promociones con `datetime.now(timezone.utc).replace(tzinfo=None)` — un naive que en
>   realidad es UTC, mal interpretado por `local_now()` como ya-local. Con `TENANT_TIMEZONE=
>   America/Bogota` (UTC-5) reproduce, solo aquí, el bug que spec 012/A-07 corrigió en el resto
>   del sistema. ACCIDENTAL — el monto cobrado no se afecta, solo la vista previa. Corregir
>   aplicando el mismo patrón ya usado en checkout/table_sessions/sales (spec 012 posee la
>   convención correcta, RN-PROMO-01).
> - **A-28** (`app/core/config.py:36-44`, `qr_context.py:121`): el invariante
>   `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` no se valida en el arranque —
>   si se viola, el barrido de sesiones (spec 010) puede cerrar mesas activas. ACCIDENTAL.
>   Corregir con un validador de arranque (Pydantic `model_validator`).
> - **A-47** (regla propia RN-CAT-26 vive en spec 003, consecuencia visible aquí): el chequeo de
>   disponibilidad del carrito es best-effort, sin reserva; en hora pico el comensal puede ver un
>   ítem disponible y que su confirmación falle. **INTENCIONAL confirmado en ronda 2 (P27-bis):
>   diseño preferido, sin invertir en reservar stock.**
>
> **Criterios de aceptación**: citar `test_qr_token.py` (contrato de tokens QR/sesión) para
> RN-CART-24/25/26, y `test_table_sessions.py` (a pesar del nombre, prueba
> `cart.service.open_session`: unión a sesión existente, desambiguación, reingreso con token de
> sesión cerrada → **401**) para RN-CART-01/02/21.
>
> **Fuera de alcance**: qué pasa una vez que el pedido se envía (specs 008/009); el cobro (spec
> 010); el motor de cálculo de promociones que consume esta spec (spec 012).

---

### → 008-confirmacion-cobro-legado-y-cancelacion-de-pedidos

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la confirmación de
> pedidos QR (el punto real de descuento de inventario), el ciclo de cobro legado por pedido
> individual, y la cancelación asimétrica del sistema POS Heladería. No es una feature nueva: es
> la especificación formal de lo que el sistema YA hace, para que sirva de contrato en la
> modernización.
>
> **Alcance** — RN-ORD-01 a RN-ORD-35, RN-ORD-65, `pos-backend/app/api/v1/orders/checkout.py`,
> `consumption.py`:
> - Bloqueo previo obligatorio para cobrar por el ciclo legado (RN-ORD-01, `block`→`pay`,
>   confirmado sin caller en la UI — `grep` cruzado en ambos repos, 0 resultados). `compute_bill`
>   de este módulo excluye solo órdenes canceladas, **no las ya pagadas** (RN-ORD-03,
>   `[DUDOSA]`) — es el **camino B** de la anomalía **A-01** (ver §3 de la propuesta de
>   partición): sin descuentos, código muerto pero peligroso si se reactiva. La convención
>   correcta (`table_sessions.compute_bill`) vive en spec **010**; A-01 se documenta aquí como el
>   camino muerto. Documentar como candidato a retirar o unificar con spec 010.
> - Confirmar un pedido (`POST /orders/{id}/confirm`) es **el único punto de descuento real de
>   inventario** para el camino QR (RN-ORD-10), válido solo desde `recibida` (RN-ORD-13); fallo
>   de stock **revierte toda la transacción** (RN-ORD-14); confirmar sin ítems consumibles está
>   **prohibido** (RN-ORD-12).
> - Reversa asimétrica de cancelación, no una reversa simétrica (RN-ORD-20 a RN-ORD-24): orden
>   `recibida` (nunca descontada) → **cero movimientos**; ítem `pendiente` → **se revierte**
>   (movimiento `'in'`); ítem `en_preparación`/`listo` → **NO vuelve al stock**, se registra como
>   **pérdida en auditoría**, no en kardex (evita doble descuento). Cancelación exige y registra
>   un motivo (RN-ORD-25).
> - **A-25 [PROTEGIDA]** (RN-ORD-65, reasignada aquí por contenido — ver §3): se retiró a
>   propósito el `PATCH` genérico de status de pedido — permitía saltarse `confirm_order` y
>   sobrestimar inventario sin aviso (`memoria-historica.md` #3, 2026-07-28, commit `5c1db9ed`,
>   Deimer Hernandez). Especificar tal cual: "no existe transición libre de status" como
>   invariante de diseño explícito, no solo ausencia de funcionalidad.
> - **A-11** (regla compartida, especificada en spec 011): la prohibición total de descuento
>   manual del cajero aplica también a este camino legado — confirmado en ronda 3 (simulada) que
>   aplica a los tres caminos de cobro por igual.
> - **A-29** (porción RN-ORD-08, `[DUDOSA]`): con 2+ combos distintos en una venta, `promotion_id`
>   no registra ninguno (descuento monetario correcto, se pierde la trazabilidad de reportes por
>   promoción). `PENDIENTE` — confirmado en P21 sin impacto práctico (no usan ese reporte).
>   Documentar sin especificar.
> - **A-38** (porción RN-ORD-31, RN-ORD-32, RN-ORD-34): el cierre de mesa en cascada delega en el
>   llamador la validación de pendientes (`PENDIENTE`); la descripción de línea de venta puede
>   quedar incompleta si el producto/variante fue borrado (`PENDIENTE`); el docstring de "único
>   punto de descuento" es preciso solo como "único punto de salida" — la reversa también
>   escribe en el kardex (precisión documental, no anomalía de comportamiento). Documentar sin
>   especificar.
> - **A-42** (porción auditoría, RN-ORD-24): la pestaña de Ajustes→Auditoría se retiró, pero
>   `record_audit()` sigue escribiendo las pérdidas por cancelación de esta spec sin interfaz que
>   lo muestre. **Confirmado en P25: nadie consulta `audit_logs` hoy.** `PENDIENTE`: decisión de
>   negocio sobre mantener el registro sin lector.
>
> **Criterios de aceptación**: citar `test_cancel_inventory.py` (`recibida`→cero movimientos;
> `pendiente`→entrada real; `en_preparación`+→sin movimiento/pérdida) como base directa de
> RN-ORD-20 a RN-ORD-24.
>
> **Fuera de alcance**: la operación de cocina y mesas físicas (spec 009); el cierre real de la
> sesión de mesa (spec 010); el constructor de venta compartido (spec 011).

---

### → 009-cocina-consolidacion-y-mesas-fisicas

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la preparación en cocina,
> la toma de pedidos por el mesero desde la terminal, la consolidación de carritos, y la
> administración de mesas físicas del sistema POS Heladería. No es una feature nueva: es la
> especificación formal de lo que el sistema YA hace, para que sirva de contrato en la
> modernización.
>
> **Alcance** — RN-ORD-36 a RN-ORD-64, RN-ORD-66, `pos-backend/app/api/v1/orders/kitchen.py`,
> `consolidation.py`, `tables_advanced.py`:
> - `estado_cocina` transiciona `pendiente→en_preparación→listo`, o `anulado`; **el salto directo
>   `pendiente→listo` está permitido a propósito** ("quien toma el pedido es quien lo prepara",
>   RN-ORD-36). Anular un ítem `pendiente` revierte inventario; uno en curso no se revierte (ya
>   se consumió).
> - **A-04**: `add_item_to_table` (`orders/consolidation.py:199`, **el único camino con botón
>   real en la terminal del mesero en producción**) **no pasa `variant=variant`** a
>   `load_valid_options` (regla RN-CAT-33, spec 004), así que no valida `min_select`/`max_select`
>   en el camino real de uso diario. Reconstruido con `git log`/`git show`: regresión de fusión
>   entre la rama de corrección `03469ca` (2026-08-03) y la rama de combos `ee94f30`
>   (2026-08-04, autor LeonardoGomezz) que partió de una copia previa a la corrección. **BUG
>   HISTÓRICO CON DEPENDIENTES — testimonio de negocio confirma merma real hace ~15 días en
>   sabores/toppings elegibles (P4, `entrevista-negocio.md`), coincidiendo exactamente con el
>   patrón predicho.** Corrección: pasar `variant=variant` en `consolidation.py:199` (una
>   línea, ya se hizo una vez en `03469ca` y se perdió en el merge). No retroactivo — no hay
>   forma de recalcular inventario ya consumido incorrectamente.
> - **A-16**: `transition_kitchen` y `void_item` **no validan** el `status` del pedido padre —
>   funcionan igual aunque la orden esté `pagada`, `cancelada` o `bloqueada` (a diferencia de
>   `mark_order_ready`, que sí valida parcialmente, RN-ORD-38/39). ACCIDENTAL. La porción
>   `PENDIENTE` (RN-ORD-37: ¿debe `bloqueada` impedir avance de cocina?) **se cerró en la ronda 2
>   (P16-bis): mitigado por el ritmo de trabajo actual — "prácticamente imposible" que
>   coincidan**. Corregir en modernización replicando la validación de `mark_order_ready`, sin
>   urgencia operativa.
> - Una mesa puede tener **varias órdenes abiertas simultáneas** — no hay índice único de "una
>   orden abierta por mesa" (RN-ORD-50). Consolidar exige carritos abiertos con ítems
>   (RN-ORD-46); el precio en consolidación **se copia** del carrito, no se recalcula
>   (RN-ORD-47).
> - **A-26**: `move_order` exige que la mesa destino esté **completamente sin órdenes activas**,
>   más estricto que el modelo general (RN-ORD-58, `[DUDOSA]`) — la porción `PENDIENTE` **se
>   cerró en ronda 2 (P20-bis): no usan la función de mover pedidos — riesgo latente, no
>   activo**. Manejador de `IntegrityError` huérfano de una constraint ya removida (RN-ORD-60,
>   ACCIDENTAL); `merge_orders` conserva el primer `merged_group_id` con un `SELECT` **sin
>   `ORDER BY`** (no determinista) cuando hay grupos preexistentes en colisión (RN-ORD-63,
>   ACCIDENTAL). Corregir ambos en modernización (retirar el manejador o documentarlo; definir
>   regla explícita de qué grupo sobrevive, p. ej. menor `created_at`).
> - **A-48**: el KDS (pantalla de cocina separada) se deprecó tres semanas después de describirse
>   como plan vigente en el CHANGELOG de v1.0.0 (`memoria-historica.md` #12, 2026-08-07, commit
>   `d52f024c` + migración `c5d6e7f8a9b0`, que **eliminó el estado `entregado`**). **INTENCIONAL
>   confirmado en P28: decisión operativa deliberada — un solo dispositivo funciona mejor.**
>   Especificar tal cual.
> - No puede haber dos mesas con el mismo número (RN-ORD-66).
>
> **Criterios de aceptación**: ningún script de los 12 existentes cubre `kitchen.py`/
> `consolidation.py`/`tables_advanced.py` de forma dedicada — **gap de caracterización
> prioritario**, dado que aquí vive A-04, la anomalía con evidencia más fuerte de todo el
> reconocimiento (`git log` + testimonio de negocio de merma real).
>
> **Fuera de alcance**: la confirmación que descuenta inventario y la cancelación (spec 008); el
> cierre de cuenta de la mesa (spec 010); la validación de opciones que el caller de esta spec
> omite (regla propia en spec 004).

---

### → 010-sesion-de-mesa-reparto-y-cierre-de-cuenta

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la sesión de mesa, el
> reparto de cuenta entre comensales, el cierre unificado/dividido, y el barrido automático de
> sesiones abandonadas del sistema POS Heladería. No es una feature nueva: es la especificación
> formal de lo que el sistema YA hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-MESA-01 a RN-MESA-27, RN-SCHED-01 a RN-SCHED-09,
> `pos-backend/app/api/v1/table_sessions/service.py`, `app/core/scheduler.py`:
> - `close_session` usa **lock optimista `FOR UPDATE`** para impedir cobro doble de dos
>   `POST /close` concurrentes (RN-MESA-01). Es **la implementación vigente y correcta** de
>   "cuánto se le debe cobrar a esta mesa" — excluye pedidos anulados/pagados y aplica
>   promociones (regla que las anomalías **A-01** de spec 008, camino muerto, y spec 009, camino
>   `group_bill` activo en mesas fusionadas, citan como divergencia — ver §3 de la propuesta de
>   partición). No se puede cerrar sin al menos un pedido cobrable (RN-MESA-03), ni con pedidos
>   sin confirmar o ítems en curso (RN-MESA-04).
> - El reparto de cuenta es **por ítem/unidad**, nunca por división porcentual automática
>   (RN-MESA-05). `billing_mode=unified` exige pagos en la **raíz**; `billing_mode=split` exige
>   **bloques**, con montos de raíz **prohibidos** — se rechazan si vienen poblados en `split`
>   (RN-MESA-10, RN-MESA-11).
> - **A-15 [PROTEGIDA]**: cuatro huecos de seguridad cerrados **antes** de dar a los cajeros la
>   capacidad de armar bloques de pago manualmente (`memoria-historica.md` #11, 2026-08-04,
>   commit `42b5dec3`, Deimer Hernandez): (1) comensales repetidos en `splits` causaban **doble
>   cobro y doble factura**; (2) montos en la raíz con `billing_mode='split'` se ignoraban en
>   silencio (el cajero **perdía la propina**); (3) importes negativos no se validaban; (4) el
>   bloque sin comensal salía **sin nombre** en la factura. **Confirmado en P12: sin ventana de
>   exposición — la capacidad se usó después del blindaje.** Especificar tal cual; los cuatro
>   chequeos deben convertirse en casos de test explícitos.
> - El vuelto solo puede salir del pago en efectivo, nunca de un pago electrónico "de más"
>   (RN-MESA-21). Cerrar una sesión es **una única transacción todo-o-nada** (RN-MESA-22); no
>   escribe movimientos de caja manuales, para no contar el dinero dos veces (RN-MESA-23).
> - **A-11** (regla compartida, spec 011): la prohibición total de descuento manual del cajero
>   aplica también al cierre unificado y dividido de esta spec — confirmado en ronda 3 (simulada)
>   que aplica a los tres caminos por igual.
> - **A-17** (porción mesa, `table_sessions/service.py:38-55,335,370,403`):
>   `add_participant`/`remove_participant`/`set_assignments` no toman lock (a diferencia de
>   `close_session`, que sí, RN-MESA-02 `[DUDOSA]`). La pregunta sobre serialización externa
>   **sigue sin decisión concluyente** — no forma parte de las preguntas cerradas en ronda 1-2 ni
>   en la ronda 3 (simulada); debe incluirse en la próxima ronda real. Corregir con
>   `with_for_update()` consistente con `close_session`.
> - **A-29** (porción RN-MESA-15): mismo mecanismo que en spec 008/011: con 2+ combos distintos
>   se pierde `promotion_id`. `PENDIENTE`, sin impacto práctico confirmado (P21).
> - **A-38** (porción RN-MESA-13, RN-MESA-24): una mesa de un solo comensal puede cerrarse en
>   `split` sin restricción de mínimo, equivalente en la práctica a `unified` disfrazado
>   (`PENDIENTE`); no se puede quitar un comensal con productos ya asignados aunque estén
>   anulados o su pedido ya no sea cobrable (`PENDIENTE`). Documentar sin especificar.
> - **RN-SCHED**: una mesa se cierra por abandono a los **30 minutos exactos** de inactividad de
>   todos sus comensales (RN-SCHED-01); una sesión con consumo se cierra por **tope duro a las 6
>   horas** de abierta, sin importar actividad (RN-SCHED-02); una sesión vencida con pedidos
>   facturables pendientes de cobro **NO** se cierra — solo expulsa a los comensales
>   (RN-SCHED-03). El barrido corre en todos los tenants; un tenant con error no detiene el
>   barrido de los demás (RN-SCHED-06); lock distribuido en Redis con TTL = mitad del intervalo
>   (RN-SCHED-07). **A-28** (regla propia en spec 007, consecuencia aquí): si el invariante
>   `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` se viola, este barrido puede
>   cerrar mesas activas.
>
> **Criterios de aceptación**: citar `test_split_blindaje.py` (los cuatro blindajes de A-15) —
> **es la base más sólida de todo el reconocimiento para proteger una regla `PROTEGIDA`** — y
> `test_table_release.py` (condiciones de liberación de mesa: nadie activo **Y** nada que cobrar,
> ambas condiciones necesarias).
>
> **Fuera de alcance**: la confirmación de pedidos y su reversa (spec 008); la cocina y mesas
> físicas (spec 009); el constructor de venta compartido y la factura (spec 011).

---

### → 011-ventas-de-mostrador-y-facturacion

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la venta de mostrador, el
> constructor de venta compartido por los cuatro caminos de cobro, y la emisión de factura
> interna del sistema POS Heladería. No es una feature nueva: es la especificación formal de lo
> que el sistema YA hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-VENTA-01 a RN-VENTA-17, RN-FACT-01 a RN-FACT-07,
> `pos-backend/app/api/v1/sales/builder.py`, `app/api/v1/invoices/service.py`:
> - Total de venta = `subtotal − descuento + impuesto + propina`, **sin redondeo intermedio**
>   (RN-VENTA-02); nunca puede ser negativo (RN-VENTA-03); el pago debe **cubrir el total, sin
>   excepción** (RN-VENTA-04); el vuelto solo sale del efectivo (RN-VENTA-05); solo se cobra
>   contra turno de caja **abierto** (RN-VENTA-07).
> - **A-11 [regla propia — dueña de la spec]**: el `discount` de cualquier venta
>   (`sales/schemas.py:63`) no tiene `le=` en el schema; el único freno es que el total no quede
>   negativo (`sales/builder.py:132-136`); `create_sale` solo exige `Depends(get_current_user)`
>   (contrastado con `create_payment_method` en el mismo router, que sí exige
>   `require_tenant_admin`). ACCIDENTAL (asimetría verificable). **Tratamiento redefinido y
>   endurecido en P5 (`entrevista-negocio.md`): el cajero no debe poder aplicar descuento manual
>   en absoluto** — más estricto que un tope numérico, para prevenir el error accidental de
>   "regalar" una venta. **Confirmado en ronda 3 (simulada, P30): aplica a los tres caminos de
>   cobro por igual** — mostrador aquí; mesa unificada/dividida en spec 010; legado en spec 008.
>   No retroactivo: no se pueden recalcular ventas ya cobradas con descuento excesivo.
> - **A-12** (`sales/consumption.py:46-51`, regla RN-VENTA-11): la venta de mostrador cobra
>   variantes **sin receta ni opción configurada, sin bloquear ni descontar nada** — a diferencia
>   del camino QR, que bloquea con **409** (RN-CAT-34, spec 003). ACCIDENTAL, el propio
>   comentario del código lo describe como "un agujero" que crece con cada nueva variante mal
>   cargada. **Alcance estimado en P8: 6-20 productos activos sin receta** (cifra aproximada, no
>   verificada). Corregir replicando el bloqueo de spec 003. No retroactivo.
> - **A-14** (`invoices/schemas.py:40-43` Python vs `sales/service.py:142-172` SQL): el número de
>   factura se formatea distinto — Python (`f"{n:06d}"`) nunca trunca, SQL (`lpad(...,6,'0')`) sí
>   trunca a **6 caracteres**; divergen matemáticamente desde el consecutivo **1.000.000** (caso
>   `1234567` → `"FAC-123456"` no encontrado, prueba determinista). BUG A SECAS. **Prioridad
>   recuperada tras la ronda 3 (simulada): la verificación técnica del 2026-08-16 confirmó que
>   NO existe ningún mecanismo de reinicio anual en el código (`app/core/scheduler.py` solo
>   registra `sweep_orphan_table_sessions` y `expire_promotions`) — pendiente de ratificación
>   real con la gestoría antes de darlo por cerrado en sentido estricto.** Adoptar ampliar el
>   padding a **7+ dígitos** en ambos lados como corrección segura.
> - **A-14a [PROTEGIDA]** (`sales/builder.py:174-179`, RN-FACT-01): toda venta pagada emite
>   automáticamente su factura **en la misma transacción de base de datos**
>   (`memoria-historica.md` #6, 2026-07-29, commit `27711065`, Deimer Hernandez: "hay cuatro
>   formas de cobrar... emitir por separado en cada una garantizaba que alguna se quedara fuera —
>   que es exactamente lo que pasaba" — "20 ventas reales, cero facturas"). **Confirmado en P11:
>   el `pay_order` legado (spec 008) sigue sin uso real — lo que el negocio usa es la cuenta
>   dividida (spec 010).** Especificar tal cual.
> - **A-29** (porción RN-VENTA-14): mismo mecanismo que en spec 008/010: con 2+ combos, se pierde
>   `promotion_id`. `PENDIENTE`, sin impacto práctico (P21).
> - **A-41** (`pos-terminal.store.ts:497`, `const tax = 0`): impuestos fijos en **$0**,
>   deprecados a propósito (`memoria-historica.md` #10, 2026-08-03, commit `8166ea9e`, Leonardo
>   Gomez, mensaje "fix(tables): deprecate editable tax field"). **INTENCIONAL confirmado en P2:
>   correcto, no discriminan impuesto en el ticket.** Especificar tal cual — con nota de que es
>   la entrada de mayor riesgo fiscal/legal residual del registro pese a su bajo impacto técnico.
> - **A-43** (`invoices/service.py:30-69`): `issue_for_sale` se documenta idempotente, pero el
>   patrón `SELECT`+`INSERT` no tiene lock ni captura de `IntegrityError`; solo un llamador hoy
>   (dentro de la misma transacción) limita la exposición real. ACCIDENTAL. Corregir solo si se
>   introduce un segundo llamador fuera de esa transacción.
> - **A-49**: verificación técnica (2026-08-16, `entrevista-negocio.md` §8): no existe ningún
>   mecanismo de reinicio del consecutivo. **Pendiente de ratificación real** — el testimonio que
>   descarta la premisa de negocio de "se reinicia cada año" (P10) es simulado.
>
> **Criterios de aceptación**: citar `test_facturacion.py` ("una factura por venta, no por
> pedido — un `split` de dos comensales → dos facturas"; "consecutivo sin huecos ni repeticiones,
> por prefijo"; "atomicidad: si el cobro falla, ni venta ni factura, y el contador no avanza")
> como base directa de RN-FACT-01/02/03 (A-14a, PROTEGIDA).
>
> **Fuera de alcance**: el cierre de cuenta de mesa que llama a esta spec (spec 010); la
> confirmación/cancelación de pedidos (spec 008); el arqueo de caja que agrega estas ventas (spec
> 006).

---

### → 012-motor-de-evaluacion-de-promociones-y-combos

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del motor de cálculo de
> descuentos y combos del sistema POS Heladería — vigencia, mejor promoción por línea, y
> expansión de combo. No es una feature nueva: es la especificación formal de lo que el sistema
> YA hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-PROMO-01 a RN-PROMO-45, `pos-backend/app/api/v1/promotions/service.py`:
> - **A-07 [PROTEGIDA — dueña de la spec]**: motor reescrito con tres cambios estructurales
>   (`memoria-historica.md` #15, 2026-08-07, commit `2e94a3ad`, autor genérico `refactor
>   <dev@local>`, no identificable): (1) la vigencia se evalúa en **hora local del tenant**
>   (antes UTC — "un 20% los martes empezaba el lunes a las **19:00** locales, que es justo
>   cuando una heladería vende", RN-PROMO-01); (2) `evaluate_detailed` devuelve **desglose por
>   línea** en vez de un escalar; (3) ante promociones empatadas, gana la de mayor **`priority`
>   explícita**, no siempre el descuento mayor (RN-PROMO-13, RN-PROMO-14: desempate final por
>   antigüedad, `created_at`). Especificar tal cual. Nota de gobernanza: el autor no es
>   identificable por nombre en el historial.
> - Ventana horaria con **cruce de medianoche**, límites **inclusivos** (RN-PROMO-02). Filtro por
>   día de la semana en CSV, **0=lunes** (RN-PROMO-04). Target de producto **gana sobre** target
>   de categoría (RN-PROMO-06). Descuento percent = porcentaje exacto **sin redondeo
>   intermedio** (RN-PROMO-08); fixed **topado al total de línea** (nunca negativo, RN-PROMO-09).
>   `qty_price` descuenta **solo paquetes completos**; el remanente va a precio normal
>   (RN-PROMO-10). Descuento porcentual **máximo 100%** deja la línea en **$0 exacto**
>   (RN-PROMO-20). Redondeo del descuento total ocurre **una sola vez, al final**, con
>   **ROUND_HALF_UP a 2 decimales** (RN-PROMO-21). El motor automático **excluye siempre** el
>   tipo combo (RN-PROMO-16) y `buy_x_get_y` (no implementado en el cálculo, RN-PROMO-45).
> - Combo agrupa líneas por `combo_id` y descuenta **solo bundles completos** (RN-PROMO-24); usa
>   el precio **MÍNIMO** cuando la misma variante aparece con precios distintos en el carrito
>   (RN-PROMO-25); `expand_combo` exige que **TODAS** las variantes componentes estén activas
>   (RN-PROMO-28).
> - **A-08** (regla propia de esta spec, manifestación en spec 007): la convención de hora local
>   (RN-PROMO-01) no se aplicó en `cart/service.py` ni `menu/router.py` — ver spec 007.
> - **A-09** (`promotion-pricing.util.ts`, "port" documentado línea por línea del motor Python):
>   el POS de staff evalúa vigencia con el **reloj del dispositivo**, sin conversión a
>   `TENANT_TIMEZONE` (el cliente no tiene ese dato). Diferencia cuantitativa documentada:
>   **$8.000 mostrado vs $6.400 cobrado** (diferencia de $1.600/unidad). **Mitigado
>   operativamente (P6: relojes de terminal verificados y fijados a `America/Bogota`)** — riesgo
>   de código sin corregir, sin incidente activo.
> - **A-10** (`promotions/service.py:268` vs `promotion-pricing.util.ts:118-121,160`): el
>   desempate del frontend usa "la primera del array" en vez de `created_at` (más antigua) como
>   el backend. ACCIDENTAL, sin efecto visible hoy (ninguna pantalla expone el nombre de la
>   promoción ganadora en un empate).
> - **A-37** (porción evaluación, RN-PROMO-11, 15, 19, 26, 27): `qty_price`/combo **nunca generan
>   descuento negativo** aunque el paquete configurado sea más caro que lo normal — oculta en
>   silencio una mala configuración; un descuento calculado en **$0** descalifica la promoción
>   como candidata, sin distinguir "deshabilitada a propósito" de "error de captura"; la cantidad
>   se **trunca a entero**, no se redondea; un combo que deja de ser vigente entre carrito y
>   cobro **no avisa al cajero**, simplemente no descuenta. Las cinco `PENDIENTE`, sin impacto
>   económico demostrado. Documentar sin especificar.
> - **A-46** (`core/config.py:17-21`, `promotions/service.py:51-54`, R17): `_tz()` lee un
>   **único** `TENANT_TIMEZONE` global de la instancia, no por tenant (limitación reconocida en
>   el propio comentario del código). ACCIDENTAL, sin impacto hoy. **Cerrado sin urgencia en P26:
>   sin planes de expansión a otra zona horaria.**
> - **A-36** (porción RN-PROMO-03, 33, 51): asimetría de precisión — `starts_at` es datetime
>   exacto, `ends_at` es solo fecha; el solapamiento de horario solo se detecta con precisión si
>   **ambas** promociones definen `start_time`; la ventana con cruce de medianoche no está
>   cubierta por test en su segundo límite exacto. `PENDIENTE`, sin impacto económico.
>   Documentar sin especificar.
>
> **Criterios de aceptación**: citar `test_promotions_rules.py` — **el único de los 12 scripts
> que corre en CI** (`.github/workflows/deploy.yml:14-22`), cubriendo explícitamente vigencia en
> hora local (no UTC) y ventana con cruce de medianoche. Es el candidato más maduro de todo el
> reconocimiento a convertirse en golden master formal.
>
> **Fuera de alcance**: cómo se administra una promoción (creación, edición, máquina de estados —
> spec 013); el consumo de esta spec en el menú/carrito (spec 007) y en cada camino de cobro
> (specs 008, 010, 011).

---

### → 013-administracion-de-promociones-vigencia-y-estados

> Spec de ingeniería inversa: documenta el comportamiento EXISTENTE de la administración de
> promociones y combos (creación, edición, validación de forma, máquina de estados) del sistema
> POS Heladería. No es una feature nueva: es la especificación formal de lo que el sistema YA
> hace, para que sirva de contrato en la modernización.
>
> **Alcance** — RN-PROMO-46 a RN-PROMO-78, RN-SCHED-10/11,
> `pos-backend/app/api/v1/promotions/router.py`, `schemas.py`, `app/core/scheduler.py`:
> - Solo el estado **`active`** habilita el descuento automático (RN-PROMO-46). Máquina de
>   estados con **transiciones fijas** (RN-PROMO-53); el cambio de forma (`type`, `targets`,
>   `combo_items`) solo se permite en **`draft`** (RN-PROMO-40, RN-PROMO-56); duplicar
>   **siempre** crea la copia en `draft`, sin importar el estado de origen (RN-PROMO-42).
> - Descuento porcentual acotado a **≤100** en **triple capa** (schema + service + BD,
>   RN-PROMO-62); `qty_price` exige `min_qty≥2` (de la promoción, triple capa igual,
>   RN-PROMO-63); `priority` acotado entre **0 y 1000 inclusive** (RN-PROMO-73). `TargetIn`
>   requiere **exactamente uno**: `product_id` **XOR** `category_id` (RN-PROMO-71). Un combo
>   exige **al menos 2** productos distintos, sin duplicados, y **no puede usar** `targets`
>   (RN-PROMO-72). El nombre de la promoción debe ser **único** en creación, actualización y
>   duplicado (RN-PROMO-78).
> - El solapamiento entre promociones es **solo advertencia**, nunca bloquea create/update
>   (RN-PROMO-30, RN-PROMO-60).
> - **A-30** (`promotions/schemas.py:187`, `router.py:72-74`, `service.py:572-575,533-540`): dos
>   vectores de `IntegrityError` no controlado en `PATCH`: (1) `{"name": null}` pasa la
>   validación de Pydantic (`min_length` no aplica a `None`) y el chequeo de unicidad del router
>   (solo valida "si `name is not None`"), llegando a `commit()` con una columna **NOT NULL**
>   (RN-PROMO-75, ACCIDENTAL — el propio código ya corrigió un bug equivalente para la unicidad
>   del nombre, pero no cubrió este caso). (2) `targets` repetidos (mismo producto/categoría en
>   la misma promoción) solo protegidos por un **índice único parcial en PostgreSQL**, sin
>   validador de aplicación (RN-PROMO-76, `PENDIENTE` — depende de si existe manejo genérico de
>   `IntegrityError` fuera de este módulo, no verificado). Corregir en modernización validando
>   ambos casos en la capa de servicio con mensajes de negocio dedicados. Sin riesgo
>   retroactivo.
> - **A-37** (porción administración, RN-PROMO-41/54, RN-PROMO-68): reenviar el mismo estado
>   actual (incluido **`finished→finished`**) es un **no-op silencioso** que no pasa por la tabla
>   de transiciones; una promoción puede crearse directamente **`active`/`paused`** sin pasar por
>   `draft`. `PENDIENTE`, sin impacto económico. Documentar sin especificar.
> - **A-39** (`core/scheduler.py:224-236` vs `promotions/service.py:91-99`, RN-SCHED-11): el job
>   de medianoche (`expire_promotions`, `CronTrigger(hour=0, minute=0)`) compara `ends_at < now`
>   con `now=datetime.now(timezone.utc)` — **UTC absoluto con datetime completo**, distinto del
>   criterio de `_valid_now` (spec 012: hora local + solo fecha). Puede marcar **`finished`** una
>   promoción que la evaluación real seguiría considerando vigente, **desfase de hasta 1.5 días
>   en UTC-5**. ACCIDENTAL — el propio comentario del job se autodescribe "puramente
>   informativo": no afecta el cobro real (spec 012 sigue gobernándolo), solo la etiqueta de
>   estado que ve el admin. Corregir unificando el criterio de corte con `_valid_now`. Sin
>   riesgo retroactivo.
>
> **Criterios de aceptación**: `test_promotions_rules.py` cubre parcialmente la vigencia
> (compartida conceptualmente con spec 012), pero **ningún script cubre la máquina de estados**
> ni la validación de forma del `PATCH`/`PATCH /shape` de forma dedicada. Gap de caracterización
> — A-30 (un **500 real no controlado**, no solo un caso límite) debería ser el primer candidato
> a test explícito de esta spec.
>
> **Fuera de alcance**: cómo se calcula el descuento en tiempo real (spec 012); el consumo de las
> promociones administradas aquí en el menú/carrito (spec 007) y en cada camino de cobro (specs
> 008, 010, 011).

---

## 6. Tabla resumen

| Spec | Nombre | Reglas RN-* | Anomalías (propias + cross-ref) | Dueño de negocio | Characterization test más cercano |
|---|---|---|---|---|---|
| 001 | Identidad y acceso del personal | RN-AUTH ×10 | A-18, A-21(personal), A-22, A-23[P] | Dueño/gerente | ninguno (gap) |
| 002 | Catálogo: productos, variantes y precios | RN-CAT-01..11 | A-44 | Admin. de catálogo | `test_variantes_duplicadas.py` |
| 003 | Consumo de inventario por receta y opción | RN-CAT-12..26,34,35 | A-02[P], A-03, A-04(regla), A-33, A-34, A-47(regla) | Jefe de cocina | `test_variant_option_groups.py`, `test_receta_obligatoria.py` |
| 004 | Grupos de opciones: selección y tolerancia | RN-CAT-27..33,36..39 | A-05[P], A-06, A-32, A-04(cross a 009) | Admin. de catálogo | ninguno (gap) |
| 005 | Inventario y compras | RN-INV ×23, RN-CAT-40/41 | A-13, A-31, A-35 | Encargado de compras/inventario | ninguno (gap) |
| 006 | Caja y arqueo | RN-CASH ×17 | A-17(caja), A-20, A-40 | Cajero jefe | ninguno (gap; RN-CASH-17 es requisito nuevo) |
| 007 | Menú público y carrito del comensal (QR) | RN-MENU ×9, RN-CART ×27 | A-08, A-21(comensal), A-24[P], A-28, A-36(cart), A-47(cross) | TI + dueño/gerente | `test_qr_token.py`, `test_table_sessions.py` |
| 008 | Confirmación, cobro legado y cancelación | RN-ORD-01..35,65 | A-01(camino B), A-11(cross), A-25[P], A-29(cross), A-38(cross), A-42(auditoría) | Dueño/gerente | `test_cancel_inventory.py` |
| 009 | Cocina, consolidación y mesas físicas | RN-ORD-36..64,66 | A-04(manifestación), A-16, A-26, A-48 | Jefe de cocina / mesero-supervisor | ninguno (gap) |
| 010 | Sesión de mesa: reparto y cierre | RN-MESA ×27, RN-SCHED-01..09 | A-01(convención), A-11(cross), A-15[P], A-17(mesa), A-29(cross), A-38(cross) | Cajero jefe | `test_split_blindaje.py`, `test_table_release.py` |
| 011 | Ventas de mostrador y facturación | RN-VENTA ×17, RN-FACT ×7 | A-11(regla), A-12, A-14, A-14a[P], A-29(cross), A-41, A-43, A-49 | Dueño/gerente + contador | `test_facturacion.py` |
| 012 | Motor de evaluación de promociones y combos | RN-PROMO-01..45 | A-07[P], A-08(cross), A-09, A-10, A-36(promo), A-37(eval), A-46 | Admin. de promociones / dueño | `test_promotions_rules.py` (única en CI) |
| 013 | Administración de promociones: vigencia y estados | RN-PROMO-46..78, RN-SCHED-10/11 | A-30, A-37(admin), A-39 | Admin. de catálogo | ninguno (gap) |

**[P] = regla `[PROTEGIDA]`**, especificar tal cual, no tocar.

**Excluidas de la partición, justificadas en §4**: A-19, A-27, A-42(porción Horarios), A-45,
`memoria-historica.md` #1.

**Total verificado**: 333 reglas `RN-*` asignadas, 49 anomalías `A-*` + `A-14a` asignadas o
excluidas con justificación, 0 huérfanas.
