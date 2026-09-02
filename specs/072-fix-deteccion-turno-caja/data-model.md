# Phase 1 — Data Model

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-02

## Resumen: cero cambios de esquema, cero campos nuevos

**Esta corrección no crea, modifica ni elimina ninguna tabla, columna, entidad ni relación —
tanto en `pos-backend` como en el modelo de datos del cliente.** No lleva migración. No tiene
estrategia de rollback de datos porque no toca datos. El Principio VIII no se activa.

Las entidades ya existentes que esta corrección solo **lee de una forma distinta**:

```text
CashRegister (una o varias por tenant)
└── CashShift (abierto/cerrado, uno por caja a la vez)
```

- **`CashRegister`**: sin cambio. Se sigue leyendo con `GET /cash/registers`
  (`CashService.listRegisters()`), ahora también invocado desde `discoverOpenShift()` cuando el
  camino rápido de `localStorage` no resuelve nada (research.md D3).
- **`CashShift`**: sin cambio de forma ni de ciclo de vida (`open`/`closed`, spec 006). Se sigue
  leyendo con `GET /cash/shifts/current?cash_register_id=` (`CashService.getCurrentShift(id)`),
  ahora invocado potencialmente una vez **por cada** caja del tenant durante el descubrimiento,
  en vez de una sola vez para la caja que `localStorage` ya conocía.

## Único cambio de forma: el signal compartido `CashService.shift`

No es un cambio de **modelo de datos** (sigue siendo `CashShift | null`,
[cash.interface.ts](../../../pos-heladeria/src/app/modules/cash-register/interfaces/cash.interface.ts)),
sino de **quién lo puede poblar y cuándo**:

| Antes | Después |
|---|---|
| Solo se poblaba desde `localStorage['cash.register']` (vía `restoreShift()`) o al operar una caja explícitamente (`operateRegister()`). | Además se puebla automáticamente cuando existe **exactamente un** turno abierto en el tenant, descubierto listando las cajas (`discoverOpenShift()`, research.md D3). |
| Con más de un turno abierto simultáneamente y sin `localStorage`, quedaba en `null` igual que con cero turnos — indistinguible. | Con más de un turno abierto y sin `localStorage`, sigue quedando en `null` **a propósito** (FR-004) — no cambia el resultado, sí la razón: ahora es una decisión explícita de no adivinar, no un efecto colateral de nunca haber consultado. |

## Persistencia local (`localStorage`)

- **Clave**: `cash.register` (sin cambio de nombre).
- **Antes**: solo se escribía al hacer clic en "Operar" (`operateRegister()`).
- **Después**: además se escribe automáticamente cuando `discoverOpenShift()` resuelve un único
  turno abierto — así la próxima verificación en ese mismo navegador toma el camino rápido sin
  volver a listar todas las cajas. Sigue siendo un **acelerador**, nunca la única fuente de verdad
  (FR-001).

**Compatibilidad con datos existentes**: total. Ningún tenant necesita ninguna acción — un
navegador que ya tenía la clave correcta en `localStorage` sigue tomando el camino rápido
exactamente como hoy; uno que no la tenía (el caso del defecto) ahora la obtiene por descubrimiento
en vez de quedarse permanentemente en `null`.

**Estrategia de rollback**: revertir el commit del frontend. Ningún dato persistido (backend ni
`localStorage`) queda en un estado que dependa de este cambio para ser válido.
