# Retroalimentación — Ejercicio 08: El Proyecto Final

**Alumno:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 60 / 100

---

## Resumen general

El sistema integra FastAPI, DuckDB, SQLite, un pipeline de CSV y Docker en una
estructura coherente. Los dos endpoints nuevos del E08 (`/transactions/ingest`
y `/anomalies`) están presentes. El `decisions.md` cubre las cuatro preguntas
del enunciado. Los problemas concretos: falta `/transactions/batch` que estaba
en el enunciado explícitamente; el pipeline tiene un bug crítico en
`validate.py` donde los statuses válidos son `{"approved", "failed"}` en lugar
de `{"completed", "failed", "pending"}`, lo que hace que el 100% de los
registros del dataset real sean rechazados; el endpoint `/anomalies` llama a
`get_sqlite_db()` directamente en lugar de usar la conexión inyectada; y
`setup_db.py` tiene el mismo `if_exists="replace"` del E07 — no incorpora el
feedback de ese ejercicio.

---

## Criterio 1 — Funcionalidad completa `24 / 40`

**Endpoints presentes (7) vs requeridos (7+pipeline):**

| Endpoint | Requerido | Estado |
|---|---|---|
| GET `/health` | ✓ | ✓ presente |
| GET `/analytics/summary` | ✓ | ✓ presente |
| GET `/analytics/top-merchants` | ✓ | ✓ presente |
| GET `/users/{user_id}/transactions` | ✓ | ✓ presente |
| GET `/users/{user_id}/stats` | ✓ | ✓ presente |
| POST `/transactions/batch` | ✓ | ✗ ausente |
| GET `/anomalies` | ✓ | ✓ presente (con bug) |
| POST `/transactions/ingest` (pipeline) | ✓ | ✓ presente |

`/transactions/batch` fue reemplazado por `/transactions/ingest` en lugar de
agregarlo como endpoint adicional. El enunciado requería ambos: el batch insert
del E04 más el pipeline de CSV del E08.

**Bug crítico en `/anomalies` — llama a `get_sqlite_db()` directamente:**
```python
@app.get("/anomalies")
def detect_anomalies(threshold: int = Query(5, ge=1), db: sqlite3.Connection = Depends(get_sqlite_db)):
    conn = get_sqlite_db()  # ← ignora 'db' inyectado, llama la función directamente
    cursor = conn.cursor()
```
`get_sqlite_db()` es un generador de FastAPI (`yield`). Llamarlo directamente
fuera del contexto de dependencias crea una conexión sin gestión del ciclo de
vida — o falla con `StopIteration`, o abre una conexión que nunca se cierra.
El parámetro correcto es usar la `db` inyectada: `cursor = db.cursor()`.

**Bug crítico en `validate.py` — statuses incorrectos:**
```python
valid_status = {"approved", "failed"}
```
El schema del sistema usa `{"completed", "failed", "pending"}`. Cualquier
CSV con transacciones de status `"completed"` (la mayoría del dataset real)
será rechazado con "Invalid values found in 'status' column" antes de llegar
a la base de datos. El pipeline devuelve `success: False` para todos los
archivos del dataset principal. Esto inutiliza la funcionalidad de ingestión
para el caso de uso real.

**`setup_db.py` idéntico al E07 — `if_exists="replace"` sin corregir:**
```python
df.to_sql("transactions", conn, if_exists="replace", index=False)
```
La retroalimentación del E07 señaló explícitamente que `replace` borra y
recrea la tabla en cada corrida, lo que no es idempotente. No se incorporó.

**Pipeline `/transactions/ingest` funciona para el caso simple:**
- Verifica existencia del archivo ✓
- Llama a `validate_dataframe()` ✓ (aunque con statuses incorrectos)
- `INSERT OR IGNORE` ✓
- Retorna `success`, `rows_inserted`, `errors` ✓

---

## Criterio 2 — Calidad técnica `19 / 35`

**`iterrows()` en `pipeline/ingest.py` — pendiente del E05:**
```python
insert_data = [
    (row["transaction_id"], ...)
    for _, row in df.iterrows()
]
```
La retroalimentación del E05 señaló este patrón. Con un CSV de 100,000 filas,
`iterrows()` puede tardar segundos. La alternativa es `df.to_records()` o
construir la lista de tuplas con `list(df.itertuples(index=False, name=None))`.

**Tests: funcionan solo dentro del contenedor con paths hardcodeados:**
```python
def override_sqlite_db():
    return sqlite3.connect("/data/transactions.db")

def override_duckdb_db():
    conn.execute("CREATE OR REPLACE VIEW gold_transactions AS SELECT * FROM '/data/benchmark_1m.parquet'")
```
Las rutas `/data/transactions.db` y `/data/benchmark_1m.parquet` son rutas
absolutas dentro del contenedor. Los tests no corren en la máquina del
desarrollador sin Docker. El README documenta esto correctamente (`docker exec
-it api_service bash && pytest`), pero para verificación del evaluador se
necesita el stack completo corriendo.

**5 tests cubren los paths principales:**
1. `test_health` — verifica 200 y campos ✓
2. `test_analytics_summary` — verifica 200 y campos ✓
3. `test_user_not_found` — verifica 404 con usuario inválido ✓
4. `test_ingest_invalid_file` — verifica manejo de archivo inexistente ✓
5. `test_anomalies_endpoint` — verifica estructura de respuesta ✓

Faltantes críticos según el enunciado:
- Test de deduplicación (mismo `transaction_id` → no sobreescribe)
- Test de rechazo de schema inválido en el batch
- Test de anomalías con un threshold real (no solo estructura)
- Test de SLA de latencia

**Código organizado en módulos:**
```
app/     ← API (main.py, db.py, cache.py, models.py)
pipeline/ ← ingesta (ingest.py, validate.py)
tests/   ← test_system.py
```
La separación entre `app/` y `pipeline/` es correcta. ✓

**Configuración desde variables de entorno en el código principal ✓** — el
hardcoding está solo en los tests, no en la app.

**docker-compose usa v1 syntax (`docker-compose up --build`)** — mismo que E07,
sin actualizar a `docker compose up --build`.

---

## Criterio 3 — decisions.md `17 / 25`

**Cubre las cuatro preguntas del enunciado:**
1. Tecnologías por capa ✓ (FastAPI, SQLite, DuckDB, Pandas, Docker)
2. Trade-offs ✓ (SQLite vs PostgreSQL, DuckDB vs Spark, cache en memoria)
3. Escalabilidad a 100M filas ✓ (PostgreSQL, BigQuery/Snowflake, Airflow/Kafka)
4. Monitoreo en producción ✓ (/health, logging JSON, métricas clave, alertas)

**Lo que falta para mayor profundidad:**

El enunciado pide "evidencia de los ejercicios anteriores". El decisions.md
menciona los ejercicios ("basado en la lógica implementada en el E06", "tal
como se hizo en el E07") pero no cita mediciones concretas. Por ejemplo:
"El E03 demostró que SQLite con idx_user_timestamp cumple SLA P1-P4 <1ms
— por eso se mantiene para consultas de usuario" sería evidencia concreta.

Los trade-offs son correctos pero genéricos. "SQLite: configuración mínima,
menor complejidad operativa" no explica qué se sacrifica en este sistema
específico para esta carga específica.

La sección de monitoreo lista métricas correctas pero describe el `/health`
que ya existe — la pregunta pedía cómo detectar un problema *antes* de que lo
reporte un usuario, lo que implica proactividad (alertas en P95 de latencia,
no solo cuando falla el healthcheck).

**El documento tiene aproximadamente 650 palabras** — cumple el mínimo de 600.

---

## Sobre el uso de herramientas de IA

El decisions.md está estructurado con encabezados y tiene las secciones
correctas, pero el contenido de trade-offs tiene el patrón de "ventajas /
desventajas" genéricas que no responden la pregunta de por qué esta decisión
para este sistema. `validate.py` con `{"approved", "failed"}` es el tipo de
error que aparece cuando el código se genera sin conectarlo al schema real del
sistema — y es el mismo patrón que los ejercicios anteriores (VALID_STATUSES
definida pero desconectada en E06, CACHE_TTL declarada pero hardcodeada en E07).
El E08 necesitaba resolver esos patrones, no repetirlos.

---

## Pregunta de seguimiento

> Tres correcciones antes de entregar:
>
> 1. En `/anomalies`, reemplaza `conn = get_sqlite_db()` por `cursor = db.cursor()`.
>
> 2. En `validate.py`, cambia `valid_status = {"approved", "failed"}` por
>    `valid_status = {"completed", "failed", "pending"}` y ejecuta el pipeline
>    contra el CSV del E01 para verificar que ahora retorna `success: True`.
>
> 3. Agrega `/transactions/batch` al E08 (ya lo tenías en E07 — es una línea de
>    importación y el endpoint copiado). ¿Por qué el enunciado pide tenerlo
>    si ya tienes `/transactions/ingest`? ¿Cuál es la diferencia entre insertar
>    un batch de JSON vs ingestar un CSV?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Funcionalidad completa | 40% | 24 / 40 |
| Calidad técnica | 35% | 19 / 35 |
| decisions.md | 25% | 17 / 25 |
| **Total** | **100%** | **60 / 100** |

---

Las tres correcciones de la pregunta de seguimiento (bug en /anomalies,
statuses incorrectos en validate.py, agregar /transactions/batch) son las
acciones más urgentes. El `decisions.md` está completo en estructura — la
diferencia entre 17/25 y 22/25 es agregar mediciones de ejercicios anteriores
como evidencia en lugar de afirmaciones genéricas.