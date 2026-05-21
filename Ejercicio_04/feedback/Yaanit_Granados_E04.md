# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumna:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 82 / 100

---

## Resumen general

El ejercicio está completo: los 6 endpoints funcionan, el lifespan está bien implementado, el cache tiene TTL con métricas reales, hay 9 tests que validan contenido y no solo status codes, y el benchmark demuestra el impacto del cache con números concretos. Hay un bug de seguridad en el endpoint de top-merchants y el benchmark solo cubre 2 de los 6 endpoints. Son los dos puntos que bajan la calificación de forma más significativa.

---

## Criterio 1 — 6 endpoints funcionales con datos reales `21 / 25`

Los 6 endpoints están implementados y el patrón de conexiones es correcto: inicialización en el lifespan, inyección vía `Depends()`, sin conexiones dentro de los endpoints. El `INSERT OR IGNORE` + `db.total_changes` para contar insertos reales es una solución limpia y no obvia.

**Bug crítico — SQL injection en `/analytics/top-merchants`:**

```python
if country:
    query += f" WHERE country_code = '{country}'"
```

La interpolación de string directamente en la query es una vulnerabilidad de SQL injection. Un request con `country=MX' OR '1'='1` modifica la query sin ningún control. La forma correcta es parámetros vinculados:

```python
# Correcto
if country:
    query += " WHERE country_code = ?"
    result = db.execute(query + f" GROUP BY ...", [country])
```

Con DuckDB y la query actual hay un problema adicional: la query construye `SELECT ... FROM gold_transactions WHERE country_code = 'MX' GROUP BY ...` pero el `GROUP BY` y `ORDER BY` están fuera del condicional, lo que significa que si no hay `country`, la query termina sin `WHERE` y con un `GROUP BY` que viene de una concatenación. Revisar que el string final sea válido SQL en todos los casos.

**`user_id` como `str` en SQLite:** el endpoint recibe `user_id: str` del path y lo pasa como string al parámetro de SQLite. La base del E3 almacena `user_id` como `INTEGER`. SQLite tiene afinidad de tipos y generalmente hace la coerción, pero es frágil — si el índice `idx_user_timestamp` está sobre un INTEGER y se consulta con un string, el planner puede no usar el índice. Los tests usan `user_id=10` (string "10" vs integer 10) y pasan, pero es un riesgo que vale corregir.

---

## Criterio 2 — Architecture decision justificado `17 / 20`

El `architecture_decision.md` cubre los cuatro puntos del ejercicio: lifespan, separación de backends por endpoint, cache TTL, y pipeline de ingesta. Las justificaciones son correctas y específicas: DuckDB para scans analíticos sobre Parquet, SQLite con B-tree para point lookups, `INSERT OR IGNORE` para deduplicación a nivel de base sin leer primero.

El punto de la justificación más sólido es el del lifespan: "abrir una conexión incurre en un costo de I/O de 50-150ms, lo que hace matemáticamente imposible cumplir los SLAs". Es correcto y el benchmark lo confirma.

Tres puntos menos: el README menciona variables de entorno configurables (SQLITE_DB_PATH, PARQUET_FILE_PATH, etc.) pero `db.py` tiene los paths hardcodeados sin leer ninguna variable de entorno. Lo que el README promete y lo que el código hace no coinciden — el mismo patrón de E01 y E02 donde el reporte describe algo que el código no implementa.

---

## Criterio 3 — Cache con TTL y medición `18 / 20`

El cache es correcto. TTL con eager eviction (elimina la clave cuando expira, no en el siguiente acceso), hit_rate calculado desde el inicio del servidor, uptime en segundos, y el singleton compartido via `Depends()`. Los TTLs son diferenciados por endpoint: 60s para summary, 30s para top-merchants — eso es una decisión de diseño real.

El impacto del cache está demostrado con números concretos:
- `/analytics/summary`: cold 62ms → warm p95 3.32ms
- `/analytics/top-merchants`: cold 375ms → warm p95 3.60ms

Dos puntos menos: el cache no es thread-safe. Si uvicorn arrancara con múltiples workers (`--workers 4`), cada worker tendría su propio `cache_engine` singleton en memoria sin compartir estado. Para un solo worker es correcto, pero la documentación no menciona esta limitación.

---

## Criterio 4 — Tests con pytest `18 / 20`

9 tests, todos validan contenido y no solo status code. La cobertura es buena:
- Happy path para los 6 endpoints ✅
- 404 para usuario inexistente ✅
- 422 para batch con amount negativo ✅
- Paginación fuera de rango (200 con lista vacía) ✅
- Deduplicación: primer insert = 1, segundo insert = 0 ✅
- Test de cache con timing (cold < 600ms, warm < 50ms) ✅

El test de deduplicación usa `time.time()` para generar un `transaction_id` único, lo que es correcto para garantizar que la primera inserción siempre sea nueva.

Dos puntos menos: el test de timing del cache (`test_02`) tiene un margen de 600ms para cold y 50ms para warm — son márgenes muy holgados que hacen el test casi imposible de fallar. Un test de SLA más ajustado (ej. cold < 500ms, warm < 20ms como dice la spec) haría el test más útil como guardia de regresión.

---

## Criterio 5 — Benchmark de latencia con cache `10 / 15`

El benchmark cubre `GET /analytics/summary` y `GET /analytics/top-merchants` con cold request + 100 warm requests, reportando p50/p95/p99. Los números demuestran el impacto del cache claramente.

El problema es que el benchmark **no cubre los 4 endpoints restantes**: `/health`, `/users/{user_id}/transactions`, `/users/{user_id}/stats`, y `POST /transactions/batch`. El ejercicio pide benchmark de latencia end-to-end del sistema — incluyendo los endpoints de SQLite que deberían demostrar que responden bajo su SLA de 80ms. Sin esos números, no hay evidencia de que la capa OLTP cumple sus SLAs.

Cinco puntos menos por esa cobertura incompleta.

---

## Sobre el uso de herramientas de IA

El código es funcional y estructurado. El bug de SQL injection es característico de código generado sin revisión de seguridad — los modelos de lenguaje frecuentemente generan f-strings en queries SQL sin señalar el riesgo. La inconsistencia entre las variables de entorno documentadas en el README y el código hardcodeado en `db.py` sigue el patrón de entregas anteriores. Los tests tienen voz propia en sus docstrings ("Pydantic guardians", "crushing strict latency SLAs") que es más AI que humano.

---

## Pregunta de seguimiento

> Tienes un SQL injection en `/analytics/top-merchants`. Corrígelo usando parámetros vinculados en DuckDB. ¿DuckDB acepta `?` como placeholder como SQLite, o usa una sintaxis diferente? Prueba que el fix no rompe los tests existentes.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 21 / 25 |
| Architecture decision justificado | 20% | 17 / 20 |
| Cache con TTL y medición | 20% | 18 / 20 |
| Tests con pytest | 20% | 18 / 20 |
| Benchmark de latencia con cache | 15% | 10 / 15 |
| **Total** | **100%** | **82 / 100** |

---

El sistema funciona end-to-end y el cache demuestra su impacto con números reales. El SQL injection y el benchmark incompleto son los dos puntos más concretos para corregir antes de un proyecto de producción.