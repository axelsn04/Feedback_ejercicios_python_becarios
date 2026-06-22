# Retroalimentación — Ejercicio 08: El Proyecto Final

**Alumno:** Ernesto Cuapantecatl  
**Fecha de revisión:** Mayo 2026  
**Calificación:** 91 / 100

---

## Resumen general

Entrega sólida y cohesiva. El sistema integra correctamente los componentes de E04–E07: DuckDB sobre Parquet para analytics, SQLite con índices para transaccional, pipeline ETL del E06 reutilizado como funciones importables, y Docker multi-stage del E07. El `decisions.md` es el segundo mejor del módulo — todas las afirmaciones tienen número de ejercicio y cifra concreta como respaldo, y el análisis de trade-offs es honesto, incluyendo la inconsistencia DuckDB/SQLite que el propio alumno identifica. Los 14 tests pasan y cubren los flujos críticos. Los gaps son concretos: sin non-root en Dockerfile (patrón del E07 que no se resolvió), el SLA test mide solo una muestra (no p95 estadístico), y el `decisions.md` no termina de cerrar el argumento sobre ingesta incremental a 100M filas.

---

## Criterio 1 — Funcionalidad completa `37 / 40`

**Los 7 endpoints requeridos + pipeline + Docker:**

| Endpoint | Estado | Nota |
|---|---|---|
| GET `/health` | ✓ | `uptime_s`, `cache_hit_rate`, `transaction_count`, `connections` |
| GET `/analytics/summary` | ✓ | `by_country` + `by_category`, cache con TTL 60s |
| GET `/analytics/top-merchants` | ✓ | `limit` + `country` filter, cache por clave compuesta |
| GET `/users/{id}/transactions` | ✓ | `date_from`, `date_to`, `page`, `page_size`, 404 limpio |
| GET `/users/{id}/stats` | ✓ | `total_amount`, `transaction_count`, `top_category`, `country_code` |
| GET `/anomalies/failed-transactions` | ✓ | `threshold` + `days` parametrizables |
| POST `/ingest/csv` | ✓ | Pipeline E→T→L, reporte con `rejected_by_type`, invalidación de cache |
| Pipeline ETL | ✓ | Reutilizado como funciones importables desde el endpoint HTTP |
| Docker | ✓ | Multi-stage, healthcheck, volúmenes, setup idempotente |

**`/ingest/csv` integrado con invalidación de cache — decisión correcta:**
```python
if load_result['inserted'] > 0:
    cache.invalidate_prefix('analytics:')
```
Cache DuckDB sobre Parquet estático no refleja inserciones en SQLite, pero al menos los endpoints analíticos se invalidan cuando hay ingesta nueva. El `decisions.md` documenta honestamente la inconsistencia residual: el Parquet no se actualiza, así que `analytics/summary` sigue sin ver los datos nuevos aunque el cache se limpie.

**`/anomalies` con `days` parametrizable — bien ejecutado:**
```python
@app.get("/anomalies/failed-transactions")
def anomalies_failed(
    threshold: int = Query(5, ge=1, ...),
    days:      int = Query(30, ge=1, le=365, ...),
):
    ...
    rows = conn.execute(
        "... AND timestamp >= datetime('now', ?) ...",
        (f'-{days} days', threshold),
    )
```
`ge=1` en `threshold` a diferencia de Antonio (`ge=0`) — decisión razonable, pero significa que no se puede usar `threshold=0` para verificar que hay usuarios en la base. Ninguno es incorrecto, son trade-offs distintos.

**Un punto menos: sin `cached` field en las respuestas de analytics.** El cache funciona correctamente, pero la respuesta no expone si el dato es fresco o de caché. Los clientes no pueden distinguir. Antonio lo implementó; aquí está ausente. No es un requisito del spec, pero es una pieza que completa el contrato.

**Un punto menos: `API_PORT` no configurable en compose.** El puerto está hardcodeado como `"8000:8000"`. Dos caracteres habrían resuelto esto: `"${API_PORT:-8000}:8000"`.

---

## Criterio 2 — Calidad técnica `31 / 35`

**14 tests con cobertura de flujos críticos:**

El SLA test mide una sola muestra, no p95:
```python
def test_analytics_summary_sla(client):
    client.get("/analytics/summary")  # warm cache
    start = time.perf_counter()
    resp  = client.get("/analytics/summary")
    elapsed_ms = (time.perf_counter() - start) * 1000
    assert elapsed_ms < 20, f"SLA violated: {elapsed_ms:.1f}ms > 20ms"
```
Una sola muestra puede pasar aunque el p95 real supere 20ms (si esa ejecución particular fue rápida). El test de Antonio usa 50 muestras y `statistics.quantiles` para p95 real. Con una sola muestra el test verifica que *una* request fue rápida, no que el endpoint sea confiablemente rápido.

**`check_same_thread=False` + WAL mode — configuración correcta para FastAPI:**
```python
_sqlite_conn = sqlite3.connect(db_path, check_same_thread=False)
_sqlite_conn.execute('PRAGMA journal_mode=WAL')
_sqlite_conn.execute('PRAGMA synchronous=NORMAL')
```
`check_same_thread=False` es necesario porque FastAPI puede ejecutar handlers en threads distintos. WAL mode permite lecturas concurrentes sin bloquear escrituras.

**`load.py` recibe la conexión existente — decisión correcta:**
```python
def load(rows: list[dict], conn: sqlite3.Connection) -> dict:
    ...
    with conn:
        conn.executemany('INSERT OR IGNORE INTO transactions VALUES (...)', values)
```
Reutilizar la conexión de la app evita conflictos de bloqueo y garantiza que el WAL y `row_factory` ya están configurados. El README documenta explícitamente esta diferencia con el E06.

**`_require_env` con `sys.exit` — falla rápido y claro:**
```python
def _require_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        sys.exit(f"ERROR: la variable de entorno {name} es requerida. "
                 f"Revisa .env.example para los valores esperados.")
    return value
```
El sistema no arranca silenciosamente con configuración incorrecta. El mensaje incluye qué variable falta y dónde buscar el valor esperado.

**Dockerfile multi-stage con `--prefix=/install` — patrón del E07 aplicado:**
```dockerfile
FROM python:3.11-slim AS builder
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /install /usr/local
```
Imagen final sin pip ni cache de build. El mismo patrón del E07, aplicado consistentemente.

**Sin usuario non-root — tercer ejercicio consecutivo con este gap:**
```dockerfile
# Ausente:
# RUN adduser --disabled-password appuser
# USER appuser
```
E06 no tenía Dockerfile. E07: sin non-root. E08: sin non-root. Es el mismo Dockerfile del E07 con el contexto de build ajustado (`COPY app/ ./app/` en lugar de `COPY ../app`). Dos líneas que convierten la imagen en apta para producción real.

**Parametrización de errores transform — bien estructurada pero sin pytest.mark.parametrize:**
`transform.py` valida 6+ tipos de error correctamente. Los tests de ingesta CSV cubren UUID inválido y amount negativo en `test_csv_ingest_with_errors`, pero no hay un test parametrizado que cubra cada tipo de error por separado. Si se rompe la validación de `country_code`, los tests actuales no lo detectarían.

---

## Criterio 3 — decisions.md `23 / 25`

**Todas las afirmaciones con cifras de ejercicios anteriores:**

| Sección | Evidencia concreta |
|---|---|
| FastAPI vs DRF | "p50 de 2ms en warm" del E04, arranque más rápido para contenedores |
| DuckDB | "projection pushdown lee solo el 7.4% del Parquet" del E02 |
| SQLite | "respondió en 0.079ms para P2, 303× más rápido que DuckDB" del E03 |
| Cache | "cold 140ms, warm 2ms" — TTL 60s justificado por patrón de tráfico |
| Docker | Setup idempotente, volúmenes, imagen <300MB del E07 |
| 100M filas | PostgreSQL con índices parciales, Parquet particionado, Redis, ingesta incremental |
| Monitoreo | 5 métricas con umbrales numéricos específicos |

**El trade-off OLTP/OLAP documentado honestamente:**
> "Un detalle importante: los endpoints analíticos consultan el Parquet original de 1M de registros, no los datos nuevos que se insertan por CSV en SQLite."

Esto es exactamente lo que hay que documentar: la limitación que un usuario podría confundir con un bug. El sistema no la esconde, la anuncia. Está en el `decisions.md` *y* aparece mencionada en el README.

**Monitoreo proactivo con umbrales concretos:**
La sección de monitoreo tiene 5 métricas, cada una con un umbral numérico: p95 de latencia, hit rate del cache (alerta si cae de 95% a 30%), cuarentena >20% de rechazo, espacio en disco al 80%, cero tolerancia a 5xx. El umbral de cuarentena (`rows_rejected / rows_extracted > 0.2`) es proactivo: detecta el problema antes de que un usuario lo note.

**Un punto menos: la sección de 100M filas no cierra el argumento de ingesta incremental.** Menciona correctamente que el setup de 30 minutos sería inaceptable y propone "un proceso continuo que lea Parquet desde S3". Pero no especifica cómo se ejecutaría ese proceso con DuckDB, que es la herramienta ya en el stack:
```sql
COPY (SELECT * FROM transactions WHERE ...)
TO 'data/year=2026/month=06/'
(FORMAT PARQUET, PARTITION_BY month);
```
La solución técnica existe dentro del stack ya elegido. Un párrafo sobre esa operación habría cerrado el razonamiento.

**Un punto menos: sin `response_model=` en los endpoints.** FastAPI puede validar la respuesta contra un modelo de salida y emitir errores de serialización claros. Sin él, un bug que devuelva un campo con tipo incorrecto genera un 500 genérico en lugar de un error descriptivo en desarrollo.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Funcionalidad completa | 40% | 37 / 40 |
| Calidad técnica | 35% | 31 / 35 |
| decisions.md | 25% | 23 / 25 |
| **Total** | **100%** | **91 / 100** |

---

El sistema integra correctamente todos los componentes del módulo y el `decisions.md` tiene el nivel de rigor correcto: cifras reales, trade-offs honestos, monitoreo con umbrales. El patrón que queda pendiente es el mismo desde el E07: `USER appuser` en el Dockerfile. No es una línea difícil — es la diferencia entre una imagen que funciona y una imagen lista para producción.