# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumno:** Bryan Alexander Gomez Miranda
**Fecha de revisión:** Mayo 2026
**Calificación:** 93 / 100

---

## Resumen general

La entrega más completa del grupo para E04 en términos de arquitectura documentada, tests y la decisión de invalidar el cache después de un POST. Cuatro cosas distinguen esta entrega del resto: el `architecture_decision.md` es el más honesto del grupo (documenta que se probó SQLite primero y se midió antes de elegir DuckDB), la invalidación del cache al insertar está implementada y documentada, los 14+ tests incluyen validación de ordenamiento y tests de SLA separados por endpoint, y todos los endpoints tienen `response_model=` en Pydantic. El único bug concreto: SQL injection en `/analytics/top-merchants`.

---

## Criterio 1 — 6 endpoints funcionales con datos reales `21 / 25`

**Invalidación del cache después del POST batch** — única implementación correcta de este patrón junto con Ernesto. Después de insertar:

```python
cache.invalidate("summary")
```

Esto garantiza que la próxima llamada a `/analytics/summary` recalcula desde DuckDB con los datos nuevos en lugar de devolver un resumen desactualizado. El `architecture_decision.md` explica el límite correctamente: el Parquet del E1 es estático, así que solo se invalida `summary` (que lee SQLite implícitamente a través del flujo), no el cache de DuckDB.

**`INSERT OR IGNORE` con deduplicación en dos pasos** — correcto. Pre-verifica qué IDs ya existen en la base, inserta solo los nuevos, y retorna `BatchResponse` con `inserted`, `duplicates`, e `invalid`.

**Response models tipados para todos los endpoints** — `SummaryResponse`, `TopMerchantsResponse`, `UserTransactionsResponse`, `UserStatsResponse`, `BatchResponse`, `HealthResponse`. Serialización validada por Pydantic en la respuesta, no solo en el request. El más completo del grupo en este punto.

**`PRAGMA threads=4` en DuckDB** — activa paralelización explícita del motor. Pequeño detalle que puede hacer diferencia en agregaciones complejas en hardware multi-core.

**SQL injection en `/analytics/top-merchants`:**
```python
where = f"WHERE country_code = '{country.upper()}'" if country else ""
rows  = db.duck.execute(f"""... {where} ...""")
```
Es el mismo bug que Yaanit. Un request con `country=MX' OR '1'='1` modifica la query. La corrección es usar parámetros vinculados — en DuckDB con `?` como placeholder:
```python
# Correcto
if country:
    rows = db.duck.execute("... WHERE country_code = ? ...", [country.upper()])
```

**Sin pre-warming de DuckDB en el lifespan** — Ernesto hace `SELECT COUNT(*) FROM transactions` al arrancar para cargar el Parquet en memoria. Sin eso, la primera llamada cold absorbe el costo de I/O del archivo.

---

## Criterio 2 — Architecture decision justificado `20 / 20`

El `architecture_decision.md` es el mejor del grupo por una razón concreta: documenta que se probó SQLite primero para `/analytics/summary` y se midió el resultado antes de elegir DuckDB.

> "Probé implementarlo con SQLite primero — la query tarda entre 280ms y 350ms sin índices útiles para agregaciones globales. Con DuckDB sobre el Parquet la misma query tarda entre 80ms y 150ms en frío."

Ese tipo de razonamiento basado en datos — no en suposición — es el estándar para un `architecture_decision.md`. Ningún otro alumno del grupo documentó el experimento negativo (la alternativa que se descartó con medición).

La nota sobre la limitación del sistema: "los datos nuevos solo existen en SQLite y se reflejarán en analytics si hay un pipeline de sincronización" — reconoce honestamente que la invalidación del cache resuelve el problema a corto plazo pero no la consistencia a largo plazo entre SQLite y DuckDB.

La justificación por qué `/health` no toca bases de datos y siempre cumple SLA, y por qué las conexiones deben estar en el lifespan (con el argumento cuantificado: "50-200ms de overhead por request") cierra el documento de forma completa.

---

## Criterio 3 — Cache con TTL y medición `19 / 20`

**`time.monotonic()`** en lugar de `time.time()` — pequeña mejora de precisión: `monotonic()` es garantizado no-decreasing y no se ve afectado por ajustes del reloj del sistema.

**`invalidate(key)`** implementado y usado en el POST batch — el método existe y se llama en el lugar correcto.

**TTL de 5 minutos (300s)** para analytics — más largo que el resto del grupo (mayoría usa 60s). La justificación es correcta: datos históricos del Parquet del E1 no cambian entre requests. El trade-off es que si se insertaran datos nuevos por batch, el summary mostraría datos hasta 5 minutos desactualizados — mitigado por la invalidación explícita.

**Impacto medido:**
- `/analytics/summary`: cold 60ms → warm p50 0.61ms → **99x de mejora**
- `/analytics/top-merchants`: cold 25ms → warm p50 0.65ms → **39x de mejora**

Un punto menos: no hay `threading.Lock()` — Angel lo implementa. Bajo carga concurrente con múltiples workers, el cache puede tener race conditions en `_hits` y `_misses`.

---

## Criterio 4 — Tests con pytest `20 / 20`

14+ tests con cobertura real. Dos tests únicos en el grupo:

**`valid_user_id` fixture** — en lugar de hardcodear `user_id=1`, el fixture consulta la base real para obtener el usuario con más transacciones. Es el mismo principio de "parámetros reales" que aplican Ulises y Antonio en E03: un usuario con muchas transacciones representa el peor caso del SLA.

**`test_top_merchants_default` valida ordenamiento:**
```python
assert data["merchants"][0]["total_amount"] >= data["merchants"][-1]["total_amount"]
```
El único test del grupo que verifica que el resultado está correctamente ordenado (no solo que tiene la estructura correcta). Esta es la validación que diferencia un test que verifica el contrato del endpoint de uno que solo comprueba que no explota.

**`test_analytics_summary_cache`** valida dos condiciones: que warm es más rápido que cold Y que warm < 20ms. La mayoría del grupo solo valida la segunda.

**`test_batch_deduplication`** verifica `inserted==0` y `duplicates==1` en el segundo insert del mismo ID — valida la lógica de deduplicación, no solo que el endpoint responde 200.

**`test_batch_invalid_status`** como test separado — prueba que `status: "unknown_status"` retorna 422, separado del test de `amount < 0`. Cobertura más granular.

**SLA tests separados** para `/health` (p95 < 50ms con 10 repeticiones) y `/analytics/summary` warm (< 20ms).

---

## Criterio 5 — Benchmark de latencia con cache `13 / 15`

El reporte incluye p50/p95/p99 cold vs warm, análisis por endpoint, y nota sobre la metodología (TestClient in-process). Los tiempos son los mejores del grupo:

- `/analytics/summary`: cold 60ms, warm p50 0.61ms
- `/analytics/top-merchants`: cold 25ms, warm p50 0.65ms
- `/users` endpoints: p50 < 1ms

La nota sobre la diferencia entre E3 (80-160ms) y E4 (< 1ms) para los mismos patrones SQLite es el análisis más preciso: "aquí el page cache está caliente y la conexión se reutiliza desde el lifespan sin overhead de apertura".

Dos puntos menos: el benchmark no incluye `POST /transactions/batch` (Angel lo mide con 500 registros y obtiene p50 28ms). Para un sistema que promete SLA < 2s en esa operación, no medirlo es una omisión relevante.

---

## Sobre el uso de herramientas de IA

Documentar que SQLite fue probado primero y descartado con medición, implementar la invalidación del cache, y escribir `test_top_merchants_default` con validación de ordenamiento son decisiones que requieren pensar el sistema, no generar código. El SQL injection en DuckDB es el tipo de bug que aparece cuando el código se escribe sin revisión de seguridad. En conjunto: código y tests genuinos, con un punto ciego en seguridad de queries.

---

## Pregunta de seguimiento

> En `architecture_decision.md` documentas que la invalidación del cache después del batch resuelve el problema a corto plazo. Si el sistema recibiera 100 batches por minuto y cada batch invalida el cache de `summary`, ¿cuántas veces ejecutaría DuckDB la query de agregación global en un minuto? ¿Cómo cambiarías el diseño para que múltiples batches en ráfaga no recalculen el summary en cada uno?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 21 / 25 |
| Architecture decision justificado | 20% | 20 / 20 |
| Cache con TTL y medición | 20% | 19 / 20 |
| Tests con pytest | 20% | 20 / 20 |
| Benchmark de latencia con cache | 15% | 13 / 15 |
| **Total** | **100%** | **93 / 100** |

---

El `architecture_decision.md` con el experimento negativo de SQLite, la invalidación del cache después del POST, y los tests con `valid_user_id` y validación de ordenamiento hacen esta la entrega más cuidada del grupo en diseño. El único gap concreto a corregir antes de llevar esto a producción es el SQL injection en el filtro de país.