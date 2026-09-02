# Contrato: `CashService.discoverOpenShift()` (FR-001 a FR-004)

**Normativo.** Reemplaza `CashService.restoreShift()`
(`pos-heladeria/src/app/modules/cash-register/services/cash.service.ts:50-63`). Mismos dos call
sites (`PosTerminalStore.ensureCheckoutDataLoaded()` y `CashSessionStore.init()`, rama admin), sin
cambio de firma (`(): Promise<void>`, sin argumentos, puebla la señal compartida `shift`).

---

## 1. Algoritmo

```text
discoverOpenShift():
  regId := localStorage.getItem('cash.register')
  si regId no es null:
    intentar shift := getCurrentShift(regId)
    si tuvo éxito: shift.set(shift); RETORNAR
    # si falló (404, esa caja puntual sin turno), seguir al descubrimiento completo

  regs := listRegisters()
  resultados := para cada r en regs, en paralelo: intentar getCurrentShift(r.id), o null si falla
  abiertos := resultados sin los null

  si abiertos.length == 1:
    shift.set(abiertos[0])
    localStorage.setItem('cash.register', abiertos[0].cash_register_id)
  si no:
    shift.set(null)   # 0 turnos: bloquea (FR-003). Más de 1: ambiguo, exige "Operar" (FR-004).
```

## 2. Tabla de casos

| `localStorage['cash.register']` | Turnos abiertos en el tenant | Resultado de `shift` | Motivo |
|---|---|---|---|
| Apunta a una caja con turno abierto | (no importa) | Ese turno | Camino rápido, sin listar cajas — sin cambio de comportamiento respecto de hoy. |
| Ausente, o apunta a una caja sin turno | Exactamente 1 (en cualquier caja) | Ese turno | Descubrimiento — **nuevo**, corrige el defecto reportado (FR-001, FR-002). |
| Ausente, o apunta a una caja sin turno | 0 | `null` | Sin cambio — sigue bloqueado (FR-003). |
| Ausente, o apunta a una caja sin turno | 2 o más | `null` | Sin cambio en el resultado — pero ahora es una decisión explícita de no adivinar, no un efecto colateral de nunca haber consultado (FR-004). |

## 3. Qué NO cambia

- `getCurrentShift(cashRegisterId)` y `listRegisters()` — mismos endpoints, mismas firmas, sin
  tocar `pos-backend`.
- El significado de la clave `localStorage['cash.register']` — sigue siendo "la última caja
  operada/confirmada desde este navegador", ahora también escrita automáticamente cuando el
  descubrimiento resuelve un único turno (para que la próxima verificación tome el camino rápido).
- El guard de `ensureCheckoutDataLoaded()` (`this.cash.shift() ? null : this.cash.discoverOpenShift()`)
  — sigue evitando pedirlo dos veces una vez resuelto (spec 059 FR-003).

## 4. Fuera de alcance

- Ningún endpoint nuevo en `pos-backend`; ver research.md D3 para las alternativas descartadas.
- Ninguna interfaz de desambiguación nueva para el caso de 2+ turnos abiertos — sigue siendo
  "Operar" desde el módulo de Caja (User Story 3, sin cambio).
