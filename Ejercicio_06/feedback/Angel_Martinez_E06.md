# Retroalimentación — Ejercicio 06: El Pipeline de Datos

**Alumno:** Angel Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 88 / 100

---

## Resumen general

El pipeline más sólido del grupo en E06. Cuatro decisiones lo distinguen: `PRAGMA
journal_mode=WAL` en `load.py` (el único alumno del módulo que lo implementa en
el pipeline), `cursor.rowcount` para contar insertados y duplicados por fila en
lugar de `total_changes`, soporte multi-error en `transform.py` (una transacción
puede reportar más de un motivo de rechazo simultáneamente), y un test de
atomicidad transaccional con `force_error_at` — ningún otro alumno probó que
el rollback funciona. Los gaps son dos: `STATUSES` está definida en
`transform.py` pero nunca se usa para validar, y los archivos `data_source.py`
y `extract.py` no fueron entregados (se infieren de los tests y el pipeline).

---

## Criterio 1 — Capas separadas y cohesivas `22 / 25`

**Nota:** `data_source.py` y `extract.py` no fueron entregados. La separación
se infiere de los imports en `pipeline.py`, los tests que los ejercitan, y el
README. El análisis de este criterio es parcial.

**Lo que se puede verificar:**

`transform.py` valida y no normaliza. `load.py` inserta y no valida. `pipeline.py`
orquesta sin mezclar lógica de capas. `load.py` incluye `init_database()` —
crea la tabla e índices si no existen. Esto es ligeramente mixto (carga +
inicialización), pero es una conveniencia defensiva razonable: garantiza que la
base siempre está lista antes de insertar, sin requerir un paso de setup manual.

**`PRAGMA journal_mode=WAL` en `load.py` — el único en el grupo:**
```python
conn.execute("PRAGMA journal_mode=WAL;")
conn.execute("PRAGMA synchronous=NORMAL;")
```
Implementado al nivel correcto: la conexión de la capa de carga, no en un
settings.py de Django ni en un script separado. Con WAL, los lectores concurrentes
no bloquean escritores y vice versa — relevante para un pipeline que podría
correr en paralelo con consultas analíticas. `synchronous=NORMAL` reduce los
fsync sin sacrificar la durabilidad por WAL.

**`init_database()` crea los índices del E3:**
```python
cursor.execute("CREATE INDEX IF NOT EXISTS idx_user_timestamp ON transactions (user_id, timestamp DESC);")
cursor.execute("CREATE INDEX IF NOT EXISTS idx_country_user ON transactions (country_code, user_id);")
```
Nombres correctos, dirección `DESC` en `timestamp` ✓. El pipeline es
auto-contenido — funciona sobre una base nueva sin requerir el E3 previo.

Tres puntos menos por los archivos faltantes.

---

## Criterio 2 — Validación y cuarentena `24 / 30`

**Multi-error por transacción — decisión arquitectónica superior.**
`validate_transaction()` retorna una lista de motivos, no solo uno:
```python
def validate_transaction(tx: dict) -> list[str]:
    reasons = []
    ...
    if amount < 0.01 or amount > 5000.00:
        reasons.append("amount_out_of_range")
    if tx["category"] not in CATEGORIES:
        reasons.append("invalid_category")
    ...
    return reasons
```
Una transacción con amount inválido Y categoría inválida acumula dos errores. El
resto del grupo usa `elif` y solo captura el primer error. `rejection_reasons` en
el JSON de cuarentena es una lista, no un string — la estructura es más honesta
sobre la realidad de los datos.

**UUID validado específicamente como versión 4:**
```python
val = uuid.UUID(str(tx_id))
if val.version != 4:
    reasons.append("invalid_uuid")
```
Un UUID v1 o v3 es un UUID válido pero no cumple el schema del módulo. El resto
del grupo solo verifica que el string sea parseable como UUID, no que sea v4.

**Timestamp timezone-aware con manejo del sufijo `Z`:**
```python
if ts_str.endswith("Z"):
    clean_ts = ts_str[:-1] + "+00:00"
```
`datetime.fromisoformat()` en Python < 3.11 no acepta la notación `Z`. Este
manejo explícito evita que registros con timestamp en formato `2026-05-01T12:00:00Z`
pasen como error de timestamp cuando en realidad son válidos.

**Quarantine con timestamp de cuarentena y lista de motivos:**
```json
{
  "transaction": {...},
  "rejection_reasons": ["amount_out_of_range", "invalid_category"],
  "quarantined_at": "2026-05-28T21:45:30.456789+00:00"
}
```
Más informativo que `{"row": {...}, "error": "tipo"}` — permite trazar exactamente
cuándo fue cuarentenado cada registro.

**`STATUSES` definida pero nunca usada — bug silencioso:**
```python
STATUSES = {"completed", "failed", "pending"}
```
La constante existe, pero `validate_transaction()` no tiene ningún check de
`status`. Una transacción con `status="cancelled"` o `status=None` (si `None`
no está en los required_fields check) pasa sin rechazarse. El test
`test_validation_and_quarantine` no lo detecta porque todas las transacciones
de prueba tienen `status="completed"`.

**Sin validación de rangos `user_id` (1–50,000) y `merchant_id` (1–10,000).**

---

## Criterio 3 — Idempotencia verificada `20 / 20`

**`test_idempotency` — la demostración más correcta del grupo:**
```python
tx_lote = [
    {"transaction_id": str(uuid.uuid4()), ...},  # UUID fijo para toda la prueba
    {"transaction_id": str(uuid.uuid4()), ...}
]

inserted_1, duplicates_1 = load_to_sqlite(tx_lote, temp_dirs["db_path"])
assert inserted_1 == 2
assert duplicates_1 == 0

inserted_2, duplicates_2 = load_to_sqlite(tx_lote, temp_dirs["db_path"])
assert inserted_2 == 0
assert duplicates_2 == 2
```
UUIDs generados una vez y reutilizados — no usa `uuid.uuid4()` en cada llamada.
Verifica el estado final de la base (`inserted=0` en el segundo run), no solo
que el segundo insert "no rompe nada".

**`cursor.rowcount` para conteo preciso por fila:**
```python
if cursor.rowcount > 0:
    inserted_count += 1
else:
    duplicate_count += 1
```
`rowcount` es 0 cuando `INSERT OR IGNORE` ignora una fila por conflicto. Más
preciso que `total_changes` (que puede ser afectado por triggers u otras
operaciones).

**`test_transactional_atomicity` — único en el grupo:**
```python
def load_to_sqlite(valid_transactions, db_path, force_error_at=None):
    for i, tx in enumerate(valid_transactions):
        if force_error_at is not None and i == force_error_at:
            raise RuntimeError("Simulated database failure for transactional testing.")
```
El test inyecta un fallo en mitad del batch y verifica que `COUNT(*) == 0`
después del rollback. Ningún otro alumno del grupo probó que el `conn.rollback()`
realmente funciona — solo declararon que lo implementaron. Angel lo prueba con
evidencia.

---

## Criterio 4 — Reporte de ejecución `22 / 25`

**Reporte más completo del grupo:**
```json
{
    "run_id": "20260528_214530",
    "timestamp": "2026-05-28T21:45:30.123456+00:00",
    "filas_extraidas": 500,
    "filas_validas": 450,
    "filas_rechazadas": 50,
    "filas_rechazadas_por_tipo_de_error": {...},
    "filas_insertadas": 420,
    "filas_duplicadas": 30,
    "tiempo_total": 0.045612
}
```
`run_id` y `timestamp` UTC como campos adicionales — trazabilidad por ejecución.
`filas_rechazadas_por_tipo_de_error` incluye el tipo `missing_fields` además
de los cinco básicos.

**`test_pipeline_orchestration` verifica las invariantes en los tests:**
```python
assert report["filas_validas"] + report["filas_rechazadas"] == 50
assert report["filas_insertadas"] + report["filas_duplicadas"] == report["filas_validas"]
```
Correcto — pero el spec pide `assert` en el pipeline mismo para que se verifiquen
en producción, no solo en tests. Si el pipeline corre sin tests, las invariantes
no se validan. Tres líneas en `run_pipeline()` resolverían esto.

**Dashboard con ANSI colors en consola** — el output CLI más elaborado del grupo.
Muestra volumetría, anomalías, persistencia y métricas de rendimiento (filas/segundo).
Funcional y útil en producción real.

---

## Sobre el uso de herramientas de IA

El `force_error_at` en `load_to_sqlite()` — un parámetro de test inyectado en
código de producción para habilitar atomicidad verificable — es el tipo de
decisión de ingeniería que requiere pensar en la testabilidad del sistema como
parte del diseño, no como un afterthought. El UUID v4 check (`val.version != 4`),
el manejo del sufijo `Z` en timestamps, y el multi-error support son detalles
que demuestran lectura de la especificación con atención a los casos borde.
El `STATUSES` definido pero no usado es el tipo de error que se detecta
ejecutando los tests con datos adversariales — un test con `status="cancelled"`
habría capturado el gap inmediatamente.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 7:

> `STATUSES = {"completed", "failed", "pending"}` está definida en `transform.py`
> pero nunca se usa en `validate_transaction()`. Agrega la regla de validación,
> corre el test `test_validation_and_quarantine` — ¿pasa sin cambios? Si pasa,
> ¿qué dice eso sobre la cobertura del test? Agrega un séptimo caso al test con
> `status="cancelled"` y verifica que retorna `invalid_status` en
> `rejection_reasons`. ¿Cambia la invariante `len(valid) == 0`?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Capas separadas y cohesivas | 25% | 22 / 25 |
| Validación y cuarentena | 30% | 24 / 30 |
| Idempotencia verificada | 20% | 20 / 20 |
| Reporte de ejecución | 25% | 22 / 25 |
| **Total** | **100%** | **88 / 100** |

---

El `test_transactional_atomicity` con `force_error_at` y el WAL mode en la capa
de carga son los dos puntos técnicamente más avanzados del grupo en E06. Los
tres gaps concretos para el E07: activar la validación de `status` (una línea
y un test), agregar rangos de `user_id` y `merchant_id`, y mover los `assert`
de invariantes al `run_pipeline()` además de los tests.