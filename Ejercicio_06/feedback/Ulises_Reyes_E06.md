# Retroalimentación — Ejercicio 06: El Pipeline de Datos

**Alumno:** Ulises Reyes
**Fecha de revisión:** Mayo 2026
**Calificación:** 92 / 100

---

## Resumen general

El pipeline más completo del grupo en E06. Seis decisiones lo distinguen: las
invariantes matemáticas verificadas con `assert` dentro de `pipeline.py` — el
único alumno del módulo que las tiene en el código de producción y no solo en
los tests; el parámetro `now` en `transform()` para hacer los tests de
timestamp deterministas; UUID4 validado por regex estructural en lugar de
intentar parsear; `--seed` en el CLI para corridas reproducibles;
`parse_errors` como métrica separada de `rejected` en el reporte; y 33 tests
distribuidos por capa con separación explícita de responsabilidades verificada
en código. Los gaps son dos: `VALID_STATUSES` está definida pero nunca se usa
en `transform()`, y `data_source.py` y `extract.py` no fueron entregados.

---

## Criterio 1 — Capas separadas y cohesivas `23 / 25`

**Nota:** `data_source.py` y `extract.py` no fueron entregados. La separación
se infiere de los tests, las importaciones en `pipeline.py`, y el `decisions.md`.

**`test_extract_no_business_rules` — verificación explícita de la separación:**
```python
def test_extract_no_business_rules(self, standard_batch):
    rows_with_neg = [r for r in standard_batch if r["amount"] < 0]
    result, parse_errors = extract(rows_with_neg)
    neg_in_result = [r for r in result if r["amount"] < 0]
    assert len(neg_in_result) > 0, (
        "Extract no debe rechazar amounts negativos — "
        "eso es responsabilidad de transform"
    )
```
Este test verifica la propiedad más crítica de la arquitectura ETL: extract no
aplica reglas de negocio. No es suficiente que el código esté separado en
archivos — hay que demostrar que la separación es funcional. Ningún otro alumno
del grupo tiene un test con esta intención explícita.

**Funciones validadoras individuales en `transform.py`:**
```python
def _check_amount(amount)   -> str | None: ...
def _check_category(category) -> str | None: ...
def _check_country(country_code) -> str | None: ...
def _check_timestamp(timestamp, now) -> str | None: ...
def _check_transaction_id(tid) -> str | None: ...
```
Cada validador es testeable de forma independiente. El encadenamiento con `or`
en `transform()` es limpio y expresivo:
```python
reason = (
    _check_null_fields(row)
    or _check_transaction_id(...)
    or _check_amount(...)
    or _check_category(...)
    or _check_country(...)
    or _check_timestamp(..., now)
)
```

**`decisions.md` con 7 decisiones documentadas con evidencia.** La Decisión 5
(idempotencia) incluye los resultados medidos con `seed=42`: `inserted=180,
duplicates=0` en la primera corrida; `inserted=0, duplicates=180` en la segunda.
La Decisión sobre clasificación de errores documenta la interacción entre
extract y transform cuando `data_source` inyecta un string vacío como UUID —
`extract` lo convierte a `None`, `transform` lo clasifica como `null_field`
en lugar de `invalid_transaction_id`. Ese nivel de detalle requiere haber
corrido el pipeline y observado el comportamiento real.

Tres puntos menos por los archivos faltantes.

---

## Criterio 2 — Validación y cuarentena `24 / 30`

**UUID4 por regex estructural — la validación más precisa del grupo:**
```python
_UUID4_RE = re.compile(
    r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$',
    re.IGNORECASE,
)
```
El patrón codifica tres restricciones en una sola operación: el formato de 5
grupos, el dígito `4` en la posición correcta (versión 4), y los bits
variantes `[89ab]` en la posición del variant. Es más eficiente que
`uuid.UUID(str(val)).version` porque no construye un objeto Python — solo
evalúa el regex. `test_rejects_invalid_uuid` lo verifica explícitamente.

**Parámetro `now` para tests deterministas:**
```python
def transform(rows, now=None):
    if now is None:
        now = datetime.now(timezone.utc).replace(tzinfo=None)
```
Inyectar el "tiempo actual" como parámetro convierte `_check_timestamp` en
una función pura testeable. El test `test_rejects_future_timestamp` construye
un timestamp que es definitivamente futuro sin depender del reloj del sistema:
```python
future = (datetime.now(timezone.utc) + timedelta(hours=3)).strftime(...)
```
Y `test_accepts_near_future_timestamp` verifica que la tolerancia de 1 hora
está implementada correctamente — ningún otro alumno prueba el caso borde.

**Motivo de rechazo con el valor que causó el error:**
```python
"amount=-50.0 fuera del rango [0.01, 5000.00]"
"category='Gambling' no está en el set válido"
"timestamp '2030-01-01 00:00:00' es futuro (límite: 2024-03-15 15:30:00)"
```
El test `test_rejection_has_exact_reason` lo verifica:
```python
valid, rejected = transform([self._make_valid_row(amount=-99.50)])
assert "-99.5" in rejected[0]["rejection_reason"]
```
Un log de cuarentena con el valor específico que causó el rechazo es
directamente accionable — no hay que buscar la fila para entender qué falló.

**Cuarentena con `rejection_reason` y `rejection_type` como campos separados.**
`rejection_reason` es el texto humano (auditoría). `rejection_type` es la
categoría del reporte (agregación). `classify_rejection()` hace el mapeo
de forma centralizada.

**`VALID_STATUSES` definida pero nunca usada — bug silencioso:**
```python
VALID_STATUSES: frozenset[str] = frozenset({"completed", "failed", "pending"})
```
Igual que en el E05, está definida pero `transform()` no tiene un
`_check_status()`. Una fila con `status="cancelled"` pasa sin rechazarse.
El test `test_all_rejected_have_reason_and_type` no lo detecta porque todos
los datos de prueba tienen `status="completed"`.

**Sin rangos de `user_id` (1–50,000) y `merchant_id` (1–10,000).**

---

## Criterio 3 — Idempotencia verificada `20 / 20`

**`test_load_idempotent` — demostración correcta con tres verificaciones:**
```python
inserted1, duplicates1 = load(valid, db_path=tmp_db)
count_after_1 = conn.execute("SELECT COUNT(*) FROM transactions").fetchone()[0]

inserted2, duplicates2 = load(valid, db_path=tmp_db)
count_after_2 = conn.execute("SELECT COUNT(*) FROM transactions").fetchone()[0]

assert inserted2 == 0
assert duplicates2 == inserted1
assert count_after_1 == count_after_2
```
Tres invariantes verificadas: el conteo de inserciones, el conteo de
duplicados, y el estado final de la base. El tercer assert (`count_after_1 ==
count_after_2`) es el más importante porque verifica el estado de la base,
no solo los valores de retorno de la función.

**`test_pipeline_idempotent` — idempotencia end-to-end:**
Corre el pipeline completo dos veces con `seed=42` y verifica `report2["inserted"] == 0`
más el conteo en DB. Es la demostración más completa del grupo.

**Pre-query de IDs existentes en `load.py` — conteo exacto sin depender de
`rowcount`:**
```python
existing_ids = set(
    conn.execute(
        "SELECT transaction_id FROM transactions WHERE transaction_id IN (...)",
        [r["transaction_id"] for r in rows],
    ).fetchall()
)
new_rows = [r for r in rows if r["transaction_id"] not in existing_ids]
```
`cursor.rowcount` puede ser poco confiable con `INSERT OR IGNORE` en algunas
versiones de SQLite. La pre-query garantiza que los conteos de `inserted` y
`duplicates` son siempre exactos.

**`conn.executemany()` para batch insert — eficiente y atómico:**
Un solo statement SQL con N valores en lugar de N statements individuales.
El `with conn:` maneja `BEGIN`/`COMMIT`/`ROLLBACK` implícitamente.

---

## Criterio 4 — Reporte de ejecución `25 / 25`

**Invariantes verificadas con `assert` en `pipeline.py` — el único en el grupo:**
```python
assert len(extracted) + len(parse_errors) == batch_size
assert len(valid) + len(rejected) == len(extracted)
assert inserted + duplicates == len(valid)
```
Si alguna invariante se rompe, el pipeline lanza una excepción con el mensaje
específico antes de guardar el reporte. Esto garantiza que nunca se guarda un
reporte con números que no cuadran. La excepción temprana hace el debugging
inmediato.

**`invariants` en el reporte JSON como confirmación explícita:**
```json
"invariants": {
    "extracted_eq_valid_plus_rejected": true,
    "inserted_plus_duplicates_eq_valid": true
}
```
El evaluador no tiene que calcular las sumas manualmente — el reporte las
confirma. `test_pipeline_invariants` los verifica en código.

**`parse_errors` como métrica separada:**
`extract.py` puede rechazar filas por errores de formato (timestamp que no
puede parsearse, amount que no es un número). `transform.py` rechaza por
reglas de negocio. El reporte separa ambos. Esta distinción es técnicamente
correcta — un error de formato en extract y un amount fuera de rango en
transform son problemas de naturaleza diferente.

**`params` section en el reporte:**
```json
"params": {
    "batch_size": 500,
    "error_rate": 0.1,
    "seed": 42,
    "db_path": "..."
}
```
Una corrida es reproducible exactamente si se tienen sus parámetros. El reporte
los documenta para que cualquier corrida pueda reproducirse.

**`quarantine_file` en el reporte** — el path al archivo de cuarentena del día
está incluido, así no hay que buscarlo.

**`test_pipeline_report_saved`** verifica todos los campos del reporte,
incluyendo `invariants` y `by_error`.

---

## Sobre el uso de herramientas de IA

Los `assert` en `pipeline.py`, el parámetro `now` en `transform()` para tests
deterministas, la regex de UUID4 con los bits variantes `[89ab]`, el
`test_extract_no_business_rules`, y el `decisions.md` con los resultados
medidos de `seed=42` son decisiones que requieren comprensión profunda del
sistema. La nota sobre la interacción entre `extract.py` y `transform.py` en
la clasificación de errores (`"" → None → null_field` en lugar de
`invalid_transaction_id`) es el tipo de observación que solo se ve corriendo
el pipeline y mirando los outputs reales. El `VALID_STATUSES` sin usar es el
único gap que no tiene explicación obvia — es el mismo bug que en el E05
donde las constantes se definen pero la regla no se conecta.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 7:

> `VALID_STATUSES` está definida en `transform.py` pero `transform()` no tiene
> `_check_status()`. Agrega el validador, conéctalo en la cadena `or`, y corre
> el test `test_rejects_null_field`. ¿Por qué ese test sigue pasando sin cambios?
> Luego agrega un test nuevo que envíe `status="cancelled"` y verifica que es
> rechazado. Finalmente: el `_ERROR_KEY_MAP` en `classify_rejection()` no tiene
> una entrada para `status` — ¿cómo lo clasificaría? ¿Qué cambio adicional
> necesitas?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Capas separadas y cohesivas | 25% | 23 / 25 |
| Validación y cuarentena | 30% | 24 / 30 |
| Idempotencia verificada | 20% | 20 / 20 |
| Reporte de ejecución | 25% | 25 / 25 |
| **Total** | **100%** | **92 / 100** |

---

Los `assert` en `pipeline.py`, el parámetro `now` para tests deterministas y
los 33 tests con separación explícita de responsabilidades son las decisiones
que hacen esta entrega la más completa del grupo en E06. El único gap concreto
es `VALID_STATUSES` sin conectar — la pregunta de seguimiento lleva directamente
a la corrección y a entender por qué el `classify_rejection()` también necesita
actualizarse.