# Quickstart — validación de la spec 071

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-02

Guía de validación ejecutable: qué correr y qué esperar para dar la feature por completa
(Principio X). No contiene implementación — los detalles normativos están en
[contracts/](./contracts/) y las tareas en `tasks.md`.

---

## 0. Precondición (Principio II) — ya satisfecha

```bash
grep -n "^### A-69 " specs/000-reconocimiento/registro-de-anomalias.md
```

Debe devolver una línea. Si no aparece, la implementación de **US4** (edición en `Pausada`) no
está autorizada — las otras tres historias (US1-US3) no dependen de esta entrada, porque no son
decisiones de negocio.

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

Las pruebas automatizadas del backend no necesitan PostgreSQL: corren sobre SQLite en memoria con
los fixtures de `cart_fixtures.py`.

## 2. Batería automatizada

```bash
# Backend
cd ../pos-backend
python -m unittest app.characterization_tests.test_promotions_rules_admin -v
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v   # suite completa

# Frontend
cd ../pos-heladeria
npm test
```

**Criterio de aceptación de la batería**: 0 fallos, 0 characterization test roto sin
autorización. `test_ca2_cambiar_reglas_de_una_activa_bloquea` debe seguir en verde probando
`Activa` (research.md D5) — si se pone en rojo, algo relajó `Activa` por error.

## 3. Validación manual por historia

Usar el tenant de prueba con el catálogo de la captura original (Cono sencillo - Única, Gaseosa -
Única, Banana Split Especial - Pequeña, Canasta - Pequeña, dos productos de prueba sin precio).

### US1 — Resumen de regla con nombre de producto

1. Crear una regla "Precio de paquete", cantidad mínima 2, valor $12.000, seleccionar solo
   "Gaseosa - Única".
2. Colapsarla (botón "Editar" en otra regla, o guardar).
3. **Esperado**: "Precio de paquete - Paga $12.000 llevando 2 unidades Gaseosa - Única."
   ([contracts/resumen-de-regla.md](./contracts/resumen-de-regla.md) §3, fila 1).
4. Repetir con "Banana Split Especial - Pequeña" → el texto cambia en consecuencia (fila 2).
5. Vaciar el conjunto de una regla → "Sin productos seleccionados."

### US2 — Selección por búsqueda

1. Abrir una regla con el conjunto vacío. **Esperado**: no se lista ninguna variante debajo de
   "CONJUNTO (0)".
2. Escribir "gaseosa" en "Buscar variante…". **Esperado**: aparece un listado de resultados con
   las coincidencias del catálogo completo.
3. Marcar "Gaseosa - Única". **Esperado**: aparece de inmediato en "CONJUNTO (1)".
4. Elegir una categoría específica sin escribir texto. **Esperado**: el listado de resultados
   muestra todas las variantes de esa categoría (contracts/busqueda-y-seleccion.md §2, fila 3).
5. Volver el filtro a "Todas las categorías" y borrar el texto. **Esperado**: el listado de
   resultados queda vacío (fila 1) — el de "CONJUNTO" no cambia.
6. Desmarcar "Gaseosa - Única" desde "CONJUNTO". **Esperado**: desaparece de ahí de inmediato.

### US3 — Regla nueva al principio

1. Con una promoción de dos reglas ya creadas, presionar "+ Agregar regla".
2. **Esperado**: la nueva regla ocupa la posición 1; las dos anteriores pasan a la 2 y la 3, en
   el mismo orden relativo que tenían.

### US4 — Editar el conjunto en `Pausada`

1. Crear y activar una promoción con una regla y un producto seleccionado.
2. Con la promoción `Activa`, intentar marcar/desmarcar un producto, o agregar/quitar una regla.
   **Esperado**: sigue bloqueado, igual que hoy.
3. Pasar la promoción a `Pausada`.
4. Agregar un producto al conjunto de la regla existente y guardar. **Esperado**: se persiste
   (verificar con `GET /promotions/{id}` o recargando la pantalla).
5. Con la misma promoción `Pausada`, presionar "+ Agregar regla", configurar tipo/valor/cantidad
   mínima/conjunto de la regla nueva, y guardar. **Esperado**: se agrega y queda en la posición 1
   (US3 aplica también aquí).
6. Intentar cambiar el tipo, el valor o la cantidad mínima de la regla que **ya existía** al
   pausar. **Esperado**: sigue bloqueado (inputs deshabilitados).
7. Con dos o más reglas, quitar una. **Esperado**: se quita al guardar.
8. Dejar la promoción con una sola regla e intentar quitarla. **Esperado**: el botón "Quitar
   regla" no aparece (o está deshabilitado) — una promoción siempre conserva al menos una regla.
9. Reactivar la promoción (`Pausada → Activa`). **Esperado**: el cobro de una venta de prueba usa
   el conjunto/las reglas ya corregidas, sin rastro de la configuración anterior.

## 4. Regresión explícita (lo que no debe cambiar)

- Una promoción `Finalizada` sigue sin ninguna edición.
- Una promoción `Borrador` sigue completamente editable, sin las restricciones de `Pausada`.
- El texto de `condition_text` que sirve el backend (listado de administración, menú QR,
  terminal) no cambia — sigue siendo el de la spec 066.
- Ninguna venta, factura ni pedido ya emitido cambia de importe o de representación.
