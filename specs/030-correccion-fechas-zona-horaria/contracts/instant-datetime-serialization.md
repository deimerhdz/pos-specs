# Contrato transversal: formato UTC explícito para todo instante absoluto de la API

Aplica a los ~13 campos de las once entidades de Key Entities (ver `data-model.md`). No es un
endpoint nuevo — es una garantía de formato que **todos** los endpoints que ya devuelven uno de estos
campos deben cumplir después de esta spec.

## Antes de esta spec

```json
{
  "id": "b3e2...",
  "sold_at": "2026-08-24T12:53:07.123456",
  "status": "paid"
}
```

`sold_at` es una cadena ISO 8601 **sin offset** — representa un instante UTC (el dato ya es UTC en la
base de datos), pero nada en la cadena lo dice. Un cliente que la reciba (`new Date(...)` en
JavaScript, `datetime.fromisoformat(...)` en Python) la interpreta según su propia convención: en
Angular/JS, una cadena sin offset y sin `Z` se trata como **hora local del navegador**, no como UTC —
la causa raíz del defecto reportado.

## Después de esta spec

```json
{
  "id": "b3e2...",
  "sold_at": "2026-08-24T12:53:07.123456+00:00",
  "status": "paid"
}
```

`sold_at` lleva el offset `+00:00` explícito (equivalente a sufijo `Z`). Cualquier cliente,
Angular incluido, puede convertirlo correctamente a la zona horaria que necesite sin adivinar
(`FR-003`, Historia 1 Escenario 2).

## Campos cubiertos

Los ~13 campos listados en `data-model.md` → "Las once entidades" — `Sale.sold_at`,
`CustomerOrder.created_at`, `Payment.paid_at` (nuevo en el schema), `PaymentAttempt.created_at`/
`resolved_at`, `CashShift.opened_at`/`closed_at`, `CashMovement.occurred_at`,
`CashPartialCount.counted_at`, `InventoryMovement.moved_at`, `TableSession.opened_at`/`closed_at`,
`SessionParticipant.joined_at`/`closed_at`, `Invoice.issued_at`, `Purchase.purchased_at`,
`AuditLog.at`. Cada uno se declara con el tipo `UtcDatetime` (`app/core/timezone.py`) en su schema de
respuesta en vez de `datetime` — ver `plan.md` → Project Structure para el fichero exacto de cada uno.

## Campo explícitamente excluido

`Promotion.starts_at`/`ends_at` — representan una hora de pared local recurrente, no un instante
absoluto (`FR-009`). Su formato de transporte **no cambia** en esta spec.

## Compatibilidad

Cambio de formato, no de valor ni de nombre de campo — un cliente que ya parseaba la cadena anterior
como ISO 8601 (cualquier parser estándar, incluido `Date` de JavaScript y `datetime.fromisoformat` de
Python 3.11+) sigue funcionando: el offset es una extensión válida del mismo formato, no una ruptura.
El único comportamiento que cambia es el que ya era el defecto (interpretación implícita como hora
local sin offset) — que es precisamente lo que esta spec corrige.
