# Feature Specification: Orden personalizado de categorías en el filtro del menú QR

**Feature Branch**: `067-orden-categorias-menu-qr`

**Created**: 2026-09-01

**Status**: Draft

**Input**: User description: "necesito que al modulo de categorias puedas definirle la opcion
establecer un orden y de acuerdo a ese orden se muestren en el filtro del menu qr de mayor a
menor, ese orden debe definirse en el formulario de creacion y edicion"

## Clarifications

### Session 2026-09-01

- Q: Al desplegar este feature, ¿qué valor de orden deben recibir las categorías que ya existían antes (creadas sin este campo)? → A: Asignar valores que reproduzcan el orden que las categorías tienen hoy en el filtro (p. ej. secuenciales según su orden de creación), para que la secuencia visible no cambie al desplegar.
- Q: Cuando un administrador crea una categoría nueva sin especificar un valor de orden, ¿en qué posición del filtro del menú QR debe aparecer por defecto? → A: Al principio — se le asigna el valor más alto disponible (máximo actual + 1), de modo que aparece de primera en el filtro apenas se crea.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Definir el orden al crear o editar una categoría (Priority: P1)

Como administrador del negocio, al crear o editar una categoría quiero poder asignarle un valor de
orden, para controlar en qué posición aparece esa categoría dentro del filtro de categorías del
menú QR que ven los comensales.

**Why this priority**: Sin la posibilidad de definir el orden desde el formulario, no existe forma
de que el resto de la funcionalidad (mostrar el filtro ordenado) tenga ningún efecto. Es el punto
de entrada obligatorio de todo el feature.

**Independent Test**: Puede probarse creando una categoría nueva asignándole un valor de orden, y
editando una categoría existente para cambiar su valor de orden, verificando en ambos casos que el
valor queda guardado.

**Acceptance Scenarios**:

1. **Given** el formulario de creación de categoría, **When** el administrador ingresa un nombre y
   un valor de orden y guarda, **Then** la categoría se crea con ese valor de orden asociado.
2. **Given** una categoría existente, **When** el administrador abre su formulario de edición y
   cambia el valor de orden, **Then** el nuevo valor queda guardado y sustituye al anterior.
3. **Given** el formulario de creación de categoría, **When** el administrador deja el campo de
   orden vacío y guarda, **Then** el sistema le asigna automáticamente el valor más alto disponible
   (máximo actual + 1), de modo que la categoría aparece de primera en el filtro del menú QR.
4. **Given** el formulario de creación o edición, **When** el administrador ingresa un valor de
   orden no numérico o negativo, **Then** el sistema rechaza el guardado y muestra un mensaje de
   error claro.

---

### User Story 2 - Ver el filtro del menú QR ordenado de mayor a menor (Priority: P1)

Como comensal que escanea el QR de la mesa, quiero que el filtro de categorías del menú se muestre
según el orden configurado por el negocio (de mayor a menor valor de orden), para encontrar más
fácilmente las categorías que el negocio considera más relevantes.

**Why this priority**: Es el valor de negocio central del feature: sin este comportamiento, definir
un orden en el formulario no tiene ningún efecto visible para el comensal.

**Independent Test**: Puede probarse configurando varias categorías activas con distintos valores de
orden y verificando que el filtro del menú QR las lista de mayor a menor valor.

**Acceptance Scenarios**:

1. **Given** tres categorías activas con valores de orden 10, 5 y 1, **When** el comensal abre el
   filtro de categorías del menú QR, **Then** las categorías se listan en la secuencia 10, 5, 1.
2. **Given** dos categorías activas con el mismo valor de orden, **When** se muestran en el filtro,
   **Then** se ordenan entre sí alfabéticamente por nombre para mantener una secuencia consistente.
3. **Given** una categoría inactiva con un valor de orden alto, **When** se genera el filtro del
   menú QR, **Then** esa categoría no aparece, respetando la regla existente de que solo se listan
   categorías activas.
4. **Given** un cambio reciente en el valor de orden de una categoría, **When** un comensal carga o
   recarga el menú QR, **Then** el filtro refleja el nuevo orden sin pasos adicionales de
   publicación por parte del negocio.

---

### User Story 3 - Ver el orden configurado desde la administración de categorías (Priority: P3)

Como administrador, quiero ver el valor de orden asignado a cada categoría en el listado de gestión
de categorías, para poder revisar y ajustar la secuencia sin tener que abrir el menú QR para
comprobarla.

**Why this priority**: Es una mejora de usabilidad para quien administra varias categorías, pero no
es indispensable para que el filtro del menú QR funcione ordenado; el negocio puede validar el
resultado directamente en el menú QR si esta historia no se entrega primero.

**Independent Test**: Puede probarse abriendo el listado de administración de categorías y
verificando que el valor de orden de cada categoría es visible junto a sus demás datos.

**Acceptance Scenarios**:

1. **Given** el listado de administración de categorías, **When** el administrador lo abre,
   **Then** cada categoría muestra su valor de orden actual.

---

### Edge Cases

- ¿Qué ocurre cuando dos o más categorías comparten el mismo valor de orden? Se desempata
  ordenándolas alfabéticamente por nombre.
- ¿Qué ocurre con las categorías creadas antes de este feature, que no tienen un valor de orden
  definido? El sistema les asigna un valor de orden que reproduce la secuencia que ya tienen hoy en
  el filtro del menú QR, para que su posición no cambie con el despliegue.
- ¿Qué ocurre si el administrador deja el campo de orden vacío al crear una categoría? El sistema le
  asigna el valor más alto disponible (máximo actual + 1), de modo que aparece de primera en el
  filtro, en lugar de bloquear el guardado.
- ¿Qué ocurre si solo existe una categoría activa? Se muestra sola en el filtro, sin importar su
  valor de orden.
- ¿Qué ocurre si una categoría con un valor de orden alto se marca como inactiva? Deja de aparecer
  en el filtro del menú QR, igual que cualquier otra categoría inactiva.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El formulario de creación de categoría DEBE incluir un campo para definir un valor de
  orden numérico, usado para determinar la posición de la categoría en el filtro del menú QR.
- **FR-002**: El formulario de edición de categoría DEBE permitir modificar el valor de orden de una
  categoría existente.
- **FR-003**: El sistema DEBE validar que el valor de orden ingresado sea un número entero no
  negativo, rechazando valores no numéricos o negativos con un mensaje de error claro.
- **FR-004**: Si el administrador no especifica un valor de orden al crear una categoría, el sistema
  DEBE asignarle automáticamente el valor más alto disponible (el valor de orden máximo entre las
  categorías existentes más uno), de forma que la categoría nueva aparezca de primera en el filtro
  del menú QR y el campo no sea un bloqueo obligatorio para crear la categoría.
- **FR-005**: El filtro de categorías del menú QR DEBE mostrar las categorías activas ordenadas de
  mayor a menor según su valor de orden configurado.
- **FR-006**: Cuando dos o más categorías activas tengan el mismo valor de orden, el sistema DEBE
  desempatar entre ellas ordenándolas alfabéticamente por nombre, para mantener una secuencia
  determinística.
- **FR-007**: Las categorías inactivas DEBEN seguir excluidas del filtro del menú QR
  independientemente de su valor de orden, conforme al comportamiento existente del menú.
- **FR-008**: Los cambios en el valor de orden de una categoría DEBEN reflejarse en el filtro del
  menú QR la próxima vez que un comensal cargue o recargue el menú, sin pasos adicionales de
  publicación.
- **FR-009**: Las categorías existentes creadas antes de este feature DEBEN recibir, durante la
  puesta en producción, un valor de orden que reproduzca la secuencia que ya tienen hoy en el
  filtro del menú QR (por ejemplo, valores secuenciales según su orden de creación), de modo que la
  secuencia visible para el comensal no cambie al desplegar el feature.

### Key Entities

- **Categoría**: Agrupación de productos que se muestra en el filtro del menú QR. Incorpora un
  nuevo atributo, "orden de visualización" (valor numérico definido en el formulario de creación o
  edición), que determina su posición dentro del filtro: a mayor valor, aparece primero. Conserva
  sus atributos existentes, como nombre y estado activo/inactivo.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un administrador puede definir o cambiar el orden de una categoría desde el
  formulario de creación o edición en menos de 30 segundos.
- **SC-002**: El 100% de las categorías activas visibles en el filtro del menú QR respeta el orden
  configurado (de mayor a menor valor).
- **SC-003**: El 0% de las categorías existentes antes del feature desaparece o genera error en el
  filtro del menú QR tras la puesta en producción.
- **SC-004**: Un comensal que abre el filtro del menú QR ve siempre la misma secuencia de
  categorías para una configuración de orden dada, sin variaciones entre cargas distintas.

## Assumptions

- El valor de orden aplica únicamente al filtro de categorías del menú QR; otros lugares donde se
  listan categorías (por ejemplo, selectores de categoría en pantallas del punto de venta) quedan
  fuera del alcance de este feature salvo que se solicite explícitamente más adelante.
- Se permite que dos categorías compartan el mismo valor de orden; el desempate es alfabético por
  nombre, sin bloquear el guardado por valores duplicados.
- Las categorías creadas antes de este feature reciben, durante la puesta en producción, un valor
  de orden que reproduce la secuencia que ya tienen hoy en el filtro del menú QR, para que el
  despliegue del feature no altere el orden que el comensal ya está viendo.
- El valor de orden es un número entero de libre elección para el administrador (sin obligar
  unicidad), lo que permite dejar espacios entre valores para reordenar categorías en el futuro sin
  tener que reeditar todas las demás.
- Solo las categorías activas se ven afectadas por este orden, ya que las categorías inactivas ya
  están excluidas del menú QR según el comportamiento existente del sistema.
