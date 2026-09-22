# Research: Exportación de Inventario a Excel

**Feature**: `086-exportacion-excel-inventario` | **Fecha**: 2026-09-22

Este documento resuelve las incógnitas técnicas (`NEEDS CLARIFICATION`) detectadas al
llenar el Technical Context del plan, mediante investigación del código existente en
`pos-backend` y `pos-heladeria`. No se investigó nada por fuera de esos dos repos y de
`pos-specs`.

---

## 1. Librería para generar `.xlsx` en el backend

**Decision**: agregar `openpyxl` a `requirements.txt` de `pos-backend`.

**Rationale**: no existe hoy ninguna librería de generación de Excel en el backend
(`grep -rniE "openpyxl|xlsxwriter|pandas|\.xlsx" app/` → 0 resultados). `openpyxl` es la
opción más liviana de las evaluadas: escribe `.xlsx` nativo (formato exigido por FR-002),
soporta celdas numéricas nativas (`Decimal`/`float`, no texto — requisito de FR-006) y
UTF-8 en las cadenas de forma transparente (el formato `.xlsx` es XML comprimido en UTF-8
por especificación, no requiere configuración adicional para tildes/eñes — resuelve FR-007
sin trabajo extra). No trae dependencias transitivas pesadas (a diferencia de `pandas`, que
arrastra `numpy` solo para escribir un archivo).

**Alternatives considered**:
- `pandas` (`DataFrame.to_excel`): descartado — agrega `numpy`/`pandas` completos al
  backend (impacto en tamaño de imagen Docker y superficie de dependencias) solo para
  generar un archivo con 7 columnas; no aporta valor sobre `openpyxl` para este caso de uso.
- `xlsxwriter`: descartado — capacidades equivalentes a `openpyxl` para este alcance
  (celdas numéricas, UTF-8), pero `openpyxl` es la opción más usada en el ecosistema
  FastAPI/SQLAlchemy y no aporta ninguna ventaja adicional relevante aquí.
- Generar el XML del `.xlsx` a mano: descartado — reinventar el formato OOXML para un caso
  de uso estándar no se justifica (Principio IX: se prefiere la librería estándar/madura
  cuando resuelve el problema sin sobrecoste relevante).

**Justificación de dependencia nueva (Principio IX)**: problema que resuelve — no existe
forma de generar `.xlsx` en el backend hoy; por qué no basta la librería estándar — Python
no tiene un generador de OOXML en `stdlib`; alternativas consideradas — ver arriba;
impacto en mantenimiento/seguridad/despliegue — `openpyxl` es una librería madura, sin
dependencias nativas (pure Python), amplio uso en el ecosistema; su huella en la imagen
Docker es marginal.

---

## 2. Patrón de descarga de archivo binario (backend)

**Decision**: construir el `.xlsx` completo en memoria (`io.BytesIO`) y devolverlo con
`fastapi.Response` (no `StreamingResponse`), `media_type=
"application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"` y encabezado
`Content-Disposition: attachment; filename="inventario_<YYYY-MM-DD>.xlsx"`.

**Rationale**: no existe precedente de descarga de archivo en el backend — `StreamingResponse`
solo se usa hoy para *Server-Sent Events* (`app/api/v1/realtime/router.py:187,221`,
`media_type="text/event-stream"`), no para servir un binario completo; `FileResponse` no se
usa en absoluto. Dado que el spec asume explícitamente que "el volumen actual de insumos
... no representa un riesgo de rendimiento relevante para una generación síncrona"
(Assumptions, spec.md), construir el workbook completo en memoria antes de responder es
más simple que un streaming incremental y, como efecto secundario, garantiza el edge case
de la spec ("no debe entregar un archivo descargado corrupto o incompleto" ante un error a
mitad de la generación): si algo falla mientras se arma el workbook, la excepción se lanza
antes de que FastAPI envíe cualquier byte de respuesta, y el manejador de errores por
defecto devuelve un 500 JSON normal — nunca un `.xlsx` truncado.

**Alternatives considered**:
- `StreamingResponse` con generador incremental: descartado para este alcance — añade
  complejidad (habría que streamear filas de un workbook que `openpyxl` normalmente
  construye completo en memoria; el modo *write-only* de `openpyxl` sí soporta streaming,
  pero no hay necesidad de esa complejidad al volumen actual de insumos, según Assumptions
  del spec). Puede revisarse en una spec futura si el volumen de insumos crece.
- `FileResponse` contra un archivo temporal en disco: descartado — introduce I/O de disco y
  limpieza de archivos temporales sin necesidad, cuando el archivo cabe cómodamente en
  memoria.

---

## 3. Autorización del endpoint de exportación

**Decision**: el nuevo endpoint usa `Depends(require_tenant_admin)` (backend) además de
heredar `Depends(require_module_access("inventario"))` ya aplicado a nivel de router.

**Rationale**: el spec (Assumptions) establece que "el rol autorizado para ver la tabla de
Inventario ... es el mismo autorizado para exportarla; esta funcionalidad no introduce un
permiso nuevo y diferenciado". Se investigó cuál es ese rol hoy:
- **Frontend**: la ruta `/dashboard/inventario` está protegida con
  `canActivate: [roleGuard([UserRole.ADMIN]), planModuleGuard('inventario')]`
  (`pos-heladeria/src/app/modules/dashboard/routes.ts:208`) — solo el rol `ADMIN` puede
  entrar a la pantalla de Inventario.
- **Backend**: no existe un rol "Gestor de Inventario" (`ROLE_NAMES = (SUPER_ADMIN, ADMIN,
  CASHIER, MESERO)`, `app/core/db.py:230`). El endpoint `GET /inventory/items` que hoy lista
  el inventario solo exige `get_current_user` (cualquier usuario autenticado) más el guard
  de plan a nivel de router — la restricción real a "solo ADMIN" vive hoy únicamente en el
  guard de ruta del frontend, no en el backend.

Para que FR-009 ("el sistema MUST rechazar la generación del archivo ... para cualquier
usuario que no tenga el rol autorizado") se cumpla también del lado servidor —y no solo
ocultando el botón en el frontend—, el nuevo endpoint aplica explícitamente
`require_tenant_admin` (el mismo dependency ya usado en `create_item`/`update_item`/
`adjust_item` de este mismo router, `app/api/v1/inventory/router.py`), que verifica
`user.role.name == "ADMIN"`. Esto no modifica el comportamiento del endpoint existente
`GET /inventory/items` (Principio II: comportamiento existente protegido) — es una regla de
autorización nueva y propia del endpoint nuevo, no una restricción retroactiva sobre el
endpoint de listado.

**Alternatives considered**:
- Reusar exactamente el mismo guard que `GET /inventory/items` (solo `get_current_user`):
  descartado — dejaría el endpoint de exportación accesible a cualquier usuario autenticado
  del tenant (CASHIER, MESERO) aunque hoy no puedan ver la pantalla de Inventario, violando
  la intención de FR-009 y de las Assumptions del spec.
- Crear un rol nuevo "Gestor de Inventario": descartado explícitamente por las Assumptions
  del spec ("no introduce un permiso nuevo y diferenciado").

---

## 4. Cálculo de la columna "Estado"

**Decision**: la columna "Estado" del archivo exportado replica exactamente la regla que
hoy calcula `inventory-page.component.ts` en pantalla — dos valores posibles, "Bajo mínimo"
u "OK" — usando la fórmula `activo AND stock_actual <= stock_mínimo` para "Bajo mínimo",
"OK" en cualquier otro caso (incluidos los insumos inactivos, que siempre muestran "OK" hoy
sin importar su nivel de stock).

**Rationale**: el spec (Assumptions) da por hecho que la columna "Estado" hoy tiene tres
valores de ejemplo ("Normal", "Stock bajo", "Agotado"), pero la investigación del código
actual encontró que **no es así**: el frontend (`inventory-page.component.ts:491-493`,
`isLow(i) { return i.active && i.current_stock <= i.min_stock; }`, renderizado en las
líneas 171-177 como badge "Bajo mínimo" / "OK") solo calcula dos estados, y ese cálculo
incluye la condición `i.active` — un insumo inactivo con stock en cero se muestra igual como
"OK", no como un tercer estado "Agotado". La misma regla vive espejada en el backend para el
filtro `low_stock` (`app/api/v1/inventory/service.py:41-42` y el endpoint
`/inventory/items/low-stock`, `app/api/v1/inventory/router.py:55-65`): `active == True AND
current_stock <= min_stock`.

El texto de las Assumptions ("Los valores que hoy calcula y muestra la columna 'Estado' ...
son los mismos que se exportan; esta funcionalidad no introduce nuevas reglas para calcular
el estado de un insumo") es la instrucción operativa real, por encima del ejemplo entre
paréntesis que la acompaña: exportar exactamente lo que la pantalla muestra hoy, sin
inventar un tercer estado "Agotado" que no existe en el sistema actual. Introducir un
umbral nuevo (p. ej. `current_stock == 0` → "Agotado") sería una regla de negocio nueva no
autorizada por este spec (Principio II: el comportamiento existente — incluida la fórmula de
"Estado" — sigue protegido salvo decisión de negocio explícita, y esta spec no registra
ninguna en `registro-de-anomalias.md`).

**Alternatives considered**:
- Implementar los tres estados "Normal"/"Stock bajo"/"Agotado" tal como sugiere el
  paréntesis de las Assumptions: descartado — constituiría una regla de negocio nueva
  (dónde está el corte entre "Stock bajo" y "Agotado") que ni el spec ni ninguna decisión
  registrada en `registro-de-anomalias.md` definen; violaría Principio II.
- Excluir la condición `active` del cálculo de "Bajo mínimo" para el export (mostrar el
  estado "real" de stock de un insumo inactivo): descartado — cambiaría el significado de
  "Estado" respecto a lo que la misma fila mostraría hoy en pantalla si el filtro de
  activos lo dejara ver, rompiendo la garantía de fidelidad de FR-005 ("los mismos valores
  que hoy se muestran ... en la tabla").

---

## 5. Etiquetas de columnas "Tipo" y "Unidad"

**Decision**: "Tipo" se exporta con la misma etiqueta ya usada en pantalla
(`typeLabel()`, `inventory-page.component.ts:488-490`): `"Materia prima"` para
`raw_material`, `"Empacado"` para `packaged`. "Unidad" se exporta con la abreviatura de la
unidad de medida (`unitAbbr()`, columna `unit_measures.abbreviation` en el backend, join por
`unit_measure_id`) — el mismo valor que hoy ve el usuario en la columna "Unidad" de la
tabla, no el nombre largo de la unidad.

**Rationale**: FR-005 exige que el archivo contenga "los mismos valores que hoy se
muestran ... en la tabla de Inventario en pantalla". El backend no expone hoy estas
etiquetas ya traducidas (`InventoryItemResponse` solo trae el código interno `type` y el
`unit_measure_id`, sin resolver) — la traducción ocurre hoy en el frontend. Para el export,
que se genera enteramente en el backend, esa traducción se replica en un helper nuevo del
backend (no se reutiliza código TypeScript desde Python), pero usando exactamente los mismos
textos literales que ya existen en `pos-heladeria`, para no crear una segunda fuente de
verdad divergente con etiquetas distintas a las que el usuario ve en pantalla.

**Alternatives considered**:
- Exportar el código interno crudo (`raw_material`/`packaged`) en la columna "Tipo":
  descartado — no es "el mismo valor que hoy se muestra en pantalla" (FR-005); el usuario
  del reporte (auditoría/contabilidad externa) no conoce los códigos internos del sistema.

---

## 6. Patrón de descarga de blob en el frontend

**Decision**: `InventoryService` agrega un método `exportItems()` que llama
`HttpClient.get(...)` con `responseType: 'blob'` y `observe: 'response'` contra el nuevo
endpoint (mismo `baseUrl` ya usado por `fetchItemsPage()`, que ya viaja con el
interceptor de autenticación existente — el mismo que agrega el Bearer token a
`GET /inventory/items` hoy). El componente dispara la descarga creando un
`URL.createObjectURL(blob)` y un `<a download>` sintético, sin depender de ningún bucket ni
CORS externo.

**Rationale**: no existe hoy en el frontend ningún uso de `HttpClient` con
`responseType: 'blob'` (`grep -rn "responseType" src/` → 0 resultados) — es un patrón
nuevo. El más cercano en el código actual es `downloadImage()`
(`transfer-details-step.component.ts:375-390`), que descarga una imagen QR ya alojada en un
bucket público vía `fetch(url, {mode: 'cors'})` — pero ese caso no aplica aquí: el archivo
de exportación no es un recurso estático en un bucket, es generado bajo demanda por un
endpoint **autenticado** del propio backend (requiere el Bearer token del usuario, ver
sección 3), así que debe pedirse con `HttpClient` (que ya pasa por el interceptor de
autenticación de la app) y no con `fetch` directo. Se conserva del ejemplo existente el
patrón de creación de blob URL + `<a download>` + `URL.revokeObjectURL()` para liberar
memoria tras la descarga.

**Alternatives considered**:
- `fetch()` directo al endpoint con el token leído manualmente: descartado — duplicaría a
  mano lo que el interceptor HTTP de Angular ya resuelve para toda la app (adjuntar el
  Bearer token, manejar refresh/401), introduciendo una segunda vía de autenticación sin
  necesidad.
- Abrir la URL del endpoint directamente en una pestaña nueva (`window.open`): descartado —
  el endpoint requiere el header `Authorization: Bearer <token>`, que no puede adjuntarse a
  una navegación de `window.open`/enlace directo sin exponer el token en la URL.

---

## 7. Nombre del archivo descargado

**Decision**: `inventario_<YYYY-MM-DD>.xlsx`, con la fecha del día en que se genera el
archivo (fecha del servidor, `date.today()`), calculada en el backend y enviada en el
encabezado `Content-Disposition`. El frontend usa ese mismo nombre (leído del encabezado de
la respuesta) como nombre del archivo descargado; si por algún motivo el encabezado no
llega, arma el mismo patrón de nombre en el cliente como respaldo (`date` local del
navegador).

**Rationale**: cubre la Assumption del spec ("El nombre del archivo descargado incluye la
fecha de generación ... el formato exacto del nombre queda a criterio de la fase de
planeación"). Generar el nombre en el backend (fuente única) y que el frontend simplemente
lo respete evita que servidor y cliente calculen fechas distintas (p. ej. por zona horaria)
para el mismo archivo.

**Alternatives considered**: incluir hora además de fecha (`inventario_2026-09-22_14-30.xlsx`):
descartado — no lo exige ninguna historia ni criterio de aceptación del spec; un nombre más
simple por fecha es suficiente para diferenciar descargas del mismo día a simple vista, que
es lo único que la Assumption pide.

---

## 8. Retroalimentación de error en el frontend

**Decision**: si la petición de exportación falla (por ejemplo, error 5xx del backend a
mitad de la generación), el componente muestra un mensaje de error con el `ToastService` ya
existente y usado hoy en `inventory-page.component.ts` (`this.toast.error(...)`,
`shared/feedback/toast.service.ts`) — no se introduce un mecanismo de notificación nuevo.

**Rationale**: `ToastService` ya está inyectado en `inventory-page.component.ts` (línea 389,
`private readonly toast = inject(ToastService)`) y se usa para otras acciones de esa misma
pantalla; reutilizarlo para el error de exportación mantiene un único patrón de
retroalimentación de error en el componente, consistente con el edge case del spec ("el
sistema debe informar el error al administrador").

**Alternatives considered**: `alert()` nativo del navegador: descartado — el proyecto ya
tiene un sistema de notificaciones propio (`ToastService`) usado consistentemente en toda la
app; usar `alert()` sería inconsistente con el resto del producto.
