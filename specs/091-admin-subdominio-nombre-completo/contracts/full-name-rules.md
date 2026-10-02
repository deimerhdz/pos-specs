# Contrato: reglas del nombre completo + vectores de prueba compartidos

**Spec**: FR-013…FR-017, FR-025 | **Decisión**: [research.md](../research.md) D5

Esta es la **única definición** de la regla. Se implementa dos veces —`app/core/person_name.py` (Python) y `src/app/shared/validators/full-name.validator.ts` (TypeScript)— y **ambas suites de prueba recorren la tabla de vectores de abajo** (mismas entradas, mismos resultados). Si una implementación diverge, falla su prueba.

## Algoritmo

Entrada: texto o ausente.

1. **Preparar**: si es ausente → tratar como `""`. Aplicar `trim` (espacios Unicode en los extremos) y luego normalización **NFC**. Este es el valor que se valida **y** el que se guarda.
2. **Obligatorio**: si el resultado es `""` → error `required`.
3. **Longitud**: contar **puntos de código** (no unidades UTF-16). Si es < 2 o > 100 → error `length`.
4. **Formato**: debe cumplir todo esto, si no → error `format`:
   - cada carácter es: **letra latina**, espacio U+0020, apóstrofe `'` (U+0027) o `’` (U+2019), o guion `-` (U+002D);
   - contiene **al menos dos letras latinas**.
5. Si pasa todo → válido; el valor es el del paso 1 (con espacios internos conservados, sin cambiar mayúsculas — FR-025).

Se devuelve **solo el primer error** en el orden `required → length → format` (FR-015).

**Letra latina** = rangos explícitos (no `\p{L}`): `A-Z`, `a-z`, `U+00C0–U+00D6`, `U+00D8–U+00F6`, `U+00F8–U+00FF`, `U+0100–U+024F` (Latinas Extendidas A/B), `U+1E00–U+1EFF` (Latinas Extendidas Adicionales). Quedan fuera `×` (U+00D7), `÷` (U+00F7), dígitos, puntuación, emoji, controles (tab, salto de línea), `<`, `>`, `&`, comillas dobles, y todo otro alfabeto.

## Mensajes (español de Colombia)

| Error | Texto |
|---|---|
| `required` | `El nombre es obligatorio` |
| `length` | `El nombre debe tener entre 2 y 100 caracteres` |
| `format` | `El nombre solo puede contener letras, espacios, apóstrofes y guiones, y al menos dos letras` |

En el servidor la excepción es `ValueError(<texto>)`; Pydantic la expone como `Value error, <texto>` en `detail[0].msg` (422). En pantalla se muestra el texto sin prefijo.

## Vectores de prueba (entrada → resultado)

`✓ valor` = válido, devuelve `valor` guardado. Los espacios visibles se muestran con `·` donde importan.

| # | Entrada | Resultado |
|---|---|---|
| 1 | `María Pérez` | ✓ `María Pérez` *(Escenario 1)* |
| 2 | `José Ñañez O'Brien-Díaz` | ✓ igual *(Escenario 5)* |
| 3 | `O’Brien` (apóstrofe tipográfico) | ✓ igual |
| 4 | `François`, `Zoë`, `Müller`, `Søren`, `Łukasz`, `Đorđe`, `Nguyễn` | ✓ igual (cada una) |
| 5 | `Al` (2 caracteres) | ✓ `Al` |
| 6 | `a`×100 | ✓ igual |
| 7 | `·Ana·` | ✓ `Ana` *(recorte, Historia 4.6)* |
| 8 | `María···Pérez` (3 espacios internos) | ✓ igual, sin tocar el interior |
| 9 | `Mari` + U+0301 + `a` (tilde combinada, NFD) | ✓ `María` (NFC) |
| 10 | *(ausente / `null`)* | ✗ `required` *(Escenario 7)* |
| 11 | `` (vacío) | ✗ `required` *(Escenario 2)* |
| 12 | `·····` (solo espacios) | ✗ `required` *(Escenario 3)* |
| 13 | `A` (1 carácter) | ✗ `length` *(Escenario 4)* |
| 14 | `-` (1 carácter) | ✗ `length` |
| 15 | `a`×101 | ✗ `length` *(Escenario 4)* |
| 16 | `1`×101 (dígitos, también largo) | ✗ `length` (el largo gana, FR-015) |
| 17 | `<script>alert(1)</script>` | ✗ `format` *(Escenario 6)* |
| 18 | `Ana3`, `Ana@`, `Ana.`, `Ana,` | ✗ `format` |
| 19 | `Ana 😀` | ✗ `format` |
| 20 | `Иван`, `李雷`, `محمد` | ✗ `format` |
| 21 | `---` (solo guiones) | ✗ `format` (sin dos letras) |
| 22 | `''` (solo apóstrofes) | ✗ `format` |
| 23 | `A-` (una letra + guion) | ✗ `format` (solo una letra) |
| 24 | `Ana⇥Pérez` (tab) / `Ana⏎Pérez` (salto de línea) | ✗ `format` |
| 25 | `A×B` | ✗ `format` |
| 26 | `😀`×60 (60 puntos de código, 120 unidades UTF-16) | ✗ `format` (no `length`: se cuentan puntos de código) |

## Uso por superficie

| Superficie | Obligatoriedad | Fuente |
|---|---|---|
| `POST /invitations` (servidor) | siempre | `person_name.normalize_full_name` |
| Formulario "Invitar usuario" | siempre | `fullNameValidator` |
| Formulario "Nuevo usuario" del Super Admin | **crear**: siempre. **Editar**: vacío permitido; si se escribe, debe ser válido (Historia 6.4) | `fullNameValidator` (+ opción `optional`) |
