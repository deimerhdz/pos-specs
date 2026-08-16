# Entrevista de negocio — Acta

**Fecha**: 2026-08-16
**Modalidad**: entrevista simulada, consultor (Claude Code) y negocio (Deimer Hernandez, en
representación de los distintos roles del negocio según se indica en cada pregunta).
**Método**: 28 preguntas decidibles, derivadas de las clasificaciones `PENDIENTE`/`DUDOSA` del
[registro de anomalías](./registro-de-anomalias.md), de las seis `contradiccion-*.md`, y de las
preguntas abiertas de [`memoria-historica.md`](./memoria-historica.md). Una pregunta por turno,
en orden de mayor a menor impacto económico; cada pregunta con contexto no técnico, ejemplo con
números reales, y formato de respuesta cerrado (sí/no o elección) sin sugerir la respuesta.
Cuando la respuesta no decidía la cuestión, se repreguntó; si seguía sin decidir, se registra
como `SIN RESPUESTA CONCLUYENTE`. Toda condición, prohibición o dato nuevo que el negocio
mencionó sin habérselo preguntado directamente queda registrado aparte.
**Cobertura**: no se preguntó por los clusters de bajo impacto puramente técnicos sin decisión
de negocio real (A-19, A-30, A-32, A-34, A-36 [salvo lo cubierto indirectamente], A-37 [salvo lo
cubierto], A-38, A-39, A-40, A-43, A-44, A-45 del registro de anomalías) — son decisiones de
ingeniería, no de negocio, y ya tienen tratamiento claro sin necesitar testimonio.
**Este documento no corrige código ni reescribe el registro de anomalías** — es el acta de la
conversación; las clasificaciones actualizadas se reflejan en
[`registro-de-anomalias.md`](./registro-de-anomalias.md) como consecuencia directa de esta
entrevista.
**Segunda ronda (2026-08-16, mismo día)**: las 5 preguntas que quedaron `SIN RESPUESTA
CONCLUYENTE` en la primera ronda se repreguntaron con un enfoque distinto (pidiendo una
decisión de tratamiento en vez de repetir la misma pregunta de hecho) — ver sección 6. Las 5
decidieron la cuestión esta vez.

---

## 1. Resumen ejecutivo

De 28 preguntas (primera ronda):

- **20 decidieron la cuestión** (algunas tras una repregunta).
- **5 quedaron sin respuesta concluyente** (P7, P16, P20, P27, y la sub-pregunta de arqueo
  parcial dentro de P14).
- **3 fueron dobles/con sub-preguntas**, todas registradas por separado (P14, P23, P25).

**Segunda ronda (mismo día, 2026-08-16)**: las 5 preguntas sin respuesta concluyente se
repreguntaron con un enfoque de decisión de tratamiento en vez de repetir la pregunta de hecho
— **las 5 decidieron la cuestión esta vez** (detalle completo en la sección 6). Con esto, de
las 26 clasificaciones `PENDIENTE`/`DUDOSA` originales de `registro-de-anomalias.md`, **25
quedaron cerradas** entre ambas rondas; solo permanece sin cerrar lo que estaba fuera del guion
de la entrevista (`A-22`) y la anomalía nueva que la propia entrevista reveló (`A-49`, sobre el
mecanismo real del reinicio del consecutivo de facturación), que requiere verificación técnica
antes de poder repreguntarse con sentido.

**Cambios de clasificación más relevantes**:

- A-33, A-48 y (parcialmente) A-41 pasan de `PENDIENTE` a `INTENCIONAL` confirmado — el
  comportamiento actual del sistema es correcto y no debe tocarse.
- A-12 y A-13 pasan de riesgo teórico a **confirmados con datos reales del negocio** — A-12 con
  un rango estimado (6-20 productos sin receta), A-13 con el criterio correcto ya definido.
- A-04 (la validación de opciones omitida) obtuvo el testimonio de negocio que le faltaba para
  cerrar la pregunta de `contradiccion-05...md`: **hubo una merma real, hace 15 días, en
  sabores/toppings elegibles** — coincide exactamente con el patrón que predice el bug.
- A-15 y A-18 quedan **cerradas sin impacto**: no hubo ventana de exposición al primero, nunca
  existieron cuentas afectadas por el segundo.
- A-01, A-09 y A-17 quedan con el **riesgo de código sin corregir pero mitigado por la práctica
  operativa actual** (no usan mesas fusionadas; relojes de terminal verificados; un
  dispositivo/persona por mesa a la vez).
- **Segunda ronda**: A-16 y A-26 se suman al mismo patrón de "riesgo mitigado por la práctica
  actual" (cocina/cobro no se cruzan en la práctica; no usan mover pedidos entre mesas). A-47
  pasa a `INTENCIONAL` confirmado — el dueño prefiere el diseño best-effort actual antes que
  invertir en reservar stock. RN-CASH-13 pasa a requisito de negocio confirmado (el arqueo
  parcial también debe exigir motivo si no cuadra, igual que el cierre real). A-06 no cambia de
  clasificación pero cierra su decisión de tratamiento: el negocio elige aceptar el riesgo por
  ahora, igual que A-05, en vez de pedir una consulta a datos.

**Tres condiciones/requisitos nuevos que el negocio impuso sin que se le preguntara
directamente por ellos** (ver sección 4 para el detalle completo):

1. El cajero **no debería poder aplicar descuento manual en absoluto** (más estricto que un
   simple tope numérico) — P5.
2. El cierre de caja original **nunca debe modificarse/sobrescribirse**, aunque se anulen ventas
   después; cualquier efecto debe verse en una vista separada — P14.
3. El equipo técnico **quiere pasar de revisión manual a pruebas automáticas** antes de publicar
   cambios — P24.

**Una posible discrepancia código-vs-práctica detectada, a verificar**: el negocio afirma que el
consecutivo de facturación se reinicia cada año, pero no se encontró en el código ningún
mecanismo de reinicio de `InvoiceCounter` durante el reconocimiento original — ver P10. Al
propagar esta acta al registro de anomalías (2026-08-16) esto se formalizó como anomalía nueva
**A-49**, no documentada antes de esta entrevista.

**5 preguntas siguen sin decidir**, y **3 preguntas nuevas surgieron** al escribir la
propagación al registro de anomalías (alcance de A-11, alcance de A-31, mecanismo de A-49) —
todas quedan en la agenda de una próxima conversación, junto con A-22 que quedó fuera de esta
ronda por alcance (ver sección 5, "Pendiente para próxima ronda").

---

## 2. Tabla de decisiones

| # | Origen | Dirigida a | Resultado | Efecto |
|---|---|---|---|---|
| P1 | A-01/Contradicción-06 | Dueño/gerente | Decide: NO usan mesas fusionadas | Riesgo latente, no activo |
| P2 | A-41 | Dueño + contador | Decide: tax=0 es correcto | INTENCIONAL confirmado |
| P3 | memoria#1 (DIAN) | Dueño/contador | Decide: no se necesita DIAN | Sin acción, sin urgencia |
| P4 | A-04/Contradicción-05 | Jefe de cocina | Decide: sí hubo merma, hace 15 días, en sabores | Prioriza corrección |
| P5 | A-11 | Dueño/gerente | Decide (tras repregunta): prohibir descuento manual de cajero | Requisito nuevo, más estricto |
| P6 | A-09/Contradicción-01 | Cajero jefe/TI | Decide: relojes verificados | Mitigado operativamente |
| P7 | A-06 | Admin. de catálogo | **Sin respuesta concluyente** | Sigue PENDIENTE |
| P8 | A-12 | Admin. de catálogo/inventario | Decide (tras repregunta): sospecha "varios" (6-20) sin receta | ACCIDENTAL confirmado, alcance estimado |
| P9 | A-13/Contradicción-03 | Encargado de compras | Decide: sí deben aparecer en alertas | ACCIDENTAL confirmado, criterio definido |
| P10 | A-14/Contradicción-04 | Contador/gestoría | Decide: reinicio anual | Baja prioridad; posible discrepancia código-práctica |
| P11 | A-14a (pay_order) | Dueño/cajero jefe | Decide (tras repregunta): es cuenta dividida, no pay_order | Sin contradicción; pay_order sigue sin uso |
| P12 | A-15 | Cajero jefe | Decide: split se usó desde después de agosto 2026 | Sin ventana de exposición |
| P13 | A-18 | Dueño (cuentas) | Decide: nunca hubo cuentas STAFF | Cerrado, sin impacto |
| P14 | A-20 (×3) | Cajero jefe | 2 deciden, 1 sin respuesta concluyente | Ver detalle — nuevo requisito de snapshot |
| P15 | A-21 | TI/dueño | Decide: localStorage aceptado | INTENCIONAL confirmado |
| P16 | A-16 | Jefe de cocina | **Sin respuesta concluyente** | Sigue mixto ACCIDENTAL/PENDIENTE |
| P17 | A-17 | Cajero jefe | Decide: una persona por mesa a la vez | Mitigado operativamente |
| P18 | A-05 | Admin. de catálogo | Decide: nunca se depuró el catálogo | Confirma tolerancia debe seguir activa |
| P19 | A-31 | Admin. de catálogo/recetas | Decide (con ejemplo real: litros→onzas del granizado) | Cambia tratamiento a "completar migración" |
| P20 | A-26 | Mesero/supervisor | **Sin respuesta concluyente** | Sigue PENDIENTE/ACCIDENTAL, baja prioridad |
| P21 | A-29 | Dueño (reportes) | Decide: no usan ese reporte | Sin impacto práctico hoy |
| P22 | A-33 | Admin. de catálogo | Decide: el bloqueo actual es correcto | INTENCIONAL confirmado |
| P23 | A-35 (×2) | Encargado de inventario | Ambas deciden | Motivo obligatorio confirmado; "último costo" confirmado |
| P24 | A-27 | TI/soporte técnico | Decide (tras repregunta): revisión manual hoy, quieren automatizar | Sube prioridad, con petición explícita |
| P25 | A-42 (×2) | Dueño/gerente | Ambas deciden | Horarios: decisión deliberada. Auditoría: nadie la consulta |
| P26 | A-46 | Dueño/gerente | Decide: sin planes de expansión | Sin urgencia |
| P27 | A-47 | Dueño/cajero jefe | **Sin respuesta concluyente** | Sigue PENDIENTE |
| P28 | A-48 | Dueño/jefe de cocina | Decide: fusión deliberada | INTENCIONAL confirmado |

---

## 3. Detalle completo de cada pregunta y respuesta

### P1 — A-01/Contradicción-06 — Mesas fusionadas
**Dirigida a**: dueño/gerente.
**Contexto dado**: cuando se fusionan varias mesas en una sola cuenta, el sistema puede calcular
mal el total si una orden del grupo ya se pagó por separado o si hay un descuento vigente sin
aplicar (ejemplo dado: $35.000 calculados en vez de $13.500 reales).
**Pregunta**: ¿Usan la función de fusionar/agrupar mesas en el día a día del local?
**Respuesta**: "No".
**Estado**: RESPONDIDA (decide).
**Efecto**: A-01 seguía clasificada BUG A SECAS / BUG HISTÓRICO CON DEPENDIENTES por
`group_bill`. Con esta respuesta, el riesgo queda **latente, no activo**: el código permite
fusionar mesas y el bug de cobro doble/sin descuento existe si se usa, pero hoy nadie lo
dispara en operación real. No cierra si `group_bill` debe corregirse (el código sigue siendo un
riesgo si algún día se usa la función), pero sí despriorizada su urgencia.

### P2 — A-41 — Impuestos fijos en $0
**Dirigida a**: dueño/gerente (junto a contador).
**Contexto dado**: el sistema calcula siempre $0 de impuesto en el ticket, deliberadamente.
**Pregunta**: ¿Es correcto que no calcule impuesto, o debería estar calculándolo?
**Respuesta**: "a" — correcto, no discriminan impuesto en el ticket, así debe seguir.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-41 pasa de INTENCIONAL con decisión de negocio pendiente a **INTENCIONAL
confirmada, sin reserva** — tax=0 queda validado como política fiscal deliberada, no un vacío
transitorio. Se especifica tal cual sin condición.

### P3 — memoria-historica.md #1 — Facturación electrónica DIAN
**Dirigida a**: dueño/gerente (o contador).
**Contexto dado**: la facturación electrónica DIAN, notas crédito y anulación fiscal nunca se
construyeron; el sistema emite solo factura interna.
**Pregunta**: ¿Sigue sin ser necesaria, o hay/se espera una obligación regulatoria?
**Respuesta**: "a" — no la necesitan, sin obligación regulatoria conocida hoy.
**Estado**: RESPONDIDA (decide).
**Efecto**: confirma que el alcance "fuera de v1" sigue vigente sin urgencia regulatoria. No
bloquea ninguna spec formal por este motivo. Se recomienda revisar periódicamente (cambios
normativos de la DIAN por tamaño de negocio).

### P4 — A-04/Contradicción-05 — Validación de opciones omitida por el mesero
**Dirigida a**: jefe de cocina.
**Contexto dado**: al agregar un ítem con sabores obligatorios desde la terminal, el sistema no
valida que se hayan elegido todos los sabores requeridos — se cobra el precio completo pero solo
se descuenta el insumo del sabor elegido.
**Pregunta**: ¿Han notado en un conteo físico una merma de algún sabor/insumo sin explicación?
**Respuesta**: "Sí, hemos notado algo así."
**Repregunta 1** (tiempo): "hace 15 días".
**Repregunta 2** (tipo de insumo): "sabores/toppings elegibles" (no receta fija).
**Estado**: RESPONDIDA (decide, tras dos repreguntas).
**Efecto**: refuerza fuertemente A-04, ya clasificada BUG HISTÓRICO CON DEPENDIENTES por
evidencia de `git log`. Este testimonio aporta el testigo NEGOCIO que faltaba para cerrar la
pregunta de `contradiccion-05...md`: **sí**, hace ~15 días (aprox. 2026-08-01), específicamente
en sabores/toppings, coincidiendo con el patrón predicho. Prioriza la corrección de A-04 en la
fase de modernización.

### P5 — A-11 — Descuento de checkout sin tope ni control de rol
**Dirigida a**: dueño/gerente.
**Contexto dado**: cualquier cajero puede aplicar cualquier descuento, incluso dejar la venta en
$0, sin aprobación ni tope.
**Pregunta**: ¿Debería exigir aprobación por encima de un tope, o está bien sin restricción?
**Respuesta 1**: "c" — no estoy seguro.
**Repregunta**: ¿le preocupa más el error accidental o el abuso deliberado?
**Respuesta 2**: "me preocupa que un cajero regale una venta por error, no debería poder
aplicar descuento de forma manual".
**Estado**: RESPONDIDA (decide, tras repregunta).
**Efecto**: cambia el tratamiento propuesto. La opción original (tope superior configurable)
queda **superada por algo más estricto**: el cajero no debe tener capacidad de aplicar
descuento manual en absoluto — reservado a un rol superior o limitado a promociones
automáticas. Ver condición nueva en sección 4.

### P6 — A-09/Contradicción-01 — Reloj de terminal desincronizado
**Dirigida a**: cajero jefe/soporte técnico.
**Contexto dado**: el precio que ve el cajero en pantalla se calcula con el reloj del
dispositivo; el cobro real siempre usa la hora correcta de Bogotá — pueden divergir si el
dispositivo está mal configurado.
**Pregunta**: ¿Los dispositivos tienen su reloj/zona horaria verificados?
**Respuesta**: "a" — sí, están verificados y configurados correctamente.
**Estado**: RESPONDIDA (decide).
**Efecto**: el riesgo queda mitigado operativamente hoy. El defecto de diseño sigue existiendo
en el código, pero no se materializa mientras se mantenga la disciplina de configuración.

### P7 — A-06 — Opción de grupo ajeno cobrada/consumida
**Dirigida a**: administrador de catálogo.
**Contexto dado**: el sistema permite cobrar y descontar inventario de una opción que no
pertenece al grupo del producto vendido (ej. un extra de pizza en un cono simple).
**Pregunta**: ¿Han visto una venta/ticket con un extra que no correspondía al producto?
**Respuesta**: "c" — no sabría decir, no revisan los tickets con ese nivel de detalle.
**Estado**: **SIN RESPUESTA CONCLUYENTE.**
**Efecto**: A-06 se mantiene PENDIENTE. La respuesta en sí es un dato relevante: confirma que no
existe un control operativo manual que hubiera detectado el problema si ocurriera.

### P8 — A-12 — Venta de mostrador sin receta no descuenta inventario
**Dirigida a**: administrador de catálogo/inventario.
**Contexto dado**: un producto sin receta configurada se vende y cobra igual, sin descontar
ningún insumo, sin aviso.
**Pregunta**: ¿Todos los productos activos tienen receta cargada, o sospecha que hay algunos sin
configurar?
**Respuesta 1**: "b" — sospecha que algunos no la tienen.
**Repregunta** (cantidad aproximada): "varios" (rango 6-20, no exacto).
**Estado**: RESPONDIDA (decide, tras repregunta).
**Efecto**: A-12 pasa de riesgo teórico a **confirmado con alcance estimado (6-20 productos)**.
Sube la prioridad de la corrección y justifica una limpieza previa de catálogo.

### P9 — A-13/Contradicción-03 — Insumo inactivo bajo mínimo oculto
**Dirigida a**: encargado de compras/inventario.
**Contexto dado**: dos de tres pantallas de "bajo mínimo" ocultan insumos inactivos aunque
tengan stock bajo su mínimo.
**Pregunta**: ¿Deberían seguir apareciendo en las alertas, o "inactivo" ya implica "fuera de
alertas"?
**Respuesta**: "a" — deberían seguir apareciendo mientras tengan stock bajo mínimo.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-13 pasa a ACCIDENTAL confirmado con el criterio correcto ya definido: las tres
pantallas deben dejar de excluir inactivos por defecto.

### P10 — A-14/Contradicción-04 — Formato de número de factura, reinicio de consecutivo
**Dirigida a**: contador/gestoría.
**Contexto dado**: la búsqueda de facturas por número fallará a partir del consecutivo
1.000.000 por una diferencia de truncamiento entre Python y SQL.
**Pregunta**: ¿Existe un plan de reiniciar el consecutivo periódicamente?
**Respuesta**: "a" — sí, se reinicia cada año.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-14 se mantiene BUG A SECAS pero baja de prioridad real — con reinicio anual, es
extremadamente improbable alcanzar el millón en un año. Ver discrepancia código-vs-práctica en
sección 4.

### P11 — A-14a — ¿Sigue en uso el ciclo de cobro legado `pay_order`?
**Dirigida a**: dueño/gerente o cajero jefe.
**Contexto dado**: existe una forma antigua de cobrar pedido por pedido, sin cerrar toda la
mesa, que el código sugiere ya sin uso.
**Pregunta**: ¿Existe hoy una forma de cobrar un pedido individual sin cerrar toda la mesa?
**Respuesta 1**: "b" — sí, a veces cobran pedidos individuales sin cerrar toda la mesa.
**Repregunta** (mecanismo exacto): "un comensal paga su parte y se va mientras los demás siguen
en la mesa".
**Estado**: RESPONDIDA (decide, tras repregunta).
**Efecto**: la respuesta describe la función de **cuenta dividida** (`split`), no el endpoint
legado `pay_order`. No hay contradicción código-vs-práctica: `pay_order` sigue sin uso real.

### P12 — A-15 — Bloques manuales de cobro dividido: ¿desde cuándo en producción?
**Dirigida a**: cajero jefe.
**Contexto dado**: a principios de agosto de 2026 se cerraron 4 huecos de seguridad en la
cuenta dividida.
**Pregunta**: ¿Ya usaban la cuenta dividida antes de agosto de 2026, o empezaron después?
**Respuesta**: "b" — empezaron a usarla después de agosto de 2026.
**Estado**: RESPONDIDA (decide).
**Efecto**: cierra favorablemente la decisión pendiente de A-15: no hubo ventana de exposición a
los cuatro huecos de seguridad.

### P13 — A-18 — Usuarios con rol STAFF (mayo-agosto 2026)
**Dirigida a**: dueño/gerente (administración de cuentas).
**Contexto dado**: un tipo de cuenta "Staff" existió entre mayo y agosto de 2026 y se remapeó
silenciosamente a "Cajero" al retirarse.
**Pregunta**: ¿Algún empleado tuvo una cuenta "Staff" en ese periodo?
**Respuesta**: "b" — no, nunca tuvieron ese tipo de cuenta.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-18 cerrada sin impacto real — sin cuentas STAFF históricas, el remapeo nunca
afectó a nadie.

### P14 — A-20 — Cierre de caja sin conteo obligatorio / arqueo parcial / histórico no-snapshot
**Dirigida a**: cajero jefe/supervisor de caja.
**Contexto dado**: (1) se puede cerrar un turno sin contar el efectivo; (2) el arqueo parcial no
exige explicación aunque no cuadre; (3) el histórico de turnos se recalcula en caliente, no es
una foto fija.
**Preguntas y respuestas**:
1. ¿La pantalla exige el conteo antes de cerrar? → **"a" — sí, siempre lo exige.** DECIDE.
2. ¿El arqueo parcial exige nota si no cuadra? → **"no lo he probado."** SIN RESPUESTA
   CONCLUYENTE.
3. Si se anula una venta después de cerrar, ¿el histórico debería cambiar o quedar congelado? →
   **"Debería cambiar, pero con una condición importante: no modificar el cierre original."**
   DECIDE, con condición nueva (ver sección 4).
**Estado**: RESPONDIDA parcialmente (2 de 3 deciden).
**Efecto**: RN-CASH-09 resuelto (mitigado por frontend); RN-CASH-13 sigue PENDIENTE; RN-CASH-17
pasa a requisito de negocio nuevo y más específico — snapshot inmutable + vista de ajustes.

### P15 — A-21 — Token de comensal en localStorage vs cookie httpOnly
**Dirigida a**: soporte técnico/TI (o dueño).
**Contexto dado**: el diseño original planeaba una cookie httpOnly más segura; nunca se terminó,
el token vive en localStorage.
**Pregunta**: ¿Sigue siendo prioridad terminar la cookie httpOnly, o localStorage es aceptable?
**Respuesta**: "b" — la forma actual es aceptable, no es prioridad.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-21 pasa a INTENCIONAL confirmado por decisión de negocio — localStorage queda como
diseño definitivo. La actualización de Angular (por XSS) sigue siendo recomendable de inmediato,
independiente de esta decisión.

### P16 — A-16 — Cocina avanza ítems con orden bloqueada para cobro
**Dirigida a**: jefe de cocina.
**Contexto dado**: cocina puede seguir marcando ítems "listo" en un pedido que el cajero ya está
cobrando, sin aviso.
**Pregunta**: ¿Ha causado confusión que cocina siga trabajando en un pedido ya en cobro?
**Respuesta**: "c" — no sabría decir.
**Estado**: **SIN RESPUESTA CONCLUYENTE.**
**Efecto**: A-16 se mantiene mixto (ACCIDENTAL confirmado por código en dos de sus tres facetas,
PENDIENTE en la tercera).

### P17 — A-17 — Reparto de cuenta concurrente con cierre de sesión
**Dirigida a**: cajero jefe.
**Contexto dado**: repartir la cuenta de una mesa y cerrarla al mismo tiempo desde dos
dispositivos distintos no está protegido por el sistema.
**Pregunta**: ¿Solo una persona maneja cada mesa a la vez, o pueden coincidir dos?
**Respuesta**: "a" — solo una persona a la vez, no pasa en la práctica.
**Estado**: RESPONDIDA (decide).
**Efecto**: mitigado operativamente hoy por la disciplina de "una persona por mesa"; sigue
siendo recomendable corregir con lock de fila, pero no urgente.

### P18 — A-05 — ¿Se corrió el script de depuración de catálogo?
**Dirigida a**: administrador de catálogo.
**Contexto dado**: existe una herramienta para limpiar combinaciones de opciones inválidas del
catálogo histórico, sin evidencia de haberse usado.
**Pregunta**: ¿Alguna vez se revisó/limpió el catálogo con ese criterio?
**Respuesta**: "b" — no, nunca se ha hecho.
**Estado**: RESPONDIDA (decide).
**Efecto**: confirma que `STRICT_OPTION_SELECTION` debe permanecer en `False`; refuerza el
riesgo residual de A-06 y A-32.

### P19 — A-31 — Necesidad de conversión de unidades entre dimensiones distintas
**Dirigida a**: administrador de catálogo/recetas.
**Contexto dado**: existe un módulo de conversión de unidades (ej. gramos↔mililitros) sin
terminar de construir y sin uso hoy.
**Pregunta**: ¿Necesitan convertir entre unidades de distinto tipo en una receta?
**Respuesta 1**: "a" — sí, la necesitan.
**Repregunta** (ejemplo real): "para el caso del granizado tenemos un cliente que compra la
materia prima en litros, pero vende el producto en vasos de onzas".
**Estado**: RESPONDIDA (decide, con ejemplo real).
**Efecto**: cambia el tratamiento de A-31: ya no es candidato a retirar, sino a **completar la
migración**. Nota técnica: litros↔onzas es, en rigor, conversión dentro de la misma dimensión
(volumen), más simple que masa↔volumen — conviene confirmar si además necesitan conversión
entre dimensiones realmente distintas.

### P20 — A-26 — Mover orden a mesa con orden activa
**Dirigida a**: mesero/supervisor de sala.
**Contexto dado**: mover un pedido a una mesa con otro pedido activo está bloqueado, más
estricto que el resto del sistema.
**Pregunta**: ¿Alguna vez el sistema les bloqueó mover un pedido a una mesa con otro pedido en
curso?
**Respuesta**: "c" — no sabría decir / no usan la función de mover pedidos.
**Estado**: **SIN RESPUESTA CONCLUYENTE.**
**Efecto**: A-26 se mantiene sin cambio de clasificación; el dato sugiere baja prioridad
práctica (no usan la función).

### P21 — A-29 — Trazabilidad de combos múltiples en reportes
**Dirigida a**: dueño/gerente (uso de reportes).
**Contexto dado**: con 2+ combos distintos en una venta, no queda registrado ningún combo
específico en el reporte de promociones.
**Pregunta**: ¿Usan algún reporte de ventas por combo/promoción?
**Respuesta**: "b" — no, no lo revisan.
**Estado**: RESPONDIDA (decide).
**Efecto**: queda sin impacto práctico hoy; documentar sin especificar y despriorizar.

### P22 — A-33 — Grupo opcional único bloquea venta si nadie elige
**Dirigida a**: administrador de catálogo.
**Contexto dado**: un producto con un grupo de opciones "opcional" que es su única fuente de
consumo bloquea la venta si nadie elige nada (ejemplo: cono simple sin sabor elegido).
**Pregunta**: ¿Debería venderse igual, o el sistema tiene razón en bloquear?
**Respuesta**: "b" — el sistema tiene razón en bloquear.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-33 pasa a INTENCIONAL confirmado — el bloqueo actual es correcto, se especifica
tal cual.

### P23 — A-35 — Motivo de ajuste obligatorio; costo promedio vs último
**Dirigida a**: encargado de inventario/compras.
**Preguntas y respuestas**:
1. ¿El motivo de un ajuste manual debería ser obligatorio? → **"debería ser obligatorio."**
2. ¿Prefieren costo de última compra o promedio ponderado? → **"el costo de la última
   compra."**
**Estado**: RESPONDIDA (decide, ambas partes).
**Efecto**: RN-INV-11 pasa a requisito de negocio confirmado (obligatorio); RN-INV-17 pasa a
INTENCIONAL confirmado (el comportamiento actual, "último costo", es el deseado).

### P24 — A-27 — CI del backend / proceso de publicación
**Dirigida a**: soporte técnico/TI.
**Contexto dado**: un spec de reportes del frontend estuvo roto sin detectar hasta que alguien
lo notó a mano; el CI del backend solo corre 1 de 12 scripts de test.
**Pregunta**: ¿Alguien revisa antes de publicar, o se publica directo?
**Respuesta 1**: "a" — sí, alguien revisa antes de publicar.
**Repregunta** (mecanismo): "actualmente es una revisión manual, pero me gustaría correr
pruebas automáticas".
**Estado**: RESPONDIDA (decide, tras repregunta).
**Efecto**: la revisión manual reduce pero no elimina el riesgo. Petición explícita del negocio
de pasar a pruebas automáticas — sube la prioridad de A-27 con mandato directo.

### P25 — A-42 — Módulo de Horarios retirado / consumo de auditoría
**Dirigida a**: dueño/gerente.
**Preguntas y respuestas**:
1. ¿Horarios se retiró por decisión de producto o problema técnico? → **"se retiró la pestaña
   de horarios porque ya no se necesitaba."**
2. ¿Alguien consulta el registro de auditoría hoy? → **"No, ya no lo revisamos."**
**Estado**: RESPONDIDA (decide, ambas partes).
**Efecto**: Horarios — decisión deliberada, habilita migración de borrado de la tabla huérfana.
Auditoría — confirma que nadie consulta `audit_logs` hoy, ni por UI ni por otra vía.

### P26 — A-46 — Planes de multi-tenant con zonas horarias distintas
**Dirigida a**: dueño/gerente.
**Pregunta**: ¿Hay planes de expansión a otra zona horaria?
**Respuesta**: "b" — no, seguirán solo en la zona horaria actual.
**Estado**: RESPONDIDA (decide).
**Efecto**: confirmado sin urgencia; se documenta la limitación sin especificar corrección.

### P27 — A-47 — Pedido rechazado tarde en hora pico (stock no reservado)
**Dirigida a**: dueño/gerente o cajero jefe.
**Contexto dado**: el sistema no reserva stock al armar el carrito; en hora pico un pedido puede
rechazarse al confirmar aunque se mostró como disponible.
**Pregunta**: ¿Han tenido quejas de comensales por esto en horas de mucha gente?
**Respuesta**: "c" — no sabría decir.
**Estado**: **SIN RESPUESTA CONCLUYENTE.**
**Efecto**: A-47 se mantiene PENDIENTE sin cambio.

### P28 — A-48 — KDS: ¿descartado o fusionado a propósito con terminal de mesas?
**Dirigida a**: dueño/gerente o jefe de cocina.
**Contexto dado**: la pantalla separada de cocina (KDS) se retiró tres semanas después de
describirse como plan vigente en el CHANGELOG.
**Pregunta**: ¿Fue por decidir que un solo dispositivo funciona mejor, o porque ya no se
necesitaba el concepto?
**Respuesta**: "a" — decidieron que un solo dispositivo/terminal compartido funciona mejor.
**Estado**: RESPONDIDA (decide).
**Efecto**: A-48 pasa a INTENCIONAL confirmado — decisión operativa deliberada, se especifica
tal cual.

---

## 4. Información nueva no solicitada (condiciones, prohibiciones, datos)

Estas son afirmaciones que el negocio hizo por su cuenta, sin que se le preguntara
directamente por ellas, y que tienen consecuencia sobre el alcance de futuras specs:

1. **PROHIBICIÓN (P5)**: el cajero no debería poder aplicar descuento manual en absoluto —
   condición explícita que redefine el alcance de la corrección de A-11: no basta con un tope
   numérico, hay que retirar o restringir el campo de descuento manual para el rol cajero.
   Pendiente de aclarar si aplica igual a mostrador, mesa unificada y dividida.
2. **DATO (P8)**: estimación de "varios" (6-20) productos activos sin receta configurada —
   cifra aproximada, no verificada; candidata directa a una consulta SQL antes de activar
   cualquier bloqueo de venta-sin-receta.
3. **DATO (P10)**: el consecutivo de facturación se reinicia cada año — dato operativo no
   documentado en ningún fichero de reconocimiento hasta ahora. **Posible discrepancia
   código-vs-práctica**: el reconocimiento original no encontró ningún mecanismo de reinicio en
   `InvoiceCounter` — si el reinicio es real, alguien lo hace manualmente por fuera del sistema,
   o el reconocimiento original no vio ese código. Requiere verificación técnica.
4. **REQUISITO (P14)**: el cierre de caja original nunca debe modificarse/sobrescribirse una
   vez hecho, aunque se anulen ventas después — cualquier efecto de esas anulaciones debe verse
   en una vista separada, no reemplazando el número original. Aplica como principio general de
   auditoría de caja.
5. **PETICIÓN (P24)**: deseo explícito de incorporar pruebas automáticas al proceso de
   publicación de cambios — tratar como requisito adicional en la fase de modernización.
6. **DATO (P19)**: caso real y concreto de necesidad de conversión de unidades (litros de
   materia prima → onzas de venta) para el producto "granizado" — candidato a primer caso de
   prueba si se retoma la funcionalidad de conversión de unidades.
7. **DATO (P11/P17)**: confirma el patrón operativo "un dispositivo/una persona por mesa a la
   vez" como práctica general del local — relevante como contexto para varias preguntas de
   concurrencia (A-17, A-26).
8. **DATO (P25)**: no hay ningún proceso externo (revisión manual de base de datos, por
   ejemplo) que use el registro de auditoría hoy — confirma que `audit_logs` se escribe sin
   ningún lector.

---

## 5. Pendiente para próxima ronda

Esta sección se actualizó al propagar esta acta al [registro de anomalías](./registro-de-anomalias.md)
(2026-08-16) y de nuevo tras la segunda ronda del mismo día (sección 6), que cerró las 5
preguntas que antes vivían aquí. Lo que queda son preguntas que surgieron al escribir esa
propagación, más una que quedó fuera del guion original por alcance. Mientras esta sección no
quede vacía, es la agenda de la siguiente conversación con el negocio.

1. **A-49 (nueva)** — el negocio afirmó en P10 que el consecutivo de facturación se reinicia
   cada año, pero ningún fichero de reconocimiento encontró ese mecanismo en el código. Antes de
   repreguntar, corresponde una verificación técnica (¿existe algún script o procedimiento real
   fuera del repositorio?); si no aparece nada, la pregunta de negocio pendiente es: ¿quién
   ejecuta el reinicio, cómo (manual, script no versionado, automático), y qué garantiza que
   ocurra todos los años sin falta?
2. **A-11, alcance** — el negocio prohibió el descuento manual del cajero "en absoluto" (P5),
   pero no se le preguntó si esa prohibición aplica igual a los tres caminos de cobro
   (mostrador, mesa unificada, mesa dividida) o solo al que tenía en mente al responder.
3. **A-31, alcance** — el ejemplo real que dio el negocio (litros→onzas del granizado, P19) es,
   en rigor, conversión dentro de la misma dimensión (volumen). Falta confirmar si además
   necesitan alguna vez convertir entre dimensiones realmente distintas (p. ej. masa↔volumen,
   gramos↔mililitros), que es el caso que de verdad justificaría completar `core/units.py` tal
   como está diseñado hoy.
4. **A-22 (fuera de guion, no nueva pero sigue sin cubrirse)** — rate-limiting de
   `/auth/login`, refresh que sobrevive al logout, truncamiento de contraseñas a 72 bytes.
   Quedó fuera de las 28 preguntas por un descuido de alcance, no por decisión del negocio —
   debe incluirse en el guion de la próxima ronda, no solo repreguntarse si sale espontáneamente.

---

## 6. Segunda ronda — cierre de las 5 preguntas sin respuesta concluyente (2026-08-16)

Mismo día de la primera ronda. Método: para cada pregunta que había quedado `SIN RESPUESTA
CONCLUYENTE`, se repreguntó cambiando el enfoque — en vez de repetir la pregunta de hecho ("¿ha
pasado esto?", que ya había recibido "no sabría decir"), se pidió una decisión de tratamiento
("dado que no lo sabemos, ¿cómo procedemos?"). Formato cerrado, sin sugerir la respuesta, igual
que la primera ronda. Las 5 decidieron la cuestión.

### P7-bis — A-06 — Tratamiento de la opción de grupo ajeno mientras no hay datos
**Dirigida a**: dueño/gerente (repregunta; la original iba a administrador de catálogo, que no
pudo decidir por no revisar tickets a ese detalle).
**Contexto dado**: con la tolerancia de catálogo activa, el sistema permite cobrar y descontar
inventario de una opción de un grupo que el producto vendido no ofrece (ej.: un cono simple con
"extra de pepperoni" cobrado y descontado, aunque el cono no tenga ese grupo configurado).
**Pregunta**: sin poder revisar tickets a mano, ¿cómo prefiere que se trate esto mientras no
haya más información? (pedir consulta a datos / aceptar el riesgo por ahora / corregir
directamente / seguir sin poder decidir).
**Respuesta**: "Aceptar el riesgo por ahora" — mientras el catálogo no esté depurado (igual que
A-05), dejarlo así, no es prioridad.
**Estado**: RESPONDIDA (decide el tratamiento; no resuelve si el comportamiento original era
"residuo aceptado" o "vector no previsto" — el negocio no distinguió entre ambas, decidió cómo
proceder en su lugar).
**Efecto**: A-06 mantiene su clasificación `PENDIENTE` en sentido estricto (no reúne los 2/3
testigos que exige `INTENCIONAL`), pero su "decisión de negocio pendiente" queda cerrada: no se
prioriza corrección ni consulta a datos; se trata con el mismo criterio ya aceptado en A-05.

### P14-bis — RN-CASH-13 — ¿Debe el arqueo parcial exigir motivo si no cuadra?
**Dirigida a**: dueño/gerente (repregunta; la original iba a cajero jefe, que no lo había
probado en la práctica).
**Contexto dado**: el arqueo parcial (revisar el efectivo sin cerrar el turno) no exige que el
cajero explique una diferencia (ej. $180.000 esperados, $165.000 contados); el cierre real sí
exige esa nota.
**Pregunta**: ¿debería el arqueo parcial exigir la misma explicación obligatoria que el cierre
final cuando hay diferencia, o son casos distintos a propósito?
**Respuesta**: "Sí, exigir explicación igual."
**Estado**: RESPONDIDA (decide).
**Efecto**: RN-CASH-13 pasa de `DUDOSA`/`PENDIENTE` a **requisito de negocio confirmado**: el
arqueo parcial debe exigir una nota obligatoria cuando la diferencia sea distinta de cero,
igual que el cierre real. Se suma al requisito ya confirmado de RN-CASH-17 (snapshot inmutable,
P14 primera ronda) como parte del mismo endurecimiento del módulo de caja.

### P16-bis — A-16 — ¿Cocina y cobro se cruzan en la práctica?
**Dirigida a**: dueño/gerente (repregunta; la original iba a jefe de cocina, que no sabría
decir).
**Contexto dado**: cocina puede seguir marcando ítems "listo" en un pedido que el cajero ya está
cobrando, sin aviso del sistema (ej.: cajero cerrando la mesa 4 mientras cocina marca "listo" un
helado de esa misma mesa).
**Pregunta**: pensando en el ritmo de trabajo diario, ¿le parece que esto podría pasar, o el
flujo de trabajo hace prácticamente imposible que coincidan esos momentos?
**Respuesta**: "No, es prácticamente imposible."
**Estado**: RESPONDIDA (decide).
**Efecto**: cierra la porción `PENDIENTE` de A-16 (RN-ORD-37) — mitigado por el ritmo de trabajo
actual, mismo patrón que A-01/A-09/A-17 (riesgo de código sin corregir, pero sin incidente real
porque la operación no lo dispara). La porción ya `ACCIDENTAL` de A-16 (RN-ORD-38/39,
inconsistencia de código verificable frente a `mark_order_ready`) no cambia — sigue siendo
recomendable corregir en fase de modernización, sin urgencia operativa.

### P20-bis — A-26 — ¿Usan mover pedidos entre mesas?
**Dirigida a**: dueño/gerente (repregunta; la original iba a mesero/supervisor, que no sabría
decir o no usa la función).
**Contexto dado**: `move_order` bloquea mover un pedido a una mesa con otro pedido activo, más
estricto que el resto del sistema (que sí permite varias órdenes por mesa).
**Pregunta**: confirmando lo anterior, ¿el equipo usa alguna vez la función de mover un pedido
de una mesa a otra, o simplemente no se usa en el local?
**Respuesta**: "No la usamos."
**Estado**: RESPONDIDA (decide).
**Efecto**: cierra A-26 (RN-ORD-58) como **riesgo latente, no activo** — mismo patrón que A-01:
el código restringe algo que nadie ejerce en la práctica. El resto de A-26 (RN-ORD-60 manejador
huérfano, RN-ORD-63 no determinismo de `merge_orders`), ya `ACCIDENTAL` confirmado por código,
no cambia.

### P27-bis — A-47 — ¿Reservar stock preventivamente o mantener el diseño actual?
**Dirigida a**: dueño/gerente (repregunta; la original iba a dueño/cajero jefe, que no sabría
decir si hubo quejas).
**Contexto dado**: el chequeo de disponibilidad al armar el carrito es "best-effort" — no
reserva stock; en hora pico dos comensales pueden ver el mismo ítem disponible y solo uno
confirmarlo (ej.: quedan 3 conos de mango, dos los ven disponibles, el segundo en confirmar se
queda sin el sabor).
**Pregunta**: si tuviera que elegir hoy entre invertir en reservar stock preventivamente (más
complejo) o dejarlo como está (más simple, con el costo ocasional de un pedido rechazado tarde),
¿qué preferiría?
**Respuesta**: "Dejarlo como está."
**Estado**: RESPONDIDA (decide).
**Efecto**: A-47 pasa a **`INTENCIONAL` confirmado** (2/3 testigos: CÓDIGO + NEGOCIO, mismo
patrón que A-05/A-15/A-21) — el diseño best-effort es una decisión de negocio deliberada, no
solo una caracterización del propio código. Se especifica tal cual: no reservar stock
preventivamente, aceptando el costo ocasional de un pedido rechazado tarde en hora pico.

---

## 7. Próximos pasos sugeridos

1. Verificar técnicamente la afirmación del punto 3 de la sección 4 (reinicio anual del
   consecutivo de facturación) contra el código real de `InvoiceCounter` — registrada como
   anomalía nueva **A-49** en el registro de anomalías. Este paso no depende del negocio y
   debería resolverse antes de repreguntar en la próxima ronda.
2. Correr (o encargar) la consulta SQL que confirme cuántos productos activos de mostrador no
   tienen receta configurada (dimensiona A-12, hoy solo estimado en "6-20").
3. ~~Actualizar el [registro de anomalías](./registro-de-anomalias.md) con las clasificaciones
   resueltas por esta entrevista~~ — hecho (2026-08-16): ver las tablas "Actualización —
   entrevista de negocio" y "Actualización — segunda ronda" al inicio de ese documento, más la
   nueva entrada A-49.
4. ~~Programar una segunda ronda breve para las 5 preguntas sin respuesta concluyente~~ — hecho
   el mismo día (sección 6): las 5 decidieron la cuestión. Queda pendiente una tercera ronda,
   más corta, para las 4 preguntas de la sección 5 (A-49, alcance de A-11, alcance de A-31,
   A-22 fuera de guion), cuando haya oportunidad de observar la operación directamente o revisar
   datos. El registro de anomalías confirma que, mientras esa lista no esté vacía, el
   reconocimiento no está listo para pasar a specs formales sobre
   las áreas que toca.
