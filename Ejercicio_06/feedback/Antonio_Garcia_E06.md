# Retroalimentación — Ejercicio 06: El Pipeline de Datos

**Alumno:** Antonio Garcia
**Fecha de revisión:** Mayo 2026
**Calificación:** 96 / 100

---

## Resumen general

La entrega más completa del grupo en E06. Cuatro decisiones la distinguen: es
el único alumno que implementa las 10 reglas de validación del spec (8 base +
`status` + rangos de `user_id`/`merchant_id`); `data_source.py` genera UUIDs
determinísticos con el RNG del seed — no con `os.urandom` — lo que garantiza
idempotencia real end-to-end; las constantes de dominio (`CATEGORIES`, `COUNTRIES`,
`STATUSES`) viven una sola vez en `data_source.py` y `transform.py` las importa
directamente; y el argumento `--batch-size` valida el rango 100-1000 del spec,
algo que ningún otro alumno hizo. Los dos gaps concretos: las tres reglas
adicionales (`invalid_status`, `invalid_user_id`, `invalid_merchant_id`) no
tienen tests de capa y el pipeline no verifica las invariantes con `assert`.

---

## Criterio 1 — Capas separadas y cohesivas `24 / 25`

**Responsabilidades limpias en los cinco archivos:**

`extract.py` normaliza y nunca rechaza — `test_extract_passes_through_unparseable_timestamp`
lo verifica explícitamente:
```python
rec = extract_record(_valid_raw(timestamp="not-a-date"))
assert rec["timestamp"] == "not-a-date"
```
Un timestamp no parseable sale de `extract.py` intacto para que `transform.py`
lo rechace como `invalid_timestamp`. Esa es la separación correcta — el mismo
principio que `test_extract_no_business_rules` de Ulises, pero para otro tipo
de dato.

`transform.py` valida y nunca normaliza. Importa las constantes de dominio de
`data_source.py` en lugar de duplicarlas:
```python
from data_source import CATEGORIES, COUNTRIES, STATUSES
CATEGORY_SET = set(CATEGORIES)
COUNTRY_SET = set(COUNTRIES)
STATUS_SET = set(STATUSES)
```
Solución al gap de duplicación que Yaanit y Angel tenían. Un solo punto de
verdad para los dominios de validación.

`load.py` inserta y nada más. Inicializa el schema desde `schema.sql` en
`init_db()` — el único alumno del grupo con el DDL separado en un archivo SQL,
lo que hace el schema legible e independiente del código Python.

**`_gen_uuid4(rng)` en `data_source.py` — idempotencia end-to-end real:**
```python
def _gen_uuid4(rng: random.Random) -> str:
    b = bytearray(rng.randbytes(16))
    b[6] = (b[6] & 0x0f) | 0x40  # version 4
    b[8] = (b[8] & 0x3f) | 0x80  # variante RFC 4122
    return str(uuid.UUID(bytes=bytes(b)))
```
`uuid.uuid4()` llama a `os.urandom()` que es no-determinístico. Esta función
usa el generador seeded — mismo seed → mismas 16 bytes → mismo UUID. El
`test_simulate_batch_is_deterministic` verifica que `a == b` para `seed=42`
(igualdad exacta, incluyendo los UUIDs). Esto significa que dos corridas del
pipeline con el mismo seed producen los mismos `transaction_id` y la segunda
corrida correctamente reporta `inserted=0` porque todos los IDs ya están en
la base.

**`--batch-size` validado en el rango del spec:**
```python
if not 100 <= size <= 1000:
    raise ValueError("size debe estar entre 100 y 1000 (limite del PDF).")
```
`test_simulate_batch_size_bounds` verifica que `size=50` y `size=1500` levantan
`ValueError`. Este chequeo es el equivalente del `--batch-size` validation del
spec que el enunciado pedía explícitamente.

Un punto menos: `transform.py` importa de `data_source.py`. En producción, las
constantes del dominio vivirían en un `schema.py` independiente para que cada
capa sea deployable sin depender de otra. Para el ejercicio es pragmático, pero
crea un acoplamiento de dirección inversa (transform → data_source) que no es
natural en el flujo ETL.

---

## Criterio 2 — Validación y cuarentena `29 / 30`

**10 reglas implementadas — las más completas del grupo:**

```python
def _validate(record, now):
    # 1) null_field — campos requeridos
    # 2) invalid_uuid — UUID4 válido (version check)
    # 3) invalid_timestamp — timestamp parseable
    # 4) future_timestamp — no más de 1h en el futuro
    # 5) invalid_amount_type — amount numérico
    # 6) amount_out_of_range — [0.01, 5000.00]
    # 7) invalid_category — whitelist 10 valores
    # 8) invalid_country — whitelist 15 países
    # 9) invalid_status — whitelist 3 valores ← único en el grupo
    # 10) invalid_user_id — rango 1-50,000 ← único en el grupo
    # 11) invalid_merchant_id — rango 1-10,000 ← único en el grupo
```

Las reglas 9-11 son las que faltaban en todos los demás alumnos.

**Orden de validación correcto con justificación documentada:**
Los campos requeridos se checan primero para que `amount=None` retorne
`null_field` y no `amount_out_of_range`. El orden importa y está documentado
en el código.

**`invalid_timestamp` como tipo separado de `future_timestamp`:**
Si `fromisoformat` falla → `invalid_timestamp`. Si parsea pero está en el
futuro → `future_timestamp`. Son dos motivos de rechazo distintos y el reporte
los distingue. Ningún otro alumno separó estos dos casos.

**`invalid_amount_type` como tipo separado de `amount_out_of_range`:**
Mismo patrón: `amount="abc"` → `invalid_amount_type`. `amount=-5.0` →
`amount_out_of_range`. El test `test_pipeline_rejects_all_error_types_in_quarantine`
ejecuta con `error_rate=1.0` y verifica que todos los tipos del spec aparecen
en el reporte.

**Quarantine con `rejected_at`, `reason`, `record`:**
```python
quarantine_fh.write(json.dumps({
    "rejected_at": now.isoformat(timespec="seconds"),
    "reason": reason,
    "record": record,
}) + "\n")
```
El test lo verifica estructuralmente:
```python
assert line["reason"] == "invalid_category"
assert line["record"]["category"] == "Groceries"
```

**Un punto menos:** Las reglas 9-11 (`invalid_status`, `invalid_user_id`,
`invalid_merchant_id`) están implementadas correctamente pero no tienen tests
de capa en `TestTransform`. `data_source.py` no inyecta esos tipos de error
(el `ERROR_TYPES` lista 7 tipos y no incluye status/rangos), así que el
pipeline end-to-end tampoco los ejerce. Las reglas existen y son correctas —
solo no tienen cobertura de test.

---

## Criterio 3 — Idempotencia verificada `20 / 20`

**`test_pipeline_is_idempotent` con cuatro verificaciones:**
```python
first  = run_pipeline(**kwargs)
count1 = _read_count(db)
second = run_pipeline(**kwargs)
count2 = _read_count(db)

assert count1 == count2
assert second["loaded"]["inserted"] == 0
assert second["loaded"]["duplicates_skipped"] == first["loaded"]["inserted"]
assert first["rejected"] == second["rejected"]   ← también verifica que los rechazos son iguales
```
La cuarta verificación (`first["rejected"] == second["rejected"]`) es única —
confirma que ambas corridas rechazaron exactamente los mismos registros por los
mismos motivos, lo que solo es posible gracias al seed determinístico.

**`test_load_skips_duplicates_by_transaction_id`:**
```python
load([txn], db_path)
stats = load([txn], db_path)
assert stats == {"received": 1, "inserted": 0, "duplicates_skipped": 1}
assert _read_count(db_path) == 1
```
Verifica el estado final de la base (COUNT=1), no solo los conteos de retorno.

**`test_load_is_atomic_on_error`:**
Elimina `amount` de un registro y verifica que `COUNT(*) == 0` después de que
el insert falla. `with con:` en `load.py` garantiza el rollback — el test
confirma que funciona.

---

## Criterio 4 — Reporte de ejecución `23 / 25`

**Estructura más rica del grupo:**
```json
{
  "run_id": "20260529_165703",
  "started_at": "2026-05-29T16:57:03",
  "total_seconds": 0.017,
  "source": {"type": "simulated", "batch_size": 500, "error_rate": 0.15, "seed": 42},
  "extracted": 500,
  "valid": 425,
  "rejected": {"total": 75, "by_reason": {...}},
  "loaded": {"received": 425, "inserted": 425, "duplicates_skipped": 0},
  "db": "...",
  "quarantine_file": "..."
}
```

`source` section documenta los parámetros exactos de la corrida — reproducible
exactamente si se conocen. `quarantine_file` incluido. `loaded` como objeto con
los tres contadores.

`test_pipeline_run_report_structure` verifica todos los campos del reporte y
las dos invariantes matemáticas.

**Las invariantes están en los tests pero no en `pipeline.py`:**
El README documenta que las sumas cuadran, y los tests las verifican con
`assert`. Pero `pipeline.py` no tiene `assert` antes de guardar el reporte.
Si una corrida produjera números incorrectos (bug en alguna capa), el reporte
se guardaría con datos erróneos sin levantar excepción. Dos líneas en `run()`
lo resuelven:
```python
assert report["extracted"] == report["valid"] + report["rejected"]["total"]
assert report["loaded"]["inserted"] + report["loaded"]["duplicates_skipped"] == report["loaded"]["received"]
```

**`--source-file` para leer desde JSONL externo** — permite usar el pipeline
con datos reales sin modificar el código. El README documenta el flujo completo
con `data_source.py --output` + `pipeline.py --source-file`. Ningún otro alumno
implementó esto.

---

## Sobre el uso de herramientas de IA

El `_gen_uuid4(rng)` con manipulación explícita de bytes de UUID para hacerlo
determinístico, la separación de `invalid_timestamp` y `future_timestamp` como
tipos distintos, las tres reglas adicionales (status, user_id, merchant_id) que
el resto del grupo dejó sin implementar, y el `test_pipeline_rejects_all_error_types_in_quarantine`
con `error_rate=1.0` son decisiones que requieren comprensión del dominio. El
uso de `from data_source import CATEGORIES` para evitar duplicación sin
documentar explícitamente el trade-off (coupling) es el único punto donde el
razonamiento no está completamente articulado.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 7:

> Las reglas 9-11 (`invalid_status`, `invalid_user_id`, `invalid_merchant_id`)
> están implementadas pero no tienen tests de capa en `transform`. Agrega un
> test para cada una siguiendo el patrón de `test_transform_rejects_invalid_category`.
> Luego: `data_source.py` no inyecta esos errores en `ERROR_TYPES` — ¿qué
> pasaría en `test_pipeline_rejects_all_error_types_in_quarantine` si
> añadieras los tres tipos al generador? ¿Pasaría el test sin cambios?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Capas separadas y cohesivas | 25% | 24 / 25 |
| Validación y cuarentena | 30% | 29 / 30 |
| Idempotencia verificada | 20% | 20 / 20 |
| Reporte de ejecución | 25% | 23 / 25 |
| **Total** | **100%** | **96 / 100** |

---

La combinación de determinismo real (UUID desde seed), 10 reglas de validación
completas y separación de `invalid_timestamp`/`invalid_amount_type` como tipos
distintos hacen esta la entrega más completa del grupo. Los dos cambios para el
E07: agregar tests para las tres reglas adicionales, y `assert` en `pipeline.py`
para que los invariantes se verifiquen en producción.