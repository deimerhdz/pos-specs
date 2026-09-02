# Quickstart — validación de la spec 066

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-01

Guía de validación ejecutable: qué correr y qué esperar para dar la feature por completa
(Principio X). No contiene implementación — los detalles normativos están en
[contracts/](./contracts/) y las tareas en `tasks.md`.

---

## 0. Precondición bloqueante (Principio II)

**Antes de escribir la primera línea de código**, verificar que las tres decisiones de negocio
están registradas:

```bash
grep -nE "^### A-6[678] " specs/000-reconocimiento/registro-de-anomalias.md
```

Deben salir **tres** líneas (A-66, A-67, A-68). Hoy el registro llega hasta **A-65**: el `grep`
no devuelve nada. Sin esas tres entradas —escritas a mano, con quién decidió y cuándo, en el
formato de A-62 a A-65— el cambio de comportamiento **no está autorizado** y la implementación no
puede arrancar. Las tareas de esta feature solo verifican que existen; no las redactan.

---

## 1. Preparar el entorno

```bash
# Backend
cd ../pos-backend
docker compose up -d postgres redis
uvicorn app.main:app --reload          # http://localhost:8000

# Frontend
cd ../pos-heladeria
npm start                              # http://localhost:4200
```

Ambos repositorios están en `develop`. Las pruebas automatizadas del backend no necesitan
PostgreSQL: corren sobre SQLite en memoria con los fixtures de `cart_fixtures.py`.

---

## 2. Batería automatizada

```bash
# Backend — suite completa (incluye characterization tests)
cd ../pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v

# Backend — funciones puras de promociones (texto, redondeo, reparto)
python -m app.scripts.test_promotions_rules

# Frontend
cd ../pos-heladeria
npm test
```

### Criterio de aceptación de la batería

| Debe pasar | Por qué |
|---|---|
| Toda la suite del backend, en verde | Principio X. |
| `test_promotions_service.py` **sin modificar ningún aserto de importe** | Es la verificación de SC-006: el total cobrado no cambia en ningún escenario. Si un aserto de importe hay que tocarlo, el diseño se salió del alcance — parar y revisar. |
| Los **cinco** tests de texto, actualizados citando esta spec | Principio III. Ver §3. |
| El test de no-regresión de la terminal | Ver §7. |

---

## 3. Tests afectados (Principio III)

Ninguno lleva el prefijo bajo veto `"CONGELA comportamiento actual:"` — verificado con
`grep -rn "CONGELA comportamiento actual" …` sobre los cuatro ficheros el 2026-09-01. Se
actualizan citando esta spec, en el mismo commit.

| # | Test | Aserto actual | Nuevo |
|---|---|---|---|
| 1 | `test_menu_router.py::test_vigente_se_anuncia_con_texto_legible` | `Llevando 2 de estas 8 variantes pagas $12.000` | `Llevando 2 entre sabor-0, sabor-1, sabor-2 y 5 más pagas $12.000` |
| 2 | `test_promotions_rules_admin.py::test_ca1_ca6_paquete_nace_borrador_con_condicion` | `Llevando 2 de estas 8 variantes pagas $12.000` | `Llevando 2 entre licor-0, licor-1, licor-2 y 5 más pagas $12.000` |
| 3 | **`test_promotions_router.py::test_el_header_no_cambia_la_forma_de_la_respuesta`** | `10% en estas 1 variantes` | ⚠️ **no listado en la spec.** Su variante se crea sin `name`, así que el fixture le pone `variante-{uid}` (no determinista): hay que **pasarle un `name` explícito** y afirmar contra él. |
| 4 | `app/scripts/test_promotions_rules.py` (§5, `_regla_texto`) | Los cuatro textos por conteo | `_regla_texto` pasa a suministrar el mapa de nombres; se cubre la [tabla de casos](./contracts/texto-condicion.md) completa, incluido el respaldo por conteo. |
| 5 | `promotions-page.component.spec.ts:56,61` | `Llevando 2 de estas 3 variantes` / `10% en estas 3 variantes` | Firma nueva `ruleConditionPreview($index)` + textos por nombres. |

`promotion-pricing.util.spec.ts:34` menciona `condition_text` **solo como dato de fixture**; ningún
aserto lo lee. No es un test afectado.

---

## 4. Escenarios manuales — Historia 1 (cartel del menú QR, FR-001 a FR-006)

**Preparación**: promoción activa, dentro de su ventana, con tres reglas de `package_price`.

| # | Conjunto de la regla | Abrir el menú QR y leer | Cubre |
|---|---|---|---|
| 4.1 | 8 variantes, todas `Pequeño 8oz`, `min_qty` 2, `$12.000` | `Llevando 2 Pequeño 8oz pagas $12.000` | CA1 |
| 4.2 | **1** variante `Pequeño 8oz` | `Llevando 2 Pequeño 8oz pagas $12.000` — **nunca** `de estas 1 variantes` | CA2, el defecto reportado |
| 4.3 | `Pequeño 8oz` + `Mediano 12oz` + `Grande 16oz`, `$15.000` | `Llevando 2 entre Grande 16oz, Mediano 12oz y Pequeño 8oz pagas $15.000` | CA3 |
| 4.4 | 5 nombres distintos | tres primeros **en orden alfabético** + `y 2 más` | CA4 |
| 4.5 | `percent` 10%, `min_qty` 1, todas `Pequeño 8oz` | `10% en Pequeño 8oz` | CA5 |
| 4.6 | Promoción activa pero fuera de su franja horaria | El cartel **no** muestra ninguna de sus reglas | CA6 |

> **Cuidado con 4.3 y 4.4**: el orden es **alfabético** (FR-002), no el orden en que se
> seleccionaron las variantes ni el orden por tamaño. `Grande 16oz` va antes que
> `Pequeño 8oz`.

**Comprobación cruzada de SC-005**: para la misma regla, el texto del cartel, el de la terminal y
el de la columna «condición» del listado de administración deben ser **idénticos carácter por
carácter**. Copiar y pegar los tres y comparar.

---

## 5. Escenarios manuales — Historia 2 (modal, FR-007 a FR-012) ⭐

Es la historia que corrige el defecto de importe. **Precio normal de `Pequeño 8oz`: `$8.000`.**

| # | Regla vigente | El modal debe mostrar | Cubre |
|---|---|---|---|
| 5.1 | `package_price`, `min_qty` 2, `$12.000` | `$8.000` + línea `2 x $12.000 · $6.000 c/u` | CA1 |
| 5.2 | La misma | Agregar **1** unidad → el carrito la cobra a **`$8.000`** (el equivalente es informativo) | CA2, FR-011 |
| 5.3 | `percent` 15%, `min_qty` 3, sobre `Mediano 12oz` de `$11.000` | `$11.000` + `3 x -15% · $9.350 c/u` (**sin `≈`**: el exacto es entero) | CA3 |
| 5.4 | `package_price`, `min_qty` 1, `$6.000` | `$8.000` **tachado** + `$6.000` vigente; el total del botón usa `$6.000` | CA4, **FR-010/A-68** |
| 5.5 | La de 5.4 | Cobrar el pedido → el importe cobrado **coincide** con el que mostró el modal | CA5, **SC-003** |
| 5.6 | `percent` 10%, `min_qty` 1 | Tachado + descuento **sin cambio de importes**, más la línea `1 x -10% · $7.200 c/u` (FR-008 con `n` = 1) | CA6, no-regresión |
| 5.7 | Producto con 3 presentaciones, solo 1 en el conjunto | **Solo** esa fila lleva información de promoción | CA7 |
| 5.8 | Promoción activa, fuera de su franja | **Ninguna** fila muestra información | CA8 |

### 5.9 — La rama sin tachado de FR-015 (no es letra muerta)

`_guard_package_is_discount` rechaza al crear una regla `package_price` con
`value >= min_qty × precio más barato`, así que **este caso no se puede montar directamente**. La
guarda corre en `create`, `update_shape` y `change_status` — **no** cuando el catálogo cambia un
precio ([research.md D-6](./research.md)). Para llegar ahí:

1. Crear y **activar** una regla `package_price`, `min_qty` 1, `$6.000` sobre `Pequeño 8oz`
   (`$8.000`). Pasa la guarda.
2. En el catálogo, **bajar el precio** de `Pequeño 8oz` a `$5.000`.
3. Abrir el menú QR.

**Esperado**: la presentación muestra `$6.000` (el valor de la regla, que es lo que se cobra),
**sin tachado y sin ninguna señal de descuento**. Nunca `$5.000` tachado junto a `$6.000`: eso
sería anunciar un aumento como ahorro.

### 5.10 — Marca de aproximado (FR-009)

| Regla | Precio normal | Esperado |
|---|---|---|
| `package_price`, `min_qty` 3, `$13.000` | cualquiera | `3 x $13.000 · ≈ $4.333 c/u` |
| `percent` 15%, `min_qty` 3 | `$11.000` | `3 x -15% · $9.350 c/u` (**sin** `≈`) |
| `percent` 12,5%, `min_qty` 2 | `$8.700` | `2 x -12.5% · ≈ $7.613 c/u` (exacto `7612,5` → medio hacia arriba) |

La marca aparece **siempre que el importe exacto no sea entero en pesos**, en los dos tipos.

---

## 6. Escenarios manuales — Historia 3 (tarjetas, FR-013 a FR-015)

| # | Producto | Esperado | Cubre |
|---|---|---|---|
| 6.1 | Al menos una variante en una regla `package_price` vigente | Insignia `🎉 Promo` | CA1, **el caso que hoy no produce señal** |
| 6.2 | Al menos una variante en una regla `percent` vigente | **La misma** insignia `🎉 Promo`, no una distinta por tipo | CA2 |
| 6.3 | Ninguna variante cubierta | **Sin** insignia | CA3 |
| 6.4 | Promoción activa pero fuera de su ventana | **Sin** insignia | CA4 |
| 6.5 | `percent` `min_qty` 1 vigente | Precio tachado + precio con descuento **y además** la insignia genérica | CA5, FR-015 |

**Comprobación de SC-004**: recorrer la carta completa y contar. Los productos con insignia deben
ser **exactamente** los que tienen al menos una variante cubierta por una regla vigente en ese
instante — ni uno más, ni uno menos.

---

## 7. Escenarios manuales — Historia 4 (terminal y administración, FR-016 a FR-018)

| # | Superficie | Esperado | Cubre |
|---|---|---|---|
| 7.1 | Terminal, modal del producto | Condición `Llevando 2 Pequeño 8oz pagas $12.000` + equivalente `$6.000 c/u`, con las **mismas palabras** que el menú | CA1, FR-016 |
| 7.2 | Administración, listado | La **misma** frase en la columna de condición | CA2 |
| 7.3 | Administración, formulario | El resumen previo a guardar usa el mismo formato, calculado sobre las variantes **seleccionadas en ese momento** | CA3, FR-018 |
| 7.4 | Administración, conjunto con >3 nombres | Resume igual que el menú (tres + `y N más`), y la lista completa de variantes con su precio sigue estando en el resumen | CA4 |
| 7.5 | Terminal, catálogo | La insignia por producto **conserva** `-10%` / `Paquete $12.000` — la genérica es exclusiva del menú QR | CA5, FR-016 |

### 7.6 — No-regresión de importes en la terminal (FR-017) 🔒

Es el riesgo principal del diseño ([research.md D-10](./research.md)) y **debe tener test
automatizado**, no solo verificación manual.

Con una regla `package_price` `min_qty` 1 de `$6.000` vigente sobre `Pequeño 8oz` (`$8.000`):

1. Abrir el modal del producto **en la terminal**.
2. **Esperado**: muestra `$8.000` (precio normal), **no** `$6.000`. La línea de promoción sí
   aparece.
3. Agregar la línea al pedido → el precio unitario del borrador es `$8.000`.
4. El descuento efectivo lo aplica el **preview del cobro** del backend, como hasta hoy.

Si la terminal empieza a mostrar `$6.000` en el modal, `MenuService` está mapeando
`discounted_price` y hay que quitarlo.

---

## 8. Lista de verificación de cierre

Un spec se considera completado cuando (Constitución, «Flujo de Trabajo de Evolución Funcional»):

- [x] A-66, A-67 y A-68 registrados en `registro-de-anomalias.md` con quién y cuándo (§0).
- [x] Los cinco tests de texto actualizados, citando esta spec en el commit (§3).
- [x] Suite del backend en verde, **sin tocar ningún aserto de importe** de la spec 063 (SC-006).
- [x] `npm test` en verde.
- [x] Test de no-regresión de la terminal, en verde (§7.6).
- [x] Escenarios manuales §4 a §7 verificados en el entorno local.
- [x] SC-001 a SC-007 comprobados: ningún descriptor por conteo salvo FR-006 (SC-001); 100%/0% de
      presentaciones marcadas (SC-002); mostrado = cobrado (SC-003); 100%/0% de insignias (SC-004);
      texto idéntico en tres superficies (SC-005); total cobrado sin cambios (SC-006);
      el comensal identifica promociones sin abrir producto (SC-007).
- [x] Sin migraciones (no debe haber ningún fichero nuevo en `alembic/versions/`).
- [x] Sin dependencias nuevas (`requirements.txt` y `package.json` sin cambios).
