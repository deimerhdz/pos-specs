---

description: "Tareas de la spec 089 — adicionales del Menú QR, cierre de mesa, impresión de caja y total inmediato en el POS (US1–US5)"
---

# Tasks: Correcciones de Adicionales del Menú QR, Cierre de Mesa, Impresión de Caja y Total Inmediato en el POS

**Input**: Documentos de diseño de `/specs/089-fix-adicionales-cierre-mesa-caja-pos/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D19), [data-model.md](./data-model.md), [contracts/](./contracts/) (`addons-per-line.md`, `session-closed.md`, `pos-total-immediate.md`), [quickstart.md](./quickstart.md)

**Tests**: El proyecto usa *characterization tests* (`app/characterization_tests/`, `python -m unittest`, sesiones SQLite en memoria vía `fixtures.py::new_session`) como árbitro de comportamiento (Principio III); `pos-heladeria` usa `ng test` (Karma/Jasmine). El plan enumera los archivos de prueba, así que las tareas de prueba de **backend** van **antes** de la implementación de cada historia y deben fallar contra el código actual. En **frontend** cada tarea incluye su `.spec.ts` (se escribe primero dentro de la misma tarea). Los tests que hoy congelan el cobro/consumo por unidad del adicional del Menú QR o el modal "El total cambió" se actualizan **en el mismo commit**, citando **A-94** / **A-96** y con evidencia de que el resto de la suite sigue verde (plan.md, fila III). La **impresión** solo se verifica de verdad en la vista previa del navegador (FR-024): no hay prueba unitaria que la sustituya.

**Organización**: Las tareas se agrupan por historia de usuario en el **orden de prioridad de `spec.md`** (US1 P1 → US2 P1 → US3 P2 → US4 P2 → US5 P3). Ojo: el plan numera las historias de otra manera en su prosa ("Historia 4 = cierre de mesa", etc.); **manda la numeración de la spec**, que es la de este archivo. Todas las rutas fueron verificadas contra el código real de ambos repos el 2026-09-30 (la única ruta que no existe aún es `fold_line_addons.py`, que es NUEVA).

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: A qué historia de usuario pertenece (US1–US5)
- Cada tarea incluye la ruta exacta del archivo, relativa a `../pos-backend`, `../pos-heladeria` o `pos-specs` (esta carpeta) según corresponda

## Convenciones de ruta

- `../pos-backend/app/...` — API FastAPI + SQLAlchemy (Python 3.12). Tests en `../pos-backend/app/characterization_tests/`; se ejecutan desde `../pos-backend` con `python -m unittest app.characterization_tests.<modulo> -v` (suite completa: `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`)
- `../pos-heladeria/src/app/...` — SPA Angular 20 (TypeScript); pruebas con `npx ng test --watch=false --browsers=ChromeHeadless --include='<glob>'`. Abreviatura en este archivo: `tables/` = `../pos-heladeria/src/app/modules/tables/`
- Sin prefijo — archivo dentro de este repo (`pos-specs`)

## Nota sobre gobernanza (Principios II/III/XI) y ramas

- **A-94, A-95 y A-96 ya están registradas** en `specs/000-reconocimiento/registro-de-anomalias.md` (antes de implementar). Falta **A-97** (hallazgo documental de `set_assignments`, T001).
- Dos valores por defecto que la spec dejó al plan **se confirman con el negocio antes de desplegar** (T002): (a) el cambio de total de última hora se resuelve con aviso no bloqueante + segundo Cobrar; (b) un 401 por **vencimiento** conserva la pantalla de nombre y solo el **cierre** lleva a "gracias". No bloquean el desarrollo, sí el despliegue.
- **Ramas (Principio XIV)**, en `../pos-backend` **y** `../pos-heladeria`: `feat/089-qr-addons-per-line` (US1 y US3, desde `develop`); `fix/089-table-close-pos-total-print` (US2 y US4, creada **desde `feat/089-…`** una vez terminada US1, porque comparten `public-menu.component.ts`, `pos-terminal.store.ts`, `payment-attempt-review-panel.component.ts`, `cart/service.py`, `consolidation.py` y `kitchen.py`); y `fix/089-cash-report-print` (US5, desde `develop`, sin archivos compartidos). Se trabajan en secuencia; no hay rebase entre ramas hermanas.
- **Despliegue**: **backend con migración primero**, luego frontend (el frontend nuevo tolera respuestas sin los campos nuevos con `line_total ?? unit_price × quantity + addons_total`). Los commits **solo** se hacen cuando el usuario lo pida (Principio XV): commits pequeños, en inglés, Conventional Commits, sin marcas de IA; la **migración va en su propio commit, el primero** (Principio VI).

---

## Phase 1: Setup

**Propósito**: gobernanza, ramas y línea base antes de tocar código.

- [X] T001 Registrar **A-97** en `specs/000-reconocimiento/registro-de-anomalias.md` (`### A-97 — [HALLAZGO — spec 089] ...`, mismo formato que A-94…A-96; confirmar el número con `grep -oE "^### A-[0-9]+" specs/000-reconocimiento/registro-de-anomalias.md | sort -t- -k2 -n | tail -1`): `table_sessions/service.py::set_assignments` ("repartir por unidades") clona las opciones de `OrderItemOption` **sin su `quantity`** (queda en 1). Es un defecto preexistente ajeno a esta spec (Principio V): solo se **documenta**, no se corrige; indicar que con la regla nueva los adicionales (`addons_total` y opciones `per_line`) se quedan **solo en la fila original** (research D5/D5-b)
- [ ] T002 Confirmar con el negocio los dos valores por defecto de la fila II del plan (a: aviso no bloqueante + segundo Cobrar; b: 401 por vencimiento → pantalla de nombre, cierre → "gracias") y dejar constancia (quién y cuándo) en las entradas **A-96** y **A-95** del registro, quitando el "por confirmar". Si el negocio elige otra cosa, ajustar `contracts/pos-total-immediate.md` / `contracts/session-closed.md` antes de las tareas de US2 / US4. No bloquea el desarrollo; **bloquea el despliegue**
- [X] T003 Crear en `../pos-backend` y en `../pos-heladeria` (ambos en `develop`, árbol limpio: verificar con `git status`) la rama `feat/089-qr-addons-per-line` desde `develop` (US1 y US3; Principio XIV). La rama `fix/089-table-close-pos-total-print` (US2 y US4) se crea **desde `feat/089-…`** —el Principio XIV manda partir de la rama en la que se está— cuando US1 esté terminada, porque T042–T047 dependen de T017, T019, T027, T037 y T039. US5 (impresión) no comparte archivos con las demás: puede ir en su propia rama `fix/089-cash-report-print` desde `develop`
- [X] T004 Línea base: correr las suites completas de ambos repos **sin cambios** (`python -m unittest discover -s app/characterization_tests -p 'test_*.py'` en `../pos-backend`; `npx ng test --watch=false --browsers=ChromeHeadless` en `../pos-heladeria`) y crear `specs/089-fix-adicionales-cierre-mesa-caja-pos/implementation-notes.md` con el número de tests y el resultado de cada suite (evidencia exigida por el Principio III para actualizar los `CONGELA`)

**Checkpoint**: A-97 registrada, ramas creadas, línea base verde documentada.

---

## Phase 2: Foundational (esquema; bloquea US1 y US3)

**Propósito**: las cuatro columnas aditivas y su migración, **sin cambiar ningún comportamiento** (defaults `0`/`false` reproducen el cálculo actual). Contrato en [data-model.md](./data-model.md). US2, US4 y US5 **no** dependen de esta fase y pueden empezar tras el Setup.

**⚠️ CRITICAL**: US1 y US3 no pueden empezar hasta terminar esta fase. Migración y comportamiento **nunca** van en el mismo commit (Principio VI).

- [X] T005 [P] Añadir `QR_ADDONS_PER_LINE: bool = True` a `Settings` en `../pos-backend/app/core/config.py` (bandera de reversa de research D2/D7; solo gobernará la **creación** de líneas nuevas, nunca la lectura, el consumo ni el cobro; sin variable de entorno obligatoria)
- [X] T006 [P] Crear `../pos-backend/app/characterization_tests/test_line_addons_migration.py`: (a) `CartItem.addons_total` y `OrderItem.addons_total` toman `Decimal("0")` por defecto y `CartItemOption.per_line` / `OrderItemOption.per_line` toman `False` al crear filas sin indicarlos; (b) una línea "histórica" (sin los campos nuevos) devuelve `line_total == unit_price × quantity` exactamente igual que antes (SC-003, Principio VII); (c) `addons_total` negativo es rechazado por el `CHECK` (si el motor de test SQLite lo aplica; si no, comprobar el `CheckConstraint` en `__table__.constraints`). Fallará hasta T008
- [X] T007 Crear la migración `../pos-backend/alembic/versions/<rev>_089_line_addons.py` con `down_revision = 'deb68185a606'` y `@for_each_tenant_schema` (mismo patrón que `deb68185a606_087_drop_cash_partial_counts.py`): `upgrade()` idempotente (revisar con `inspect` si la columna ya existe) que añade `cart_items.addons_total` y `order_items.addons_total` (`NUMERIC(12,2) NOT NULL DEFAULT 0`), `cart_item_options.per_line` y `order_item_options.per_line` (`BOOLEAN NOT NULL DEFAULT false`) y los `CHECK (addons_total >= 0)` `ck_cart_item_addons_total_nonneg` / `ck_order_item_addons_total_nonneg`; **sin `UPDATE` de backfill**. `downgrade()` elimina columnas y constraints **solo si** no existe ninguna fila con `addons_total > 0` en `cart_items` y `order_items` de ese esquema; si existe, aborta con un mensaje que apunta **solo** a `QR_ADDONS_PER_LINE=false` (única reversa soportada una vez existen pedidos con `addons_total > 0`; `fold_line_addons` limpia únicamente `cart_items` y no habilita el downgrade; data-model §5)
- [X] T008 Añadir las columnas a los modelos: `CartItem.addons_total` y `CartItemOption.per_line` en `../pos-backend/app/models/cart_item.py`; `OrderItem.addons_total`, `OrderItemOption.per_line` y la propiedad `OrderItem.line_total` (`unit_price × quantity + addons_total`) en `../pos-backend/app/models/order_item.py`, con `server_default` igual al de la migración y el `CheckConstraint` correspondiente. Depende de T007
- [X] T009 Verificar la migración (quickstart §1) en un esquema de pruebas de PostgreSQL 16: anotar conteo de `order_items` y un total de venta conocido, `alembic upgrade head` (4 columnas con `\d tenant.order_items`), comprobar conteo igual y `SELECT count(*) FROM tenant.order_items WHERE addons_total <> 0` = 0, `alembic downgrade -1` (corre sin error con cero líneas con adicionales) y `upgrade head` otra vez. Registrar el resultado en `implementation-notes.md`. Confirmar además que `test_line_addons_migration.py` (T006) queda en verde

**Checkpoint**: esquema listo, comportamiento sin cambios; la suite de la línea base sigue verde.

---

## Phase 3: User Story 1 — Adicionales cobrados por las unidades elegidas, no por producto (Priority: P1) 🎯 MVP

**Goal**: en el Menú QR, `line_total = unit_price × quantity + addons_total` (2 hamburguesas de $15.000 + 1 tocino de $3.000 = **$33.000**), con precio, comanda, inventario, promociones y las cinco superficies de cobro coherentes; líneas históricas y de la terminal POS intactas.

**Independent Test**: agregar en el Menú QR 2 hamburguesas con 1 tocino ⇒ línea y carrito en $33.000; llegar al POS con ese total en terminal, checkout, "Pagos por confirmar" y detalle de venta; comanda "2 hamburguesas · Tocino x1"; inventario del tocino descuenta 1 (quickstart §2).

### Tests para User Story 1 (escribir primero; deben fallar contra el código actual) ⚠️

- [X] T010 [P] [US1] Actualizar `../pos-backend/app/characterization_tests/test_catalog_line_pricing.py` (citar **A-94**): casos nuevos con `ChosenOption(per_line=True)` — 2 × 15.000 + tocino x1 → `line_total` 33.000; cantidad 3 → 48.000; 1 producto + adicional x2 → 21.000; promoción 20 % con adicional → 27.000 (el descuento solo sobre 30.000); opción de grupo **incluido** (`per_line=False`) sigue por unidad; `compute_line_price` con `per_line=False` da el mismo resultado que hoy; línea histórica (`addons_total=0`) → 36.000 sin cambio; línea de **combo** (`combo_id`) con adicional → (precio del componente × cantidad) + adicionales. Los `CONGELA` de este archivo que dependan de la forma por unidad del adicional del Menú QR se actualizan citando A-94; los de la terminal POS/mostrador quedan **intactos**
- [X] T011 [P] [US1] Actualizar `../pos-backend/app/characterization_tests/test_catalog_consumption_plan.py` (citar **A-94**): `plan_line_consumption` con `per_line=True` descuenta `per_unit × chosen.quantity` (1, no 2, para 2 hamburguesas con 1 tocino); receta y grupo incluido siguen `× quantity`; **deduct == reverse**: el consumo por línea sigue la marca de la fila aunque después cambie `option_groups.pricing_type` (research D1); un adicional con precio $0 también consume por línea (D2)
- [X] T012 [P] [US1] Actualizar `../pos-backend/app/characterization_tests/test_cart_service.py` (citar **A-94**): `add_item` con grupo `con_recargo` guarda `unit_price` = solo presentación, `addons_total` = Σ y marca `per_line=True`; grupo incluido → `per_line=False`; con `settings.QR_ADDONS_PER_LINE=False` la línea nueva sale con la regla histórica (`addons_total=0`, extras dentro de `unit_price`); `serialize_cart` devuelve `line_total` 33.000 y el total de carrito correcto; promoción + adicional; una línea histórica (creada con `addons_total=0`) y una nueva conviven en el mismo carrito y el total es la suma; una línea de **combo** con adicional suma bien y `submit_cart` conserva `combo_id`; `submit_cart` copia `addons_total` y `per_line` al pedido sin recalcular
- [X] T013 [P] [US1] Actualizar `../pos-backend/app/characterization_tests/test_orders_consolidation.py` (citar **A-94**): `consolidate_table` copia `addons_total`, `per_line` y `combo_id` (líneas de combo con adicional incluidas); un pedido con una línea histórica + una línea nueva suma bien y descuenta inventario con la marca de cada fila; el descuento y la reversa de esas filas cuadran
- [X] T014 [P] [US1] Actualizar `../pos-backend/app/characterization_tests/test_orders_service.py` (citar **A-94**): `order_sale_lines`/`compute_bill` con `addons_total` producen `SaleLine.line_total` = 33.000 y `base_unit_price == unit_price` en líneas nuevas (la promoción nunca descuenta adicionales, D3); `OrderItemResponse` expone `addons_total` y `line_total`; las opciones de la respuesta (base de la comanda) conservan `quantity` = la cantidad elegida del adicional (x1 para 2 hamburguesas con 1 tocino, FR-006); las líneas de la terminal POS/mostrador (`per_line=False`) no cambian de total
- [X] T015 [P] [US1] Crear `../pos-backend/app/characterization_tests/test_fold_line_addons.py`: la simulación (por defecto) lista sin escribir; `--apply` pliega `addons_total / quantity` dentro de `unit_price` en `cart_items` solo cuando la división es exacta a 2 decimales y deja `addons_total=0` y `per_line=false`; las no exactas se listan para resolución manual sin tocarse; **no** modifica `order_items` ni ventas

### Implementación de User Story 1 — backend

- [X] T016 [US1] En `../pos-backend/app/catalog_engine/core.py` (contracts/addons-per-line.md §1): `ChosenOption` gana `per_line: bool = False`; añadir `compute_unit_price(variant, options)` (presentación + Σ extra × quantity de las opciones `per_line=False`), `compute_addons_total(options)` (Σ de las `per_line=True`) y `line_total(unit_price, quantity, addons_total=0) = unit_price × quantity + addons_total`; `compute_line_price` **conserva firma y resultado** para todo llamador existente (= `compute_unit_price + compute_addons_total`). Hace pasar T010
- [X] T017 [US1] En `../pos-backend/app/catalog_engine/consumption.py`: `plan_line_consumption` calcula, para opciones con `per_line=True`, `per_unit × chosen.quantity` (sin `× quantity` de la línea); receta de la variante y opciones `per_line=False` siguen `× quantity`. Como descuento, reversa y disponibilidad pasan por esta función, las tres cuentas cuadran (FR-007). Actualizar el docstring. Hace pasar T011. Depende de T016
- [X] T018 [US1] En `../pos-backend/app/api/v1/orders/checkout.py`: crear el helper `chosen_from_rows(db, item)` que reconstruye `ChosenOption(option, quantity, per_line)` **leyendo `per_line` de la fila** `OrderItemOption`, y reemplazar `_item_options` (~l.131) por él; todo llamador de descuento/reversa (`deduct_order_items`, `reverse_order_item`, anulación) usa la marca de fila. Depende de T016
- [X] T019 [US1] En `../pos-backend/app/api/v1/orders/kitchen.py`: reemplazar la copia local de `_item_options` (~l.38) por `chosen_from_rows` de T018, y en el reemplazo de línea copiar `addons_total` y las marcas `per_line`. Depende de T018
- [X] T020 [P] [US1] En `../pos-backend/app/api/v1/cart/schemas.py`: `CartItemResponse.addons_total: Decimal = 0` y `CartItemOptionResponse.per_line: bool = False` (aditivos; ningún campo existente cambia de tipo)
- [X] T021 [US1] En `../pos-backend/app/api/v1/cart/service.py`, `add_item` y `update_item`: marcar `per_line=True` en las opciones de grupos `pricing_type == 'con_recargo'` **solo si** `settings.QR_ADDONS_PER_LINE`; guardar `unit_price = compute_unit_price(...)` y `addons_total = compute_addons_total(...)`; persistir `CartItemOption.per_line`; al editar (`update_item`) reemplazar las filas de opciones con sus marcas nuevas. Una opción con cantidad 0 no crea fila. Depende de T016 y T020
- [X] T022 [US1] En `../pos-backend/app/api/v1/cart/service.py`, lectura y envío: `serialize_cart`, `_cart_promo_lines` (restar de `unit_price` solo opciones `per_line=False`, D3), `_cart_line_discount` (`discounted_line_total = line_total − descuento`; `discounted_unit_price = (discounted_line_total − addons_total) / quantity`), `_cart_consumption` (usar las marcas de la fila) y `submit_cart` (copiar `addons_total` y `per_line` al `OrderItem`/`OrderItemOption`) usan `line_total`. Hace pasar T012. Depende de T021
- [X] T023 [US1] En `../pos-backend/app/api/v1/cart/router.py`: el total del evento `order.created` (~l.161) se calcula con `line_total` (`unit_price × quantity + addons_total`), no con `unit_price × quantity`. Depende de T022
- [X] T024 [P] [US1] En `../pos-backend/app/api/v1/orders/schemas.py`: `OrderItemResponse.addons_total: Decimal = 0` y `OrderItemResponse.line_total: Decimal` (nuevo; antes el frontend lo calculaba) y `OrderItemOptionResponse.per_line: bool = False`
- [X] T025 [P] [US1] En `../pos-backend/app/api/v1/sales/builder.py`: `SaleLine` gana `addons_total` (default 0); `line_total = unit_price × quantity + addons_total`; `base_unit_price` resta del `unit_price` solo las opciones con `per_line=False`; el snapshot JSONB `options` de `SaleItem` gana `"per_line": true` en líneas nuevas (ausente = `false` en las antiguas). `sale_items` **sin columna nueva** (data-model §1)
- [X] T026 [US1] En `../pos-backend/app/api/v1/orders/checkout.py`: `compute_bill` (~l.237: arma `BillItemLine` con `unit_price × quantity` en línea, incluido el reparto `split` por comensal) usa `line_total(unit_price, quantity, addons_total)`; `order_sale_lines` (~l.296-303) pasa `addons_total` a `SaleLine` y añade `"per_line"` al dict `options` del snapshot (lo declarado en T025); `release_table` sin cambio. Hace pasar T014. Depende de T018 y T025
- [X] T027 [US1] En `../pos-backend/app/api/v1/orders/consolidation.py`: `consolidate_table` copia `addons_total` y `per_line` (nunca los recalcula) y usa `chosen_from_rows`; `add_item_to_order` (mesero) sigue con `per_line=False` (decisión de negocio "solo Menú QR"). Hace pasar T013. Depende de T018
- [X] T028 [US1] En `../pos-backend/app/api/v1/table_sessions/service.py`: `compute_bill` (vía `SaleLine`) usa `addons_total`; `set_assignments` ("repartir por unidades", ~l.590-610): la fila original conserva `addons_total` y sus opciones `per_line`; las filas nuevas nacen con `addons_total = 0` y **sin** opciones `per_line`; las opciones `per_line=false` se clonan como hoy. Añadir un comentario que cite **A-97** para el defecto previo de `OrderItemOption.quantity` **sin corregirlo**. Depende de T025
- [X] T029 [P] [US1] En `../pos-backend/app/api/v1/orders/tables_advanced.py`: confirmar que los cálculos de cuenta dividida pasan por `SaleLine`/`line_total` y ajustar cualquier `unit_price × quantity` suelto restante. Depende de T025
- [X] T030 [US1] Crear `../pos-backend/app/scripts/fold_line_addons.py` (emergencia, data-model §5 nivel 3): `python -m app.scripts.fold_line_addons` = **simulación** por defecto; `--apply` pliega solo `cart_items` con `addons_total` divisible entre `quantity` a 2 decimales; lista las no exactas; no toca `order_items` ni ventas. Sigue el estilo de los demás scripts de `app/scripts/`. Hace pasar T015
- [X] T031 [US1] Checkpoint de backend: correr la suite completa de `../pos-backend`; barrer `grep -rnE "unit_price\s*\*\s*(item\.|line\.|ci\.)?quantity|\.unit_price \* " ../pos-backend/app --include=*.py` para confirmar que **no queda** ninguno de los once sitios de research D1 multiplicando `unit_price × quantity` sin `addons_total`; verificar que los `CONGELA` actualizados citan A-94 y que el resto sigue verde; anotar el resultado en `implementation-notes.md`

### Implementación de User Story 1 — frontend

- [X] T032 [P] [US1] En `tables/interfaces/dining.interface.ts` y `tables/interfaces/diner.interface.ts` (y `dining.interface.spec.ts` si tipa estos modelos): añadir `addons_total`, `line_total` en ítems de pedido, y `per_line` en opciones (todos opcionales/aditivos); crear un helper puro `lineTotal(item)` = `item.discounted_line_total ?? item.line_total ?? item.unit_price * item.quantity + (item.addons_total ?? 0)` en el mismo archivo o en un util existente de `tables/` (el descuento persistido ya incluye los adicionales, research D3), con su prueba: línea sin descuento, con promoción y adicional, histórica y sin los campos nuevos
- [X] T033 [P] [US1] En `tables/components/product-select.component.ts` (+ `product-select.component.spec.ts`): `@Input() addonsPerLine = false`; con `true`, el total del pie usa la regla nueva (`precio × cantidad del producto + Σ adicionales de grupos con recargo`, sin auto-escalar el adicional al cambiar la cantidad del producto); con `false` (terminal POS/pedido manual) el cálculo actual no cambia. Pruebas: 2 × 15.000 + tocino x1 → 33.000; cantidad 3 → 48.000; `addonsPerLine=false` → comportamiento anterior
- [X] T034 [P] [US1] En `tables/services/dining-cart.service.ts` (+ spec): `CartLine` gana `addonsTotal`; el total del carrito y el de cada línea se calculan con `line_total` (`?? unit_price × quantity + addons_total`); cambiar la cantidad del producto no toca `addonsTotal`. Depende de T032
- [X] T035 [US1] En `tables/components/cart.component.ts` (+ spec): la etiqueta "c/u" usa `unit_price` (presentación, sin adicionales), cada adicional se muestra con su multiplicador exacto ("Tocino x1", spec 087 FR-013) y el total de línea usa `line_total`. Depende de T034
- [X] T036 [US1] En `tables/pages/public-menu.component.ts` (+ `public-menu.component.spec.ts`): `itemLineTotal`/`itemOriginalLineTotal` y el historial del comensal leen `line_total` (con el fallback de T032); la fila del historial `{{ item.quantity }} × {{ item.unit_price }}` (~l.351) se mantiene como precio de la presentación y los adicionales se muestran como fila aparte con su multiplicador, de modo que la suma visible cuadre con el total; pasar `[addonsPerLine]="true"` al `ProductSelectComponent` del Menú QR. Depende de T032 y T033
- [X] T037 [US1] En `tables/services/pos-terminal.store.ts` (+ `pos-terminal.store.spec.ts`): las lecturas de `itemUnitPrice`/subtotales de líneas de pedido existentes (~l.967-990, ~1725, ~2380-2394) usan `lineTotal(item)`; las líneas que arma la propia terminal (borrador) **no cambian** (regla por unidad, decisión de negocio). También suman `addons_total` el `subtotal` de la línea 986 (que puede partir de un `unitPrice` descontado) y la rama de combos de la línea ~2393; la rama con `discounted_line_total` (~l.2386) ya es autoritativa. Pruebas: pedido con una línea histórica + una nueva del Menú QR suma correctamente, y línea con promoción y adicional suma `discounted_line_total` sin perder el adicional. Depende de T032
- [X] T038 [P] [US1] En `tables/components/pos-order-panel.component.ts` (+ spec): el total de cada línea del pedido usa `lineTotal(item)`. Depende de T032
- [X] T039 [P] [US1] En `tables/components/payment-attempt-review-panel.component.ts` (~l.371) (+ spec): el subtotal de cada línea usa `lineTotal(item)` en lugar de `unit_price × quantity` (solo la lectura del importe; el modal se trata en US2 T047). Depende de T032
- [ ] T040 [US1] Checkpoint de US1: `ng test` completo de `../pos-heladeria` en verde y recorrido manual de quickstart §2 (pasos 1–9: mismo total en las cinco superficies, comanda, inventario descuenta 1, reversa devuelve 1, pedido anterior intacto, `QR_ADDONS_PER_LINE=false`). Anotar en `implementation-notes.md`

**Checkpoint**: US1 completa y verificable por sí sola; backend + frontend coinciden en $33.000.

---

## Phase 4: User Story 2 — Total inmediato y sin modales al agregar productos a una orden abierta en el POS (Priority: P1)

**Goal**: al guardar productos en una orden abierta (mesa, domicilio, para llevar) el total visible se actualiza al instante (estimado local) y el servidor lo confirma; se retiran los tres modales "El total cambió"; nunca se cobra un importe que el cajero no vio; el agotado nombra el producto.

**Independent Test**: orden de $20.000, agregar 1 gaseosa de $5.000 ⇒ total visible $25.000 en < 0,2 s sin diálogos; elegir método de pago y Cobrar ⇒ venta por $25.000 sin "El total cambió" (quickstart §5).

### Tests para User Story 2 ⚠️

- [X] T041 [P] [US2] Ampliar `../pos-backend/app/characterization_tests/test_orders_service.py` (citar **A-96**; mismo archivo que T014, ejecutar después): al agregar una línea cuyo insumo se agota (`create_order`, `add_item_to_order` y el reemplazo de `kitchen.py`) la respuesta es `400` con `detail = {"error": "«<producto · presentación>» está agotado: falta <insumo>", "producto": ..., "insumo": ...}` y **no** queda ninguna línea a medias ni descuento parcial; el status sigue siendo el de `InsufficientStockError`

### Implementación de User Story 2 — backend (agotado con nombre, D18)

- [X] T042 [US2] Crear en `../pos-backend/app/catalog_engine/consumption.py`, junto a `variant_label`, un helper `sold_out_detail(variant, exc)` que devuelve el `detail` de `contracts/pos-total-immediate.md` §6 a partir de `InsufficientStockError`; usarlo en `../pos-backend/app/api/v1/orders/service.py::create_order` capturando el error del descuento y re-lanzándolo con ese detalle (mismo status 400). Depende de T017 (mismo archivo de consumo, ya con `per_line`)
- [X] T043 [US2] En `../pos-backend/app/api/v1/orders/consolidation.py::add_item_to_order`: capturar el error del descuento y re-lanzarlo con `sold_out_detail` (T042). Hace pasar la parte de T041 de agregar líneas. Depende de T027 y T042
- [X] T044 [P] [US2] En `../pos-backend/app/api/v1/orders/kitchen.py` (reemplazo de línea): mismo manejo con `sold_out_detail`. Depende de T019 y T042

### Implementación de User Story 2 — frontend

- [X] T045 [US2] En `tables/services/pos-terminal.store.ts` (+ `pos-terminal.store.spec.ts`, citar **A-96**): añadir `checkoutPreviewEstimate = signal<{subtotal:number; total:number} | null>(null)` y reescribir el flujo de `saveOrder()` según contracts/pos-total-immediate.md §2: (1) antes de la primera llamada publicar `estimate = total confirmado + Σ draftLines.subtotal` (síncrono); (2) enviar las líneas secuenciales con `submitting=true` (serializa altas concurrentes); (3) éxito → `reload()` + `loadCheckoutPreview(orderId)` y limpiar el estimado, sin diálogo aunque el servidor difiera; (4) fallo en una línea → limpiar el estimado, `loadCheckoutPreview(orderId)` (refleja las líneas que sí se guardaron) y toast con `detail.error` (nombra el producto agotado); (5) servidor sin respuesta → igual que (4) y, si el preview también falla, `checkoutPreview = null` ("Calculando el total…", Cobrar deshabilitado). Pruebas: estimado publicado antes del primer POST; limpiado tras el preview; fallo revierte y muestra el nombre del producto; tres altas seguidas suman una sola vez; sin servidor no queda estimado como confirmado (FR-031). Depende de T037 (mismo archivo)
- [X] T045b [US2] En `tables/services/pos-terminal.store.ts` (+ spec, citar **A-96**): FR-025 también cubre **quitar y reemplazar**. `voidPersistedItem` (~l.1820) y el reemplazo de línea publican, antes de llamar al servidor, `checkoutPreviewEstimate = confirmado.total − lineTotal(línea)` (en el reemplazo: menos la línea vieja, más la nueva); tras `reload()` llaman a `loadCheckoutPreview(selectedOrderId)` y limpian el estimado; si el servidor falla, revierten al confirmado como en T045 (4) y (5). La confirmación "¿Anular este ítem?" **se conserva** (no es el modal de total). Pruebas: anular una línea baja el total al instante; un fallo revierte; ningún `confirm.ask` de total. Añadir el flujo a contracts/pos-total-immediate.md §2b. Depende de T045
- [X] T046 [US2] En `tables/components/pos-checkout-panel.component.ts` (+ spec, citar **A-96**): `checkout()` sin `confirm.ask` según contracts §3 — no cobrar mientras haya estimado, esté cargando o no haya preview; volver a pedir el preview; si `fresh.total != visible` mostrar aviso **no bloqueante dentro del panel** ("El total cambió: ahora es $X (antes $Y). Revisa el importe y vuelve a pulsar Cobrar.") y **no cobrar**; el siguiente Cobrar cobra el importe visible; Cobrar deshabilitado con `checkoutPreviewEstimate() != null`; indicador discreto "actualizando…" junto al total; el aviso se descarta al volver a pulsar Cobrar, al cambiar de orden o al llegar un preview sin diferencia; la barra `checkoutPreviewStale` (evento `session.bill_changed`) se **conserva**. Reescribir los casos del spec que esperaban el modal. Depende de T045 y T045b (mismo archivo de store)
- [X] T047 [P] [US2] En `tables/components/payment-attempt-review-panel.component.ts` (+ spec, citar **A-96**): `reconfirmIfTotalChanged()` sin `confirm.ask`; si el total cambió, actualiza `checkoutPreview`, muestra el aviso no bloqueante **en la tarjeta** y **no resuelve** el pago (el siguiente clic usa el total nuevo); `actionsBlocked()` deja de exigir `totalChangeAck` (sigue bloqueando mientras carga o no hay total autoritativo); el texto informativo `totalChanged()` (difiere del declarado por el comensal) se conserva sin exigir acuse (research D17, FR-030). Depende de T039 (mismo archivo)
- [X] T048 [P] [US2] En `tables/pages/manual-order-page.component.ts` (+ spec, citar **A-96**): `confirm()` sin modal — si al volver a pedir `draft-preview` el total difiere, actualizar la cifra, mostrar el aviso no bloqueante y **no crear** el pedido hasta un segundo clic (FR-015a de la spec 073 → FR-028)
- [ ] T049 [US2] Checkpoint de US2: `grep -rn "El total cambió" ../pos-heladeria/src` solo debe encontrar textos del aviso no bloqueante y la barra `checkoutPreviewStale`, **ningún** `confirm.ask` con ese título (SC-010); `ng test` completo en verde; recorrido manual de quickstart §5 pasos 1–11 (mesa, domicilio y para llevar; anular línea; agotado con nombre; cambio externo; backend detenido), midiendo con las herramientas de rendimiento del navegador que el total estimado aparece en < 0,2 s (SC-008). Anotar en `implementation-notes.md`

**Checkpoint**: US1 y US2 (las dos P1) funcionan y se prueban de forma independiente.

---

## Phase 5: User Story 3 — Editar los adicionales de un producto ya agregado al carrito (Priority: P2)

**Goal**: el cliente edita o quita los adicionales y la nota de una línea del carrito sin eliminarla, en ≤ 3 interacciones, con el total recalculado por la regla de US1.

**Independent Test**: producto con un adicional en el carrito ⇒ "Editar adicionales" ⇒ cambiar/quitar ⇒ guardar ⇒ misma línea (producto, presentación, cantidad) con total nuevo; cancelar no llama al backend (quickstart §3).

**Dependencias**: requiere US1 (regla de cálculo y campos nuevos); el backend ya soporta `PATCH /cart/items/{id}` (research D6), el trabajo es sobre todo frontend.

### Tests para User Story 3 ⚠️

- [X] T050 [P] [US3] Ampliar `../pos-backend/app/characterization_tests/test_cart_service.py` (mismo archivo que T012, ejecutar después): `update_item` con `options` reemplaza la selección, recalcula `unit_price`/`addons_total` y marca `per_line`; quitar todos los adicionales deja la línea sin eliminarla ($30.000 en el ejemplo de 2 hamburguesas); una opción con cantidad 0 equivale a quitarla (sin fila "x0"); selección que viola mínimo/máximo/tope por opción → 422 con el mensaje del grupo; adicional inactivo o sin stock → 422/409; un comensal **no** puede editar líneas de otro participante ni líneas ya enviadas (404); editar dos líneas hasta que queden idénticas no las fusiona

### Implementación de User Story 3

- [X] T051 [US3] En `../pos-backend/app/api/v1/cart/service.py::update_item`: corregir lo que T050 revele (cantidad 0 como quitar, mensajes 422/409 claros, sin fusión de líneas). Si T050 pasa sin cambios, dejar constancia en `implementation-notes.md`. Depende de T022 y T050
- [X] T052 [P] [US3] En `tables/services/dining-cart.service.ts` (+ spec): `updateItem(itemId, options, notes)` que llama `PATCH /cart/items/{id}` con `{options, notes}` (la `quantity` no cambia) y actualiza la línea en su lugar. Depende de T034
- [X] T053 [P] [US3] En `tables/components/product-select.component.ts` (+ spec): en modo edición (`initialSelection`) ignorar `existingQtyFor` y el paso de promoción (spec 081), porque la cantidad no cambia; marcar como **"no disponible"** el adicional inactivo o sin stock (calculable desde `menu-lookup`) y bloquear "Guardar cambios" hasta quitarlo o reemplazarlo; permitir dejar la selección de adicionales vacía. Depende de T033
- [X] T054 [US3] En `tables/components/cart.component.ts` (+ `cart.component.spec.ts`): acción **"Editar adicionales"** por línea (icono + texto, objetivo táctil ≥ 44 px) emitiendo un evento con la línea; no se ofrece si la línea ya se envió ni si su producto/presentación está desactivado (solo quitar). Depende de T035
- [X] T055 [US3] En `tables/pages/public-menu.component.ts` (+ spec): al recibir el evento de T054 abrir `ProductSelectComponent` con `initialSelection` (variante, cantidad, opciones y notas de la línea) y `[addonsPerLine]="true"`; **Guardar cambios** → `dining-cart.updateItem` (T052); **Cancelar** o cerrar no llama al backend y deja la línea intacta; si el backend responde 422 mostrar su mensaje **sin cerrar** el selector; cada comensal solo ve y edita su propio carrito. Depende de T036, T052, T053 y T054
- [ ] T056 [US3] Checkpoint de US3: `ng test` y suite de backend en verde; recorrido manual de quickstart §3 pasos 1–8 (incluida la cuenta de interacciones: abrir, modificar, guardar = 3). Anotar en `implementation-notes.md`

**Checkpoint**: US1, US2 y US3 funcionan de forma independiente.

---

## Phase 6: User Story 4 — Al cerrar la mesa, el cliente ve "¡Gracias por tu visita!" y su sesión queda invalidada (Priority: P2)

**Goal**: **todo** camino que cierra la sesión notifica al comensal; el Menú QR muestra siempre la pantalla de gracias (sin historial, recibo ni botón), limpia carrito y sesión, y rechaza cualquier acción con ese token, incluso al reabrir la mesa.

**Independent Test**: con un teléfono conectado, cerrar la mesa por cada camino (a–e) y verificar en < 5 s la pantalla de gracias, 401 con `X-Session-State: closed` al reintentar y que solo un nuevo escaneo crea sesión (quickstart §4).

### Tests para User Story 4 ⚠️

- [X] T057 [P] [US4] Ampliar `../pos-backend/app/characterization_tests/test_table_sessions_router.py` (contracts/session-closed.md §4): `release_table` publica exactamente un `session.closed` con `reason="released"` **después** del commit y ninguno si responde 409 por órdenes sin cerrar; `try_release_if_empty` cierra y dispara el evento con `reason="empty"` desde `leave_session`, desde el otro llamador de `cart/service.py` y desde `qr_context._abandon_expired`; los caminos que ya emitían (`paid`, `swept`) no cambian; el 401 lleva `X-Session-State: closed` **solo** en los dos casos de cierre (`Sesión no activa`, `La mesa ya no tiene esta sesión abierta…`) y **no** en vencimiento (`Sesión expirada.`, inactividad, duración máxima); el cuerpo y los mensajes de los 401 no cambian; el token de la sesión anterior sigue en 401 tras reabrir u ocupar la mesa (FR-017); FR-016: con la sesión cerrada, **cada una** de las acciones del comensal (enviar pedido, iniciar pago, adjuntar comprobante, cancelar, actualizar/editar el carrito) responde 401 (test parametrizado sobre esa lista), incluso con cuentas pendientes
- [X] T058 [P] [US4] Ampliar `../pos-backend/app/characterization_tests/test_table_sessions_service.py`: `try_release_if_empty` devuelve `list[TableSession]` (vacía = no cerró; truthiness preservada para quien solo evaluaba `if try_release_if_empty(...)`); `notify_sessions_closed` publica un evento por sesión y **no lanza** si el publicador falla (best-effort)

### Implementación de User Story 4 — backend

- [X] T059 [US4] En `../pos-backend/app/api/v1/table_sessions/service.py` y `../pos-backend/app/core/events.py`: crear `notify_sessions_closed(tenant_id, sessions, reason)` (best-effort, mismo `try/except` que `events.publish`; publica `events.session_closed` por sesión); cambiar `try_release_if_empty(db, table_session_id)` (~l.88) de `bool` a `list[TableSession]`; documentar en el docstring de `events.session_closed` que `reason ∈ paid | swept | released | empty`. Hace pasar T058
- [X] T060 [US4] En `../pos-backend/app/api/v1/cart/service.py`: en los **dos** llamadores de `try_release_if_empty` (entre ellos `leave_session`), hacer `commit` y **después** llamar a `notify_sessions_closed(..., reason="empty")` (el evento nunca antes del commit). Depende de T059 y T022 (mismo archivo, `cart/service.py`)
- [X] T061 [P] [US4] En `../pos-backend/app/api/v1/orders/router.py::release_table` (~l.565): tras el commit llamar a `notify_sessions_closed(tenant.id, closed_sessions, reason="released")` con las sesiones que cerró `close_table_sessions`; `close_table_sessions` **no** cambia (corre antes del commit). Depende de T059
- [X] T062 [US4] En `../pos-backend/app/core/qr_context.py`: (a) `_abandon_expired` commit + `notify_sessions_closed(..., reason="empty")` usando el `tenant` del contexto; (b) los dos `HTTPException(401)` de cierre (`participant.status != "open"` y sesión no activa/distinta) llevan la cabecera `X-Session-State: closed` (`headers={"X-Session-State": "closed"}`), sin tocar `detail`; los de vencimiento/duración/firma **no** la llevan. Depende de T059
- [X] T063 [P] [US4] En `../pos-backend/app/main.py`: añadir `"X-Session-State"` a `expose_headers` de `CORSMiddleware`. Hace pasar T057 junto con T060–T062

### Implementación de User Story 4 — frontend

- [X] T064 [P] [US4] En `tables/services/diner.service.ts` (+ `diner.service.spec.ts`): `call()` lee `X-Session-State` en un 401 y lanza `DinerSessionExpiredError` con `closed: boolean` (`true` solo si la cabecera vale `closed`); los mensajes actuales no cambian (los tests existentes siguen verdes); añadir `closed` al tipo del error en `tables/interfaces/diner.interface.ts` si está declarado allí
- [X] T065 [US4] En `tables/pages/public-menu.component.ts` (+ spec): `type MenuView` gana `'closed'`; plantilla de la vista con logo/nombre del negocio como `'exited'`, texto exacto **"¡Gracias por tu visita! Esperamos verte pronto."** y **sin** botones, enlaces, historial ni recibo; `expireSession(motivo: 'closed' | 'expired', mensaje?)`; `session.closed` (cualquier `reason`) → `'closed'`; un 401 con `closed = true` (sondeo, `resync`/`reconnected`, carga inicial `ngOnInit`, `showCartError`, envío, pago) → `'closed'`; un 401 sin la cabecera conserva la vista `'name'` con mensaje; si el comensal ya salió voluntariamente (`isExited`) se conserva `'exited'` sin cambios; al entrar a `'closed'`: `stopPolling`, `disconnectRealtime`, `tokenStore.clear()`, `cart.clear()`, `cart.clearDiner()`, `myOrders.set([])`, `orderError.set(null)`. Pruebas: vista por evento, vista por 401 con y sin cabecera, carrito y token limpios, sin controles, `exited` intacto, un pago por transferencia en curso no bloquea la pantalla (FR-019). Depende de T064; si US3 ya se hizo, ejecutar después de T055 (mismo archivo), pero no es un requisito lógico
- [ ] T066 [US4] Checkpoint de US4: suite de backend y `ng test` en verde; recorrido manual de quickstart §4 con los cinco caminos (a–e) anotando el tiempo hasta la pantalla (< 5 s por evento; en el barrido con pedidos por cobrar, ≤ 10 s por el sondeo, SC-005), reintento con token viejo ⇒ 401 con `X-Session-State`, teléfono sin conexión al cierre (SC-006), reabrir la mesa no revive la sesión, varios comensales, transferencia pendiente sigue en "Pagos por confirmar", y el que salió con "Salir" sigue viendo su pantalla. Anotar en `implementation-notes.md`

**Checkpoint**: US1–US4 funcionan de forma independiente.

---

## Phase 7: User Story 5 — Reporte de cierre de caja legible al imprimir (Priority: P3)

**Goal**: el reporte de cierre de turno se imprime completo, en negro sobre blanco, sin hojas en blanco, con tema claro y oscuro; recibos y hoja de QR no cambian.

**Independent Test**: cerrar un turno, "Imprimir / Exportar reporte" y comprobar en la **vista previa real** de Chrome y Firefox (claro/oscuro, 1 y ≥ 2 hojas) que todo se ve (quickstart §6).

**Nota**: independiente de US1–US4; puede empezar en cualquier momento tras el Setup. La causa **se reproduce antes de corregir** (FR-024).

- [X] T067 [US5] **Diagnóstico** (research D13, FR-024): reproducir la hoja en blanco con el código actual en Chrome (vista previa + `Emulate CSS media type: print`) y Firefox, tema claro y oscuro, turno cerrado con y sin movimientos; capturar el árbol computado de `app-dashboard-layout > div`, `main` y `app-cash-report` (alturas, `overflow`, `margin-left`, colores) y registrar en `implementation-notes.md` **cuál** de las hipótesis (1) shell `h-screen overflow-hidden`, (2) colores heredados del tema, (3) `lg:ml-64` con sidebar `fixed`, o una combinación, es la causa real. **No tocar CSS hasta tener este registro**; si la causa difiere de las hipótesis, ajustar T068/T069 y `research.md` D13
- [X] T068 [US5] En `../pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts` (+ `dashboard-layout.component.spec.ts`): en `@media print` **acotado a `body.printing-cash-report`** (junto al `:host-context` de la spec 087) liberar `height`, `overflow` y `margin-left` de los contenedores del shell, `position: static` y `display: block` en `main`; sin afectar recibo de venta, recibo de mesa ni hoja de QR (FR-023). Ajustar según la causa confirmada en T067. Depende de T067
- [X] T069 [US5] En `../pos-heladeria/src/app/modules/cash-register/components/cash-report.component.ts` (+ `cash-report.component.spec.ts`): en impresión forzar fondo `#fff` y color `#000` en `app-cash-report` y sus descendientes con `print-color-adjust: exact`, neutralizando colores de estado de baja legibilidad (`tagClass`/`diffClass`) y `break-inside: avoid` en filas de tablas; el menú lateral, encabezado y botones siguen ocultos y el nombre de archivo `<tenant-slug>-<DD-MM-YYYY>` (spec 087) no cambia. Solo pruebas de clase/estilo del componente (ningún test unitario ejecuta la impresión real). Depende de T067
- [ ] T070 [US5] Verificación en la vista previa real (quickstart §6 pasos 1–6): Chrome **y** Firefox, tema claro y oscuro, turno con movimientos, turno con ≥ 2 hojas (todas con contenido, sin hoja vacía intermedia), turno sin movimientos (encabezados y totales en cero, no hoja vacía), "Guardar como PDF" con el nombre correcto, y recibo de venta / recibo de mesa / hoja de QR **sin cambios**. Anotar resultado y navegadores en `implementation-notes.md`. Depende de T068 y T069

**Checkpoint**: las cinco historias completas.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Propósito**: verificación final y documentación (Principio X).

- [X] T071 Suite completa de `../pos-backend` (`python -m unittest discover -s app/characterization_tests -p 'test_*.py'`) en verde y comparada con la línea base de T004 (solo suben los tests nuevos; ningún `CONGELA` cambió sin cita a A-94/A-96)
- [X] T072 Suite completa de `../pos-heladeria` (`npx ng test --watch=false --browsers=ChromeHeadless`) en verde y comparada con la línea base de T004; `npx ng build` sin errores de tipos
- [ ] T073 Recorrido completo de [quickstart.md](./quickstart.md) §1–§7 y comprobar la tabla de criterios de salida (SC-001…SC-010); listar en `implementation-notes.md` los recorridos manuales que queden pendientes y por qué
- [X] T074 Completar `specs/089-fix-adicionales-cierre-mesa-caja-pos/implementation-notes.md`: causa real de la impresión (T067), decisiones tomadas y su porqué, **orden de despliegue** (backend con migración primero, luego frontend; sin volver a un binario anterior una vez existan líneas con `addons_total > 0`, usar `QR_ADDONS_PER_LINE=false`), pendientes de confirmación con el negocio (T002) y cualquier desviación respecto al plan
- [X] T075 Verificación de trazabilidad (Principio XII): `grep -n "A-94\|A-95\|A-96\|A-97" specs/000-reconocimiento/registro-de-anomalias.md` encuentra las cuatro entradas; `grep -rn "A-94\|A-96" ../pos-backend/app/characterization_tests ../pos-heladeria/src` confirma que todo test `CONGELA` modificado cita su decisión; `git status` en `pos-specs`, `../pos-backend` y `../pos-heladeria` — **no commitear** salvo que el usuario lo pida (Principio XV)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias. T002 no bloquea el desarrollo, sí el despliegue.
- **Foundational (Phase 2)**: depende del Setup; **bloquea US1 y US3** (columnas y modelos). **No** bloquea US2, US4 ni US5.
- **US1 (Phase 3)**: depende de Phase 2.
- **US2 (Phase 4)**: independiente de US1 en lógica; comparte archivos (`orders/service.py`, `consolidation.py`, `kitchen.py`, `pos-terminal.store.ts`, `payment-attempt-review-panel.component.ts`), así que sus tareas T042/T043/T044/T045/T045b/T047 se ejecutan **después** de las tareas de US1 que tocan el mismo archivo (T017, T027, T019, T037, T039) y en la rama creada desde `feat/089-…` (ver T003).
- **US3 (Phase 5)**: depende de US1 (regla de cálculo, `product-select`, `cart.component`, `public-menu`).
- **US4 (Phase 6)**: independiente; comparte `cart/service.py` (T060 tras T023) y `public-menu.component.ts` (T065 tras T055 si US3 ya se hizo).
- **US5 (Phase 7)**: totalmente independiente; el diagnóstico T067 puede hacerse desde el primer día.
- **Polish (Phase 8)**: depende de las historias que se vayan a entregar.

### User Story Dependencies

- **US1 (P1)**: tras Foundational. Sin dependencias de otras historias.
- **US2 (P1)**: tras Setup. Sin dependencia lógica de US1.
- **US3 (P2)**: tras US1.
- **US4 (P2)**: tras Setup. Sin dependencia lógica de otras.
- **US5 (P3)**: tras Setup. Sin dependencia de otras.

### Within Each User Story

- Tests de backend primero y fallando; después modelo/motor → servicio → endpoints/esquemas → frontend.
- La migración (T007) y los cambios de comportamiento (T016+) **nunca** en el mismo commit.
- Cada historia se cierra con su checkpoint manual antes de pasar a la siguiente prioridad.

### Parallel Opportunities

- Setup: T001, T002 y T003 en paralelo; T004 tras T003.
- Foundational: T005 y T006 en paralelo; T007 → T008 → T009.
- US1 tests T010–T015 (archivos distintos) todos en paralelo; T020, T024, T025, T029 en paralelo con lo demás una vez cumplidas sus dependencias; en frontend T032 primero y luego T033, T034, T038 y T039 en paralelo.
- US4: T057 y T058 en paralelo; T061, T063 y T064 en paralelo entre sí.
- US5 (T067) en paralelo con cualquier otra historia.
- Con dos personas: una lleva US1 → US3 (rama `feat/…`) y otra US5 (rama `fix/089-cash-report-print`, independiente) hasta que US1 esté lista; US2 y US4 (rama `fix/089-table-close-pos-total-print`, desde `feat/…`) se hacen después.

---

## Parallel Example: User Story 1

```bash
# Tests de backend de US1 (archivos distintos), juntos:
Task: "Actualizar test_catalog_line_pricing.py con los casos 33.000 / 48.000 / adicional x2 / promoción / histórica (A-94)"
Task: "Actualizar test_catalog_consumption_plan.py con per_line y deduct == reverse (A-94)"
Task: "Actualizar test_cart_service.py con add/update per_line y total de carrito (A-94)"
Task: "Actualizar test_orders_consolidation.py con copia de marcas y línea histórica + nueva (A-94)"
Task: "Actualizar test_orders_service.py con SaleLine y addons_total (A-94)"
Task: "Crear test_fold_line_addons.py"

# Esquemas y builder (archivos distintos), juntos:
Task: "cart/schemas.py: addons_total y per_line"
Task: "orders/schemas.py: addons_total, line_total y per_line"
Task: "sales/builder.py: SaleLine con addons_total"
```

---

## Implementation Strategy

### MVP First (US1 + US2, las dos P1)

1. Setup → Foundational (esquema, sin cambio de comportamiento).
2. US1 completa (backend + frontend) → **STOP y VALIDAR** con el quickstart §2 (el error de dinero más grave queda corregido).
3. US2 (total inmediato) → validar quickstart §5. Con US1 + US2 las dos P1 están entregadas.

### Incremental Delivery

1. Setup + Foundational → esquema listo.
2. US1 → validar → desplegable (backend con migración primero, luego frontend).
3. US2 → validar → desplegable.
4. US3 → validar (edición de adicionales).
5. US4 → validar (cierre de mesa por los cinco caminos).
6. US5 → validar en vista previa real (impresión).
7. Polish y `implementation-notes.md`.

Cada historia agrega valor sin romper las anteriores; si algo falla en producción, `QR_ADDONS_PER_LINE=false` apaga la regla nueva de US1 sin perder datos.

---

## Notes

- **[P]** = archivos distintos, sin dependencia pendiente.
- **[Story]** enlaza cada tarea con su historia para trazabilidad (Principio XII).
- Verificar que los tests de backend fallan antes de implementar y que los `CONGELA` actualizados citan **A-94** / **A-96**.
- **No commitear** salvo que el usuario lo pida; cuando lo haga: commits pequeños por historia y por unidad (migración; modelo/fórmula; consumo; cada superficie de cobro; evento; vista; CSS; store), en inglés, Conventional Commits, sin marcas de IA.
- Evitar: tareas vagas, conflictos en el mismo archivo entre tareas [P], dependencias entre historias que rompan su independencia, y corregir CSS de impresión sin haber reproducido antes la causa.

---

## Phase 9: Convergence

**Origen**: `/speckit-converge` (2026-09-30). `tasks.md` se generó **antes** de la segunda clarificación de `/speckit-clarify` (FR-015a/b/c, FR-017a, SC-011, US4 escenarios 10–13, research D9b, `contracts/session-closed.md` §3), así que T057–T066 no la cubren: hoy el cierre de mesa solo hace `tokenStore.clear()` y no deja la marca por pestaña, una recarga pide el nombre y abre una sesión nueva, "Salir" y el cierre tienen dos borrados distintos y el texto de acceso denegado sigue siendo el antiguo. Todas estas tareas son de `../pos-heladeria` (`tables/` = `../pos-heladeria/src/app/modules/tables/`), citan **A-95** (enmendada) y van en la rama de US4 (`fix/089-table-close-pos-total-print`, o `feat/089-…` si aún no se dividió). Las tareas abiertas T002, T040, T049, T056, T066, T070 y T073 **siguen vigentes** y no se duplican aquí.

- [X] T076 [US4] Crear `DinerTokenStore.endAccess(tableToken)` en `tables/services/diner-token.store.ts` (+ `diner-token.store.spec.ts`, citar **A-95**) como **único** borrado de fin de acceso, según research D9b y contracts/session-closed.md §3: (a) `clear()` del token de sesión; (b) elimina de `localStorage` y de `sessionStorage` **toda** clave con prefijo `pos.diner.` (token de sesión, `checkout_progress.*`, cualquier dato de comensal) y expira defensivamente cualquier cookie `pos.diner*`; (c) al final escribe **solo** `markExited(tableToken)`. Falla silenciosa con almacenamiento bloqueado (la purga en memoria igual ocurre). Pruebas: tras `endAccess` no queda ninguna clave `pos.diner.*` salvo `pos.diner.exited_token` = token público en `sessionStorage`; `localStorage` y cookies quedan sin datos del comensal; con `sessionStorage` bloqueado no lanza. `markExited`/`isExited` se conservan. per FR-015a (missing)
- [X] T077 [US4] En `tables/pages/public-menu.component.ts` (+ spec, citar **A-95**): `exit()` (~l.1404) y `expireSession('closed')` (~l.1657) llaman a `tokenStore.endAccess(this.token)` en lugar de `clear()` / `markExited()+clear()` (una sola lógica, sin dos copias); `exit()` pasa a mostrar la vista `'closed'` (gracias) en lugar de `'exited'` al instante; un evento o 401 tardío que llegue con la vista ya en `'closed'` **no** la cambia a `'exited'` (hoy `expireSession` consulta `isExited` y la pisaría). Conservar el orden de `expireSession` (detener sondeo/SSE, vaciar carrito, `myOrders`, `orderError` y cerrar overlays). Depende de T076. per FR-015a / FR-015c (partial)
- [X] T078 [US4] En `tables/pages/public-menu.component.ts` → `ngOnInit` (+ spec, citar **A-95**): en el `catch` de la carga inicial (`cart.load()`/`refreshOrders()`, ~l.1029), un `DinerSessionExpiredError` con `closed === true` (token de una sesión anterior en `localStorage` o en `?s=`, con la mesa libre, cerrada o reocupada) ejecuta `endAccess(this.token)` + vista `'exited'` (acceso denegado), **no** `'closed'`; el 401 sin la cabecera conserva `'name'` con su mensaje. Un 401 `closed` **durante el uso** (sondeo, acción, `resync`/`reconnected`, evento) sigue yendo a `'closed'`. Reescribir el caso del spec ~l.921 ("un 401 CON cabecera al cargar… lleva a gracias"), que hoy congela lo contrario de lo exigido. Depende de T077. per FR-017a / FR-015b / US4-AC13 (contradicts)
- [X] T079 [US4] En `tables/pages/public-menu.component.ts` (plantilla de `view() === 'exited'`, ~l.114-130) (+ spec, citar **A-95**): una **única** pantalla estática de acceso denegado con el texto exacto "Por favor, escanea nuevamente el código QR de la mesa para ingresar al menú" (sin el encabezado "¡Gracias por tu visita a …!", sin campo de nombre, menú, botón ni enlace), con `data-testid="vista-acceso-denegado"`; reemplaza el texto "Acceso finalizado, vuelve a escanear el código QR de tu mesa" y se actualizan las pruebas que lo esperaban (~l.168 y ~l.957-966 de `public-menu.component.spec.ts`). Actualizar el comentario HTML de la plantilla. per FR-015b (contradicts)
- [X] T080 [US4] Pruebas de comportamiento de SC-011 / US4 escenarios 10–12 en `tables/pages/public-menu.component.spec.ts` (citar **A-95**): (a) tras `session.closed` por cada `reason` (`paid`, `swept`, `released`, `empty`) y tras "Salir", el estado del navegador es **idéntico** (sin `pos.diner.session_token`, `checkout_progress.*`, datos de comensal ni cookies; solo `pos.diner.exited_token` = token público en `sessionStorage`); (b) recargar (nuevo `ngOnInit`) con la marca presente ⇒ vista `'exited'` y **cero** llamadas a `openSession`/`confirmName`; (c) pestaña sin marca (sin `sessionStorage`) ⇒ `'name'` y sesión nueva posible; (d) `sessionStorage` bloqueado ⇒ la pantalla de gracias se muestra sin excepción; (e) "Salir" ya hecho y luego llega `session.closed` ⇒ no cambia de pantalla. Depende de T077–T079. per SC-011 (missing)
- [ ] T081 [US4] Recorrido manual de `quickstart.md` §4 pasos 5–10 (lo que T066 no detalla): para cada camino de cierre a–e, revisar en DevTools → Application que solo queda `pos.diner.exited_token` en `sessionStorage`; F5, "Atrás" y "Adelante" en la misma pestaña muestran la pantalla de acceso denegado y la pestaña de red no registra ningún `POST` de apertura de sesión; una pestaña nueva/escaneo nuevo entra por el nombre; `?s=<token viejo>` y `localStorage` con token viejo, con la mesa libre, cerrada y reocupada ⇒ `401` con `X-Session-State: closed` y acceso denegado. Anotar el resultado en `implementation-notes.md` (puede hacerse junto con T066). Depende de T080. per SC-011 (partial)
- [X] T082 [US5] Completar `research.md` D13 con la causa **confirmada** de la hoja en blanco, que difiere de las tres hipótesis originales según `implementation-notes.md` Fase 7 (telón del menú móvil `fixed inset-0 lg:hidden` pintado encima del reporte porque el navegador pagina a ~816 px; el shell `h-screen overflow-hidden` solo explica el recorte; el tema oscuro no existe en la app), tal como exigía T067 ("ajustar … research.md D13"). Solo documentación. per FR-024 / T067 (partial)
