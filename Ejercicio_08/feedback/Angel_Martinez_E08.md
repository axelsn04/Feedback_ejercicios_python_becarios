# Retroalimentación — Ejercicio 08: El Proyecto Final

**Alumno:** Angel Martinez  
**Fecha de revisión:** Mayo 2026  
**Calificación:** 93 / 100

---

## Resumen general

La entrega más completa de Angel en el módulo. 29 tests en tres archivos (15 API, 9 pipeline, 5 anomalías), `DatabaseManager` con gestión de conexiones dual bien encapsulada, `db_manager.get_duckdb_cursor()` que retorna un cursor clonado por request (thread-safety correcta), y un `decisions.md` con cifras reales de ejercicios anteriores. El pattern `127.0.0.1:8000:8000` del E07 se mantiene. El gap más importante: `setup_db.py` **no es idempotente** — borra y recrea la base en cada ejecución, lo que en un compose real significa perder los datos insertados vía pipeline. El segundo gap: sin usuario non-root en el Dockerfile.

---

## Criterio 1 — Funcionalidad completa `38 / 40`

**Los 8 endpoints implementados + pipeline + Docker:**

| Endpoint | Estado | Nota |
|---|---|---|
| GET `/health` | ✓ | `status healthy/degraded`, `cache_hits`, `cache_misses`, `connections_active`, `uptime_seconds` |
| GET `/analytics/summary` | ✓ | `breakdown_by_country` + `breakdown_by_category`, cache con TTL configurable |
| GET `/analytics/top-merchants` | ✓ | `limit` + `country` filter, cache por clave compuesta |
| GET `/analytics/anomalies` | ✓ | `threshold` + `period_days=30`, umbral desde env var `ANOMALY_DEFAULT_THRESHOLD` |
| GET `/users/{id}/transactions` | ✓ | `date_from`, `date_to`, `page`, `page_size`, `total_pages`, 404 limpio |
| GET `/users/{id}/stats` | ✓ | `total_amount`, `transaction_count`, `most_frequent_category`, `country_code` |
| POST `/transactions/batch` | ✓ | Idempotente, límite de 500, `ignored_ids` en respuesta, deduplicación in-memory |
| POST `/pipeline/ingest` | ✓ | Upload CSV, pipeline E→T→L, reporte con métricas, limpieza de temp file |
| Pipeline ETL | ✓ | Capas separadas importables desde el endpoint |
| Docker | ✓ | Dos Dockerfiles (api + setup), compose con `depends_on`, healthcheck |

**`ANOMALY_DEFAULT_THRESHOLD` desde env var — único del grupo:**
```python
ANOMALY_DEFAULT_THRESHOLD = int(os.getenv("ANOMALY_DEFAULT_THRESHOLD", "5"))
...
effective_threshold = threshold if threshold is not None else ANOMALY_DEFAULT_THRESHOLD
```
El umbral por defecto es configurable sin reconstruir la imagen. Permite ajustar la sensibilidad de detección en producción vía `.env` sin tocar código.

**`/transactions/batch` con deduplicación en dos capas:**
```python
# Capa 1: deduplicación in-memory del mismo batch
seen = {}
for tx in batch:
    seen[tx.transaction_id] = tx
unique_txs = list(seen.values())

# Capa 2: verificación contra la base antes de insertar
existing_ids = {row[0] for row in cursor.fetchall()}
to_insert = [tx for tx in unique_txs if tx.transaction_id not in existing_ids]
```
Un batch que trae el mismo `transaction_id` dos veces no genera un error de constraint — se deduplica antes. Luego se verifica contra la base. La respuesta incluye `ignored_ids` para auditoría.

**`/pipeline/ingest` con sanitización de path y cleanup garantizado:**
```python
temp_filename = f"ingest_{uuid.uuid4().hex}.csv"
temp_path = os.path.join(temp_dir, temp_filename)
try:
    ...
finally:
    if os.path.exists(temp_path):
        os.remove(temp_path)
```
UUID hex como nombre de archivo previene path traversal. El `finally` garantiza que el archivo temporal se elimina incluso si el pipeline falla.

**`setup_db.py` no es idempotente — gap concreto:**
```python
if os.path.exists(sqlite_db_path):
    print(f"Existing SQLite DB found... Removing it to rebuild a clean state...")
    os.remove(sqlite_db_path)
```
El setup borra y recrea la base en cada ejecución. En el compose actual esto no es un problema porque el servicio `setup` corre una sola vez. Pero si se hace `docker compose restart setup` o se escala el servicio, todos los datos insertados vía `/pipeline/ingest` o `/transactions/batch` se pierden. El patrón correcto es verificar si la tabla ya existe y tiene datos antes de reingestar:
```python
cursor.execute("SELECT COUNT(*) FROM transactions")
if cursor.fetchone()[0] > 0:
    print("Database already populated. Skipping ingestion.")
    return
```

---

## Criterio 2 — Calidad técnica `33 / 35`

**`DatabaseManager` con cursor clonado para thread-safety:**
```python
def get_duckdb_cursor(self):
    # Returns a clone cursor to ensure safe concurrent thread usage
    return self.duckdb_conn.cursor()
```
DuckDB en modo in-memory no es thread-safe si múltiples threads comparten el mismo cursor. Retornar un cursor nuevo por request (`.cursor()` crea un contexto de ejecución independiente) resuelve esto correctamente.

**29 tests en tres archivos con cobertura estructurada:**

| Archivo | Tests | Cobertura |
|---|---|---|
| `test_api.py` | 15 | Happy path por endpoint, idempotencia batch, 404, 422, límite 500, SLA |
| `test_anomalies.py` | 5 | Default threshold, custom, threshold alto (lista vacía), estructura de respuesta, 422 con threshold=0 |
| `test_pipeline.py` | 9 | Normalización de campos, rechazo por amount/category, cuarentena JSONL, pipeline completo, idempotencia, endpoint ingest |

**Test de idempotencia del pipeline con verificación concreta:**
```python
def test_pipeline_idempotency(sample_csv_path):
    report1 = run_csv_pipeline(...)
    inserted_first = report1["filas_insertadas"]

    report2 = run_csv_pipeline(...)
    assert report2["filas_insertadas"] == 0
    assert report2["filas_duplicadas"] == inserted_first + report1["filas_duplicadas"]
```
Verifica que la segunda corrida inserta cero y que el conteo de duplicados cuadra. No solo verifica el estado final — verifica que los números son internamente consistentes.

**Test de estructura de respuesta de anomalías con invariante:**
```python
def test_anomalies_response_structure(client):
    response = client.get("/analytics/anomalies?threshold=1")
    ...
    if data["anomalies"]:
        first = data["anomalies"][0]
        assert first["failed_count"] > 1  # > threshold, no >= threshold
```
Verifica que `HAVING COUNT(*) > threshold` se comporta como `>` y no como `>=`. Un test que solo verifica la estructura sin verificar el invariante de negocio no detectaría un bug en el operador de comparación.

**SLA test con promedio de 10 muestras:**
```python
def test_health_sla_latency_under_50ms(client):
    latencies = []
    for _ in range(10):
        start = time.perf_counter()
        response = client.get("/health")
        duration = (time.perf_counter() - start) * 1000
        latencies.append(duration)
    avg_latency = sum(latencies) / len(latencies)
    assert avg_latency < 50.0
```
Mejor que 1 muestra (Ernesto), pero promedio vs p95 es una diferencia importante: un promedio de 40ms puede esconder un p95 de 180ms si hay varianza alta. El test de Antonio usa 50 muestras + `statistics.quantiles` para p95 real.

**Dockerfile con `uv` para instalación rápida — único del grupo:**
```dockerfile
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
RUN uv venv /opt/venv
RUN . /opt/venv/bin/activate && \
    uv pip install --no-cache fastapi uvicorn duckdb pydantic pydantic-core python-multipart
```
`uv` es significativamente más rápido que pip para resolver e instalar dependencias. El venv en `/opt/venv` se copia al stage final limpiamente. Ventaja: builds más rápidos en CI/CD.

**Sin usuario non-root — patrón recurrente:**
```dockerfile
# Ausente en ambos Dockerfiles:
# RUN adduser --disabled-password appuser
# USER appuser
```
Ni el `Dockerfile` ni el `Dockerfile.setup` tienen `USER appuser`. La imagen corre como root, lo que es un riesgo de seguridad en producción. Tres líneas que cierran este gap.

**`setup_json_logging()` aplicado a uvicorn — detalle correcto:**
```python
for logger_name in ("", "uvicorn", "uvicorn.access", "uvicorn.error"):
    logger = logging.getLogger(logger_name)
    handler.setFormatter(JSONFormatter())
```
Los logs de acceso de uvicorn también salen en JSON. Sin esto, los logs de la app son JSON pero los de uvicorn son texto plano, mezclando formatos en el mismo stream.

---

## Criterio 3 — decisions.md `22 / 25`

**Cifras de ejercicios anteriores como evidencia:**

| Sección | Evidencia |
|---|---|
| FastAPI vs DRF | "latencia promedio mayor en DRF por middleware de Django" del E05, "menos de 200ms en todos los endpoints analíticos" del E04 |
| DuckDB | "menos de 200ms para aggregaciones sobre 1M filas" del E04 |
| SQLite | "menos de 50ms para consultas puntuales con los índices correctos" del E03 |
| Pipeline stdlib | "pandas añade más de 100MB" — justificación de imagen liviana |
| 100M filas | PostgreSQL con particionamiento, DuckDB con particiones Parquet por mes, Celery/Airflow, Gunicorn multi-worker |
| Monitoreo | 4 métricas con umbrales numéricos (p95 >500ms, error 5xx, quarantine >20%, pipeline insert rate 0 por 1h) |

**Trade-off OLTP/OLAP documentado con consecuencia concreta:**
> "las transacciones insertadas via batch o pipeline no aparecen en los endpoints analíticos hasta que se regenere el Parquet"

La limitación está documentada y es honesta. Igual que Ernesto, el sistema no esconde el comportamiento.

**Tres puntos menos — el decisions.md no cierra dos argumentos:**

Primero, la sección de DuckDB cita "menos de 200ms" del E04 pero no incluye la comparación directa con SQLite para el mismo patrón que sí aparece en el E03 (ratio 41x o 303x). Las cifras del E02 (pandas 11.9s vs DuckDB 0.29s) habrían reforzado el argumento de forma más directa.

Segundo, la sección de 100M filas es la más débil del módulo entre los alumnos que entregaron. Menciona las tecnologías correctas (PostgreSQL, particionamiento Parquet, Celery, Kubernetes) pero no especifica qué cambia concretamente en el código actual. Falta la operación DuckDB que generaría las particiones nuevas, el índice parcial de PostgreSQL para `status = 'failed'`, o el tamaño estimado del archivo SQLite a 100M filas (≈20GB). Las tecnologías correctas sin las consecuencias concretas es la mitad del argumento.

Tercero, el monitoreo detecta el problema de calidad de datos (quarantine >20%) pero no responde la pregunta del spec: *"cómo detectarías un problema antes de que lo reporte un usuario"*. La alerta de cuarentena es reactiva (ya falló la validación). La respuesta proactiva sería monitorear la tendencia: si el p95 de `/analytics/summary` sube 2× sobre la línea base del E04, el Parquet creció y el cache se está vaciando con demasiada frecuencia.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Funcionalidad completa | 40% | 38 / 40 |
| Calidad técnica | 33 / 35 | 33 / 35 |
| decisions.md | 25% | 22 / 25 |
| **Total** | **100%** | **93 / 100** |

---

La entrega de Angel tiene la suite de tests más estructurada del grupo (tres archivos, 29 tests, cobertura de invariantes de negocio), el `DatabaseManager` mejor encapsulado, y el único uso de `uv` en el módulo. Los dos gaps que quedan: `USER appuser` en ambos Dockerfiles, y el setup que borra la base en lugar de verificar si ya tiene datos. El segundo es el más crítico en producción — un restart del servicio setup no debería borrar lo que el pipeline insertó.