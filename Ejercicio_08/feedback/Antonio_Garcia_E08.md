# Retroalimentación — Ejercicio 08: El Proyecto Final

**Alumno:** Antonio García
**Fecha de revisión:** Mayo 2026
**Calificación:** 95 / 100

---

## Resumen general

La entrega más completa del grupo en E08. El `decisions.md` es el mejor del
módulo — el único que cita mediciones reales de ejercicios anteriores como
evidencia en lugar de afirmaciones genéricas. Los 38+ tests incluyen: p95 de
latencia contra 50 muestras, parametrización de los 7 tipos de error en
transform, tests del pipeline independientes del Parquet (corren siempre), y
verificación del campo `cached` en la respuesta de analytics. El `/anomalies`
tiene `window_days` como parámetro adicional — más rico que el spec. Los tres
CACHE_TTL por endpoint están conectados desde `.env` hasta el código.
`API_PORT` configurable en compose. El único gap concreto: sin usuario
non-root en el Dockerfile.

---

## Criterio 1 — Funcionalidad completa `38 / 40`

**Los 7 endpoints requeridos + pipeline + Docker:**

| Endpoint | Estado | Nota |
|---|---|---|
| GET `/health` | ✓ | `connections`, `cache_hit_rate`, `cache_entries` |
| GET `/analytics/summary` | ✓ | `cached: True/False` en respuesta |
| GET `/analytics/top-merchants` | ✓ | `country` filter + `cached` |
| GET `/anomalies` | ✓ | `threshold` + `window_days` parametrizables |
| GET `/users/{id}/transactions` | ✓ | paginación con `n_returned` |
| GET `/users/{id}/stats` | ✓ | 404 limpio si no existe |
| POST `/transactions/batch` | ✓ | `duplicate_ids` en respuesta |
| Pipeline ETL (CSV) | ✓ | extract→transform→load, idempotente |

**`window_days` en `/anomalies` — más rico que el spec:**
```python
@app.get("/anomalies")
def get_anomalies(
    threshold: int = Query(5, ge=0),
    window_days: int = Query(30, ge=1, le=365),
```
El spec pedía solo `threshold` (N parametrizable). `window_days` convierte el
endpoint en genuinamente útil para análisis retrospectivos ("usuarios con >3
fallos en los últimos 7 días" vs "en el último año"). La restricción `ge=0`
permite threshold=0 que el test `test_anomalies_threshold_zero_returns_users`
explota para verificar que hay usuarios en la base.

**`cached: True/False` en las respuestas de analytics:**
```python
hit, value = cache.get(key)
if hit:
    return {**value, "cached": True}
# ...
return {**value, "cached": False}
```
El campo `cached` permite a los clientes saber si están viendo datos frescos
o de caché. El test `test_anomalies_cache_hit` verifica exactamente esto:
`assert r2.json()["cached"] is True`. La API y los tests están en feedback
loop — si alguien elimina el campo `cached`, el test falla.

**`duplicate_ids` en `/transactions/batch`:**
```python
return {
    "received": len(body.transactions),
    "inserted": inserted,
    "duplicates_skipped": len(duplicate_ids),
    "duplicate_ids": duplicate_ids,
}
```
Permite auditar cuáles transacciones fueron rechazadas y por qué.

**Pipeline ETL con reporte de corrida:**
El README documenta el formato exacto del reporte JSON con `run_id`, `started_at`,
`valid`, `rejected.by_reason`, `loaded.inserted`, `loaded.duplicates_skipped` —
el mismo formato que el E06 estableció y este ejercicio extiende para CSV.

---

## Criterio 2 — Calidad técnica `33 / 35`

**38+ tests con cobertura real:**

Test de SLA con p95 sobre 50 muestras — el más riguroso del grupo:
```python
def test_health_sla_p95_under_50ms(client):
    latencies = []
    for _ in range(50):
        t0 = time.perf_counter()
        r = client.get("/health")
        latencies.append((time.perf_counter() - t0) * 1000)
    p95 = statistics.quantiles(latencies, n=20)[18]
    assert p95 < 50, f"p95 /health = {p95:.2f}ms excede 50ms SLA"
```
50 muestras para estabilidad estadística; `statistics.quantiles` para p95
real. Un test con 10 muestras puede pasar accidentalmente si las 10 son
rápidas; 50 da representatividad.

**Parametrización de los 7 tipos de error en transform:**
```python
@pytest.mark.parametrize("field,bad_value,expected_reason", [
    ("amount", 9999.99, "amount_out_of_range"),
    ("category", "Groceries", "invalid_category"),
    ("country_code", "US", "invalid_country"),
    ("status", "cancelled", "invalid_status"),
    ("user_id", 99999, "invalid_user_id"),
    ("merchant_id", 99999, "invalid_merchant_id"),
    ("transaction_id", "not-a-uuid", "invalid_uuid"),
])
def test_transform_rejects_bad_values(tmp_path, field, bad_value, expected_reason):
```
Un test por tipo de error en lugar de siete funciones separadas. Si se agrega
un octavo tipo, se añade un caso al parametrize.

**Tests del pipeline independientes del Parquet:**
```python
@pytest.fixture(scope="module")
def client():
    if not DEFAULT_DB.exists() or not DEFAULT_PARQUET.exists():
        pytest.skip("Faltan datos...")
    with TestClient(app) as c:
        yield c
```
Los tests de API hacen `skip` si faltan los datos — no fallan. Los tests del
pipeline usan CSVs en memoria (`tmp_path`) y nunca tocan el filesystem real.
Resultado: `uv run pytest -k pipeline` siempre corre, sin setup previo.

**Los tres CACHE_TTL env vars conectados correctamente:**
```python
TTL_SUMMARY = float(os.environ.get("CACHE_TTL_SUMMARY", "30"))
TTL_TOP_MERCHANTS = float(os.environ.get("CACHE_TTL_TOP_MERCHANTS", "30"))
TTL_ANOMALIES = float(os.environ.get("CACHE_TTL_ANOMALIES", "30"))
```
`TTL=0` deshabilita efectivamente el cache (`if ttl <= 0: return` en
`cache.set`). Documentado en el README: "TTL 0 = off". Esto es el nivel
correcto de configurabilidad para un sistema en producción.

**`API_PORT` configurable en compose:**
```yaml
ports:
  - "${API_PORT:-8000}:8000"
```
Permite levantar múltiples instancias en diferentes puertos para testing A/B
sin modificar el compose. Ningún otro alumno lo implementó.

**`response_model=` en todos los endpoints:**
```python
@app.get("/anomalies", response_model=AnomalyResponse)
```
FastAPI valida la respuesta contra el modelo de salida y emite errores de
serialización claros en lugar de 500s genéricos.

**Sin usuario non-root — el mismo gap del E07:**
El Dockerfile es idéntico al del E07 (mismo multi-stage, mismo strip, mismo
PYTHONDONTWRITEBYTECODE) excepto que el build context ahora es `.` en lugar
de `..`. El `USER appuser` sigue sin estar.

---

## Criterio 3 — decisions.md `24 / 25`

**El mejor decisions.md del módulo — todas las afirmaciones con evidencia:**

| Sección | Evidencia concreta |
|---|---|
| FastAPI vs DRF | "p50 <0.6ms /health" vs "~120s para cargar el Parquet" del E5 |
| DuckDB | Tabla E2: pandas 11.9s/query, DuckDB 0.29s/query (ratio 41x) |
| SQLite | Tabla E3: P1 sin índice 230ms, con índice 0.18ms (1275x) |
| Cache | E4: cold=45.6ms, warm=0.6ms (ratio 76x) |
| Docker | E7: "imagen de 299.8MB alcanzable con 4 optimizaciones" |
| 100M filas | Hive partitioning + `read_parquet('data/**/*.parquet', hive_partitioning=true)` |
| Monitoreo | 5 métricas con umbrales numéricos específicos |

**La sección de monitoreo proactivo es la única del grupo que responde la
pregunta real:**
> "¿Cómo detectarías un problema *antes* de que lo reporte un usuario?"

Respuesta de Antonio: "Un aumento sostenido en p95 de `/analytics/summary`
indica que el Parquet creció y el cache se está vaciando con demasiada
frecuencia. Umbral de alerta: p95 > 2× la línea base del benchmark E4."

Eso es proactividad: detectar la tendencia antes del umbral de falla, usando
como referencia una medición histórica del mismo sistema.

**Trade-off OLTP/OLAP documentado con consecuencia concreta:**
"Un insert en el batch endpoint NO se refleja en analytics hasta que el Parquet
sea regenerado." Esta limitación se documenta en README *y* en decisions.md,
y está en el diagrama de arquitectura. El sistema no esconde el comportamiento
— lo anuncia.

**Un punto menos:** la sección de 100M filas describe correctamente la
solución técnica (Hive partitioning) pero no menciona qué herramienta
generaría esas particiones nuevas de forma incremental. Con DuckDB:
```sql
COPY (SELECT * FROM transactions WHERE ...) 
TO 'data/year=2026/month=06/' 
(FORMAT PARQUET, PARTITION_BY month);
```
Un párrafo sobre esa operación habría completado el razonamiento.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Funcionalidad completa | 40% | 38 / 40 |
| Calidad técnica | 35% | 33 / 35 |
| decisions.md | 25% | 24 / 25 |
| **Total** | **100%** | **95 / 100** |

---

El `decisions.md` es la pieza más importante del E08 según el enunciado, y esta
entrega lo toma en serio: cada decisión tiene un número del ejercicio que la
justifica. El único cambio restante antes de cerrar el módulo: `USER appuser`
en el Dockerfile — dos líneas que convierten una imagen de producción en una
imagen realmente lista para producción.