# Data Model: Corrección global de fechas, horas y zonas horarias

Esta spec no crea ninguna tabla, y modifica exactamente **una** columna de esquema
(`shared.tenants.timezone`). El resto del "modelo de datos" que aporta esta spec no es de base de
datos: es el tipo de serialización (`UtcDatetime`) que las once entidades adoptan y el estado nuevo
del cliente frontend. Se documentan aquí para trazabilidad (Principio VIII/XII).

## Tenant (`shared.tenants`) — única tabla que cambia

| Atributo | Tipo | Antes | Después |
|---|---|---|---|
| `timezone` | `String` | No existe | `NOT NULL`, `server_default='America/Bogota'`; validado con `zoneinfo.ZoneInfo(value)` en cada escritura (`@validates`, research.md Decisión 3) — un valor que no sea un nombre IANA reconocido nunca se persiste (Clarifications). |

**Compatibilidad con filas existentes** (Principio VIII): el `server_default` asigna
`'America/Bogota'` a todo tenant existente sin requerir backfill de aplicación — Historia 4, Escenario
1 de spec.md ("un tenant sin zona horaria configurada explícitamente ... usa `America/Bogota` como
valor por defecto, sin cambio de comportamiento").

**Migración**: una sola revisión Alembic ordinaria contra `shared.tenants` (no requiere el fan-out por
esquema de tenant — `Tenant` vive en `shared`, ver research.md/plan.md, Migration tooling).
`downgrade()` elimina la columna.

**Quién la escribe**: únicamente `app/scripts/set_tenant_timezone.py` (research.md Decisión 4) o la
migración misma (valor por defecto). `TenantUpdateRequest`/`PATCH /tenant` **no** incluye este campo —
ver `contracts/tenant-info-endpoint.md`.

## `UtcDatetime` — tipo de serialización (no es una tabla ni una columna)

| Aspecto | Detalle |
|---|---|
| Definición | `Annotated[datetime, PlainSerializer(lambda dt: dt.replace(tzinfo=timezone.utc).isoformat() if dt.tzinfo is None else dt.astimezone(timezone.utc).isoformat(), return_type=str)]`, en `app/core/timezone.py` |
| Entrada | El `datetime` naive que SQLAlchemy ya devuelve para cualquier columna `DateTime` de las once entidades — sin cambio de valor, solo de cómo se serializa. |
| Salida | Cadena ISO 8601 con offset explícito `+00:00` (equivalente a `Z`) — cumple `FR-003`: "identifica sin ambigüedad que es UTC". |
| Alcance | Reemplaza la anotación `datetime` en cada campo de instante absoluto de las once entidades (tabla siguiente). No se aplica a `Promotion.starts_at`/`ends_at` (`FR-009`, hora de pared local, fuera de alcance) ni a ningún campo que no represente un instante absoluto. |

## Las once entidades de Key Entities — campo, ubicación, y qué cambia

| Entidad.campo | Columna DB (sin cambio) | Schema de respuesta (cambia a `UtcDatetime`) | Filtro de rango existente |
|---|---|---|---|
| Sale.sold_at | `TIMESTAMP` naive, `server_default=func.now()` | `SaleResponse.sold_at` (`sales/schemas.py:134`) | Sí — `sales/service.py` (Decisión 5) |
| CustomerOrder.created_at | `TIMESTAMP` naive, `func.now()` | `OrderResponse.created_at` (`orders/schemas.py:182`) | No |
| Payment.paid_at | `TIMESTAMP` naive, `func.now()` (`payment.py:58`) | `PaymentResponse.paid_at` — **campo nuevo**, hoy ausente (research.md Decisión 9) | No |
| PaymentAttempt.created_at/resolved_at | `TIMESTAMP` naive | `PaymentAttemptResponse.created_at/resolved_at` (`orders/schemas.py:213-214`) | No |
| CashShift.opened_at/closed_at | `TIMESTAMP` naive, `func.now()` (`cash_shift.py:31`) | `ShiftResponse`/`ShiftSummaryResponse` (`cash/schemas.py:52-53,153-154`) | No |
| CashMovement.occurred_at | `TIMESTAMP` naive, `func.now()` (`cash_movement.py:36`) | `CashMovementResponse.occurred_at` (`cash/schemas.py:77`) | No |
| CashPartialCount.counted_at | `TIMESTAMP` naive, `func.now()` (`cash_partial_count.py:31`) | `PartialCountResponse.counted_at` (`cash/schemas.py:130`) | No |
| InventoryMovement.moved_at | `TIMESTAMP` naive, `func.now()` (`inventory_movement.py:38`) | `MovementResponse.moved_at` (`inventory/schemas.py:64`) | No |
| TableSession.opened_at/closed_at | `TIMESTAMP` naive, `func.now()` (`table_session.py:39`) | `TableSessionResponse` (`table_sessions/schemas.py:36-37`) | No |
| SessionParticipant.joined_at/closed_at | `TIMESTAMP` naive, `func.now()` (`session_participant.py:48`) | `ParticipantResponse` (`table_sessions/schemas.py:25,27`) | No |
| Invoice.issued_at | `TIMESTAMP` naive, `func.now()` (`invoice.py:43`) | `InvoiceResponse.issued_at` (`invoices/schemas.py:32`) | No — importe y demás datos de facturas ya emitidas quedan intactos (Principio VII) |
| Purchase.purchased_at | `TIMESTAMP` naive, `func.now()` (`purchase.py:36`) | `PurchaseResponse.purchased_at` (`inventory/schemas.py:125`) | No |
| AuditLog.at | `TIMESTAMP` naive, `func.now()` (`audit_log.py:25`) | `AuditLogResponse.at` (`audit/router.py:19-27`) | No — este listado no tiene filtro de rango hoy |

Nota (no bloqueante): `SC-002` de spec.md dice "las nueve entidades" pero la lista de Key Entities de
la misma spec enumera once — se usa la lista de Key Entities como fuente de verdad (es la enumeración
explícita, sin ambigüedad de qué cubre); es una discrepancia de redacción menor entre dos secciones de
la misma spec, no una ambigüedad funcional que requiriera clarificación adicional.

`Registros con TimestampMixin` (`Tenant`, `User`, y demás entidades base) también son instante
absoluto por definición (Key Entities de spec.md), pero la investigación no encontró ningún sitio del
frontend que muestre `created_at`/`updated_at` de esas entidades directamente al usuario — se dejan
con su tipo `datetime` actual en los schemas que sí las exponen (no forman parte de las tablas
anteriores porque ningún `FR`/`SC` de spec.md los cita por nombre de campo, solo la nota general de
Key Entities); si una pantalla futura llega a mostrarlos, debe adoptar el mismo `UtcDatetime` por
consistencia, pero no es parte del alcance verificable de esta spec.

## `local_day_bounds_utc(day, tz)` — no es una entidad, es una función pura

| Entrada | Salida |
|---|---|
| `day: date` (p. ej. `date(2026, 8, 24)`), `tz: ZoneInfo` (la del tenant resuelta por `resolve_timezone`) | `(start_utc, end_utc_exclusive)` — ambos `datetime` naive en UTC: la medianoche local de `day` y la medianoche local del día siguiente, ya convertidas. Reemplaza la comparación cruda `Sale.sold_at >= date_from` / `< date_to + timedelta(days=1)` en `sales/service.py` y `reports/service.py` (research.md Decisión 5). |

## Estado nuevo del cliente frontend (no es una entidad de base de datos)

| Campo | Tipo | Rol |
|---|---|---|
| `TenantInfo.timezone` | `string` | Nuevo campo en la interfaz `TenantInfo` (`tenant-info.service.ts:8-16`), poblado desde `GET /tenant` (`TenantInfoResponse.timezone`, nuevo). Fuente de zona horaria para `TenantDatePipe` y `businessToday()`. |
| `TenantDatePipe` | `Pipe` inyectable | Mecanismo único de formato (research.md Decisión 6) — sin estado propio, lee `TenantInfoService.info()` en cada `transform()`. |

## Reglas de validación

- `Tenant.timezone`: debe ser un nombre IANA reconocido por `zoneinfo` (`@validates`, research.md
  Decisión 3) — única regla de validación nueva de esta spec.
- Ninguna regla de `Sale`, `CustomerOrder`, `Payment`, `CashShift`, `CashMovement`,
  `CashPartialCount`, `InventoryMovement`, `TableSession`, `SessionParticipant`, `Invoice`,
  `Purchase` ni `AuditLog` cambia — sus columnas, tipos, defaults y constraints existentes quedan
  intactos (`FR-007`).

## Transiciones de estado

Ninguna de las once entidades gana una transición de estado nueva. La única transición relevante es
de configuración, no de dominio: `Tenant.timezone` pasa de "no existir" (antes de la migración) a
`'America/Bogota'` (tras la migración, para todo tenant existente) a, opcionalmente, un valor distinto
fijado por `set_tenant_timezone.py` — nunca se revierte automáticamente ni cambia como efecto
secundario de ninguna otra operación.
