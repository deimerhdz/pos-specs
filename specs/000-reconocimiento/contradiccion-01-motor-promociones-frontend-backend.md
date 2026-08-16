# Contradicción 01 — El motor de promociones automáticas está escrito dos veces: Python (autoritativo) y TypeScript (previsualización)

**Fecha**: 2026-08-15
**Alcance**: `pos-backend` (`app/api/v1/promotions/service.py`) y `pos-heladeria`
(`src/app/modules/promotions/services/promotion-pricing.util.ts`).
**Método**: lectura directa de ambos ficheros, comparación función por función, `grep`
cruzado de los llamadores en ambos repos. No se corrige nada (Principio III de la
Constitución); este documento solo registra el hallazgo para el registro de anomalías y el
guion de entrevista al negocio.

---

## 1. Las implementaciones implicadas

**Backend (autoritativo, decide cuánto se cobra realmente)**: `promotions/service.py`.

- `local_now()` / `_tz()` — líneas 50-68: convierte `now` a hora **local del tenant**
  (`ZoneInfo(settings.TENANT_TIMEZONE)`, por defecto `America/Bogota`, `app/core/config.py:20`).
- `_in_time_window()` — líneas 71-86.
- `_valid_now(promo, now)` — líneas 89-104: normaliza `now` con `local_now()` antes de
  comparar estado, fechas, día de la semana y ventana horaria.
- `_matching_target()` — líneas 107-124.
- `_pack_terms()` — líneas 127-138.
- `_line_discount()` — líneas 141-160.
- `_best_line_match()` / `best_line_discount()` — líneas 233-287: elige la mejor promoción
  para una línea con criterio `(priority, amount, -created_at.timestamp())` (línea 268).
- `find_overlaps()` y sus auxiliares `_ranges_overlap`, `_csv_overlap`, `_times_overlap`,
  `_scope_overlap` — líneas 448-513.

Se invoca desde los cuatro caminos de cobro real (`checkout.py`, `sales/builder.py`,
`table_sessions/service.py`) vía `evaluate`/`evaluate_detailed`, que a su vez llaman a
`active_discount_promotions` (línea 211, que ya filtra por `_valid_now`).

**Frontend (previsualización, decide qué se muestra en pantalla antes de cobrar)**:
`promotion-pricing.util.ts`, explícitamente comentado como "port" función por función:

- `inTimeWindow()` (líneas 14-20) — comentario propio: "port de `_in_time_window()`".
- `isPromoActiveNow()` (líneas 35-48) — comentario propio: "Réplica de `_valid_now()` del
  backend... El cálculo real del descuento lo sigue haciendo el backend; esto solo decide
  elegibilidad en el cliente."
- `matchingTarget()` (60-73) — "port de `_matching_target()`".
- `packTerms()` (81-84) — "port de `_pack_terms()`".
- `lineDiscount()` (94-115) — "port de `_line_discount()`".
- `bestProductDiscount()` (129-166) — "Port de `best_line_discount()`... (El tercer
  desempate del backend es `created_at`, que la lista no expone.)" (comentario propio,
  líneas 118-121).
- `findOverlaps()` y auxiliares (276-351) — "port de `find_overlaps()`".

**Dónde se usa la previsualización, en el POS de staff (`pos-terminal.store.ts`)**:

- `combos` (líneas 247-252): filtra combos vigentes con `isPromoActiveNow(p, now)` usando
  `new Date()` del navegador.
- `productDiscountBadges` (261-277): insignia `-X%`/`-$X` por producto con
  `bestProductDiscount`, con comentario propio: "el monto real que se cobra lo sigue
  calculando el backend" (línea 259).
- `cartView` (384-475 aprox.): calcula `unitPrice` de cada línea del carrito (persistida
  o borrador) con `discountedUnitPrice` — líneas 393-400, 459-466, y una tercera vez en la
  franja ~1195. Este es el número que ve el cajero como subtotal antes de cobrar.

Y en el panel de administración (`promotions-page.component.ts`): `getPromoDisplay`
(línea 1504) y `findOverlaps` (líneas 1508, 1918) deciden si una promoción se etiqueta
`live`/`out_of_window`/etc. en la lista que revisa el admin.

## 2. ¿Usan la misma convención o algoritmo?

Casi exactamente la misma, a propósito: el frontend es un "port" deliberado, con
comentarios que citan la función Python correspondiente línea por línea. La lógica de
`_matching_target`/`matchingTarget`, `_pack_terms`/`packTerms` y `_line_discount`/
`lineDiscount` son fieles la una a la otra. Pero hay dos puntos donde el propio autor del
port documentó, en el comentario, que **no pudo** replicar el comportamiento exacto:

1. **Hora de referencia.** El backend siempre evalúa `_valid_now` contra la hora **local
   del tenant** (`local_now()`, conversión explícita vía `ZoneInfo`). El frontend evalúa
   `isPromoActiveNow(promo, now)` contra un `Date` de JavaScript que el propio navegador/SO
   del terminal produce en **su** zona horaria local — no hay ninguna conversión a
   `TENANT_TIMEZONE`, porque el cliente no tiene ese dato (no viaja en ningún endpoint
   consumido por `promotion-pricing.util.ts`).
2. **Desempate de la "mejor" promoción.** El backend desempata por `created_at` (la más
   antigua gana, línea 268 de `service.py`). El frontend, por su propio comentario, no
   puede: no tiene el campo. Su desempate real, no documentado como tal, es "la primera que
   aparece en el array" (`bestProductDiscount`, línea 160: comparación estricta `>`, nunca
   `>=`, así que en un empate el primer candidato recorrido nunca es reemplazado).

## 3. Ejemplos concretos de divergencia

### 3.1 Divergencia por zona horaria — visible y con efecto en dinero

Escenario: `TENANT_TIMEZONE=America/Bogota` (UTC-5, sin horario de verano — valor por
defecto de `pos-backend/app/core/config.py:20`, sin override en `.env.example`).
Promoción `percent`, `value=20`, `status=active`, `start_time=17:00`, `end_time=19:00`,
`days_of_week=null` (todos los días), destino: categoría "Granizados". Un granizado cuesta
$8.000.

Un terminal de venta (tablet o PC de mostrador) cuyo reloj/zona horaria del sistema
operativo **no** está sincronizado a `America/Bogota` — un caso plausible y común: tablets
Android de fábrica en UTC hasta que alguien configura la zona a mano, o un PC con
"hora automática" que no detectó bien la región — marca, al mismo instante real, una hora
distinta a la de Bogotá.

- **Instante real**: 2026-08-15 22:30 UTC = 2026-08-15 17:30 hora Bogotá (dentro de la
  ventana 17:00-19:00).
- **Backend** (`_valid_now`, vía `local_now()`): convierte a Bogotá → 17:30 → **dentro** de
  la ventana → la promoción está vigente. Al cobrar (`pay_order`/`_close_unified`/
  `_close_split`/`sales/builder.py`, todos vía `promotions.evaluate`), el granizado se
  factura a $6.400 (20% de descuento).
- **Frontend** (`isPromoActiveNow`, con `new Date()` del terminal en UTC sin corregir):
  `now.toTimeString()` da "22:30" → **fuera** de la ventana 17:00-19:00 → `bestProductDiscount`
  no encuentra coincidencia → ni la insignia (`productDiscountBadges`) ni el precio del
  carrito (`cartView` → `discountedUnitPrice`) muestran descuento → el cajero ve $8.000 en
  pantalla mientras arma el pedido.

**Resultado**: el cajero ve $8.000 en el carrito durante toda la construcción del pedido
(en plena hora pico de la promoción) y al cerrar la venta el sistema cobra $6.400 — una
diferencia de $1.600 por unidad que aparece recién al confirmar el pago, sin que nada en
pantalla lo haya anticipado. El caso inverso (terminal adelantado respecto a Bogotá) es
igual de posible y produce el efecto contrario: el terminal promete un descuento que el
cobro real no aplica.

Esto es una variante nueva y distinta del bug ya registrado como **R4** en
`registro-riesgos.md` (que describe un `datetime` naive mal interpretado como local en
`cart/service.py` y `menu/router.py`, del lado del comensal/QR). Aquí el mecanismo es
distinto — no es un bug de interpretación de naive/aware en el servidor, sino una
reimplementación completa del cálculo de vigencia en el cliente, que depende del reloj y
la zona horaria del dispositivo físico del staff, algo que el servidor nunca puede
corregir ni validar.

### 3.2 Divergencia por desempate — presente en el código, sin efecto visible hoy

Dos promociones `percent`, `value=10`, `priority=5` (empatadas), ambas vigentes, mismo
destino (categoría "Helados"): "Zapatos Gratis" (`created_at=2026-01-10`, la más antigua)
y "Aniversario" (`created_at=2026-07-01`, más reciente). Para una línea de $10.000, ambas
producen el mismo `amount` ($1.000, por tener el mismo `value`).

- **Backend** (`_best_line_match`, clave `(priority, amount, -created_at.timestamp())`):
  gana la más antigua, "Zapatos Gratis" — es la que queda registrada en `Sale.promotion_id`
  y en el desglose (`LineDiscount.promotion_name`) que el cajero ve en el detalle de cobro.
- **Frontend** (`bestProductDiscount`, sin tercer criterio): conserva la primera del
  array, que llega ordenado por `GET /promotions` con `priority.desc(), name` (orden
  alfabético en el empate) — es decir, "Aniversario" (alfabéticamente antes que "Zapatos
  Gratis").

El monto numérico coincide ($1.000 en ambos casos, porque el `value` es igual), así que
**nada en pantalla lo delata hoy**: la insignia de `productDiscountBadges` solo muestra el
porcentaje (`-10%`), no el nombre de la promoción (línea 271-272 de `pos-terminal.store.ts`);
y `discountedUnitPrice` tampoco expone qué promoción ganó. La divergencia existe en el
código y en qué `Promotion` se referencia internamente, pero solo se volvería visible el
día en que alguna pantalla del staff muestre el *nombre* de la promoción aplicada antes de
cobrar, o si dos promociones empatadas en `priority` y monto tuvieran, además, textos de
detalle (`_describe`) distintos que el cajero comparara contra lo impreso en el ticket.

## 4. Cuándo se manifiesta y por qué nadie lo ha visto

- **3.1 (zona horaria)** solo se manifiesta si el reloj/zona del dispositivo del staff
  difiere de `America/Bogota`. En un negocio físico donde el personal usa siempre el mismo
  terminal ya configurado correctamente, esto puede no haber ocurrido nunca; pero es
  exactamente la clase de fallo que aparece sin aviso al añadir un terminal nuevo, un
  navegador remoto (soporte técnico accediendo desde otro país), o un dispositivo que
  perdió la hora tras un reinicio de fábrica. Ambos lados **coinciden siempre** cuando el
  dispositivo está bien configurado — lo cual es, hasta donde este documento puede
  verificar, el caso normal — así que el margen de detección es estrecho y depende de un
  detalle de configuración de hardware que nadie audita.
- **3.2 (desempate)** no se manifiesta hoy porque ninguna pantalla del staff expone la
  identidad de la promoción ganadora en un empate, solo el monto — y el monto no cambia
  entre las dos promociones tomadas como ejemplo. Se manifestaría el día que dos
  promociones autómaticas con la misma `priority` produjeran, además, un `amount` distinto
  entre sí para la misma línea en algún instante — ahí sí backend y frontend podrían
  mostrar montos distintos en el raro caso de empate exacto de `priority` y casi-empate de
  `amount` que dependen del orden de iteración, aunque no se encontró un ejemplo con datos
  reales de catálogo que lo produzca hoy: se registra como riesgo latente en el código, no
  como divergencia numérica confirmada.

## 5. Historia probable

El propio encabezado de `promotions/service.py` (líneas 1-23) narra tres cambios
estructurales de una refactorización reciente: hora local del tenant, `evaluate_detailed`
con desglose por línea, y prioridad + desempates para resolver conflictos "para que el
resultado sea reproducible". Los comentarios de `promotion-pricing.util.ts` muestran que el
port al frontend se hizo **después** y de forma consciente y cuidadosa (cada función cita
su contraparte Python), probablemente para dar al catálogo del POS y al panel de
promociones una señal visual ("esto tiene descuento") sin pagar el costo de una llamada al
backend por cada render. El propio autor del port dejó constancia explícita de las dos
piezas que no pudo trasladar: el desempate por `created_at` (comentario en línea 121) y,
implícitamente, la zona horaria (el comentario dice "el cálculo real... lo sigue haciendo
el backend", reconociendo que esto es solo una aproximación, sin mencionar directamente el
riesgo de reloj del dispositivo). Es decir: la duplicación es intencional y documentada
como aproximación de UX, no un descuido — pero la aproximación tiene un punto ciego (la
zona horaria del dispositivo) que ninguno de los dos comentarios cubre explícitamente.

---

**Pregunta abierta al negocio**: ¿los terminales de venta usados en el local tienen su
reloj y zona horaria verificados y fijados a `America/Bogota`, o dependen de la
configuración automática del sistema operativo del dispositivo? ¿Se ha usado alguna vez el
panel de promociones o el POS desde un dispositivo fuera del local (soporte remoto, prueba
desde otra ciudad/país)?
