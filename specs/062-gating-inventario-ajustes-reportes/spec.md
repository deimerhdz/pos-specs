# Feature Specification: Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado

**Feature Branch**: `062-gating-inventario-ajustes-reportes`

**Created**: 2026-08-31

**Status**: Draft

**Input**: User description: "las unidades de medida tiene relacion directa con el inventario, si el tenant no tiene el modulo de inventario habilidado, no deberia tener visible esa opcion en los ajustes y lo mismo con los reportes, no deberia de aparecer los reportes relacionados con insumos del inventario, como los insumos del stock y no estoy seguro si el margen tambien aplicaria a esa regla, analizalo"

> **Nota de alcance**: esta spec extiende la regla de acceso a módulos por plan ya
> establecida en la spec 033 (Planes de Suscripción por Tenant) — donde el módulo
> Inventario, al no estar incluido en el plan de un tenant, "desaparece de la
> navegación y no se puede ni siquiera consultar" — a dos superficies que esa spec
> no cubrió explícitamente: la pestaña "Unidades de medida" dentro de Ajustes, y la
> sección de reportes que depende de datos de inventario (insumos con stock bajo).
> También resuelve, con análisis dedicado, si el indicador de Margen debe seguir la
> misma regla.

## Clarifications

### Session 2026-08-31

- Q: Cuando un Tenant Admin sin el módulo Inventario intenta entrar directamente
  por URL a "Unidades de medida" (una pestaña anidada dentro de Ajustes, a
  diferencia de Inventario/Promociones que son pantallas de nivel superior),
  ¿a dónde lo redirige el sistema? → A: A `/dashboard` — mismo guard genérico y
  mismo destino ya usados para Inventario y Promociones (spec 033), sin
  tratamiento especial por estar anidada dentro de Ajustes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Unidades de medida desaparece de Ajustes sin el módulo Inventario (Priority: P1)

Como Tenant Admin cuyo plan no incluye el módulo Inventario, al entrar a Ajustes no
quiero ver la pestaña "Unidades de medida" — es una configuración que solo tiene
sentido para dar de alta insumos de inventario, y verla sin poder usarla para nada
es confuso y contradice la promesa de que un módulo no incluido "desaparece por
completo" (spec 033).

**Why this priority**: Es el caso concreto que motivó esta spec y el más simple de
verificar: una opción de configuración que no debería estar visible lo está.

**Independent Test**: Con un tenant en un plan sin acceso a Inventario, el Tenant
Admin abre Ajustes y verifica que la pestaña "Unidades de medida" no aparece en la
lista de pestañas, y que entrar por URL directa a esa pantalla tampoco funciona.

**Acceptance Scenarios**:

1. **Given** el plan vigente de un tenant no incluye el módulo Inventario, **When**
   el Tenant Admin abre la pantalla de Ajustes, **Then** la pestaña "Unidades de
   medida" no aparece entre las pestañas disponibles.
2. **Given** el plan vigente de un tenant no incluye el módulo Inventario, **When**
   el Tenant Admin intenta entrar directamente (URL, marcador, enlace guardado) a
   la pantalla de Unidades de Medida, **Then** el sistema lo redirige a
   `/dashboard` — mismo destino y mecanismo ya usados para el resto de pantallas
   de un módulo no incluido (spec 033, FR-008/FR-009) — no una pantalla en blanco
   ni un error técnico (Clarifications, sesión 2026-08-31).
3. **Given** el plan vigente de un tenant sí incluye el módulo Inventario, **When**
   el Tenant Admin abre Ajustes, **Then** la pestaña "Unidades de medida" aparece y
   funciona con normalidad, sin ninguna restricción adicional.
4. **Given** un tenant tenía unidades de medida creadas antes de que se le quitara
   el acceso a Inventario, **When** se le retira el acceso, **Then** esas unidades
   de medida no se borran — solo dejan de ser visibles/editables hasta que se le
   reasigne el acceso (mismo criterio de no pérdida de datos que spec 033).

---

### User Story 2 - Los reportes de insumos de inventario desaparecen sin el módulo Inventario (Priority: P1)

Como Tenant Admin cuyo plan no incluye el módulo Inventario, al abrir la pantalla
de Reportes no quiero ver la sección de insumos con stock bajo — es información
de un módulo al que no tengo acceso, y mostrarla contradice el mismo principio de
la Historia 1.

**Why this priority**: Mismo nivel de urgencia que la Historia 1 — es la segunda
mitad explícita del pedido del usuario, y expone el mismo tipo de dato
(existencias de insumos) que el módulo Inventario ya protege en su propia
pantalla.

**Independent Test**: Con un tenant en un plan sin acceso a Inventario, el Tenant
Admin abre Reportes y verifica que la sección "Insumos con stock bajo" (y su
enlace a "Gestionar insumos") no aparece en ningún lugar de la pantalla.

**Acceptance Scenarios**:

1. **Given** el plan vigente de un tenant no incluye el módulo Inventario, **When**
   el Tenant Admin abre Reportes, **Then** la sección "Insumos con stock bajo" no
   aparece en la pantalla.
2. **Given** el plan vigente de un tenant sí incluye el módulo Inventario, **When**
   el Tenant Admin abre Reportes, **Then** la sección "Insumos con stock bajo" se
   muestra con normalidad, igual que hoy.
3. **Given** el plan vigente de un tenant no incluye el módulo Inventario, **When**
   cualquier otra parte del sistema (no solo la pantalla de Reportes) intenta
   consultar ese mismo dato de insumos con stock bajo, **Then** la consulta se
   deniega con el mismo criterio que el resto de datos de un módulo no incluido
   — no solo se oculta en una pantalla y queda accesible por otra vía.

---

### User Story 3 - El indicador de Margen refleja que depende de datos de Inventario (Priority: P2)

Como Tenant Admin cuyo plan no incluye el módulo Inventario, cuando abro Reportes
no quiero ver un indicador de "Margen" que en realidad no puede calcularse sin
datos de inventario, para no tomar decisiones de negocio basadas en un número
incorrecto.

**Why this priority**: Es la parte que el propio pedido marca como incierta. El
análisis (ver más abajo) confirma que el cálculo de Margen depende enteramente de
costos de insumos de inventario, así que mostrarlo sin ese módulo no es solo una
cuestión de acceso — es un número engañoso. Es P2 porque, a diferencia de las
Historias 1 y 2, no es una fuga de datos de otro módulo sino una métrica que da un
resultado sin sentido; el impacto es de calidad de información, no de control de
acceso.

**Análisis (motivado por el pedido "no estoy seguro si el margen también
aplicaría a esa regla, analizalo")**: el indicador de Margen de Reportes se
calcula como ingresos menos costo de lo vendido, y ese costo se obtiene
exclusivamente de las recetas de producto y el costo de los insumos de
inventario. Un tenant sin el módulo Inventario no tiene forma de cargar ni
mantener esos datos, así que su costo calculado es siempre cero — el sistema
mostraría "Margen = 100% de los ingresos", una cifra falsa, no un simple dato
faltante. Por lo tanto, el Margen **sí debe tratarse bajo la misma regla que las
Historias 1 y 2**: un tenant sin Inventario no debe ver un valor de Margen
calculado a partir de datos de un módulo al que no tiene acceso.

**Independent Test**: Con un tenant en un plan sin acceso a Inventario, el Tenant
Admin abre Reportes y verifica que la tarjeta de "Margen" no aparece entre las
métricas de la pantalla (decisión tomada tras el análisis: se oculta por
completo, mismo criterio que las Historias 1 y 2 — ver FR-007).

**Acceptance Scenarios**:

1. **Given** el plan vigente de un tenant no incluye el módulo Inventario, **When**
   el Tenant Admin abre Reportes, **Then** la tarjeta de "Margen" no aparece entre
   las métricas de la pantalla.
2. **Given** el plan vigente de un tenant sí incluye el módulo Inventario, **When**
   el Tenant Admin abre Reportes, **Then** la tarjeta de "Margen" se calcula y
   muestra con normalidad, igual que hoy.
3. **Given** un tenant pierde el acceso a Inventario teniendo ventas históricas
   con margen ya calculado, **When** consulta un período de reportes anterior a la
   pérdida de acceso, **Then** el sistema aplica el mismo criterio de esta
   historia (la tarjeta de Margen no aparece) — consistente con que el módulo "no
   se puede ni siquiera consultar" mientras no se reasigne el acceso.

---

### Edge Cases

- Un tenant tiene el módulo Inventario incluido en su plan pero su plan está
  vencido (spec 033, Historia 5): se aplica el mismo bloqueo que si el módulo no
  estuviera incluido — Unidades de Medida, insumos con stock bajo y Margen se
  ocultan igual que con `inventario_access = false`.
- Un tenant nunca tuvo Inventario habilitado (nunca cargó insumos ni recetas): las
  tres superficies de esta spec se comportan igual que para un tenant al que se le
  retiró el acceso — no hay un caso especial de "nunca tuvo".
- Al Super Admin le reasigna a un tenant el acceso a Inventario: "Unidades de
  medida" reaparece en Ajustes, la sección de insumos con stock bajo reaparece en
  Reportes, y el Margen vuelve a calcularse con normalidad — todo de inmediato,
  sin que el Tenant Admin tenga que hacer nada (mismo criterio de aplicación
  inmediata que spec 033, FR-014).
- Un tenant sin Inventario intenta llegar a Unidades de Medida o a los datos de
  insumos con stock bajo por una vía que no es la pantalla (por ejemplo, un
  enlace guardado, una llamada directa, u otra pantalla que hoy los referencie):
  se le niega igual — el criterio de esta spec no es "ocultar el enlace" sino
  "el módulo no se puede ni siquiera consultar", igual que ya exige la spec 033.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST ocultar la pestaña "Unidades de medida" de la
  pantalla de Ajustes cuando el plan vigente del tenant no incluye el módulo
  Inventario, o cuando el tenant está vencido (mismo criterio que spec 033,
  FR-008/FR-009/FR-019).
- **FR-002**: El sistema MUST denegar el acceso a la pantalla de Unidades de
  Medida cuando se solicita directamente (no solo ocultar la pestaña),
  redirigiendo a `/dashboard` — mismo mecanismo y destino ya usados para el resto
  de pantallas de un módulo no incluido, sin tratamiento especial por estar
  anidada dentro de Ajustes (Clarifications, sesión 2026-08-31).
- **FR-003**: El sistema MUST denegar cualquier operación de lectura o escritura
  sobre unidades de medida (crear, editar, listar, eliminar) cuando el plan
  vigente del tenant no incluye el módulo Inventario o está vencido — la
  restricción MUST aplicar más allá de la pantalla, en el mismo punto donde ya se
  valida el acceso a otras funciones de Inventario (spec 033, FR-008).
- **FR-004**: El sistema MUST conservar las unidades de medida ya creadas por un
  tenant al que se le retira el acceso a Inventario — no se borran, solo dejan de
  ser accesibles hasta que se reasigne el acceso (mismo criterio que spec 033,
  Edge Cases).
- **FR-005**: El sistema MUST ocultar la sección de "insumos con stock bajo" de
  la pantalla de Reportes cuando el plan vigente del tenant no incluye el módulo
  Inventario, o cuando el tenant está vencido.
- **FR-006**: El sistema MUST denegar cualquier consulta al dato subyacente de
  insumos con stock bajo (no solo ocultar la sección en Reportes) cuando el plan
  vigente del tenant no incluye el módulo Inventario o está vencido.
- **FR-007**: El sistema MUST ocultar por completo la tarjeta de "Margen" de
  Reportes (no solo el número, la tarjeta entera) cuando el plan vigente del
  tenant no incluye el módulo Inventario, o cuando el tenant está vencido — mismo
  criterio de ocultamiento total que FR-001 y FR-005, decidido tras el análisis:
  sin datos de inventario el costo calculado es siempre cero, así que un número
  de margen sería una cifra falsa, no un dato simplemente ausente.
- **FR-008**: Cuando el Super Admin reasigna a un tenant el acceso al módulo
  Inventario (o renueva su plan vencido), el sistema MUST restaurar de inmediato
  la visibilidad y el funcionamiento normal de Unidades de Medida, la sección de
  insumos con stock bajo, y el cálculo de Margen — sin requerir ninguna acción
  adicional del Tenant Admin (mismo criterio de aplicación inmediata que spec
  033, FR-014/FR-020).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los tenants cuyo plan no incluye el módulo Inventario (o
  cuyo plan está vencido) no ven la pestaña "Unidades de medida" en Ajustes.
- **SC-002**: El 100% de los intentos de esos tenants de entrar directamente a la
  pantalla de Unidades de Medida, o de leer/escribir una unidad de medida por
  cualquier otra vía, son denegados.
- **SC-003**: El 100% de los tenants cuyo plan no incluye el módulo Inventario (o
  cuyo plan está vencido) no ven la sección de insumos con stock bajo en
  Reportes, ni pueden obtener ese dato por ninguna otra vía.
- **SC-004**: El 100% de los tenants cuyo plan no incluye el módulo Inventario (o
  cuyo plan está vencido) no ven la tarjeta de Margen entre las métricas de
  Reportes.
- **SC-005**: Cuando el Super Admin reasigna el módulo Inventario a un tenant (o
  renueva su plan), las tres superficies de esta spec vuelven a funcionar con
  normalidad en el siguiente intento de acceso, sin que el Tenant Admin cierre
  sesión ni espere un proceso posterior.

## Assumptions

- El módulo Inventario, sus reglas de acceso por plan, y los mensajes de
  denegación ya existentes (spec 033) son la base de esta spec — no se redefine
  aquí qué significa "tener el módulo incluido", solo se extiende dónde aplica
  esa regla.
- "Unidades de medida" es, en el sistema actual, una configuración usada
  exclusivamente para dar de alta y gestionar insumos de inventario — no la usa
  ninguna otra parte del catálogo (productos, variantes, grupos de opciones). Si
  en el futuro otra función pasara a depender de ella, dejaría de tener sentido
  ocultarla por completo y esta spec debería revisarse.
- La sección de "insumos con stock bajo" de Reportes es la única parte de esa
  pantalla que depende de datos de inventario; el resto de reportes (ventas,
  productos, categorías, cajeros, ticket promedio) no se ve afectado por esta
  spec y sigue disponible sin el módulo Inventario.
- El desglose de rentabilidad por categoría que acompaña al Margen (si una
  categoría no tiene insumos de inventario asociados) sigue el mismo tratamiento
  que se decida para la tarjeta de Margen general (FR-007) — no se trata como un
  caso aparte.
- Los roles y permisos existentes (Tenant Admin, Super Admin) no cambian con esta
  spec; solo se agregan superficies nuevas a la regla de acceso por plan que ya
  gobierna esos roles.
