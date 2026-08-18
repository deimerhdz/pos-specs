# Research: Corrección del orden de borrado de imagen en R2 (A-44)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — el Technical Context (y
la Fase 0 en general) se resolvió por completo leyendo directamente `pos-backend`, sin necesidad de
una ronda de clarificación con negocio ("A-44 es de severidad Baja y no tiene corrección pendiente
en esta spec [002]"; el "tratamiento acordado" ya fija las dos alternativas posibles). Este
documento registra las decisiones de diseño tomadas para esta delta.

## Decisión 1 — Inversión síncrona del orden, no cola asíncrona

- **Decisión**: mover `delete_object(old_key)` de antes a después de `db.commit()`, dentro de la
  misma petición HTTP y la misma función `update_product` — sin introducir un worker, cola ni
  proceso separado.
- **Rationale**: de las dos alternativas que ofrece el "tratamiento acordado" de A-44 (invertir el
  orden vs. mover el borrado a un proceso asíncrono post-commit), la inversión síncrona resuelve el
  100% del defecto observado (referencia de imagen rota tras un fallo de commit) sin agregar
  infraestructura nueva (cola, worker, dependencia) que la Constitución (Principio IV) exigiría
  justificar y aprobar por separado. El propio registro de anomalías clasifica A-44 como severidad
  Baja — no amerita la complejidad operativa de una cola para un caso raro.
- **Alternatives considered**: proceso asíncrono post-commit (cola de trabajos) — descartado para
  esta delta por requerir una dependencia/infraestructura nueva no justificada para un caso raro de
  severidad Baja; queda documentado como fuera de alcance en `spec.md` para una futura delta si el
  negocio decide que el borrado debe sobrevivir a caídas del proceso entre el commit y el borrado
  (ver Edge Case CL4 de `spec.md`, riesgo residual que esta delta no resuelve).

## Decisión 2 — Cómo verificar el nuevo orden en un characterization test

- **Decisión**: crear `app/characterization_tests/test_products_service.py` (no existe hoy ningún
  `test_products_*.py`) y mockear `app.api.v1.products.service.delete_object` con
  `unittest.mock.patch`, registrando el orden relativo de la llamada mockeada frente a un `commit`
  de la sesión de SQLAlchemy (spy sobre `Session.commit`, mismo enfoque que ya usa el repo para
  espiar `app.core.events.*` en `test_table_sessions_service.py`) para afirmar
  `delete_object` se invoca **después** de `commit`.
- **Rationale**: es el mismo patrón ya establecido en el repositorio para aislar una dependencia
  externa (R2/eventos) de un test que corre contra SQLite en memoria (`fixtures.py`), sin abrir
  conexión de red real ni necesitar credenciales de R2. Verificar el *orden relativo* de las dos
  llamadas (en vez de solo que ambas ocurran) es lo que hace el test capaz de fallar si alguien
  reintroduce el orden defectuoso de A-44 en el futuro — el mismo tipo de regresión que ya sufrió
  A-04 al perderse en un merge (ver spec 020).
- **Alternatives considered**: verificar el orden por inspección de logs — descartado, es más frágil
  y no aísla la aserción del formato del mensaje de log; usar una base de datos real con un
  `SAVEPOINT` fallido para forzar el escenario de fallo de commit — descartado, SQLite en memoria ya
  permite forzar un fallo de commit lanzando una excepción desde un mock del propio `commit` sin
  necesitar Postgres real, consistente con el resto de `app/characterization_tests/`.

## Decisión 3 — Alcance sobre los dos endpoints que comparten el servicio

- **Decisión**: la corrección se aplica una sola vez en `products.service.update_product`, que es
  el único punto llamado tanto por `PATCH /products/{id}` como por `PUT /products/{id}`
  (`replace_product`, alias) — no se duplica lógica entre los dos endpoints del router.
- **Rationale**: verificado en `app/api/v1/products/router.py` — ambas rutas (`update_product` y
  `replace_product`) llaman literalmente a `service.update_product(db, id, body)`; corregir el
  servicio una sola vez corrige ambos caminos de entrada sin lógica adicional ni duplicada.
- **Alternatives considered**: ninguna — no hay una alternativa razonable a corregir la función
  compartida una sola vez.
