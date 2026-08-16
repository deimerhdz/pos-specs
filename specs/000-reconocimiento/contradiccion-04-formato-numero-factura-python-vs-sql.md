# Contradicción 04 — El número completo de factura se formatea dos veces: una vez en Python (para mostrarlo) y otra en SQL (para buscarlo), y no truncan igual

**Fecha**: 2026-08-15
**Alcance**: `pos-backend`, `app/api/v1/invoices/schemas.py` y `app/api/v1/sales/service.py`.
**Método**: lectura directa de ambas implementaciones y verificación del comportamiento
documentado de `lpad` en PostgreSQL. Corresponde y amplía `RN-FACT-07 [DUDOSA]` de
`reglas-de-negocio.md:1612` ("No se verificó si `Invoice.full_number` coincide exactamente
con la fórmula SQL reconstruida en `list_sales_query`") — aquí se completa esa
verificación.

---

## 1. Las implementaciones implicadas

**(A) `Invoice.full_number` — Python, para mostrar/imprimir**
`app/api/v1/invoices/schemas.py:40-43`:

```python
@computed_field
@property
def full_number(self) -> str:
    return f"{self.prefix}{self.number:06d}"
```

Es un `computed_field` de Pydantic sobre `InvoiceResponse`: se calcula cada vez que la API
serializa una factura — es decir, es el número que efectivamente ve el cajero, el comensal
en su ticket y cualquier pantalla que muestre el detalle de una factura.

**(B) La misma fórmula, reconstruida en SQL para buscar** —
`app/api/v1/sales/service.py:142-172` (`list_sales_query`), líneas 165-171:

```python
if invoice_reference:
    # No hay columna "referencia": se reconstruye prefix + número (6 dígitos)
    # tal como se imprime en el ticket (ver Invoice.full_number).
    full_number = func.concat(Invoice.prefix, func.lpad(cast(Invoice.number, String), 6, "0"))
    stmt = stmt.join(Invoice, Invoice.sale_id == Sale.id).where(
        full_number.ilike(f"%{invoice_reference.strip()}%")
    )
```

El propio comentario del código (línea 166-167) declara la intención explícita de
reproducir `Invoice.full_number` en SQL, para el buscador de `GET /sales?invoice_reference=`
— la función de "buscar una venta por el número de factura impreso en el ticket".

## 2. ¿Usan la misma convención o algoritmo?

La intención es idéntica y así lo dice el comentario, pero el mecanismo de relleno de
ceros no es equivalente en el caso límite de un número que ya excede el ancho fijo de 6
dígitos:

- **Python, `f"{n:06d}"`**: el especificador `06d` fija un **ancho mínimo**. Si `n` tiene
  más de 6 dígitos, Python no trunca nada — simplemente imprime el número completo, sin
  padding adicional (`f"{1234567:06d}"` → `"1234567"`, 7 caracteres).
- **PostgreSQL, `lpad(string, length, fill)`**: según la documentación de PostgreSQL, si el
  `string` de entrada ya es más largo que `length`, `lpad` lo **trunca** para que el
  resultado mida exactamente `length` caracteres, conservando los caracteres de la
  izquierda y descartando los de la derecha (`lpad('1234567', 6, '0')` → `"123456"`, se
  pierde el último dígito).

Por debajo de 1.000.000 (es decir, mientras `Invoice.number` tenga 6 dígitos o menos)
ambas fórmulas producen exactamente el mismo texto, porque ninguna de las dos rellena ni
trunca — el número ya mide 6 caracteres o menos y el padding de ceros a la izquierda
coincide byte a byte.

## 3. Ejemplo concreto con resultado distinto

Tenant con `prefix="FAC-"` y una secuencia de facturación que, tras varios años de
operación sin reiniciar el consecutivo, llega a `number=1234567` (7 dígitos).

- **Lo que se imprime en el ticket y se muestra en el detalle de la factura** (fuente A,
  `Invoice.full_number`): `"FAC-1234567"`.
- **Lo que la gestoría o el cajero escriben en el buscador de ventas** para encontrar esa
  factura exacta, copiando el número visible en el ticket: `invoice_reference=1234567` (o
  `FAC-1234567`).
- **La comparación SQL que ejecuta la búsqueda** (fuente B):
  `concat('FAC-', lpad('1234567', 6, '0'))` = `concat('FAC-', '123456')` = `"FAC-123456"`
  (se truncó el último dígito, el `7`).
- **Resultado**: `"FAC-123456"` (lo que la consulta compara) no contiene la subcadena
  `"1234567"` ni `"FAC-1234567"` — el `ILIKE` no encuentra coincidencia. La factura existe,
  el número que la gestoría tiene en la mano es correcto, y aun así la búsqueda por número
  de factura devuelve cero resultados para esa venta.

## 4. Cuándo se manifiesta y cuándo coinciden

Coinciden exactamente para cualquier `Invoice.number` de 6 dígitos o menos — es decir,
para el primer millón de facturas de cada prefijo. Divergen únicamente a partir del
consecutivo 1.000.000 en adelante, para ese prefijo. Esto explica con precisión por qué
nadie lo ha detectado: es matemáticamente imposible que se manifieste antes de que un
tenant emita un millón de facturas bajo el mismo prefijo, un volumen que —hasta donde este
documento puede verificar por el propio contexto del proyecto (un negocio de heladería
individual, sin indicios de operar a esa escala)— está muy lejos del uso actual. El defecto
es real, está en el código hoy, y es inofensivo únicamente porque el volumen de facturación
todavía no lo activa; no depende de ninguna otra condición del sistema, config ni de una
sucesión de eventos concurrentes, a diferencia de la mayoría de los hallazgos de
`registro-riesgos.md`.

## 5. Historia probable

El comentario en `sales/service.py:166-167` ("No hay columna 'referencia': se reconstruye
prefix + número... tal como se imprime en el ticket") indica que quien escribió la
búsqueda por número de factura fue consciente de que estaba replicando una fórmula que ya
existía en otro sitio (`Invoice.full_number`), y lo hizo a propósito porque el modelo
`Invoice` no guarda el número completo como columna — solo `prefix` y `number` por
separado, y el número completo se compone al vuelo. Es una duplicación deliberada de una
fórmula de formato entre dos lenguajes distintos (una expresión de formato de Python en la
capa de presentación, una expresión SQL en la capa de consulta), motivada por la necesidad
de buscar sobre un valor que no existe como columna materializada. El autor incluso citó la
fuente de verdad en el comentario ("ver `Invoice.full_number`"), lo que demuestra
intención de mantenerlas sincronizadas — pero `lpad` y el formato `%06d` de Python no
comparten semántica de truncamiento, una diferencia de comportamiento entre dos lenguajes
que es fácil de pasar por alto porque en el 99.9999% de los casos (cualquier número por
debajo de un millón) el resultado es indistinguible.

---

**Pregunta abierta al negocio**: ¿existe algún plan de reiniciar el consecutivo de
facturación periódicamente (por ejemplo, cada año o cada resolución de la DIAN), o se
espera que crezca indefinidamente bajo el mismo prefijo? La respuesta acota si este
defecto es puramente teórico o si conviene vigilarlo a medida que la numeración avanza.
