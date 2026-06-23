# Retroalimentación — Ejercicio 06: El Pipeline de Datos

**Alumno:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 68 / 100

---

## Resumen general

Las cuatro capas están separadas correctamente — cada archivo tiene una
responsabilidad clara y ninguna hace el trabajo de otra. `extract` normaliza
sin rechazar, `transform` valida sin normalizar, `load` inserta sin validar.
`load.py` usa `conn.total_changes` para derivar los conteos de forma correcta
y la cuarentena escribe JSONL con el motivo de error. Los tres gaps concretos:
cinco de las ocho reglas de validación están implementadas (faltan `status`,
rangos de `user_id` y `merchant_id`); las categorías están en minúsculas cuando
el schema del módulo usa capitalizadas; y los tests verifican la consistencia
matemática pero no demuestran idempotencia — el pipeline corre dos veces con
datos aleatorios distintos, no con los mismos datos.

---

## Criterio 1 — Capas separadas y cohesivas `21 / 25`

**Separación correcta en los cuatro archivos:**

`extract.py` normaliza — y solo normaliza:
- Timestamp a ISO 8601 (`datetime.fromisoformat().isoformat()`) ✓
- `country_code` a mayúsculas ✓
- `amount` redondeado a 2 decimales ✓
- Copia el row antes de modificar para no mutar el original ✓
- Nunca rechaza una fila — si la normalización falla, devuelve el valor original

`transform.py` valida — y solo valida. No hay normalización mezclada. Las
categorías y países están definidas como constantes locales y la clasificación
es por secuencia de `elif` — el primer error encontrado determina el motivo.

`load.py` inserta — y solo inserta. No valida reglas de negocio ni normaliza.

`pipeline.py` orquesta — encadena las capas y escribe los resultados.

**Constantes duplicadas entre `data_source.py` y `transform.py`:**
```python
# En ambos archivos:
VALID_CATEGORIES = ["food", "electronics", ...]
VALID_COUNTRIES  = ["MX", "US", "CA", ...]
```
La pregunta del README ("¿debería separar ese fragmento?") tiene respuesta
directa: sí. Un `constants.py` con `VALID_CATEGORIES` y `VALID_COUNTRIES`
elimina la duplicación y garantiza que ambas capas usen la misma lista. Si
mañana se agrega un país, hay un solo lugar donde hacerlo. Con dos copias, una
se puede olvidar.

**Categorías en minúsculas — inconsistencia con el schema del módulo.** Tanto
`data_source.py` como `transform.py` usan `"food"`, `"electronics"`, etc. El
schema del módulo desde E1 usa `"Food"`, `"Electronics"`, etc. (capitalizadas).
El pipeline es internamente consistente (genera minúsculas, valida minúsculas),
pero si se usara con el dataset real del E1 o con la base del E3, todas las
filas fallarían la validación de categoría porque `"Food" not in {"food", ...}`.
Cambiar `VALID_CATEGORIES` a mayúsculas y agregar `.capitalize()` o
`.title()` en `extract` resolvería el problema limpiamente.

**`invalid_uuid` en `transform` pero nunca inyectado en `data_source`.**
`transform` valida `is_valid_uuid()` correctamente, pero `data_source.py`
inyecta cuatro tipos de error: `invalid_amount`, `invalid_category`,
`future_timestamp`, `null_field`. El tipo `invalid_uuid` solo aparece si
`null_field` cae sobre `transaction_id` (probabilidad de 1/8 de los errores
del ~20% de filas malas). En la práctica, el reporte casi nunca mostrará
`invalid_uuid` aunque la validación exista.

---

## Criterio 2 — Validación y cuarentena `18 / 30`

**Cinco de las ocho reglas implementadas:**

| Regla | Estado |
|---|---|
| `amount` fuera de 0.01–5,000.00 | ✓ |
| `category` fuera del set válido | ✓ (minúsculas) |
| `country_code` fuera de los 15 países | ✓ |
| `timestamp` más de 1 hora en el futuro | ✓ |
| `transaction_id` no es UUID4 válido | ✓ (no se inyecta) |
| `status` distinto de completed/failed/pending | ✗ |
| `user_id` fuera del rango 1–50,000 | ✗ |
| `merchant_id` fuera del rango 1–10,000 | ✗ |

`status` en `data_source.py` solo genera `"completed"` y `"failed"` — falta
`"pending"` del schema — y transform nunca valida el campo. Una fila con
`status="cancelled"` o `status=None` (por `null_field`) pasaría sin rechazarse.

Los rangos de `user_id` (1–50,000) y `merchant_id` (1–10,000) tampoco están
validados. `data_source` genera `user_id` en 1–1,000 y `merchant_id` en 1–500
(dentro del rango), así que ningún error natural aparece en el reporte para
estos campos — pero un batch externo con valores fuera de rango pasaría.

**La mecánica de cuarentena es correcta:**
```python
{"row": {...}, "error": "invalid_amount"}
```
Escrito a `quarantine/YYYY-MM-DD.jsonl` en modo append (`"a"`) ✓. Múltiples
ejecuciones en el mismo día acumulan en el mismo archivo. El directorio se crea
si no existe ✓.

---

## Criterio 3 — Idempotencia verificada `10 / 20`

**`INSERT OR IGNORE` en `load.py` ES idempotente** — si la misma
`transaction_id` llega dos veces, la segunda es ignorada. El mecanismo es
correcto.

**Los tests no demuestran idempotencia — demuestran consistencia matemática.**
```python
def test_pipeline_execution():
    metrics = run_pipeline(...)  # datos aleatorios con uuid.uuid4()

def test_pipeline_consistency_again():
    metrics = run_pipeline(...)  # datos distintos (UUIDs diferentes)
```
`generate_batches()` llama `uuid.uuid4()` en cada fila. Dos ejecuciones del
pipeline producen dos conjuntos de UUIDs completamente distintos. Ninguna fila
de la segunda corrida es duplicado de la primera — `rows_duplicates` es 0 en
ambas. Los asserts verifican que `rows_valid == rows_inserted + rows_duplicates`
(que siempre se cumple trivialmente cuando `duplicates=0`).

Para demostrar idempotencia, el test tiene que insertar una fila con un UUID
fijo, insertarla de nuevo, y verificar que la base tiene exactamente 1 registro:
```python
def test_idempotency():
    fixed_row = [{
        "transaction_id": "aaaaaaaa-0000-4000-a000-000000000001",
        ...
    }]
    load(fixed_row, db_path)
    load(fixed_row, db_path)  # segunda inserción
    
    conn = sqlite3.connect(db_path)
    count = conn.execute(
        "SELECT COUNT(*) FROM transactions WHERE transaction_id = ?",
        ("aaaaaaaa-0000-4000-a000-000000000001",)
    ).fetchone()[0]
    assert count == 1  # no duplicado
```

`test_output_directories()` verifica que las carpetas existen después de
ejecutar el pipeline — útil pero mínimo.

---

## Criterio 4 — Reporte de ejecución `19 / 25`

**El JSON tiene todos los campos requeridos:**
```json
{
  "rows_extracted": 500,
  "rows_valid": 400,
  "rows_invalid": 100,
  "rows_inserted": 390,
  "rows_duplicates": 10,
  "errors": {
    "invalid_amount": 35,
    "invalid_category": 31,
    "invalid_timestamp": 29,
    "invalid_country": 4,
    "invalid_uuid": 1
  },
  "total_time_sec": 0.7282
}
```

Nombrado con timestamp `run_YYYYMMDD_HHMMSS.json` ✓. Directorio `results/`
creado si no existe ✓.

**Las invariantes se verifican en los tests pero no en `pipeline.py`.**
El README documenta las dos invariantes claramente. Los tests las verifican con
`assert`. Pero el spec pide que sean `assert` dentro del pipeline mismo — si el
pipeline corre sin tests, las invariantes no se verifican. La corrección es
agregar al final de `run_pipeline()`:
```python
assert metrics["rows_extracted"] == metrics["rows_valid"] + metrics["rows_invalid"]
assert metrics["rows_valid"] == metrics["rows_inserted"] + metrics["rows_duplicates"]
```

**`errors` solo muestra los 4-5 tipos que `data_source` inyecta.** Como
`status`, `user_id` y `merchant_id` no están validados, y `invalid_uuid` rara
vez aparece, el reporte en producción mostraría un perfil incompleto de la
distribución real de errores.

---

## Sobre la pregunta del README

> ¿Debería separar el fragmento duplicado en un archivo aparte?

Sí. La regla general es: si dos módulos necesitan el mismo dato para funcionar
correctamente juntos, ese dato debería vivir en un módulo compartido. Aquí
`data_source.py` usa las categorías para *generar* datos válidos y `transform.py`
las usa para *validar* — si una lista cambia sin que la otra cambie, el
pipeline genera datos que su propio validador rechaza. Un `constants.py` o
`schema.py` con 15 líneas soluciona esto y hace la intención explícita.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 7:

> Agrega un test de idempotencia real: genera una fila con un `transaction_id`
> fijo (no `uuid.uuid4()`), cóprela dos veces al `load`, y verifica con
> `COUNT(*)` que la base tiene exactamente 1 registro. Luego corre el pipeline
> completo con `--batch-size 50 --num-batches 3` dos veces seguidas con los
> mismos datos. ¿Puedes hacer que ambas corridas usen las mismas filas? Si no
> (porque `uuid.uuid4()` no es reproducible), ¿qué cambio mínimo en
> `data_source.py` lo haría posible?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Capas separadas y cohesivas | 25% | 21 / 25 |
| Validación y cuarentena | 30% | 18 / 30 |
| Idempotencia verificada | 20% | 10 / 20 |
| Reporte de ejecución | 25% | 19 / 25 |
| **Total** | **100%** | **68 / 100** |

---

La separación de capas está bien lograda — cada archivo tiene una
responsabilidad clara. Los tres cambios concretos antes de E07: agregar las
tres reglas de validación faltantes (status, rangos de user_id y merchant_id),
mover las constantes a un módulo compartido, y reemplazar los tests de
consistencia matemática por un test de idempotencia con UUID fijo y verificación
de `COUNT(*) == 1`.