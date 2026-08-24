# Registro de anomalías — POS Heladería

**Fecha**: 2026-08-15
**Alcance**: consolidación de todo el reconocimiento e ingeniería inversa realizados sobre
`../pos-backend` y `../pos-heladeria` (ver [Constitución](../../.specify/memory/constitution.md)).
**Fuentes recorridas** (las seis, completas):

1. [`reglas-de-negocio.md`](./reglas-de-negocio.md) — 333 reglas; de ahí entran aquí las 43
   marcadas `DUDOSA`, las 12 marcadas `ACCIDENTAL`, y las `INTENCIONAL` con historia de
   incidente confirmada en `memoria-historica.md` (reglas protegidas).
2. Los seis `contradiccion-*.md`.
3. [`registro-riesgos.md`](./registro-riesgos.md) — 23 filas (R1-R23).
4. [`memoria-historica.md`](./memoria-historica.md) — 17 entradas cronológicas + "trabajo
   pendiente o abandonado".
5. `entrevista-negocio.md` y los resultados de arqueología de datos.
6. [`mapa-sistema.md`](./mapa-sistema.md) (sección 6, "zonas oscuras") y
   [`flujo-pedido-qr.md`](./flujo-pedido-qr.md) (sección 12, "hallazgos adicionales"), citados
   por varias entradas de `memoria-historica.md` y necesarios para cerrarlas sin dejarlas
   huérfanas.

**Aviso metodológico (actualizado 2026-08-16)**: al redactar la primera versión de este
documento, la fuente 5 (`entrevista-negocio.md` y arqueología de datos) no existía todavía en
este repositorio, así que el testigo **NEGOCIO** solo existía donde `memoria-historica.md`
documentaba una decisión atribuida a una persona con nombre y fecha ("NEGOCIO histórico, vía
comentario de commit"), y el testigo **DATOS** prácticamente no existía salvo lectura
determinista de código. Eso llevó a que la mayoría de lo `DUDOSA` quedara `PENDIENTE`. Desde
entonces se realizó una entrevista de negocio real (28 preguntas, acta en
[`entrevista-negocio.md`](./entrevista-negocio.md)) que aporta testigo **NEGOCIO (en vivo)**
para 20 de esas anomalías — ver la sección "Actualización — entrevista de negocio" más abajo,
que resume los cambios de clasificación resultantes. El testigo **DATOS** (arqueología de
datos vía consulta real a producción) **sigue sin existir** como proceso documentado; las
anomalías que dependían de ese testigo (notablemente A-06, A-32) siguen `PENDIENTE`.

**Regla innegociable aplicada sin excepción**: ninguna entrada se marca `INTENCIONAL` sin al
menos dos de los tres testigos (CÓDIGO, DATOS, NEGOCIO) a favor. Las entradas marcadas
**[PROTEGIDA]** son las únicas que alcanzan ese estándar hoy: tienen CÓDIGO (la implementación
y su docstring) **y** NEGOCIO histórico (una entrada nombrada y fechada de
`memoria-historica.md`). El resto de lo `INTENCIONAL` en `reglas-de-negocio.md` (~280 reglas
sin historia de incidente) queda fuera de este registro a propósito: no son anomalías, son
comportamiento documentado y sin conflicto.

**Orden**: por impacto económico estimado, de mayor a menor. Igual que en `registro-riesgos.md`,
el criterio de orden es un juicio de quien documenta; los datos de cada fila (evidencia) no lo
son. Las entradas `[PROTEGIDA]` se intercalan en el lugar que les corresponde por el impacto
que *tendría* romperlas, no porque estén rotas hoy — el tratamiento que les corresponde es
"especificar tal cual, no tocar".

---

## Actualización — entrevista de negocio (2026-08-16)

Se realizó una entrevista de negocio (28 preguntas, acta completa en
[`entrevista-negocio.md`](./entrevista-negocio.md)) que cierra 20 de las 26 clasificaciones
`PENDIENTE` que este documento tenía originalmente, más varias `DUDOSA`/`ACCIDENTAL` con
decisión de negocio abierta. **Las entradas individuales de abajo (Nivel 1-3) se dejan tal como
quedaron redactadas antes de la entrevista** — documentan el estado del análisis de código —,
y esta tabla es la capa de resultado que las actualiza. Ante cualquier lectura, esta tabla
prevalece sobre el campo "Decisión de negocio pendiente" de la entrada correspondiente.

| ID | Antes | Después de la entrevista | Detalle |
|---|---|---|---|
| A-01 | BUG A SECAS / BUG HISTÓRICO CON DEPENDIENTES | Sin cambio de clasificación — riesgo confirmado **latente, no activo** (no usan mesas fusionadas) | P1 |
| A-04 | BUG HISTÓRICO CON DEPENDIENTES | Reforzada — **testimonio de negocio confirma merma real hace ~15 días** en sabores/toppings; sube prioridad | P4 |
| A-05 | INTENCIONAL [PROTEGIDA], decisión pendiente | Decisión pendiente **cerrada**: catálogo nunca se depuró, `STRICT_OPTION_SELECTION` debe seguir en `False` | P18 |
| A-09 | PENDIENTE | Mitigado operativamente (P6) — **reabierta 2026-08-18**: corregir en modernización pese a la mitigación, ver spec 023 | P6 |
| A-11 | ACCIDENTAL, tope a definir | Tratamiento **redefinido y más estricto**: el cajero no debe poder aplicar descuento manual en absoluto (no solo un tope) | P5 |
| A-12 | ACCIDENTAL, alcance desconocido | **Alcance estimado**: 6-20 productos activos sin receta configurada; sube prioridad | P8 |
| A-13 | PENDIENTE | **ACCIDENTAL confirmado** — insumos inactivos bajo mínimo deben seguir en las alertas | P9 |
| A-14 | BUG A SECAS, prioridad alta | **Baja de prioridad** — el consecutivo se reinicia cada año (verificar mecanismo, ver nota abajo y nueva entrada **A-49**) | P10 |
| A-14a | INTENCIONAL [PROTEGIDA], decisión pendiente | `pay_order` legado confirmado **sin uso real** — lo que el negocio usa es la cuenta dividida (`split`) | P11 |
| A-15 | INTENCIONAL [PROTEGIDA], decisión pendiente | Decisión pendiente **cerrada**: sin ventana de exposición, empezaron a usarla después del blindaje | P12 |
| A-17 | ACCIDENTAL/PENDIENTE | Reparto-vs-cierre **mitigado operativamente** (una persona por mesa a la vez) | P17 |
| A-18 | PENDIENTE | **Cerrada sin impacto** — nunca hubo cuentas con rol STAFF | P13 |
| A-20 / RN-CASH-09 | DUDOSA | **Mitigado** — la pantalla siempre exige el conteo antes de cerrar | P14 |
| A-20 / RN-CASH-13 | DUDOSA | Sigue **PENDIENTE** — sin respuesta concluyente | P14 |
| A-20 / RN-CASH-17 | DUDOSA | **Requisito de negocio nuevo**: snapshot inmutable del cierre + vista de ajustes separada (más estricto que "congelar" o "recalcular") | P14 |
| A-21 | PENDIENTE | **INTENCIONAL confirmado** — localStorage es el diseño definitivo, no prioridad la cookie httpOnly | P15 |
| A-27 | ACCIDENTAL | Reforzada — hay revisión manual hoy, **el negocio pide explícitamente pasar a pruebas automáticas** | P24 |
| A-29 | PENDIENTE | Sigue PENDIENTE en código, pero **sin impacto práctico** — no usan ese reporte | P21 |
| A-31 | DUDOSA, candidato a retirar | **Cambia de tratamiento**: candidato a completar la migración (necesitan conversión litros↔onzas para el granizado) | P19 |
| A-33 | PENDIENTE | **INTENCIONAL confirmado** — el bloqueo actual es correcto | P22 |
| A-35 / RN-INV-11 | DUDOSA | **Requisito confirmado**: motivo de ajuste debe ser obligatorio | P23 |
| A-35 / RN-INV-17 | DUDOSA | **INTENCIONAL confirmado** — "último costo de compra" es el comportamiento deseado | P23 |
| A-41 | INTENCIONAL, con reserva | Reserva **cerrada** — tax=0 es política fiscal deliberada y definitiva | P2 |
| A-42 (Horarios) | PENDIENTE | **Cerrada** — retiro fue decisión de producto deliberada; habilita migración de borrado de `business_hours` | P25 |
| A-42 (Auditoría) | PENDIENTE | Confirma que **nadie consulta `audit_logs` hoy** — decisión de negocio pendiente sobre si mantenerlo | P25 |
| A-46 | PENDIENTE | **Cerrada sin urgencia** — sin planes de expansión a otra zona horaria | P26 |
| A-48 | PENDIENTE | **INTENCIONAL confirmado** — fusión de KDS y terminal fue decisión operativa deliberada | P28 |
| memoria#1 (DIAN) | Pregunta abierta | **Cerrada** — sin obligación regulatoria conocida hoy | P3 |

**Siguen sin decidir tras la primera ronda** (5): A-06 (P7), la sub-pregunta de RN-CASH-13
dentro de A-20 (P14), A-16 (P16), A-26 (P20), A-47 (P27). **Las 5 se cerraron el mismo día en
una segunda ronda** — ver tabla siguiente.

### Actualización — segunda ronda de entrevista (2026-08-16, mismo día)

Las 5 preguntas sin respuesta concluyente de la primera ronda se repreguntaron pidiendo una
decisión de tratamiento en vez de repetir la pregunta de hecho (detalle completo en
[`entrevista-negocio.md`](./entrevista-negocio.md) §6). Las 5 decidieron la cuestión.

| ID | Antes (tras 1ª ronda) | Después de la 2ª ronda | Detalle |
|---|---|---|---|
| A-06 | PENDIENTE, sin decisión de tratamiento | Sigue `PENDIENTE` en clasificación (no reúne 2/3 testigos), pero **decisión de tratamiento cerrada**: aceptar el riesgo por ahora, igual que A-05 — no se prioriza corrección ni consulta a datos | P7-bis |
| A-20 / RN-CASH-13 | PENDIENTE, sin decisión | **Requisito de negocio confirmado**: el arqueo parcial también debe exigir motivo obligatorio si la diferencia es distinta de cero, igual que el cierre real | P14-bis |
| A-16 | ACCIDENTAL (parcial) / PENDIENTE (RN-ORD-37) | Porción `PENDIENTE` **cerrada** — mitigada por el ritmo de trabajo actual (mismo patrón que A-01/A-09/A-17); la porción `ACCIDENTAL` no cambia | P16-bis |
| A-26 | PENDIENTE (RN-ORD-58) / ACCIDENTAL (resto) | Porción `PENDIENTE` **cerrada** — riesgo latente, no activo (no usan mover pedidos entre mesas); el resto `ACCIDENTAL` no cambia | P20-bis |
| A-47 | PENDIENTE (solo testigo CÓDIGO) | **`INTENCIONAL` confirmado** (2/3 testigos: CÓDIGO + NEGOCIO) — el dueño prefiere el diseño best-effort actual antes que invertir en reservar stock | P27-bis |

Con esto, de las 26 clasificaciones `PENDIENTE`/`DUDOSA` originales de este registro, **25
quedaron cerradas** entre ambas rondas. Solo permanecen sin cerrar: `A-22` (fuera del guion de
la entrevista por alcance) y `A-49` (anomalía nueva revelada por la propia entrevista, que
requiere verificación técnica antes de poder repreguntarse con sentido) — ver la lista de
PENDIENTES al final de este documento. **Actualización 2026-08-16 (ronda 3, simulada)**: estos
4 puntos (A-22, A-49, alcance de A-11, alcance de A-31) se cerraron en una tercera ronda — ver
aviso y detalle inmediatamente abajo.

### Actualización — tercera ronda, SIMULADA a petición del usuario (2026-08-16)

**Aviso de método**: a diferencia de las dos rondas anteriores, esta ronda **no es una
entrevista real con el negocio**. Se realizó porque el usuario de este repositorio pidió
explícitamente destrabar el ejercicio de partición de specs asumiendo el rol de cliente y
eligiendo la respuesta más razonable, sin esperar una conversación real. Detalle completo,
incluida la verificación técnica que la precede, en
[`entrevista-negocio.md` §8](./entrevista-negocio.md). Efecto sobre la "regla innegociable" de
dos testigos: la verificación técnica de A-49 sí es un testigo CÓDIGO real; el testimonio de
negocio de esta ronda **no** cuenta como testigo NEGOCIO genuino para esa regla — se anota así
en la entrada de A-49. **Antes de tratar estas cuatro decisiones como definitivas para
producción, deben ratificarse con el negocio real** en una próxima ronda no simulada.

| ID | Antes | Después de la ronda 3 (simulada) |
|---|---|---|
| A-49 | PENDIENTE, requería verificación técnica | Verificación técnica confirma: **no existe ningún mecanismo de reinicio** en `app/core/scheduler.py` ni en el resto del código (solo corren `sweep_orphan_table_sessions` y `expire_promotions`). Testimonio simulado descarta la premisa de P10. A-14 recupera su prioridad original; tratamiento: ampliar el padding a 7+ dígitos |
| A-11 (alcance) | Sin cerrar | La prohibición de descuento manual de cajero aplica **a los tres caminos de cobro** (mostrador, unificado, dividido), sin excepción |
| A-31 (alcance) | Sin cerrar | Solo litros↔onzas (misma dimensión, volumen); conversión entre dimensiones distintas (masa↔volumen) **queda fuera de alcance**, sin caso de uso real |
| A-22 | ACCIDENTAL (RN-AUTH-03) / PENDIENTE (RN-AUTH-08, RN-AUTH-09) | RN-AUTH-08 y RN-AUTH-09 pasan a **ACCIDENTAL confirmado** (sin infraestructura de rate-limit, refresh-en-logout no deliberado, sin validación de 72 bytes) — las tres reglas de A-22 quedan ACCIDENTAL, corregir en modernización sin reserva |

Con esto, **la lista de clasificaciones PENDIENTES queda vacía a efectos de este ejercicio** —
ver la nota de cierre al final de este documento.

**Tres requisitos/condiciones nuevos surgidos sin habérselos preguntado directamente** (detalle
completo en `entrevista-negocio.md` §4): prohibición de descuento manual para el rol cajero
(A-11); snapshot inmutable de caja + vista de ajustes separada (A-20/RN-CASH-17); petición
explícita de pruebas automáticas en el proceso de publicación (A-27).

**Discrepancia código-vs-práctica a verificar**: el negocio afirma que el consecutivo de
facturación (A-14) se reinicia cada año, pero el reconocimiento original no encontró ningún
mecanismo de reinicio en `InvoiceCounter`. Antes de despriorizar A-14 de forma definitiva,
conviene confirmar técnicamente cómo ocurre ese reinicio (¿manual, por fuera del sistema?
¿automático y no detectado en la lectura de código original?). Esta discrepancia, al no estar
documentada en ningún fichero de reconocimiento previo a la entrevista, se registra como entrada
nueva propia: **A-49** (Nivel 3, más abajo, sección "Reinicio anual del consecutivo de
facturación"), para no quedar enterrada dentro de A-14.

---

## Nivel 1 — Alto impacto económico

### A-01 — "Cuánto se le debe cobrar a esta mesa" tiene tres implementaciones, y dos ignoran promociones y órdenes ya pagadas
**Descripción**: `table_sessions.compute_bill` (correcta, en uso), `orders/checkout.compute_bill`
(sin descuentos, sin caller hoy) y `tables_advanced.group_bill` (sin descuentos, sin excluir
pagadas, **en uso real para mesas fusionadas**) responden la misma pregunta de negocio con
resultados distintos.
- **CÓDIGO**: `table_sessions/service.py:139-181` (A, correcta); `orders/checkout.py:127-180`
  (B, sin descuentos, sin caller confirmado por `grep`); `orders/tables_advanced.py:92-114`
  (C, `group_bill`, sin filtro de status, sin descuentos); `reglas-de-negocio.md` RN-ORD-02
  (INTENCIONAL, excluye anuladas), RN-ORD-03 (DUDOSA, B incluye `pagada`), RN-ORD-64 (DUDOSA,
  C no filtra por status de orden).
- **DATOS**: ninguno (no hay arqueología de datos); el ejemplo cuantitativo (159% de
  sobreestimación, $35.000 vs $13.500 reales) está calculado en `contradiccion-06...md §3`
  sobre datos hipotéticos, no una consulta real.
- **NEGOCIO**: ninguno.
**Clasificación**: **BUG A SECAS** (camino B, `orders/checkout.compute_bill`, sin caller —
código muerto pero peligroso si se reactiva) + **BUG HISTÓRICO CON DEPENDIENTES** (camino C,
`group_bill`, confirmado en uso real por `table.service.ts:133` para la función de mesas
fusionadas).
**Depende de esto**: cualquier negocio que use "fusionar/agrupar mesas" hoy (camino C, activo);
nadie identificado depende del camino B porque no tiene caller conocido, pero reactivarlo
sin arreglarlo reproduciría el mismo error.
**Tratamiento acordado**: corregir en fase de modernización. Camino C (`group_bill`) debe
excluir pagadas/canceladas y aplicar `promotions.evaluate` igual que camino A — **no
retroactivo**: cuentas de grupo ya cobradas no se recalculan, solo el cálculo hacia adelante.
Camino B se documenta como código muerto candidato a retirar (o a unificar con A) en la misma
fase que se resuelva H2/H3 de `flujo-pedido-qr.md` (el ciclo `pay_order` legado).
**Decisión de negocio pendiente**: ¿se usa la función de fusionar/agrupar mesas en el día a
día? ¿Alguna vez se cobró una orden de una mesa fusionada por separado del resto del grupo
antes de cerrar todo el grupo? (pregunta original de `contradiccion-06...md`). Responsable:
por identificar — no hay contacto de negocio documentado en este repositorio.

### A-02 — [PROTEGIDA] Corrección del doble descuento de inventario por tamaño+opción (bug histórico de 140g)
**Descripción**: el consumo por opción elegida usa una sola cantidad — la del tamaño manda
sobre la de la opción — corrigiendo un bug real donde ambas se sumaban.
- **CÓDIGO**: `app/api/v1/catalog/consumption_plan.py:116-138`; regla RN-CAT-18
  (`reglas-de-negocio.md:241-246`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #8 (2026-08-03, commit `03469cad`,
  Deimer Hernandez): "con los sabores en 80g y una ensalada pequeña en 60g, cada venta
  descontaba 140g — nadie se enteraba hasta el conteo físico".
- **DATOS**: ninguno — pregunta abierta en la propia entrada #8: ¿coincide algún conteo físico
  histórico con merma sin causa aparente antes de esta fecha?
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos: CÓDIGO + NEGOCIO histórico).
**Depende de esto**: todo el cálculo de consumo de insumos por opción en `orders`, `sales` y
`table_sessions` (todos los caminos de cobro pasan por `plan_line_consumption`); indirectamente,
la exactitud del kardex de insumos con sabores/toppings.
**Tratamiento acordado**: **especificar tal cual, no tocar**. Cualquier especificación formal
de "cálculo de consumo por línea" debe fijar explícitamente "la cantidad del tamaño manda si
está definida; si no, la de la opción; nunca se suman" como invariante de test, precisamente
porque ya se rompió una vez.
**Decisión de negocio pendiente**: ninguna para el comportamiento en sí. Queda abierta la
pregunta forense de la entrada #8 (duración del bug, si hay una merma histórica atribuible),
sin impacto en la especificación futura.

### A-03 — El docstring del modelo `VariantOptionGroup` sigue diciendo que las cantidades "se suman", contradiciendo la corrección de A-02
**Descripción**: el comentario del campo `quantity_per_option` en el modelo afirma "se suma a
`options.item_quantity`"; el código hace exactamente lo opuesto desde la corrección de A-02.
- **CÓDIGO**: `app/models/variant_option_group.py:46-49` vs
  `app/api/v1/catalog/consumption_plan.py:24-28,126-129`; regla RN-CAT-19
  (`reglas-de-negocio.md:248-253`).
**Clasificación**: **ACCIDENTAL** (contradicción textual directa; no requiere testigo de
negocio porque no es una pregunta de intención, es un hecho verificable en el propio
repositorio).
**Depende de esto**: nadie hoy — pero es exactamente el tipo de comentario obsoleto que puede
inducir a un futuro desarrollador a "corregir" A-02 de vuelta al bug original, creyendo que
sigue el comentario del modelo.
**Tratamiento acordado**: corregir en fase de modernización — actualizar el docstring del
modelo para que describa el comportamiento real de A-02. Corrección puramente documental, sin
riesgo de retroactividad porque no cambia comportamiento.
**Decisión de negocio pendiente**: ninguna.

### A-04 — La validación de selección de opciones se omite en el camino que usa realmente el mesero (`add_item_to_table`)
**Descripción**: `load_valid_options` solo valida `min_select`/`max_select`/pertenencia si se
le pasa `variant`; el camino con botón real en la terminal del mesero (`add_item_to_table`) no
se lo pasa, mientras que `create_order` (sin caller de UI conocido) sí. Reconstruido con
`git log`/`git show`: es una regresión de fusión entre una rama de corrección (`03469ca`,
2026-08-03) y una rama de funcionalidad nueva de combos (`ee94f30`, 2026-08-04, autor distinto:
LeonardoGomezz) que partió de una copia del fichero anterior a la corrección.
- **CÓDIGO**: `app/api/v1/catalog/line_pricing.py:43-66` (`load_valid_options`);
  `app/api/v1/orders/service.py:102` (SÍ pasa `variant`); `app/api/v1/orders/consolidation.py:199`
  (NO pasa `variant`, con la variable ya en ámbito dos líneas arriba); reglas RN-CAT-33
  (DUDOSA, `reglas-de-negocio.md:343-347`) y `contradiccion-05-validacion-opciones-omitida-alta-directa.md`
  completo (incluye el rastreo exacto de commits `a95fe31` → `03469ca` → `ee94f30`).
- **DATOS**: ninguno — pregunta abierta en `contradiccion-05...md`: ¿un conteo físico ha
  detectado desfase en ítems de sabor múltiple consumiendo menos insumo del debido?
**Clasificación**: **BUG HISTÓRICO CON DEPENDIENTES** — es el único hallazgo de todo el
reconocimiento con prueba directa de `git log` de cómo y cuándo se rompió (no una suposición).
**Depende de esto**: la exactitud del kardex de cualquier ítem con grupo de opciones
obligatorio que descuenta inventario (ej. "3 bolas de sabor"), vendido por el mesero desde la
terminal (el camino real de uso diario, según `flujo-pedido-qr.md`).
**Tratamiento acordado**: corregir en fase de modernización — pasar `variant=variant` en
`consolidation.py:199` (una línea, ya se hizo una vez en `03469ca` y se perdió en el merge).
**No retroactivo**: no hay forma de recalcular inventario ya consumido incorrectamente; solo
detiene el sangrado hacia adelante. Antes de corregir, considerar si conviene auditar el
catálogo con `min_select>0` + `quantity_per_option>0` para dimensionar el desfase acumulado.
**Decisión de negocio pendiente**: la pregunta de `contradiccion-05...md` (arriba). Responsable:
por identificar.

### A-05 — [PROTEGIDA] `STRICT_OPTION_SELECTION=False` es una tolerancia deliberada de migración, no un descuido
**Descripción**: se decidió explícitamente no validar por defecto que las selecciones de
opciones respeten `min_select`/`max_select`, porque "el catálogo histórico nunca se validó"
— activar el chequeo a ciegas rechazaría combinaciones ya cargadas en producción.
- **CÓDIGO**: `app/core/config.py:55-63`; regla RN-CAT-31 (`reglas-de-negocio.md:329-334`);
  script de premigración `app/scripts/opciones_fuera_de_grupo.py`; registro de riesgos **R5**
  (mismo hallazgo, con foco en el riesgo residual de menú/precio silencioso).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #9 (2026-08-03, commit `03469cad`,
  Deimer Hernandez).
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos).
**Depende de esto**: todo el catálogo histórico no depurado; apagar el flag sin antes correr
`opciones_fuera_de_grupo.py` rechazaría ventas legítimas de la noche a la mañana.
**Tratamiento acordado**: **especificar tal cual, no tocar** el default `False` sin que el
negocio confirme primero que el catálogo ya se depuró (ver decisión pendiente).
**Decisión de negocio pendiente**: ¿se llegó a correr `opciones_fuera_de_grupo.py` alguna vez?
¿Sigue el catálogo sin depurar, casi un año después de esta nota? (memoria-historica.md #9).
Mientras no se responda, el residuo de riesgo de esta tolerancia (ver A-06 y A-31) permanece
activo. Responsable: por identificar.

### A-06 — Con la tolerancia de A-05 activa, se puede cobrar y consumir inventario de una opción de un grupo que la variante no ofrece
**Descripción**: si el `gid` de la opción elegida no corresponde a ningún grupo vinculado a la
variante, ese problema nunca es bloqueante por sí solo; con `STRICT_OPTION_SELECTION=False`
pasa sin error, sigue sumando `extra_price` al cobro y, si tiene `item_quantity>0` propio,
sigue generando consumo de un insumo ajeno a la variante vendida.
- **CÓDIGO**: `app/api/v1/catalog/line_pricing.py:141-144,164-172`;
  `consumption_plan.py:116-129`; regla RN-CAT-32 (DUDOSA, `reglas-de-negocio.md:336-341`).
**Clasificación**: **PENDIENTE** — es la consecuencia directa y ya identificada de una
tolerancia deliberada (A-05), pero nadie confirmó si este caso específico (grupo totalmente
ajeno) es un residuo aceptado o un vector no previsto de esa decisión.
**Pregunta que la desbloquearía**: con `STRICT_OPTION_SELECTION=False`, ¿es tolerancia
deliberada durante la migración que se pueda cobrar y consumir inventario de una opción de un
grupo que la variante no ofrece, o es un caso que se pasó por alto al diseñar la tolerancia?
(reglas-de-negocio.md, pregunta abierta #5).
**Depende de esto**: nadie identificado hoy — riesgo latente mientras el catálogo no esté
depurado.
**Tratamiento acordado**: documentar sin especificar hasta la respuesta del negocio.

### A-07 — [PROTEGIDA] Motor de promociones reescrito: hora local del tenant, desglose por línea y prioridad explícita
**Descripción**: la vigencia pasa a evaluarse en hora local del tenant (antes UTC, corriendo
día de la semana y ventana horaria); `evaluate_detailed` devuelve desglose por línea; y ante
promociones empatadas gana la de mayor `priority` explícita, no siempre el descuento mayor.
- **CÓDIGO**: `app/api/v1/promotions/service.py:1-20,50-104,233-287`; reglas RN-PROMO-01,
  RN-PROMO-13, RN-PROMO-14, RN-PROMO-47 (todas INTENCIONAL, `reglas-de-negocio.md`);
  test dedicado en CI: `app/scripts/test_promotions_rules.py` (único script de test que sí
  corre en el pipeline — ver A-25).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #15 (2026-08-07, commit `2e94a3ad`,
  autor **`refactor <dev@local>`** — no una persona identificable en el historial): "un 20% los
  martes empezaba el lunes a las 19:00 locales, que es justo cuando una heladería vende".
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos: CÓDIGO + NEGOCIO histórico,
reforzado con test en CI — el único caso de este documento con tres señales, aunque DATOS en
sentido estricto de "consulta a producción" sigue sin existir).
**Depende de esto**: el monto cobrado en los cuatro caminos de cobro real (mostrador,
unificado, dividido, `pay_order` legado) y el reporte de promociones aplicadas.
**Tratamiento acordado**: **especificar tal cual, no tocar**. Nota de gobernanza aparte (no
bloqueante): el autor del commit que hizo este cambio no es identificable por nombre en el
historial — el negocio podría querer indagar quién lo escribió y revisó.
**Decisión de negocio pendiente**: ¿el bug de zona horaria llegó a afectar una promoción real
en producción antes del 2026-08-07 (reclamos de clientes/cajeros)? (memoria-historica.md #15).
Responsable: por identificar.

### A-08 — `cart/service.py` y `menu/router.py` siguen tratando UTC como si fuera hora local — el mismo bug que A-07 corrigió, sin corregir aquí
**Descripción**: el carrito del comensal y el menú público por QR evalúan qué promociones
están vigentes con `datetime.now(timezone.utc).replace(tzinfo=None)` — un naive que en
realidad es UTC. `promotions.local_now()` asume que un `datetime` naive **ya está** en hora
local y no lo convierte. Con `TENANT_TIMEZONE=America/Bogota` (UTC-5) esto reproduce, en estos
dos puntos exactos, el bug que A-07 documenta haber corregido en el resto del sistema.
- **CÓDIGO**: `app/api/v1/cart/service.py:52-53,205-206`; `app/api/v1/menu/router.py:82-83`;
  función mal interpretada: `app/api/v1/promotions/service.py:57-67`; registro de riesgos R4
  (`registro-riesgos.md`).
- **DATOS/NEGOCIO**: ninguno.
**Clasificación**: **ACCIDENTAL** (contraste directo con A-07, que sí se corrigió en los cuatro
caminos de cobro real — CÓDIGO por sí solo basta para ACCIDENTAL, no requiere el estándar de 2
testigos que exige INTENCIONAL).
**Depende de esto**: cualquier comensal viendo el menú/carrito por QR en horario cercano a un
límite de vigencia de promoción; el monto cobrado no se ve afectado (los caminos de cobro real
sí usan `datetime` aware) — solo lo que se muestra antes de pagar, generando reclamos y
desconfianza en el momento del cobro.
**Tratamiento acordado**: corregir en fase de modernización, aplicando exactamente el mismo
patrón que ya funciona en `checkout.py`/`table_sessions/service.py`/`sales/service.py` (pasar
`datetime` aware, o convertir explícitamente con `local_now()`). Sin riesgo de retroactividad
— es una corrección de visualización, no de dinero ya cobrado.
**Decisión de negocio pendiente**: ninguna — es una corrección técnica directa, no una
decisión de producto.

### A-09 — El POS de staff previsualiza promociones con el reloj del dispositivo, sin conversión a la zona horaria del tenant
**Descripción**: el backend evalúa vigencia siempre en hora local del tenant (A-07); el
frontend del terminal de venta (`promotion-pricing.util.ts`, un "port" deliberado y
documentado línea por línea del motor Python) evalúa contra `new Date()` del navegador/SO del
dispositivo — sin conversión a `TENANT_TIMEZONE`, porque el cliente no tiene ese dato. Un
terminal con reloj/zona mal configurada (tablet de fábrica en UTC, PC con detección de región
fallida) puede mostrar un precio distinto al que el sistema cobra al confirmar.
- **CÓDIGO**: `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`
  (comentarios propios citando la función Python correspondiente); `pos-terminal.store.ts:247-277,384-475`;
  `contradiccion-01-motor-promociones-frontend-backend.md` completo, incluido el ejemplo
  cuantitativo ($8.000 mostrado vs $6.400 cobrado, diferencia de $1.600/unidad).
- **DATOS/NEGOCIO**: ninguno.
**Clasificación**: **PENDIENTE** — el propio documento fuente lo describe como "aproximación
de UX documentada, no un descuido", pero con un punto ciego (reloj del dispositivo) que ningún
comentario cubre explícitamente; no hay testimonio de negocio que confirme si ese punto ciego
es un riesgo aceptado.
**Pregunta que la desbloquearía**: ¿los terminales de venta del local tienen su reloj y zona
horaria verificados y fijados a `America/Bogota`, o dependen de la configuración automática
del sistema operativo del dispositivo? ¿Se ha usado el POS o el panel de promociones desde un
dispositivo fuera del local? (`contradiccion-01...md`).
**Depende de esto**: cualquier cajero que arme un pedido en horario cercano a un límite de
promoción, en un terminal con reloj no verificado.
**Tratamiento acordado**: documentar sin especificar hasta la respuesta del negocio. Si la
respuesta confirma que los terminales no están verificados, la corrección natural (enviar
`TENANT_TIMEZONE` al frontend y convertir) pasa a fase de modernización sin retroactividad
(es solo previsualización).

**Actualización — reapertura de la decisión (2026-08-18)**: la respuesta P6 (relojes
verificados y fijados a `America/Bogota`) mitigó el riesgo operativamente, pero el defecto de
diseño en el código quedó sin corregir, dependiente de que esa disciplina de configuración se
mantenga en cada terminal presente y futuro — nadie la audita de forma continua. El
propietario del repositorio (deimerhdz21@gmail.com), actuando como negocio, decide **reabrir**
esta decisión y pasar el tratamiento de "mitigado operativamente, documentar sin especificar" a
**corregir en modernización**, con el mismo criterio de riesgo latente ya aplicado a A-08 (spec
022, corregida por contraste directo con A-07). Ver spec
[023-correccion-zona-horaria-pos-staff](../023-correccion-zona-horaria-pos-staff/spec.md),
que especifica la corrección sin tocar el motor de promociones del backend (A-07, protegido) ni
el desempate del frontend (A-10, anomalía distinta).

### A-10 — El desempate de la "mejor promoción" en el frontend no replica el criterio real del backend (creado, no antigüedad)
**Descripción**: el backend desempata promociones con el mismo `priority` y monto por
`created_at` (gana la más antigua); el frontend, por no tener ese campo, conserva "la primera
del array" — un desempate distinto no documentado como tal.
- **CÓDIGO**: `promotions/service.py:268` vs `promotion-pricing.util.ts:118-121,160` (comentario
  propio reconoce la limitación); `contradiccion-01...md §3.2`.
**Clasificación**: **ACCIDENTAL** (divergencia de código verificable, sin efecto visible hoy
porque ninguna pantalla expone el nombre de la promoción ganadora en un empate — se registra
como riesgo latente, no como divergencia numérica confirmada en datos reales).
**Depende de esto**: nadie hoy de forma visible.
**Tratamiento acordado**: documentar sin especificar; corregir en fase de modernización solo
si se agrega una pantalla que muestre el nombre de la promoción ganadora antes de cobrar (el
momento en que este defecto se volvería visible).
**Decisión de negocio pendiente**: ninguna en el estado actual.

### A-11 — Descuento de checkout sin tope superior ni control de rol
**Descripción**: el `discount` de cualquier venta (mostrador, unificado, dividido) no tiene
`le=` en el schema; el único freno es que el total no quede negativo. Cualquier cajero
autenticado —no solo un admin— puede aplicar un descuento igual al subtotal completo, incluso
por error de tipeo.
- **CÓDIGO**: `sales/schemas.py:63` (`discount: Decimal = Field(0, ge=0, ...)`, sin `le=`);
  `sales/builder.py:132-136` (único chequeo: `total < 0`); `sales/router.py:46-53`
  (`create_sale` solo exige `Depends(get_current_user)`, contrastado con `create_payment_method`
  en el mismo router, línea 31, que sí exige `require_tenant_admin`); registro de riesgos R2.
**Clasificación**: **ACCIDENTAL** (asimetría verificable entre dos endpoints del mismo router
que protegen con distinto rigor).
**Depende de esto**: la integridad del margen de cada venta; cualquier cajero con acceso al
sistema hoy puede regalar una venta completa sin que quede señalado.
**Tratamiento acordado**: corregir en fase de modernización — introducir tope superior
configurable y/o exigir rol para descuentos por encima de un umbral. **No retroactivo**: no se
puede recalcular ventas ya cobradas con descuento excesivo; solo se puede auditar el histórico
para dimensionar el impacto ya ocurrido (requiere una consulta de datos que hoy no existe).
**Decisión de negocio pendiente**: cerrada en la primera ronda de entrevista (P5): el cajero no
debe poder aplicar descuento manual en absoluto (más estricto que un tope). El alcance —¿aplica
a los tres caminos de cobro?— quedó cerrado en la ronda 3 (simulada, ver
`entrevista-negocio.md` §8, P30): **sí, a los tres** (mostrador, unificado, dividido), sin
excepción.

### A-12 — Venta de mostrador cobra variantes sin receta ni opción configurada, sin descontar nada de inventario
**Descripción**: a diferencia de la confirmación de pedidos por QR (que bloquea con 409 si no
hay nada que descontar), el camino de mostrador no bloquea el cobro si la variante no tiene
ninguna regla de consumo definida; la venta se cobra y factura sin generar ningún movimiento.
- **CÓDIGO**: `app/api/v1/sales/consumption.py:46-51`; regla RN-VENTA-11
  (`reglas-de-negocio.md:1532-1536`); el propio comentario del código lo describe como un
  "agujero" conocido y creciente ("con slots el agujero crece").
**Clasificación**: **ACCIDENTAL** (el propio equipo lo documenta como no deseado — no requiere
testigo de negocio adicional porque el código mismo ya declara la intención contraria).
**Depende de esto**: la exactitud del kardex de cualquier variante de mostrador sin receta
configurada; el tamaño del "agujero" crece con cada nueva variante mal cargada.
**Tratamiento acordado**: corregir en fase de modernización, replicando el bloqueo de
RN-CAT-34 (409 si el plan de consumo agregado es vacío) también en el camino de mostrador.
**No retroactivo**: no hay forma de reconstruir el inventario no descontado de ventas pasadas.
**Decisión de negocio pendiente**: ¿cuántas variantes de mostrador activas hoy no tienen
receta ni opción con consumo? (requiere una consulta que hoy no existe — candidata directa a
arqueología de datos).

### A-13 — "¿Este insumo está bajo mínimo?" se calcula con tres criterios distintos, y solo dos coinciden
**Descripción**: el filtro `low_stock` del listado general de insumos no excluye inactivos por
defecto; el endpoint dedicado `/items/low-stock` (tarjeta del dashboard) y el reporte
"Insumos por reponer" sí exigen `active=True` siempre. Un insumo desactivado con existencias
residuales bajo mínimo aparece en una pantalla y no en las otras dos, sin que ninguna se
declare "la lista completa".
- **CÓDIGO**: `inventory/service.py:24-43` (A, sin excluir inactivos por defecto);
  `inventory/router.py:51-61` (B, exige `active=True`); `reports/service.py:114-128` (C, exige
  `active=True`); regla RN-INV-15 (ACCIDENTAL, `reglas-de-negocio.md:501-506`);
  `contradiccion-03-insumo-bajo-minimo-tres-implementaciones.md` completo.
- **DATOS/NEGOCIO**: ninguno — pregunta abierta en `contradiccion-03...md`.
**Clasificación**: **ACCIDENTAL** (B y C coinciden entre sí exactamente; A diverge sin
justificación documentada — verificable por código solo).
**Depende de esto**: el dueño del negocio, que confía en la tarjeta del dashboard (B) y el
reporte (C) para decidir reposición; un insumo descontinuado con stock residual bajo mínimo
puede quedar invisible en ambas fuentes principales indefinidamente.
**Tratamiento acordado**: documentar sin especificar hasta la respuesta de negocio (afecta
directamente qué comportamiento es "correcto", no solo cuál es consistente). Una vez resuelta,
corregir en fase de modernización unificando el criterio en las tres fuentes — **sin impacto
retroactivo**, es una consulta, no un dato almacenado.
**Decisión de negocio pendiente**: cuando un insumo se marca inactivo con existencias bajo su
mínimo, ¿el negocio espera seguir viéndolo en las alertas de "bajo mínimo", o "inactivo" ya
implica "fuera de las alertas de reposición"? (`contradiccion-03...md`). Responsable: por
identificar.

### A-14 — El número de factura se formatea distinto en Python (sin truncar) y SQL (`lpad`, que sí trunca) — la búsqueda por número falla desde el consecutivo 1.000.000
**Descripción**: `Invoice.full_number` (Python, `f"{n:06d}"`) nunca trunca un número que exceda
6 dígitos; la reconstrucción SQL con `lpad(..., 6, '0')` sí trunca a 6 caracteres, descartando
el dígito sobrante. Ambas coinciden exactamente por debajo de 1.000.000; divergen a partir de
ahí, matemáticamente garantizado.
- **CÓDIGO**: `invoices/schemas.py:40-43` (Python); `sales/service.py:142-172` (SQL, con
  comentario propio citando `Invoice.full_number` como fuente de verdad — intención expresa de
  mantenerlas sincronizadas); reglas RN-FACT-05 y RN-FACT-07 (DUDOSA,
  `reglas-de-negocio.md:1600-1616`); `contradiccion-04-formato-numero-factura-python-vs-sql.md`
  completo, con la prueba determinista del caso `1234567` → `"FAC-123456"` no encontrado.
- **DATOS**: ninguno — pero a diferencia de otras entradas de este registro, aquí el propio
  documento de contradicción es una **prueba matemática cerrada**, no una suposición: no
  depende de ninguna condición de carrera ni de datos reales para confirmarse.
- Nota sobre RN-FACT-06 (DUDOSA en origen: "cobertura parcial verificada" de la corrección de
  '20 ventas reales, cero facturas'): este propio documento, al confirmar en A-01/A-07 que los
  cuatro caminos de cobro (mostrador, unificado, dividido, `pay_order` legado) usan
  `build_sale`, cierra esa duda salvo por un quinto camino no documentado — se considera
  **resuelta favorablemente**, no se lista como anomalía propia.
**Clasificación**: **BUG A SECAS** (matemáticamente cierto en el código actual, sin
condicionalidad; no requiere testigo de negocio para confirmarse como defecto, solo para
decidir el tratamiento).
**Depende de esto**: nadie hoy — es "matemáticamente imposible" que se manifieste antes de que
un tenant emita un millón de facturas bajo el mismo prefijo (un negocio de heladería individual
está lejos de ese volumen); es una bomba de tiempo, no un incidente activo.
**Tratamiento acordado**: corregir en fase de modernización, con dos opciones no excluyentes a
decidir con negocio: (a) ampliar el ancho del padding a 7+ dígitos en ambos lados, o (b)
definir una política de reinicio periódico del consecutivo. **No retroactivo** en el sentido
de que los números ya emitidos con 6 dígidos o menos no cambian; sí requiere migrar la lógica
de búsqueda antes de que el primer tenant cruce el millón.
**Decisión de negocio pendiente**: ¿existe algún plan de reiniciar el consecutivo de
facturación periódicamente (anual, por resolución DIAN), o se espera que crezca indefinidamente
bajo el mismo prefijo? (`contradiccion-04...md`). Responsable: por identificar.

### A-14a — [PROTEGIDA] Toda venta pagada emite automáticamente su factura, en la misma transacción de base de datos
**Descripción**: no existe botón separado para facturar: al confirmarse el pago, la factura se
emite dentro de la misma transacción que crea la venta; el rollback deshace venta y factura
juntas. Corrige un problema histórico documentado explícitamente en el código.
- **CÓDIGO**: `sales/builder.py:174-179`; `sales/service.py:106-137`;
  `invoices/service.py:1-11,45-53`; regla RN-FACT-01 (INTENCIONAL,
  `reglas-de-negocio.md:1574-1579`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #6 (2026-07-29, commit `27711065`,
  Deimer Hernandez): "hay cuatro formas de cobrar... emitir por separado en cada una
  garantizaba que alguna se quedara fuera — que es exactamente lo que pasaba".
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos: CÓDIGO + NEGOCIO histórico).
**Depende de esto**: los cuatro caminos de cobro (mostrador, unificado, dividido, `pay_order`
legado) y, por extensión, toda la contabilidad/gestoría que depende de que cada venta pagada
tenga su factura.
**Tratamiento acordado**: **especificar tal cual, no tocar** — "una factura sin su venta, o al
revés, sería peor que no tener factura" (razón explícita en el propio código).
**Decisión de negocio pendiente**: el comentario del código da a entender que **ya ocurrieron
facturas faltantes en producción** antes de este rediseño (las "20 ventas reales, cero
facturas" citadas en RN-FACT-06). ¿Hubo un reclamo o auditoría real que lo disparó? Y el
`pay_order` legado que menciona el propio docstring: ¿sigue en uso hoy, más allá de estar sin
caller conocido en la UI actual (ver A-01 y H2/H3 de `flujo-pedido-qr.md`)?
(memoria-historica.md #6). Responsable: por identificar.

---

## Nivel 2 — Impacto medio

### A-15 — [PROTEGIDA] Cuatro huecos de seguridad cerrados en el cobro dividido
**Descripción**: antes de dar a los cajeros la capacidad de armar bloques de pago manualmente,
se cerraron cuatro huecos "desde que se implementó el split" pero difíciles de alcanzar hasta
entonces: comensales repetidos causaban doble cobro/doble factura; montos en la raíz con
`billing_mode='split'` se ignoraban en silencio (el cajero perdía la propina); importes
negativos no se validaban; el bloque sin comensal salía sin nombre en la factura.
- **CÓDIGO**: `table_sessions/service.py:590-632` (RN-MESA-11, RN-MESA-12, ambas INTENCIONAL,
  `reglas-de-negocio.md:920-930`); test de blindaje `app/scripts/test_split_blindaje.py`.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #11 (2026-08-04, commit `42b5dec3`,
  Deimer Hernandez).
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos).
**Depende de esto**: la integridad de cualquier cobro dividido con más de un comensal.
**Tratamiento acordado**: **especificar tal cual, no tocar** — estos cuatro chequeos deben
convertirse en casos de test explícitos de la especificación formal, precisamente porque ya
fueron huecos reales.
**Decisión de negocio pendiente**: ¿la capacidad de armar bloques manuales ya está en
producción? Si es así, ¿desde cuándo, y coincide con la fecha de este endurecimiento
(2026-08-04), o quedó una ventana abierta entre que se dio la capacidad y que se blindó?
(memoria-historica.md #11). Responsable: por identificar.

### A-16 — Cocina puede mutar y anular ítems sin validar el estado del pedido padre
**Descripción**: `transition_kitchen` y `void_item` no comprueban el `status` de la
`CustomerOrder`; funcionan igual aunque la orden esté `pagada`, `cancelada` o `bloqueada`
(congelada para cobro). `mark_order_ready` sí valida el status de la orden, pero solo la
bloquea en estados terminales de pago, no en `bloqueada`.
- **CÓDIGO**: `orders/kitchen.py:43-60` (`transition_kitchen`, sin validación); `:93-176`
  (`void_item`, sin validación); `:63-90` (`mark_order_ready`, valida parcialmente); reglas
  RN-ORD-37 (DUDOSA), RN-ORD-38 y RN-ORD-39 (ambas ACCIDENTAL,
  `reglas-de-negocio.md:1314-1330`); registro de riesgos R1 (mismo patrón: sin
  `with_for_update`, contrastado explícitamente con `checkout.confirm_order` que sí lo usa).
**Clasificación**: **ACCIDENTAL** (RN-ORD-38/39, inconsistencia directa frente al cuidado
explícito de `mark_order_ready` en el mismo módulo) + **PENDIENTE** (RN-ORD-37, sin
confirmación de si permitir avanzar cocina sobre una orden `bloqueada` es deseado).
**Pregunta que desbloquearía RN-ORD-37**: ¿es deliberado que `mark_order_ready` permita
terminar de preparar ítems de una orden ya `bloqueada` para cobro? (reglas-de-negocio.md,
pregunta abierta #21).
**Depende de esto**: la consistencia entre lo que se cobró y lo que cocina reporta; sin lock de
fila (R1), una anulación y un "listo" casi simultáneos sobre el mismo ítem pueden pisarse y
descuadrar el kardex sin error visible.
**Tratamiento acordado**: corregir en fase de modernización — replicar en `transition_kitchen`
y `void_item` la misma validación de status que ya tiene `mark_order_ready`, y añadir
`with_for_update` consistente con `checkout.confirm_order`. Sin riesgo retroactivo relevante.
**Decisión de negocio pendiente**: la de RN-ORD-37 (arriba).

### A-17 — Operaciones concurrentes sin lock de fila en caja y mesas
**Descripción**: varios puntos del sistema no bloquean la fila antes de leer-y-decidir, a
diferencia del resto del código (que sí usa `SELECT...FOR UPDATE` de forma consistente en
inventario, facturación y cierre de sesión de mesa): `close_shift` y `add_movement` (caja),
`add_participant`/`remove_participant`/`set_assignments` (reparto de mesa, mientras que
`close_session` sí bloquea), y `_get_or_create_table_session` (apertura de sesión QR).
- **CÓDIGO**: `cash/router.py:90-124` (`close_shift`) y `:190-203` (`add_movement`) — registro
  de riesgos R7 y R22; `table_sessions/service.py:38-55,228,335,370,403` — R12 y regla
  RN-MESA-02 (DUDOSA, `reglas-de-negocio.md:864-869`, mismo hallazgo); `cart/service.py:58-74`
  con manejo de excepción genérico en `:93-123` — R16.
**Clasificación**: **PENDIENTE** (RN-MESA-02/R12, con pregunta de negocio explícita sobre si
algún mecanismo externo serializa esto en la práctica) + **ACCIDENTAL** (R7, R16, R22 — sin
comentario que justifique la ausencia de lock frente al resto del código, que sí lo tiene
consistentemente).
**Pregunta que desbloquearía RN-MESA-02**: ¿existe algún mecanismo externo (cliente restringido
a una pantalla por mesa, o similar) que en la práctica evite que un reparto de cuenta
concurrente con un cierre de sesión en curso lea datos a medio comitear? (reglas-de-negocio.md,
pregunta abierta #15).
**Depende de esto**: la integridad del arqueo de caja (R7, doble clic en cierre de turno
duplica denominaciones), del reparto de cuenta (R12/RN-MESA-02) y de la apertura de sesión QR
(R16, dos comensales escaneando la misma mesa a la vez reciben un 500 genérico en vez de un
409 controlado).
**Tratamiento acordado**: corregir en fase de modernización — añadir `with_for_update()`
consistente con el patrón ya establecido en `inventory/stock.py`, `invoices/service.py` y
`table_sessions.close_session`. Sin riesgo retroactivo (son condiciones de carrera, no datos
almacenados incorrectamente por sí solas).
**Decisión de negocio pendiente**: la de RN-MESA-02 (arriba).

### A-18 — El rol legado `STAFF` se remapeó silenciosamente a `CASHIER`
**Descripción**: `STAFF` —"una invención del frontend, el backend solo emite `ADMIN` y
`CASHIER`"— quedó marcado como legado y se mapea a `CASHIER` para no invalidar sesiones JWT
vivas. Su pantalla de inicio era el tablero de cocina (KDS), deprecado poco después.
- **CÓDIGO**: `pos-heladeria/src/app/core/interfaces/user.interface.ts:22-30`.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #13 (2026-08-07, commits `9510479` y
  `9d2e4bc8`, Deimer Hernandez); `STAFF` se introdujo en `9510479` (2026-05-21).
**Clasificación**: **PENDIENTE** — hay CÓDIGO y NEGOCIO histórico sobre el mecanismo, pero
falta DATOS: no se sabe si algún usuario real tuvo rol `STAFF` entre mayo y agosto, ni si el
remapeo a `CASHIER` le dio o quitó permisos sin que nadie lo notara. Sin ese dato, no puede
cerrarse si el remapeo fue inofensivo o un incidente de control de acceso silencioso.
**Pregunta que la desbloquearía**: ¿hubo algún usuario con rol `STAFF` en producción entre
mayo y agosto de 2026? Si el mapeo automático a `CASHIER` le dio permisos que antes no tenía
(o le quitó los que sí tenía), ¿alguien lo notó? (memoria-historica.md #13). Requiere una
consulta directa a la tabla de usuarios (candidata a arqueología de datos, hoy inexistente).
**Depende de esto**: cualquier cuenta creada con rol `STAFF` entre 2026-05-21 y 2026-08-07 que
siga activa.
**Tratamiento acordado**: documentar sin especificar hasta tener el dato. Si se confirma que
hubo cuentas `STAFF` con permisos ampliados o reducidos sin aviso, esto pasa a ser un hallazgo
de seguridad a tratar con prioridad alta, no solo de modernización.

### A-19 — `.env` real define `EMAIL_API_URL` dos veces, con valores de desarrollo y producción incompatibles
**Descripción**: `pos-backend/.env:13` (`http://localhost:8080`) y `:32`
(`https://app.skeilopos.com`) definen la misma variable con valores distintos en el mismo
fichero; `python-dotenv`/Pydantic-settings toma la última ocurrencia, así que el valor
efectivo depende del orden del fichero. `.env.example` no tiene esta duplicación —
específico de este entorno, no del repositorio.
- **CÓDIGO**: `mapa-sistema.md`, sección 6, hallazgo 4 (verificado directamente en el
  `.env` real, no en `.env.example`); consumo real: `app/core/mail.py:19-38`.
**Clasificación**: **ACCIDENTAL** (verificable directamente en el fichero, sin ambigüedad de
intención).
**Depende de esto**: cualquier correo transaccional (bienvenida de tenant, etc.) enviado desde
este entorno concreto — un reordenamiento futuro del `.env` cambiaría en silencio a qué API de
correo apunta el sistema.
**Tratamiento acordado**: corregir de inmediato (no requiere fase de modernización, es
higiene de configuración): eliminar la entrada duplicada del `.env` real, dejando solo el
valor que corresponda al entorno donde vive ese fichero. Sin riesgo retroactivo.
**Decisión de negocio pendiente**: ninguna — es una corrección operativa, no de producto.
Nota: relacionado con el hotfix ya conocido de `memoria-historica.md` (`8efe0fb`,
"hotfix/fix-email", URL de login mal construida en el correo de bienvenida) — no se sabe si
están relacionados, pero comparten el mismo subsistema de correo.

### A-20 — Cierre de caja sin conteo físico obligatorio; arqueo parcial no exige observación; el histórico se recalcula, no es una foto fija
**Descripción**: tres reglas relacionadas del módulo de caja: (1) `close_shift` puede aceptar
un `counted_amount` recibido directamente sin denominaciones, o incluso ninguno de los dos,
dejando el conteo en `None`; (2) `partial-count` (arqueo sin cerrar turno) no exige
`close_note` aunque la diferencia sea distinta de cero, a diferencia del cierre real que sí la
exige; (3) el histórico de turnos siempre recalcula `expected`/`difference` en el momento de
la consulta usando datos actuales, no una foto fija del cierre — si después del cierre se
anula una venta que formaba parte del cálculo, el histórico cambia retroactivamente.
- **CÓDIGO**: `cash/router.py:96-111,133-151`; `cash/service.py:56-57,76-182`; reglas
  RN-CASH-09, RN-CASH-13, RN-CASH-17 (las tres DUDOSA, `reglas-de-negocio.md:618-670`).
**Clasificación**: **PENDIENTE** (las tres) — todas dependen de si el frontend compensa lo que
el backend no exige, algo que no se puede confirmar sin ver el frontend en ejecución o sin
testimonio del negocio.
**Pregunta que las desbloquearía**: (RN-CASH-09) ¿el frontend impide cerrar un turno sin haber
enviado `counted_amount` o denominaciones? (RN-CASH-13) ¿es intencional la asimetría entre
arqueo parcial y cierre real? (RN-CASH-17) ¿existe algún mecanismo que congele el arqueo al
cerrar, o el histórico puede cambiar retroactivamente? (reglas-de-negocio.md, preguntas
abiertas #12-14).
**Depende de esto**: la gestoría, que según la Constitución del proyecto usa el conteo de
efectivo reportado por el arqueo (citado también en registro-riesgos.md R7).
**Tratamiento acordado**: documentar sin especificar hasta la respuesta del negocio.

### A-21 — Diseño documentado de cookie `httpOnly` para el comensal nunca se implementó; `localStorage` es el almacén real, agravado por vulnerabilidades XSS conocidas en Angular
**Descripción**: el frontend reconoce por escrito que el token de sesión del comensal (flujo
QR) debía viajar en una cookie `httpOnly` según "el documento de contrato", pero el backend
nunca llegó a emitir esa cookie — el token vive en `localStorage`, legible por cualquier
script que corra en la página. Lo mismo aplica al JWT de sesión del personal (access y
refresh). `@angular/core` 21.2.14 tiene 6 vulnerabilidades "high" reportadas por
`npm audit`, incluido bypass de sanitización XSS.
- **CÓDIGO**: `pos-heladeria/src/app/modules/tables/services/diner-token.store.ts:15-18`
  (comentario propio); `token-storage.service.ts:12-24`; `package.json:14-18`; salida de
  `npm audit --omit=dev`; registro de riesgos R3, R9, R11.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #5 (2026-07-28, commit `46ad3eda`,
  Deimer Hernandez).
**Clasificación**: **PENDIENTE** — hay CÓDIGO y NEGOCIO histórico sobre el hecho de que
`localStorage` es un sustituto no planeado, pero falta confirmar si sigue siendo un plan
pendiente o ya es la decisión definitiva.
**Pregunta que la desbloquearía**: ¿la cookie `httpOnly` sigue siendo el diseño que se quiere,
o `localStorage` se adoptó ya como decisión definitiva? (memoria-historica.md #5).
**Depende de esto**: cualquier sesión de comensal o de personal activa; el riesgo real depende
de que exista un XSS explotable, lo cual R3 documenta como plausible dadas las vulnerabilidades
reportadas por `npm audit` en la versión actual de Angular.
**Tratamiento acordado**: documentar sin especificar la decisión de arquitectura de
almacenamiento del token hasta la respuesta del negocio; corregir de inmediato, sin embargo,
la actualización de `@angular/core` y paquetes relacionados a una versión sin las
vulnerabilidades reportadas — eso no depende de ninguna decisión de producto.

### A-22 — Endurecimientos de autenticación pendientes: sin rate-limit de login, el refresh sobrevive al logout, contraseñas truncadas a 72 bytes sin validar longitud
**Descripción**: (1) `POST /auth/login` no tiene ningún contador de intentos fallidos ni
bloqueo temporal — cualquier número de intentos con credenciales incorrectas responde igual;
(2) el logout revoca solo el `jti` del access token vía blocklist, pero el refresh token sigue
siendo válido; (3) las contraseñas se truncan a 72 bytes (límite de bcrypt) sin validación de
longitud máxima visible en el schema de cambio de contraseña.
- **CÓDIGO**: `auth/routes.py:22-89` (login, sin lógica de intentos; `app/core/rate_limit.py`
  existe pero no se importa en `auth/routes.py`), `:152-160` (logout/blocklist);
  `app/core/utils.py:14-25` (truncamiento bcrypt); reglas RN-AUTH-03 (ACCIDENTAL), RN-AUTH-08
  y RN-AUTH-09 (ambas DUDOSA, `reglas-de-negocio.md:61-108`).
**Clasificación**: **ACCIDENTAL** (las tres reglas: RN-AUTH-03 — ausencia de bloqueo por fuerza
bruta, sin ningún comentario que la declare intencional; RN-AUTH-08 y RN-AUTH-09 cerradas en
ronda 3, simulada, ver `entrevista-negocio.md` §8 P32: ninguna de las tres omisiones es
deliberada — no hay rate-limit de infraestructura, el refresh sobreviviendo al logout no fue una
decisión a propósito, y el formulario de cambio de contraseña no valida el límite de 72 bytes).
**Depende de esto**: la resistencia del sistema a ataques de fuerza bruta sobre credenciales de
personal (admin/cajero), que tienen acceso a caja e inventario.
**Tratamiento acordado**: corregir en fase de modernización — añadir rate-limiting al login
(mismo mecanismo ya usado en `menu`), revocar el refresh junto al access en logout, y validar la
longitud máxima de contraseña acorde a 72 bytes. Sin riesgo retroactivo, sin reserva pendiente.

### A-23 — [PROTEGIDA] El refresh token relee el usuario en base de datos y exige que siga activo
**Descripción**: al renovar el access token, no se reutilizan los datos codificados en el
refresh: se vuelve a consultar el usuario por id y se exige `active==True`, para que una
cuenta desactivada o con rol cambiado no siga emitiendo access tokens válidos con datos
obsoletos durante toda la vida del refresh (hasta 7 días).
- **CÓDIGO**: `auth/routes.py:111-150,126-129`; regla RN-AUTH-07 (INTENCIONAL,
  `reglas-de-negocio.md:89-94`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #4 (2026-07-28, commit `5c1db9ed`,
  Deimer Hernandez).
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos).
**Depende de esto**: la seguridad de cualquier cuenta desactivada o con cambio de rol mientras
su refresh token sigue vigente.
**Tratamiento acordado**: **especificar tal cual, no tocar**.
**Decisión de negocio pendiente**: ¿este cambio respondió a un caso real (un empleado
desvinculado que siguió con acceso)? (memoria-historica.md #4) — no bloquea la especificación,
es solo una pregunta forense.

### A-24 — [PROTEGIDA] El tenant y la mesa en el flujo QR siempre vienen del token firmado, nunca de un id que mande el cliente
**Descripción**: se eliminaron a propósito dos endpoints legacy (`POST /sessions` con UUID
plano falsificable, `GET /menu/qr/{qr_token}` con `table_id` editable por el cliente),
reemplazados por endpoints donde tenant y mesa viajan firmados dentro del token.
- **CÓDIGO**: `cart/router.py:1-6`; `core/qr_context.py:1-6`; regla RN-CART-26 (INTENCIONAL,
  `reglas-de-negocio.md:837-841`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #2 (2026-07-28, commit `5c1db9ed`,
  Deimer Hernandez).
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos).
**Depende de esto**: la integridad de todo el flujo QR — un id de mesa o tenant falsificable
permitiría a un comensal ver/pedir en una mesa o tenant ajeno.
**Tratamiento acordado**: **especificar tal cual, no tocar**.
**Decisión de negocio pendiente**: ¿quedó algún cliente (app vieja en un dispositivo, enlace
impreso) que todavía apunte a las rutas eliminadas? (memoria-historica.md #2) — forense, no
bloqueante.

### A-25 — [PROTEGIDA] Se retiró a propósito el `PATCH` genérico de status de pedido
**Descripción**: `PATCH /{order_id}/status` permitía asignar cualquier estado a un pedido sin
validar la transición ni tocar inventario — un pedido podía pasar de `recibida` a `abierta`
esquivando `confirm_order` (el único punto que descuenta stock), dejando el inventario
sobrestimado sin que nadie se enterara. Se retiró y cada transición legítima quedó con su
propio endpoint.
- **CÓDIGO**: `orders/router.py:443-452`; regla RN-ORD-65 (INTENCIONAL,
  `reglas-de-negocio.md:1461-1465`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #3 (2026-07-28, commit `5c1db9ed`,
  Deimer Hernandez).
**Clasificación**: **INTENCIONAL [PROTEGIDA]** (2/3 testigos).
**Depende de esto**: la exactitud del inventario de cualquier pedido QR — reintroducir una vía
de transición libre de status reproduciría exactamente el bug ya corregido.
**Tratamiento acordado**: **especificar tal cual, no tocar**. La especificación formal debe
declarar explícitamente "no existe transición libre de status" como invariante de diseño, no
solo como ausencia de funcionalidad.
**Decisión de negocio pendiente**: ¿se detectó este patrón en datos reales antes de retirarlo,
o fue revisión preventiva? Si fue real, ¿cuánto inventario quedó descuadrado?
(memoria-historica.md #3) — forense, no bloqueante para la especificación futura.

### A-26 — Mesas avanzadas: `move_order` más estricto que el modelo general, manejador de error huérfano, y `merge_orders` no determinista
**Descripción**: tres hallazgos del mismo módulo (`tables_advanced.py`, "mover, fusionar y
agrupar mesas"): (1) `move_order` exige que la mesa destino esté completamente sin órdenes
activas, más estricto que el modelo general que sí permite varias órdenes por mesa; (2)
captura un `IntegrityError` que traduce a 409, pero el modelo documenta que ya no existe
ningún índice único que lo dispare — manejador huérfano de una constraint removida; (3)
`merge_orders`, si las órdenes a fusionar ya pertenecían a grupos distintos preexistentes,
conserva el primero según un `SELECT` sin `ORDER BY` (no determinista), dejando órdenes
huérfanas de su grupo original.
- **CÓDIGO**: `orders/tables_advanced.py:53-54` (RN-ORD-58, DUDOSA), `:59-63` (RN-ORD-60,
  ACCIDENTAL), `:75-89` (RN-ORD-63, ACCIDENTAL) — `reglas-de-negocio.md:1421-1452`.
**Clasificación**: **PENDIENTE** (RN-ORD-58) + **ACCIDENTAL** (RN-ORD-60, RN-ORD-63).
**Pregunta que desbloquearía RN-ORD-58**: ¿es intencional que `move_order` sea más estricto que
el modelo general de "varias órdenes por mesa", o quedó desalineado tras introducir esa
capacidad? (reglas-de-negocio.md, pregunta abierta #23).
**Depende de esto**: cualquier negocio que use mover/fusionar mesas con grupos preexistentes en
colisión — el resultado depende del orden no determinista de una consulta SQL, un
comportamiento potencialmente distinto en cada ejecución.
**Tratamiento acordado**: corregir en fase de modernización el manejador huérfano (retirarlo o
documentar por qué se conserva) y la no-determinismo de `merge_orders` (definir una regla
explícita de qué grupo sobrevive, p. ej. el de menor `created_at`). Documentar sin especificar
la restricción de `move_order` hasta la respuesta de negocio.

### A-27 — El CI del backend solo ejecuta 1 de 12 scripts de test — ya se dejó pasar un test roto real en el frontend
**Descripción**: el pipeline de CI instala solo 4 de los ~80 paquetes de `requirements.txt` y
ejecuta un único script, `test_promotions_rules.py`. Los otros 11 scripts `test_*.py`
—incluido `test_session_ttl.py`, que contiene el `assert` del invariante de A-30— no corren en
CI ni en ningún otro paso automatizado. En el frontend, el spec de `ReportsService` llevó roto
desde que el módulo de reportes migró a `/reports/*` hasta que alguien lo notó y reescribió —
mismo patrón, confirmado ya ocurrido una vez.
- **CÓDIGO**: `.github/workflows/deploy.yml:14-22`; scripts no ejecutados listados en
  `registro-riesgos.md` R6 y `mapa-sistema.md` sección 6, hallazgo 2.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #16 (2026-08-10, commit `33220db4`,
  Deimer Hernandez, sobre el spec roto de `ReportsService`).
**Clasificación**: **ACCIDENTAL** (verificable directamente en el workflow; agravado por un
precedente real ya documentado).
**Depende de esto**: ninguno de los hallazgos de este registro que ya tiene un test escrito
para su comportamiento (notablemente A-30/R14) se detectaría automáticamente si una futura
edición lo rompe — el negocio depende de que alguien lo corra a mano.
**Tratamiento acordado**: corregir en fase de modernización — incorporar al menos los scripts
que cubren comportamiento ya documentado como crítico en este registro
(`test_session_ttl.py`, `test_split_blindaje.py`, `test_cancel_inventory.py`,
`test_receta_obligatoria.py`) al pipeline de CI. Sin riesgo retroactivo.
**Decisión de negocio pendiente**: ¿desde cuándo corre CI en el frontend? Si corría, ¿por qué
el spec roto no se detectó hasta que alguien lo notó manualmente? (memoria-historica.md #16).

### A-28 — El invariante `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` no se valida en el arranque
**Descripción**: si no se cumple este invariante, el barrido de sesiones cierra mesas
**activas** sin pedidos — lo dice un comentario extenso en `config.py`, pero no hay ningún
`assert`/validador de Pydantic que lo garantice. El único chequeo existente vive en
`test_session_ttl.py`, que no corre en CI (ver A-27).
- **CÓDIGO**: `app/core/config.py:36-44` (comentario del invariante);
  `app/core/qr_context.py:121` (uso real); `app/scripts/test_session_ttl.py:39-44` (único
  chequeo); registro de riesgos R14.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #7 (2026-08-01, commit `87e6282c`,
  Deimer Hernandez): la constante se introdujo como optimización (de ~360 escrituras/hora a
  ~6) pero deja este invariante frágil.
**Clasificación**: **ACCIDENTAL** — hay CÓDIGO y NEGOCIO histórico sobre el mecanismo y su
fragilidad, pero la falta de un `assert` en el arranque es un defecto verificable por código
solo, sin necesidad de más testigos para clasificarlo como tal (la pregunta de negocio abajo
es forense, no de clasificación).
**Depende de esto**: cualquier mesa activa en pleno servicio, si alguien cambia estos dos
valores en `.env` sin conocer el invariante.
**Tratamiento acordado**: corregir en fase de modernización — añadir validación en el arranque
(Pydantic `model_validator` o equivalente) que rechace una configuración que viole el
invariante. Sin riesgo retroactivo.
**Decisión de negocio pendiente**: ¿alguna vez se tocó uno de estos dos valores en `.env` de
producción sin conocer el invariante? (memoria-historica.md #7) — forense.

### A-29 — Con dos o más combos distintos en una misma venta, no queda registrado ningún combo en `promotion_id`
**Descripción**: el mismo mecanismo se repite en los tres caminos de cobro (mesa unificada,
orden de mesa, mostrador): si las líneas cobradas usan exactamente un combo distinto,
`promotion_id` = ese combo; con cero o más de uno, se pierde la referencia (aunque el
descuento monetario de todos se sume correctamente al total — solo se pierde la trazabilidad
por promoción en reportes).
- **CÓDIGO**: `table_sessions/service.py:555-559` (RN-MESA-15); `orders/checkout.py:268-269`
  (RN-ORD-08); `sales/service.py:104` (RN-VENTA-14) — las tres DUDOSA,
  `reglas-de-negocio.md:944-948,1137-1141,1548-1551`.
**Clasificación**: **PENDIENTE** — el descuento económico es correcto; lo que falta confirmar
es si la pérdida de trazabilidad en reportes por promoción es aceptable.
**Pregunta que la desbloquearía**: cuando una venta de mesa usa dos o más combos distintos,
¿es aceptable para el negocio que la venta no registre ningún combo en `promotion_id`?
(reglas-de-negocio.md, pregunta abierta #18).
**Depende de esto**: cualquier reporte que agrupe ventas por promoción/combo específico.
**Tratamiento acordado**: documentar sin especificar hasta la respuesta del negocio.

### A-30 — `PATCH` de promociones puede disparar un `IntegrityError` no controlado
**Descripción**: dos vectores distintos del mismo módulo: (1) `PATCH {"name": null}` pasa la
validación de Pydantic (que no aplica `min_length` cuando el valor es `None`) y el chequeo de
unicidad del router (que solo valida "si `name is not None`"), llegando a `commit()` con una
columna `NOT NULL`; (2) los `targets` repetidos (mismo producto/categoría en la misma
promoción) solo están protegidos por un índice único parcial en PostgreSQL, sin validador en
Pydantic ni en el servicio.
- **CÓDIGO**: `promotions/schemas.py:187`; `promotions/router.py:72-74`;
  `promotions/service.py:572-575,533-540`; `promotions/model.py:57,166-173`; reglas
  RN-PROMO-75 (ACCIDENTAL) y RN-PROMO-76 (DUDOSA), `reglas-de-negocio.md:2001-2013`.
**Clasificación**: **ACCIDENTAL** (RN-PROMO-75 — el propio código ya documenta haber corregido
un bug equivalente para la unicidad del nombre, pero no cubrió este caso) + **PENDIENTE**
(RN-PROMO-76, depende de si existe un manejo genérico de `IntegrityError` fuera de estos
ficheros que no se verificó).
**Pregunta que desbloquearía RN-PROMO-76**: ¿existe algún manejo genérico de `IntegrityError`
que convierta un target duplicado en un 409 legible, en vez de un error no controlado?
(reglas-de-negocio.md, pregunta abierta #37).
**Depende de esto**: cualquier admin que edite una promoción vía PATCH — el resultado hoy es
previsiblemente un 500 sin mensaje útil, no un 409 accionable.
**Tratamiento acordado**: corregir en fase de modernización — validar explícitamente ambos
casos en la capa de servicio, con mensajes de negocio dedicados. Sin riesgo retroactivo.

---

## Nivel 3 — Impacto bajo

### A-31 — `core/units.py` depende de columnas inexistentes en `UnitMeasure` y no se usa en ningún lugar del código
**Descripción**: `convert()` accede a `from_unit.dimension`/`factor_to_base`, columnas que el
modelo real de `UnitMeasure` no define (solo `name`, `abbreviation`, `active`). Ninguna
búsqueda en el repositorio encuentra un import o uso de esta función — es código muerto de una
migración "Fase 1" nunca completada.
- **CÓDIGO**: `app/core/units.py:14-29` vs `app/models/unit_measure.py:1-18`; `grep` sin
  resultados de uso; reglas RN-CAT-40 (DUDOSA) y RN-CAT-41 (ACCIDENTAL,
  "HALLAZGO CRÍTICO / CÓDIGO MUERTO", `reglas-de-negocio.md:390-402`).
**Clasificación**: **ACCIDENTAL** (verificable por código solo: invocarlo con datos reales
lanzaría `AttributeError`, no el 422 que el propio módulo documenta).
**Depende de esto**: nadie identificado.
**Tratamiento acordado**: corregir en fase de modernización — completar la migración de esquema
(agregar `dimension`/`factor_to_base` a `UnitMeasure`) **solo para conversión dentro de la misma
dimensión (volumen)**, el único caso real confirmado (litros↔onzas del granizado, P19). Sin
riesgo retroactivo por ser código inerte.
**Decisión de negocio pendiente**: cerrada en la primera ronda (P19): sí es necesaria (caso real
de litros↔onzas). El alcance —¿también entre dimensiones distintas (g↔ml)?— quedó cerrado en la
ronda 3 (simulada, ver `entrevista-negocio.md` §8, P31): **no**, sin caso de uso real hoy; queda
fuera de alcance de la migración.

### A-32 — "¿Este grupo de opciones descuenta inventario?" tiene dos criterios distintos según qué función pregunte
**Descripción**: `grupos_que_descuentan` (validación de selección al alta) cuenta un grupo como
"descuenta" con solo `item_quantity>0`, sin exigir `active` ni `inventory_item_id`;
`group_discounts` (confirmación de venta) exige las tres condiciones. Una opción con cantidad
puesta pero sin insumo enlazado (catálogo a medio terminar) es "sí descuenta" para una función
y "no descuenta" para la otra, produciendo el mensaje de error equivocado
("no tiene receta configurada") cuando el problema real es un insumo no enlazado.
- **CÓDIGO**: `catalog/line_pricing.py:69-91` vs `catalog/consumption_plan.py:79-95`; regla
  RN-CAT-39 (DUDOSA, `reglas-de-negocio.md:383-388`);
  `contradiccion-02-criterio-grupo-descuenta-inventario.md` completo.
- **DATOS**: ninguno — pregunta abierta explícita: requiere una consulta directa a la base de
  datos, fuera del alcance de lectura de código.
**Clasificación**: **PENDIENTE** — coinciden siempre que el catálogo esté bien formado; la
propia `contradiccion-02...md` señala que confirmar el alcance real requiere una consulta que
hoy no existe.
**Pregunta que la desbloquearía**: ¿existen hoy en el catálogo opciones con `item_quantity`
puesto pero sin insumo de inventario enlazado (o inactivas) que sigan asignadas a algún grupo
de alguna variante vendible? (candidata directa a arqueología de datos).
**Depende de esto**: cualquier admin que reciba el mensaje de error equivocado al intentar
vender una variante con esta configuración incompleta.
**Tratamiento acordado**: documentar sin especificar hasta tener el dato; si se confirma que
existen casos así, unificar el criterio en fase de modernización (sin retroactividad, es
lógica de validación, no dato almacenado).

### A-33 — Un grupo opcional que descuenta inventario bloquea la venta si nadie elige nada, pese a que el propio código llama a eso "una decisión legítima"
**Descripción**: `validate_option_selection` permite explícitamente no elegir nada de un grupo
opcional (`min_select=0`); pero si ese grupo opcional es la única fuente de consumo,
`ensure_lines_consume_inventory` bloquea la venta con 409 — el comentario que justifica la
primera función no cubre este caso.
- **CÓDIGO**: `catalog/consumption_plan.py:174-179` (comentario) vs `:188-198,214-226`
  (lógica real); regla RN-CAT-35 (DUDOSA, `reglas-de-negocio.md:356-361`).
**Clasificación**: **PENDIENTE**.
**Pregunta que la desbloquearía**: cuando un grupo de opciones opcional es la única fuente de
consumo de inventario de una variante, ¿debería bloquearse la venta si no se elige nada, o el
bloqueo contradice la intención de "grupo opcional"? (reglas-de-negocio.md, pregunta abierta
#7).
**Depende de esto**: cualquier variante configurada con un único grupo opcional como fuente de
consumo.
**Tratamiento acordado**: documentar sin especificar.

### A-34 — El mismo insumo repetido en una receta se rechaza, pero el `DELETE` de la receta anterior ya se ejecutó antes de detectar el duplicado
**Descripción**: guardar la receta de una variante borra la receta anterior y la reemplaza; si
el nuevo payload trae el mismo insumo repetido, se rechaza — pero el `DELETE` ya se ejecutó
dentro de la misma transacción sin commit previo, dependiendo del manejo de rollback (fuera
del alcance de este análisis) para no dejar la variante sin receta.
- **CÓDIGO**: `catalog/router.py:218-225`; regla RN-CAT-14 (DUDOSA,
  `reglas-de-negocio.md:213-218`).
**Clasificación**: **PENDIENTE** — depende del comportamiento de `app/core/db.py`, no
verificado en este documento.
**Pregunta que la desbloquearía**: ¿el rollback de la transacción de `set_recipe` está
garantizado ante un 422 lanzado a mitad de la función, de forma que el `DELETE` previo nunca
llegue a persistirse?
**Depende de esto**: cualquier intento de guardar una receta con un insumo repetido.
**Tratamiento acordado**: documentar sin especificar hasta verificar `app/core/db.py`.

### A-35 — Cluster de inventario: `allow_negative` sin llamador visible, motivo de ajuste no obligatorio, costo unitario siempre sobrescrito, ajuste con delta=0 termina en 500 no controlado
**Descripción**: cuatro hallazgos menores del mismo módulo: (1) `record_movement` acepta
`allow_negative=True` pero ningún llamador visible en `inventory` lo usa así; (2) el motivo de
un ajuste manual no es obligatorio en el backend; (3) toda compra/recepción sobrescribe el
costo unitario del insumo sin promediar (sin costo promedio ponderado); (4) un ajuste con
`signed_delta=0` se rechaza con `ValueError`, pero al no haber handler dedicado propaga como
500 genérico en vez de 400/422.
- **CÓDIGO**: `inventory/stock.py:49,72` (RN-INV-05, DUDOSA); `inventory/schemas.py:47-53` y
  `stock.py:97,117` (RN-INV-11, DUDOSA); `inventory/service.py:76,149-151` (RN-INV-17, DUDOSA);
  `inventory/stock.py:102-103` sin handler en `app/main.py` (RN-INV-10, ACCIDENTAL) —
  `reglas-de-negocio.md:436-480,515-520`.
**Clasificación**: **ACCIDENTAL** (RN-INV-10, verificable: ausencia de handler confirmada) +
**PENDIENTE** (RN-INV-05, RN-INV-11, RN-INV-17).
**Preguntas que desbloquearían lo pendiente**: ¿qué endpoints invocan `allow_negative=True`?
¿El frontend exige un motivo aunque el backend no lo obligue? ¿El negocio espera "último costo
de compra" o promedio ponderado? (reglas-de-negocio.md, preguntas abiertas #9-11).
**Depende de esto**: la exactitud del costo unitario reportado (RN-INV-17, con impacto directo
en margen reportado) y la trazabilidad de ajustes manuales sin motivo (RN-INV-11, relevante
para detectar mermas/fraude).
**Tratamiento acordado**: corregir de inmediato el 500 no controlado de RN-INV-10 (agregar
handler); documentar sin especificar el resto hasta respuesta de negocio.

### A-36 — Cluster de precisión y límites sin confirmar como contractuales
**Descripción**: cinco casos límite donde el comportamiento está definido en código pero sin
test o confirmación explícita de que sea el límite deseado: (1) no hay redondeo explícito en
el precio de línea de catálogo (`RN-CAT-02`); (2) la expiración de sesión de comensal usa `<=`
en vez de `<` en el instante exacto (`RN-CART-18`, resolución de microsegundos, casi
inobservable); (3) `starts_at` de una promoción se compara con datetime completo mientras
`ends_at` solo por fecha (`RN-PROMO-03`); (4) el solapamiento de horario entre dos promociones
solo se detecta con precisión si ambas definen `start_time` (`RN-PROMO-33`); (5) la ventana
horaria con cruce de medianoche no está cubierta por test en su segundo límite exacto
(`RN-PROMO-51`).
- **CÓDIGO**: `catalog/line_pricing.py:191-196`; `core/qr_context.py:181`;
  `promotions/service.py:71-99,462-466`; `test_promotions_rules.py:92-96` — todas DUDOSA en
  `reglas-de-negocio.md`.
**Clasificación**: **PENDIENTE** (las cinco) — ninguna tiene impacto económico demostrado hoy,
todas son casos límite de precisión sin confirmación de negocio o cobertura de test.
**Depende de esto**: nadie identificado con impacto visible hoy.
**Tratamiento acordado**: documentar sin especificar; candidatas naturales a convertirse en
casos de test explícitos si se decide activar los tests hoy no ejecutados (ver A-27).

### A-37 — Cluster de casos límite del motor de promociones
**Descripción**: seis hallazgos menores del cálculo de descuentos: (1) `qty_price` nunca
genera descuento negativo si el paquete configurado es más caro que lo normal, ocultando en
silencio una mala configuración (`RN-PROMO-11`); (2) lo mismo para combos, precio=`value×bundles`
sin validar contra el costo real de los componentes (`RN-PROMO-26`); (3) un descuento
calculado en `0` descalifica la promoción como candidata, sin distinguir "promo deshabilitada
a propósito" de "error de captura" (`RN-PROMO-15`); (4) la cantidad se trunca a entero, no se
redondea, en el cálculo de cantidad mínima (`RN-PROMO-19`); (5) un combo que deja de ser
vigente entre agregarlo al carrito y cobrar no avisa al cajero — simplemente no descuenta
(`RN-PROMO-27`); (6) reenviar el mismo estado de una promoción (incluido `finished→finished`)
es un no-op silencioso que no pasa por la tabla de transiciones (`RN-PROMO-41`/`RN-PROMO-54`);
(7) una promoción puede crearse directamente `active`/`paused` sin pasar por `draft`
(`RN-PROMO-68`).
- **CÓDIGO**: `promotions/service.py:159,265-267,242,405-414,652-667`; todas DUDOSA en
  `reglas-de-negocio.md:1680-1706,1765-1771,1830-1834,1897-1901,1968-1972`.
**Clasificación**: **PENDIENTE** (las siete).
**Preguntas que las desbloquearían**: reglas-de-negocio.md, preguntas abiertas #28-30, #32-33,
#35.
**Depende de esto**: la integridad del catálogo de promociones y, en el caso de `RN-PROMO-27`,
la confianza del cajero en lo que ve antes de cobrar.
**Tratamiento acordado**: documentar sin especificar hasta respuesta de negocio.

### A-38 — Cluster de mesas y pedidos menores
**Descripción**: cinco hallazgos de bajo impacto individual: (1) una mesa de un solo comensal
puede cerrarse en modo `split` sin restricción de mínimo, equivalente en la práctica a
`unified` disfrazado (`RN-MESA-13`); (2) no se puede quitar un comensal que ya tiene productos
asignados, aunque estén anulados o su pedido ya no sea cobrable (`RN-MESA-24`); (3) el cierre
de mesa en cascada no valida pendientes por sí mismo, delega esa responsabilidad al llamador
(`RN-ORD-31`); (4) la descripción de línea de venta ("Producto - Variante") puede quedar
incompleta si el producto o la variante fueron borrados (`RN-ORD-32`); (5) el docstring de
"único punto de descuento" de `confirm_order` es preciso solo si se lee como "único punto de
salida", no "único punto que toca inventario" — la propia reversa también escribe en el
kardex (`RN-ORD-34`, autoacknowledged como no-contradicción real, solo precisión de comentario).
- **CÓDIGO**: `table_sessions/service.py:578-671,362-388`; `orders/checkout.py:210-215,311,339,419,500-503`
  — todas DUDOSA en `reglas-de-negocio.md`.
**Clasificación**: **PENDIENTE** (RN-MESA-13, RN-MESA-24, RN-ORD-31, RN-ORD-32) +
observación documental sin clasificar (RN-ORD-34, no es una anomalía de comportamiento).
**Preguntas que desbloquearían lo pendiente**: reglas-de-negocio.md, preguntas abiertas #17,
#19, #22.
**Depende de esto**: bajo impacto individual — casos límite de uso operativo diario.
**Tratamiento acordado**: documentar sin especificar.

### A-39 — El job de medianoche que marca promociones vencidas compara en UTC absoluto, distinto del criterio de evaluación en tiempo real
**Descripción**: `_valid_now` (motor de evaluación, A-07) usa hora local del tenant y compara
`ends_at` solo por fecha; el job `expire_promotions` compara `ends_at < now` con
`now=datetime.now(timezone.utc)`, sin conversión y con datetime completo — puede marcar
`finished` una promoción que `_valid_now` todavía consideraría vigente, con un desfase de
hasta un día y medio en UTC-5.
- **CÓDIGO**: `core/scheduler.py:224-236` vs `promotions/service.py:91-99`; regla RN-SCHED-11
  (ACCIDENTAL, `reglas-de-negocio.md:1080-1085`).
**Clasificación**: **ACCIDENTAL** (contradicción directa verificable en código, aunque el
propio comentario del job se autodescribe como "puramente informativo").
**Depende de esto**: solo la etiqueta de estado que ve el admin en el listado de promociones
(`status=finished`); no afecta el cobro real, que sigue gobernado por `_valid_now`.
**Tratamiento acordado**: corregir en fase de modernización, unificando el criterio de corte
con `_valid_now`. Sin riesgo retroactivo.
**Decisión de negocio pendiente**: ¿debe unificarse el criterio, o el desfase es aceptable
porque el job es "puramente informativo"? (reglas-de-negocio.md, pregunta abierta #38).

### A-40 — El alias `cash_sales`, marcado deprecado, sigue expuesto en la API de caja
**Descripción**: `cash_sales` es exactamente `ventas_efectivo` (neto del cambio entregado),
mantenido solo por compatibilidad con el frontend — pero su nombre puede sugerir el bruto de
pagos en efectivo, no el neto.
- **CÓDIGO**: `cash/service.py:180-181`; `cash/schemas.py:112`; regla RN-CASH-15 (DUDOSA,
  `reglas-de-negocio.md:653-658`); `mapa-sistema.md` sección 6, hallazgo 7 ("deuda técnica
  documentada, no accidental").
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #17 (2026-07-18, commit `927a4606`,
  Deimer Hernandez).
**Clasificación**: **PENDIENTE** — hay CÓDIGO y NEGOCIO histórico sobre que es deprecado a
propósito, pero falta confirmar si el frontend ya migró y el alias puede retirarse.
**Pregunta que la desbloquearía**: ¿sigue el frontend leyendo `cash_sales` hoy, o ya migró a
`ventas_efectivo` y este alias se puede retirar? (memoria-historica.md #17).
**Depende de esto**: cualquier consumidor del frontend que aún lea el campo deprecado.
**Tratamiento acordado**: documentar sin especificar hasta confirmar uso real; retirar en fase
de modernización si se confirma que no hay consumidores.

### A-41 — Impuestos fijos en $0 sin confirmar si es una decisión fiscal definitiva
**Descripción**: se depreció a propósito el campo de impuestos editable en la terminal — el
total ya no calcula impuestos, siempre queda en 0.
- **CÓDIGO**: `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:497`
  (`const tax = 0`).
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #10 (2026-08-03, commit `8166ea9e`,
  **Leonardo Gomez** — commit message: "fix(tables): deprecate editable tax field in table
  terminal").
**Clasificación**: **INTENCIONAL** (2/3 testigos: CÓDIGO + NEGOCIO histórico — es
inequívocamente deliberado, no un descuido), con una decisión de negocio abierta sobre si es
la política fiscal correcta.
**Depende de esto**: cualquier cálculo de total que hoy asume impuesto en cero.
**Tratamiento acordado**: **especificar tal cual** el comportamiento actual (tax=0 fijo) como
punto de partida, pero sin cerrar la especificación de facturación fiscal hasta la respuesta
de negocio — es la única entrada `INTENCIONAL` de este registro con una decisión de negocio
explícitamente pendiente encima.
**Decisión de negocio pendiente**: ¿por qué se dejó de cobrar impuestos — es una decisión
fiscal (el negocio no discrimina IVA en el ticket) o quedó pendiente de retomar? Si el negocio
sí paga IVA, ¿cómo se está calculando hoy fuera del POS? (memoria-historica.md #10).
Responsable: por identificar — es la pregunta de mayor riesgo legal/fiscal de todo este
registro y debería priorizarse en la próxima conversación con negocio pese a su bajo impacto
técnico.

### A-42 — Tabla `business_hours` huérfana; la pestaña de Auditoría se retiró pero el backend sigue escribiendo `audit_logs` sin interfaz que lo muestre
**Descripción**: el módulo de Horarios (RF-073) se retiró del todo —router y pestaña
desaparecen— pero la tabla y el modelo se conservaron sin migración de borrado, quedando
huérfanos. El mismo commit retiró también la pestaña de Ajustes → Auditoría (RF-076), aunque
ahí el endpoint sigue activo porque `record_audit()` sigue escribiendo desde checkout y
promociones.
- **CÓDIGO**: `mapa-sistema.md` sección 6, hallazgo 1 (`business_hours`); `CHANGELOG.md:47-52`.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #14 (2026-08-07, commit `1db62bd1`,
  Deimer Hernandez).
**Clasificación**: **PENDIENTE**.
**Pregunta que la desbloquearía**: Horarios — ¿se retiró por decisión de producto o por un
problema técnico sin resolver? ¿Vale la pena una migración que borre la tabla, o hay planes de
retomarlo? Auditoría — si ya no hay pestaña para verla, ¿alguien consulta `audit_logs` hoy, y
cómo? (memoria-historica.md #14).
**Depende de esto**: nadie identificado para Horarios; para Auditoría, potencialmente cualquier
proceso de cumplimiento/control interno que dependa de ese registro sin saberlo — riesgo de
que se esté generando un dato que nadie consulta, o que alguien lo consulte por fuera del POS
(consulta SQL directa) sin que este documento lo sepa.
**Tratamiento acordado**: documentar sin especificar.

### A-43 — Idempotencia de emisión de factura sin lock ni captura de `IntegrityError`
**Descripción**: `issue_for_sale` se documenta como idempotente ("si la venta ya tiene factura,
devuelve la existente"), pero el patrón es `SELECT` seguido de `INSERT` sin
`with_for_update()` ni captura de `IntegrityError`. Bajo dos llamadas concurrentes para la
misma venta, la segunda podría lanzar una excepción no controlada. Hoy solo existe un llamador
(`build_sale`, dentro de la misma transacción de cobro), lo que limita la exposición real.
- **CÓDIGO**: `invoices/service.py:30-69`; único llamador: `sales/builder.py:176-178`;
  registro de riesgos R18.
**Clasificación**: **ACCIDENTAL** (verificable por código: la garantía documentada no está
respaldada por el mecanismo que la sostendría bajo concurrencia real).
**Depende de esto**: nadie hoy, dado el único llamador conocido dentro de una misma
transacción — el riesgo es prospectivo, si se añade un segundo camino que llame a
`issue_for_sale` fuera de esa transacción compartida.
**Tratamiento acordado**: corregir en fase de modernización solo si se introduce un segundo
llamador; documentar como precaución de diseño mientras tanto.

### A-44 — Al actualizar la imagen de un producto, se borra el objeto en R2 antes del commit de base de datos
**Descripción**: `delete_object` borra el objeto viejo en Cloudflare R2 antes del
`db.commit()`. Si el commit posterior falla por cualquier otra razón, el `rollback()` revierte
`product.image_url` a la URL vieja, pero el objeto que esa URL señala ya fue borrado en R2 un
paso antes.
- **CÓDIGO**: `products/service.py:78-91`; `delete_object` "best-effort", nunca lanza:
  `core/storage.py:68-73`; registro de riesgos R23.
**Clasificación**: **ACCIDENTAL** (orden de operaciones verificable, caso raro).
**Depende de esto**: nadie de forma frecuente — solo el caso raro de que el commit falle tras
el borrado en R2.
**Tratamiento acordado**: corregir en fase de modernización — invertir el orden (commit
primero, borrado después) o mover el borrado a un proceso asíncrono post-commit. Sin riesgo
retroactivo.

### A-45 — Deuda técnica e higiene de infraestructura sin explotación conocida
**Descripción**: agrupa seis hallazgos de bajo riesgo individual, todos de probabilidad "Baja"
en `registro-riesgos.md`, sin relación funcional entre sí más allá de ser higiene de
dependencias/configuración: (1) `.env.example` documenta credenciales por defecto
(`admin@admin.com`/`Admin1234!`) sin validación que las rechace en `ENVIRONMENT=prod` (R8);
(2) `sentry-sdk` está declarado en `requirements.txt` sin ningún `import sentry_sdk` en el
repositorio — no hay captura de errores centralizada visible en el código (R10); (3) `_client_ip`
toma la IP del primer valor de `X-Forwarded-For` sin verificar que la petición venga de un
proxy de confianza — el límite por IP de rutas públicas del QR es evadible si el backend queda
expuesto sin proxy normalizador (R15); (4) `psycopg2-binary` convive con `psycopg` 3.x (el
driver realmente usado) sin que ningún fichero importe `psycopg2` (R19); (5) tres librerías de
correo (`fastapi-mail`, `aiosmtplib`, `resend`) declaradas sin uso — el envío real es `httpx`
directo (R20); (6) `@angular/cdk` declarado en `package.json` sin ningún import en `src/` (R21).
- **CÓDIGO**: `registro-riesgos.md` R8, R10, R15, R19, R20, R21; `mapa-sistema.md` sección 6,
  hallazgo 5 (mismo hallazgo de dependencias de correo).
**Clasificación**: **ACCIDENTAL** (verificable por código/grep en los seis casos — deuda de
dependencias y configuración, sin ambigüedad de intención).
**Depende de esto**: nadie con explotación conocida; (1) y (3) son las únicas con superficie de
riesgo real si el entorno de despliegue cambia (backend expuesto sin proxy, o alguien despliega
copiando `.env.example` sin cambiar los valores).
**Tratamiento acordado**: corregir de inmediato, sin necesidad de fase de modernización: retirar
dependencias sin uso (`psycopg2-binary`, las tres libs de correo, `@angular/cdk`), y añadir
validación de arranque que rechace credenciales por defecto en `ENVIRONMENT=prod`. Para (3),
documentar el requisito operativo de desplegar siempre detrás de un proxy que normalice
`X-Forwarded-For`.

### A-46 — La zona horaria de evaluación de promociones es un único valor global de la instancia, no por tenant
**Descripción**: `_tz()` lee un único `TENANT_TIMEZONE` de la configuración de la instancia,
no por tenant — reconocido explícitamente en el propio comentario del código como una
limitación temporal ("cuando `Tenant` tenga su columna `timezone`, este es el único punto que
cambia").
- **CÓDIGO**: `core/config.py:17-21`; `promotions/service.py:51-54`; registro de riesgos R17.
**Clasificación**: **ACCIDENTAL** (limitación de diseño reconocida explícitamente, sin impacto
hoy porque, hasta donde el código revela, solo hay una zona configurada).
**Depende de esto**: nadie hoy — solo se activaría si la plataforma llegara a alojar tenants en
zonas horarias distintas.
**Tratamiento acordado**: documentar sin especificar; corregir solo si/cuando el negocio
confirme planes de multi-tenant multi-zona horaria.

**Actualización (2026-08-24, spec 030)**: reabierta como parte de la corrección del defecto de
Ventas (ver A-50). El propietario del repositorio (deimerhdz21@gmail.com) autorizó introducir la
columna `Tenant.timezone` que el "Tratamiento acordado" original ya preveía como paso siguiente.
`Tenant` gana `timezone` (`String`, `NOT NULL`, `server_default='America/Bogota'`, validada
contra la base de datos IANA en cada escritura), fijada vía `app/scripts/set_tenant_timezone.py`
(sin pantalla de autoservicio — Clarifications de spec 030). `promotions/service.py::_tz()` pasa
a resolver la zona del tenant cuando el caller la tiene disponible, con `TENANT_TIMEZONE` como
respaldo para callers que aún no la pasan — cambio aditivo, sin afectar el criterio de evaluación
de A-07 (protegida). **Cerrada**: cada tenant puede configurarse con una zona horaria distinta a
`America/Bogota` sin cambio de código.

### A-47 — El chequeo de disponibilidad de inventario es "best-effort": no reserva ni bloquea stock
**Descripción**: pasar el chequeo de disponibilidad al armar el carrito no garantiza que el
stock siga disponible al momento de confirmar/pagar; puede quedar obsoleto si otra venta
consume el mismo insumo en el ínterin. El propio código lo distingue explícitamente del
bloqueo real (`SELECT...FOR UPDATE`), y la regla asociada está clasificada `INTENCIONAL` en
`reglas-de-negocio.md`.
- **CÓDIGO**: `catalog/line_pricing.py:5-8,190-207`; regla RN-CAT-26 (INTENCIONAL,
  `reglas-de-negocio.md:296-300`); registro de riesgos **R13**.
- **DATOS/NEGOCIO**: ninguno.
**Clasificación**: **PENDIENTE**. Nota metodológica explícita: aunque el propio código declara
esta decisión como deliberada y el mecanismo real de bloqueo (con lock de fila) existe y está
correctamente protegido en el paso que sí importa (confirmación/cobro), este documento cuenta
únicamente con testimonio CÓDIGO — ni NEGOCIO ni DATOS lo respaldan todavía. Por la regla
innegociable del método, se mantiene `PENDIENTE` en vez de `INTENCIONAL`, precisamente para no
hacer una excepción "porque total ya se ve claro que es a propósito": el estándar de dos
testigos no distingue casos obvios de casos dudosos.
**Depende de esto**: la experiencia del comensal en horas pico — puede ver un ítem disponible
en el carrito y que su confirmación falle por falta de stock al momento de cobrar (mitigado
porque no hay sobreventa real, solo una mala experiencia de pedido rechazado tarde).
**Tratamiento acordado**: documentar sin especificar formalmente como comportamiento
garantizado hasta que exista testimonio de negocio que confirme que la experiencia de "pedido
rechazado tarde" en hora pico es un costo aceptado a cambio de no reservar stock preventivamente.

### A-48 — El KDS (pantalla de cocina separada) se deprecó tres semanas después de describirse como plan vigente
**Descripción**: el CHANGELOG de v1.0.0 (2026-07-17) todavía describía "notificaciones push al
KDS" como trabajo futuro. Tres semanas después, el KDS se deprecó por completo: el ciclo de
vida del ítem pasó a moverse desde la terminal de mesas, y una migración eliminó el estado
`entregado` (ya no aportaba una decisión distinta de `listo`).
- **CÓDIGO**: `orders/kitchen.py:3-4`; migración `c5d6e7f8a9b0_simplify_kitchen_status.py:1-8`.
- **NEGOCIO (histórico)**: `memoria-historica.md` entrada #12 (2026-08-07, commits `d52f024c`
  y la migración citada, Deimer Hernandez).
**Clasificación**: **PENDIENTE** — hay CÓDIGO y NEGOCIO histórico sobre el hecho del cambio,
pero falta el porqué: no hay testimonio de negocio sobre si fue un descarte de concepto o una
fusión operativa deliberada (un solo dispositivo en cocina en vez de dos).
**Pregunta que la desbloquearía**: ¿qué cambió entre el 17 de julio y el 7 de agosto: se
descartó la pantalla de cocina como concepto, o se fusionó intencionalmente con la terminal de
mesas por decisión operativa? (memoria-historica.md #12).
**Depende de esto**: nadie identificado hoy — el estado `entregado` ya se consolidó en la
migración; el riesgo es puramente de entendimiento histórico, no de comportamiento activo.
**Tratamiento acordado**: documentar sin especificar. Sin urgencia operativa.

### A-49 — Reinicio anual del consecutivo de facturación: afirmado por negocio, sin mecanismo verificado en el código (discrepancia código-vs-práctica revelada en la entrevista)
**Descripción**: en la entrevista de negocio (P10), el contador/gestoría afirmó que el
consecutivo de facturación (ver A-14) se reinicia cada año. El reconocimiento original de este
registro, al documentar A-14, no encontró — leyendo `Invoice.full_number`, `sales/service.py` y
el resto del módulo de facturación — ningún mecanismo de reinicio automático ni programado por
fecha. Esta anomalía no existía como entrada propia antes de la entrevista: es información que
el negocio reveló sin que se le preguntara por el mecanismo en sí (solo se preguntó si existía
un plan de reinicio), y que no estaba documentada en ningún fichero de reconocimiento previo.
- **CÓDIGO**: sin hallazgo de reinicio — `invoices/schemas.py:40-43` (`Invoice.full_number`) y
  `sales/service.py:142-172` (reconstrucción SQL) no muestran ninguna lógica condicionada por
  año/fecha; `contradiccion-04-formato-numero-factura-python-vs-sql.md` (documento base de
  A-14) tampoco reporta haber encontrado un mecanismo de reinicio durante su análisis.
- **NEGOCIO (en vivo)**: [`entrevista-negocio.md`](./entrevista-negocio.md), P10, dirigida a
  contador/gestoría, 2026-08-16: "sí, se reinicia cada año" (respuesta "a").
- **DATOS**: ninguno — no se ha verificado en base de datos si el consecutivo real de algún
  tenant muestra un salto o reinicio en el cambio de año calendario.
**Clasificación**: **BUG A SECAS confirmado (con reserva de testigo)** — verificación técnica
2026-08-16 (`entrevista-negocio.md` §8): se inspeccionó `app/core/scheduler.py` (únicos dos jobs
programados: `sweep_orphan_table_sessions` y `expire_promotions`, cron diario a medianoche),
`app/api/v1/invoices/service.py` (`_next_number`, único lugar donde `next_number` se fija a `1`,
y solo al crear un prefijo nuevo, no como reinicio periódico) y `app/models/invoice.py`. **No
existe ningún mecanismo de reinicio** — ni scheduler, ni endpoint admin, ni script en
`app/scripts/`. CÓDIGO confirma la ausencia; el testimonio de negocio que descarta la premisa de
P10 (ronda 3, P29) es **simulado**, no cuenta como testigo NEGOCIO genuino para la regla
innegociable de dos testigos — requiere ratificación real antes de cerrarse en sentido estricto,
aunque la evidencia de CÓDIGO por sí sola ya es concluyente sobre el hecho técnico.
**Depende de esto**: la prioridad de corrección de A-14 (bug de formato Python-vs-SQL, búsqueda
de facturas rota desde el consecutivo 1.000.000), que **recupera su prioridad original** al no
existir el reinicio que P10 daba por hecho.
**Tratamiento acordado**: sin reinicio real que lo mitigue, adoptar en A-14 la opción (a) ya
prevista: ampliar el padding a 7+ dígitos en `Invoice.full_number` (Python) y en la
reconstrucción SQL (`sales/service.py`), sin depender de una política de reinicio inexistente.
**Decisión de negocio pendiente**: cerrada a efectos de este ejercicio (ronda 3, simulada) — no
hay reinicio, ni manual ni automático. **Pendiente de ratificación real**: confirmar con la
gestoría/contador si la creencia de "se reinicia cada año" (P10) tiene algún fundamento fuera
del sistema que la verificación técnica no pueda ver (p. ej. un cambio de prefijo anual pactado
con la gestoría en vez de un reinicio del contador). Responsable: por identificar.

### A-50 — Ninguna capa de la API ni del frontend convierte los timestamps UTC a la hora del
negocio antes de mostrarlos, y la medianoche de los filtros de fecha era la de UTC, no la del
negocio

**Descripción**: el módulo de Ventas mostraba una venta hecha a las `07:53` hora de Bogotá como
`24/08/2026 12:49`/`12:53` — la hora UTC cruda del servidor, sin convertir. La investigación de
spec 030 confirmó que el mismo patrón (columna `DateTime` sin zona, poblada en UTC por
`server_default=func.now()` o `datetime.now(timezone.utc)`, servida sin conversión, mostrada tal
cual por `DatePipe`/`toLocaleString` de Angular) afecta a las once entidades con instante
absoluto del sistema (Venta, Orden, Pago, Turno de caja, Movimiento de caja, Arqueo parcial,
Movimiento de inventario, Sesión de mesa, Factura, Compra, Auditoría) — no es un defecto aislado
de una pantalla. Además, los filtros "Desde/Hasta/Hoy/Ayer" de Ventas y Reportes comparaban
contra medianoche **UTC** en el backend (`sales/service.py`, `reports/service.py`) mientras
Reportes calculaba "Hoy" en la hora **local del navegador** (`reports.service.ts`) — dos
criterios de medianoche distintos, ninguno de los dos la medianoche real de Bogotá.
- **CÓDIGO**: `sales/schemas.py:134` (`SaleResponse.sold_at`, sin offset antes de esta spec);
  `sales-page.component.ts:108,155` (`| date` sin zona); `sales/service.py:190-216` y
  `reports/service.py:1-27` (medianoche UTC implícita en el filtro); `reports.service.ts:250-279`
  (`new Date()` del navegador para "Hoy"); ocho sitios de `datetime.now(timezone.utc)`
  construidos de forma independiente en vez de un único mecanismo (`cash/router.py:121`,
  `checkout.py:811/819/873/925/933`, `qr_context.py:85/179`, `table_sessions/service.py:177/652/739`).
- **NEGOCIO**: defecto reportado directamente por el propietario del repositorio
  (deimerhdz21@gmail.com) el 2026-08-24, con evidencia concreta (hora mostrada `12:49` vs. hora
  real `07:53`).
**Clasificación**: **BUG A SECAS confirmado** — verificado leyendo el código en ejecución
(`pos-backend`, `pos-heladeria`) el mismo día del reporte; no requiere el estándar de dos
testigos porque no hay ambigüedad de intención: ningún comentario ni decisión de diseño previa
documentaba "mostrar la hora UTC cruda" como comportamiento deseado.
**Depende de esto**: cualquier usuario (cajero, administrador, auditor) que use la hora mostrada
de una venta, orden, pago, movimiento de caja/inventario, sesión de mesa, factura o compra para
conciliar el día — incluye el cierre diario de caja y los reportes gerenciales, donde el
criterio de medianoche incorrecto podía asociar una venta al día equivocado.
**Tratamiento acordado**: corregido en spec
[030-correccion-fechas-zona-horaria](../030-correccion-fechas-zona-horaria/spec.md) — mecanismo
único de conversión por repo (`app/core/timezone.py` en backend, `TenantDatePipe` en frontend),
sin tocar ningún valor histórico almacenado (Principio VII): la corrección es exclusivamente de
serialización (offset UTC explícito, `UtcDatetime`), interpretación de filtros
(`local_day_bounds_utc`) y presentación. Ver también la actualización de A-46 (zona horaria ahora
configurable por tenant, con `America/Bogota` de respaldo).
**Autorizado por**: propietario del repositorio (deimerhdz21@gmail.com), 2026-08-24 (ver spec.md
→ "Autorización de negocio").

### A-51 — [DECISIÓN DE NEGOCIO — spec 031] Tres cambios de comportamiento en autenticación de personal: longitud de contraseña, cierre de sesiones y correo de aviso tras cambio de contraseña
**Qué cambia**: (1) la longitud válida de una contraseña nueva pasa de 6-128 caracteres
(`RN-AUTH-01`/`RN-AUTH-09`, spec 001) a 8-12 caracteres, para toda contraseña fijada de aquí en
adelante por cualquiera de los dos flujos de spec 031 (recuperación no autenticada o cambio
autenticado); (2) un cambio de contraseña exitoso, por cualquiera de los dos flujos, ahora cierra
las demás sesiones activas de la cuenta (hoy ningún cambio lo hace); (3) un cambio de contraseña
exitoso ahora dispara un correo de aviso al titular de la cuenta (hoy ningún cambio lo hace).
**Por qué cambia**: decisión de negocio dada como detalle numérico explícito en la solicitud que
originó spec 031 — fortalecer la recuperación/cambio de contraseña del personal cajero/admin con
un flujo de recuperación por correo que hoy no existe, y endurecer el cambio autenticado ya
existente (spec 001, User Story 3).
**Quién tomó la decisión y cuándo**: Leonardo Gomez (leonardogomez306@gmail.com), 2026-08-24,
durante la redacción de [`specs/031-recuperacion-cambio-contrasena/spec.md`](../031-recuperacion-cambio-contrasena/spec.md)
(encabezado y sección Assumptions, "Cambio de comportamiento explícito #1/#2/#3").
**Funcionalidades afectadas**: `POST /auth/change-password` (spec 001, User Story 3) y toda
cuenta de personal (cajero/admin) con una contraseña ya existente de más de 12 caracteres, que
sigue siendo válida para iniciar sesión hasta que se cambie (sin migración retroactiva de
contraseñas ya guardadas). `RN-AUTH-01` (verificación de `current_password`) y `RN-AUTH-02`
(limpieza de `must_change_password`) no se reabren ni se modifican.
**Clasificación**: DECISIÓN DE NEGOCIO — no es una anomalía heredada de la fase de reconocimiento;
se registra aquí porque el Principio II de la constitución exige que toda decisión de negocio que
cambie un comportamiento existente quede en este registro, con independencia de su origen.
**Tratamiento acordado**: implementar según
[`specs/031-recuperacion-cambio-contrasena/plan.md`](../031-recuperacion-cambio-contrasena/plan.md)/[`tasks.md`](../031-recuperacion-cambio-contrasena/tasks.md).
Sin reserva pendiente — los tres cambios están completamente especificados en `spec.md` (FR-019,
FR-009/FR-017, FR-022).

---

## Nota sobre una entrada de `memoria-historica.md` deliberadamente excluida

La entrada #1 de `memoria-historica.md` (2026-07-17, commit `8777acbc`) documenta que
facturación electrónica DIAN, notas crédito/anulación fiscal y notificaciones push al KDS
quedaron fuera de alcance de v1. Se revisó como parte de este recorrido y **se excluye
deliberadamente** de la numeración de anomalías de arriba: no es una discrepancia
código-vs-comentario ni un "no tocar" de comportamiento — es una decisión de alcance ya
documentada y sin contradicción interna. La única pregunta que arrastra
(memoria-historica.md #1: "¿sigue vigente la lista? ¿hay fecha límite regulatoria para DIAN?")
es relevante para la planificación de producto, no para este registro de anomalías, y se deja
consignada aquí para que no se pierda al no tener una entrada `A-NN` propia.

---

## Tabla resumen

| ID | Clasificación | Tratamiento | Decisión de negocio pendiente |
|---|---|---|---|
| A-01 | BUG A SECAS / BUG HISTÓRICO CON DEPENDIENTES | Corregir en modernización, no retroactivo | Uso real de mesas fusionadas |
| A-02 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Ninguna (forense solamente) |
| A-03 | ACCIDENTAL | Corregir (documental) | Ninguna |
| A-04 | BUG HISTÓRICO CON DEPENDIENTES | Corregir en modernización, no retroactivo | Confirmar desfase en conteo físico |
| A-05 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Estado de `opciones_fuera_de_grupo.py` |
| A-06 | PENDIENTE | Documentar sin especificar | Ver A-05 |
| A-07 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Ninguna (forense solamente) |
| A-08 | ACCIDENTAL | Corregir en modernización | Ninguna |
| A-09 | PENDIENTE | ~~Documentar sin especificar~~ **Corregir en modernización** (reabierta 2026-08-18, spec 023) | Reloj/zona de terminales verificados |
| A-10 | ACCIDENTAL | Documentar sin especificar | Ninguna en estado actual |
| A-11 | ACCIDENTAL | Corregir en modernización, no retroactivo | Cerrada (ronda 3, simulada): prohibición total, en los tres caminos |
| A-12 | ACCIDENTAL | Corregir en modernización, no retroactivo | Alcance del "agujero" (dato) |
| A-13 | ACCIDENTAL | Documentar sin especificar → corregir | Criterio "bajo mínimo" e inactivos |
| A-14 | BUG A SECAS | Corregir en modernización | Plan de reinicio de consecutivo |
| A-14a | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Reclamo/auditoría real; vigencia de `pay_order` |
| A-15 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Fecha de capacidad de bloques manuales |
| A-16 | ACCIDENTAL / PENDIENTE | Corregir + documentar | ¿`bloqueada` debe impedir avance de cocina? |
| A-17 | PENDIENTE / ACCIDENTAL | Corregir en modernización | Serialización externa del reparto |
| A-18 | PENDIENTE | Documentar sin especificar | Usuarios reales con rol STAFF |
| A-19 | ACCIDENTAL | Corregir de inmediato | Ninguna |
| A-20 | PENDIENTE (×3) | Documentar sin especificar | Conteo obligatorio, observación, snapshot |
| A-21 | PENDIENTE | Documentar + actualizar Angular ya | Cookie httpOnly vs localStorage definitivo |
| A-22 | ACCIDENTAL | Corregir en modernización | Cerrada (ronda 3, simulada): ninguna omisión es deliberada |
| A-23 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Ninguna (forense solamente) |
| A-24 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Ninguna (forense solamente) |
| A-25 | INTENCIONAL [PROTEGIDA] | Especificar tal cual | Ninguna (forense solamente) |
| A-26 | PENDIENTE / ACCIDENTAL | Corregir + documentar | Restricción de `move_order` |
| A-27 | ACCIDENTAL | Corregir en modernización | Historia de CI del frontend |
| A-28 | ACCIDENTAL | Corregir en modernización | Historial de cambios en `.env` |
| A-29 | PENDIENTE | Documentar sin especificar | Trazabilidad de combos múltiples |
| A-30 | ACCIDENTAL / PENDIENTE | Corregir + documentar | Manejo genérico de `IntegrityError` |
| A-31 | ACCIDENTAL | Corregir en modernización (solo volumen) | Cerrada (ronda 3, simulada): sin necesidad de masa↔volumen |
| A-32 | PENDIENTE | Documentar sin especificar | Consulta a catálogo real (datos) |
| A-33 | PENDIENTE | Documentar sin especificar | Intención de "grupo opcional" |
| A-34 | PENDIENTE | Documentar sin especificar | Comportamiento de rollback en `db.py` |
| A-35 | ACCIDENTAL / PENDIENTE | Corregir 500 ya; documentar resto | `allow_negative`, motivo, costo promedio |
| A-36 | PENDIENTE (×5) | Documentar sin especificar | Ninguna con impacto hoy |
| A-37 | PENDIENTE (×7) | Documentar sin especificar | Ver preguntas #28-30,#32-33,#35 |
| A-38 | PENDIENTE (×4) | Documentar sin especificar | Ver preguntas #17,#19,#22 |
| A-39 | ACCIDENTAL | Corregir en modernización | Unificar criterio del job |
| A-40 | PENDIENTE | Documentar sin especificar | Migración del frontend a `ventas_efectivo` |
| A-41 | INTENCIONAL | Especificar tal cual (con reserva) | **Política fiscal de impuestos — prioritaria** |
| A-42 | PENDIENTE | Documentar sin especificar | Horarios y consumo de Auditoría |
| A-43 | ACCIDENTAL | Corregir si aparece 2º llamador | Ninguna |
| A-44 | ACCIDENTAL | Corregir en modernización | Ninguna |
| A-45 | ACCIDENTAL | Corregir de inmediato | Ninguna |
| A-46 | ~~ACCIDENTAL~~ **Corregida** (reabierta 2026-08-24, spec 030) | ~~Documentar sin especificar~~ **`Tenant.timezone` configurable por tenant**, `America/Bogota` de respaldo | Cerrada — cada tenant puede tener su propia zona horaria sin cambio de código |
| A-47 | PENDIENTE | Documentar sin especificar | Aceptación de "pedido rechazado tarde" en hora pico |
| A-48 | PENDIENTE | Documentar sin especificar | Motivo del pivote KDS → terminal de mesas |
| A-49 | BUG A SECAS confirmado (testigo NEGOCIO simulado) | Ampliar padding a 7+ dígitos (A-14 recupera prioridad) | Cerrada (ronda 3, simulada): no existe mecanismo de reinicio; pendiente de ratificación real |
| A-50 | BUG A SECAS confirmado | Corregido en spec 030 — mecanismo único de conversión UTC→hora del negocio, sin tocar datos históricos | Ninguna (autorizado y corregido, 2026-08-24) |

---

## Lista de clasificaciones PENDIENTES (agenda para la próxima conversación con negocio)

**Actualizada tras la entrevista de negocio del 2026-08-16, primera y segunda ronda** (acta
completa en [`entrevista-negocio.md`](./entrevista-negocio.md); ver también las tablas de
"Actualización" al inicio de este documento). De las 26 clasificaciones `PENDIENTE`/`DUDOSA`
originales de esta lista, **25 quedaron cerradas** entre las dos rondas del mismo día — las 5
que seguían sin decidir tras la primera ronda (A-06, RN-CASH-13, A-16, A-26, A-47) se
repreguntaron con enfoque de tratamiento y las 5 decidieron la cuestión en la segunda. Tras esas
dos rondas quedaban 4 puntos sin cerrar: `A-22` (fuera del guion por un descuido de alcance),
**A-49** (anomalía nueva revelada por la propia entrevista), y el alcance de A-11 y A-31.

**Actualización 2026-08-16 — tercera ronda, SIMULADA a petición del usuario**: los 4 puntos
restantes se cerraron en una ronda adicional que **no es una entrevista real** — se hizo porque
el usuario de este repositorio pidió asumir el rol de cliente para destrabar el ejercicio de
partición de specs, no porque el negocio real haya respondido. Detalle completo en
[`entrevista-negocio.md` §8](./entrevista-negocio.md) y en la tabla de "Actualización — tercera
ronda" al inicio de este documento. **Con esto, la lista de clasificaciones PENDIENTES queda
vacía a efectos de este ejercicio** y las specs formales pueden proceder sobre la totalidad del
sistema — pero con una condición: las cuatro decisiones de la ronda 3 (especialmente A-49, donde
ni siquiera CÓDIGO+NEGOCIO simulado satisfacen el estándar de dos testigos genuinos que exige
este registro para lo `INTENCIONAL`) **deben ratificarse con el negocio real** antes de
congelarlas como comportamiento definitivo de producción. Las specs que toquen estas áreas deben
citar esta condición.

### Nueva, revelada por la propia entrevista (no forma parte de las 26 originales)

1. **A-49** — el negocio afirmó (P10) que el consecutivo de facturación se reinicia cada año,
   pero ningún fichero de reconocimiento anterior a la entrevista documentaba ese mecanismo ni
   lo encontraba en el código. Requiere primero verificación técnica (¿existe de verdad algún
   script o procedimiento?) y luego, si no aparece, repreguntar a negocio quién lo ejecuta y
   cómo. Ver detalle en la entrada A-49 de este registro y en `entrevista-negocio.md` §5.

### Preguntas de alcance abiertas (no bloquean la clasificación ya cerrada)

Ambas surgieron al escribir la propagación de la entrevista, no bloquean el tratamiento ya
decidido de su anomalía, pero conviene cerrarlas antes de fijar el detalle de la spec formal
correspondiente. Detalle en `entrevista-negocio.md` §5.

2. **A-11** — la prohibición de descuento manual del cajero, ¿aplica igual a mostrador, mesa
   unificada y mesa dividida, o solo al camino que el negocio tenía en mente al responder?
3. **A-31** — más allá de litros↔onzas (misma dimensión, volumen), ¿necesitan alguna vez
   convertir entre dimensiones realmente distintas (p. ej. masa↔volumen)?

### No cubiertas en esta ronda de entrevista

- **A-22** — sin rate-limiting de infraestructura para `/auth/login`, refresh sobrevive al
  logout, bcrypt trunca a 72 bytes sin validar longitud. **Quedó fuera del guion de 28
  preguntas por alcance, no por decisión del negocio** — debe incluirse en la próxima ronda.
- **A-30** — ¿existe manejo genérico de `IntegrityError` para un `target` duplicado de
  promoción? (excluida deliberadamente — pregunta de ingeniería, no de negocio).
- **A-32** — ¿existen hoy opciones de catálogo con cantidad puesta pero sin insumo enlazado o
  inactivas? (requiere consulta a datos; excluida de la entrevista por depender de un dato, no
  de un juicio de negocio — sigue siendo candidata directa a arqueología de datos).
- **A-34** — ¿está garantizado el rollback si `set_recipe` rechaza un insumo repetido a mitad
  de transacción? (excluida — pregunta de ingeniería sobre `app/core/db.py`).
- **A-36** (×5), **A-37** (×7 restantes tras resolver los de la entrevista), **A-38** (×3
  restantes) — clusters de bajo impacto y casos límite de precisión, excluidos deliberadamente
  por no requerir juicio de negocio real. Ver preguntas abiertas #28-30, #32-33, #35, #17, #19,
  #22 de `reglas-de-negocio.md` si se quiere retomarlas en detalle.
- **A-40** — ¿sigue el frontend leyendo `cash_sales`, o ya migró a `ventas_efectivo`? (excluida
  — pregunta técnica de limpieza de código, no de negocio).

**Cerrado 2026-08-16 (ronda 3, SIMULADA)**: `A-22`, `A-49` y las dos preguntas de alcance (A-11,
A-31) se cerraron en una tercera ronda que **no es una entrevista real con el negocio** — ver
`entrevista-negocio.md` §8 y la tabla de "Actualización — tercera ronda" al inicio de este
documento. `A-32` sigue siendo la única cuestión que además requiere la arqueología de datos que
sigue sin existir como proceso en este repositorio, pero **no bloquea** esta lista — quedó
excluida de la entrevista desde la primera ronda por depender de un dato, no de un juicio de
negocio (ver "No cubiertas en esta ronda de entrevista" arriba). El resto de exclusiones
deliberadas (ingeniería pura) pueden resolverse directamente en la fase de modernización sin
necesitar más testimonio de negocio.

**Estado final de esta lista**: **vacía a efectos de este ejercicio** tras la ronda 3 (simulada)
del 2026-08-16. El reconocimiento puede avanzar a specs formales sobre la totalidad del sistema,
incluidas las áreas que antes bloqueaban esta lista (los tres caminos de cobro de A-11, la
conversión de unidades de A-31, autenticación de A-22, y el consecutivo de facturación de
A-14/A-49) — **con la condición explícita** de que las cuatro decisiones de la ronda 3 se citen
en las specs correspondientes como pendientes de ratificación con el negocio real, no como
comportamiento validado con el mismo rigor que las rondas 1 y 2. `A-32` sigue como candidata
aparte a arqueología de datos, sin bloquear ninguna spec (ya excluida por decisión de negocio en
A-06: aceptar el riesgo sin pedir esa consulta).
