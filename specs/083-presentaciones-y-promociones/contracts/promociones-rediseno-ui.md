# Contrato: Rediseño de pantallas de Promociones

Cubre FR-009 a FR-018, US3. **Cero cambios en `app/api/v1/promotions/`** (router, service,
schemas) — confirmado en research.md D9. Este documento describe el contrato de comportamiento
de UI que el frontend debe cumplir contra los tres prototipos entregados
(`~/Escritorio/promociones/listado-promociones.html`, `formulario-crear-promocion.html`,
`configurar-promocion.html`) y contra los endpoints ya existentes de
`app/api/v1/promotions/router.py`, sin modificarlos.

## Endpoints consumidos (sin cambio, ya existentes)

| Endpoint | Uso en el nuevo flujo |
|---|---|
| `GET /promotions` | Listado (`listado-promociones.html`): nombre, resumen de reglas, vigencia, estado, filtro por pestaña (`Todas`/`Borradores`/`Activas`/`En pausa`/`Finalizadas` → `status`). |
| `POST /promotions` | Paso final de `configurar-promocion.html`: `{name, description?, starts_at, ends_at?, days_of_week?, start_time?, end_time?, rules: [{type, value, min_qty, variant_ids}]}` — el `type` se fija en `formulario-crear-promocion.html` y se reutiliza para **todas** las reglas que se agreguen después (FR-010). |
| `PATCH /promotions/{id}` | Edición de campos escalares de una promoción ya `Activa`/`Pausada` (nombre, descripción, fin de vigencia, días, horas — FR-018 permite estos). |
| `PATCH /promotions/{id}/shape` | Edición completa de `rules` — solo mientras la promoción está en `Borrador`/`Pausada` (bloqueo ya vigente, spec 063 FR-018); en `Activa` el frontend deshabilita la sección de reglas y muestra el aviso "Tipo... Fijado en creación" (FR-018). |
| `PATCH /promotions/{id}/status` | Botones Activar/Pausar/Finalizar del listado y de configuración. |
| `POST /promotions/{id}/duplicate` | Botón "Duplicar" del listado. |
| `DELETE /promotions/{id}` | Botón "Eliminar" del listado (borrador únicamente, sin cambio de regla ya vigente). |

## Pantalla 1 — Listado (`listado-promociones.html`)

Tabla con columnas **Promoción | Reglas | Vigencia | Estado | Acciones** (FR-017), una fila por
promoción de `GET /promotions`:

- **Promoción**: `name`.
- **Reglas**: resumen legible (p. ej. "Precio de paquete", o "N reglas" si hay más de una) — usa
  `promotion-condition.util.ts` ya existente, sin cambio de lógica de cálculo.
- **Vigencia**: texto en lenguaje llano ya generado hoy ("los lunes, martes... de 6:12 a.m. a
  11:15 p.m.") — sin cambio de la función que lo arma.
- **Estado**: badge (`Vigente`/`Finalizada`/`Pausada`/`Borrador`), mismo mapeo de `status` ya
  usado.
- **Acciones**: menú desplegable con Configurar, Editar, Duplicar, Pausar/Activar, Eliminar —
  mismas condiciones de habilitación ya vigentes por estado (spec 063 FR-017/FR-018).
- Filtro por pestañas de estado + búsqueda por nombre + paginación (20/50/100 por página, según el
  prototipo) — usa los mismos query params que `GET /promotions` ya acepta.
- Botón "Nueva promoción" → navega/abre la Pantalla 2.

## Pantalla 2 — Creación (`formulario-crear-promocion.html`)

Formulario mínimo, **antes** de crear el registro en el backend:

- Campo `Nombre` (texto).
- Selector `Tipo de promoción`: **porcentaje** o **precio de paquete** (`PromotionType`, sin
  cambio de enum).
- Al continuar, estos dos valores se retienen en el estado local del formulario (no se llama
  `POST /promotions` todavía — el `POST` real ocurre al terminar la Pantalla 3, con al menos una
  regla ya armada, porque `PromotionCreate.rules` exige `min_length=1`). El tipo elegido aquí se
  usa como `type` de cada regla que se agregue en la Pantalla 3 (FR-010) y no vuelve a preguntarse
  ahí.

## Pantalla 3 — Configuración (`configurar-promocion.html`)

Dos bloques, en el orden del prototipo:

**Bloque "Configuración de Vigencia y Horarios"** (FR-011):
- Días de la semana (multi-select Lunes..Domingo; vacío = todos los días → `days_of_week: null`).
- Fecha inicio (`starts_at`, requerida), fecha fin (`ends_at`, opcional).
- Hora desde / hora hasta (opcionales, ambas o ninguna — validación ya existente en
  `_VigenciaMixin._time_window_pair`, sin cambio; el frontend replica la misma regla para dar el
  error antes del submit).

**Bloque "Paso 1: ¿Qué productos participan?" + "Paso 2: Presentación y Regla de Precio"**
(FR-012 a FR-016):
1. Buscador de producto (`placeholder="Buscar producto o sabor..."`) con filtro por categoría —
   mismo mecanismo ya existente (spec 063 FR-004), reutiliza el aplanado de productos/variantes
   que ya arma `promotions-page.component.ts`.
2. Al elegir un producto, el selector "PRESENTACIÓN / TAMAÑO" lista **todas** las variantes reales
   del producto (FR-013) — nunca una lista filtrada — con esta etiqueta por variante:
   - nombre de la presentación del catálogo, si `variant.name` coincide (comparación exacta) con
     una `Presentation` activa asociada a `product.category_id` (dato ya disponible en
     `CategoryResponse.presentations`, [contrato de categorías](./categoria-herencia-producto.md));
   - `"Presentación única"` si el producto no maneja variaciones (una sola variante, mismo nombre
     que crea `ensure_default_variant`);
   - el nombre propio de la variante en cualquier otro caso (variante anterior a esta
     funcionalidad, o renombrada a mano sin coincidir con el catálogo).
3. Campos `UNIDADES` (`min_qty`, entero ≥ 1) y `PRECIO PROMOCIONAL ($ COP)` (`value`).
4. Botón "Agregar a la lista": valida localmente lo mismo que ya valida el backend en
   `_guard_package_is_discount`/`PromotionRuleIn` (rango de `percent`, `value > 0` en
   `package_price`) para dar feedback inmediato, pero la validación autoritativa sigue siendo la
   respuesta 409/422 del backend al guardar (FR-016, sin cambio de criterio).
5. Cada fila agregada se muestra con precio regular tachado + ahorro calculado (FR-015), usando
   `promotion-pricing.util.ts` ya existente.
6. Se pueden agregar varias filas (distintas presentaciones/variantes, cada una con su propia
   condición) antes de guardar (FR-014, ya soportado por `rules: list[PromotionRuleIn]`).

**Guardado**: la primera vez arma `POST /promotions` con `rules` completo; en promociones
existentes, `PATCH /promotions/{id}/shape` reemplaza `rules` completo (solo si
`Borrador`/`Pausada`).

**Promoción `Activa`** (FR-018): la pantalla de configuración carga igual, pero:
- El tipo se muestra fijo con el aviso "Tipo seleccionado... Fijado en creación" (texto del
  prototipo), sin control editable.
- Los campos de cada regla ya guardada (tipo, valor, unidades mínimas, conjunto de variantes)
  quedan de solo lectura.
- Vigencia (fin, días, horas) y nombre/descripción siguen editables vía `PATCH /promotions/{id}`.

## Fuera de este contrato

- El motor de cálculo (`evaluate_variant_sets` y todo `app/api/v1/promotions/service.py`): sin
  cambios, ver research.md D9.
- El consumo de promociones en Menú QR/terminal (specs 066/071/081): sin cambios, spec.md
  §Assumptions.
