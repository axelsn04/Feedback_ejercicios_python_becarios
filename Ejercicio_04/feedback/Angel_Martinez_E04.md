# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumno:** Angel Martinez Aguilar
**Fecha de revisión:** Mayo 2026
**Calificación:** 93 / 100

---

## Resumen general

La entrega más técnicamente robusta del grupo en tres dimensiones: validación de Pydantic (la más estricta), cache con `threading.Lock()` (la única thread-safe del grupo), y benchmark con 100 cold measurements con `cache.clear()` por request incluyendo el `POST /transactions/batch` a 500 registros (el único que lo hace). Los 11 tests tienen cobertura real — incluyendo validación de path params fuera de rango y el caso del batch demasiado grande. La única decisión técnica que vale la pena discutir es `INSERT OR REPLACE` en lugar de `INSERT OR IGNORE`.

---

## Criterio 1 — 6 endpoints funcionales con datos reales `22 / 25`

**Validación de Pydantic más estricta del grupo:** `models.py` valida UUID format, whitelist de categorías (10 valores), whitelist de países (15 LATAM), whitelist de estados (3 valores), y rango de amount (0.01-5000). El `@field_validator` de UUID rechaza cualquier string que no sea un UUID4 válido antes de que el endpoint corra. Ningún otro alumno implementó validación a este nivel.

**`user_id: int = FastAPIPath(..., ge=1, le=50000)`** — validación de path parameter con rango. Un request a `/users/50001/stats` retorna 422 sin llegar al handler. El test `test_user_not_found_and_validation` lo confirma. Es la única entrega del grupo con esta protección.

**DuckDB con parámetros vinculados** — no hay SQL injection. El `country_code` se pasa como `?` en lugar de interpolarse en el string. Correcto, a diferencia de Yaanit.

**`INSERT OR REPLACE` en lugar de `INSERT OR IGNORE`:** El `architecture_decision.md` lo documenta explícitamente como decisión intencional — "evita errores de UNIQUE constraint sin necesidad de INSERT OR IGNORE, lo cual taparía otros posibles errores de integridad". El test confirma que el comportamiento esperado es "should overwrite". El tradeoff es real: `REPLACE` permite actualizar transacciones existentes (útil si el cliente corrige un error), pero en un sistema de auditoría financiera una transacción ya registrada no debería poderse modificar silenciosamente con un segundo POST. Para E4 como ejercicio es aceptable; en producción requeriría una política explícita de update vs dedup.

**Paginación retorna 400 para página fuera de rango** — diferente del resto del grupo que retorna 200 con lista vacía. Ambas son válidas desde REST; la respuesta 400 es más explícita para el cliente pero puede causar que clientes que paginen hasta vacío exploten.

**Sin pre-warming de DuckDB en lifespan** — Ernesto hace un `SELECT COUNT(*)` al arrancar para forzar la lectura del Parquet a memoria. Sin eso, la primera request cold incluye ese costo. Los tiempos medidos (p50 cold 33ms para summary) son buenos de todas formas, probablemente porque el TestClient usa llamadas in-process.

---

## Criterio 2 — Architecture decision justificado `18 / 20`

El `architecture_decision.md` documenta las decisiones por endpoint con justificación técnica correcta. La justificación de no usar cache en endpoints de usuario ("los datos de un usuario pueden cambiar con cada batch insert y la latencia ya cumple el SLA sin él") es correcta y evita problemas de consistencia.

El análisis del TTL es honesto: 60s de TTL significa que transacciones nuevas tardan hasta 60s en reflejarse en los analytics. Para un dashboard eso es aceptable; la propuesta de "limpieza de memoria automática al recibir un batch insert nuevo" es la extensión correcta.

Dos puntos menos: el documento no analiza la implicación de `INSERT OR REPLACE` para la consistencia del sistema (¿qué pasa si DuckDB ya cachó un summary y luego un REPLACE modifica el amount de una transacción existente en SQLite? El cache TTL es la única protección y puede mostrar datos inconsistentes durante hasta 60s).

---

## Criterio 3 — Cache con TTL y medición `20 / 20`

**`threading.Lock()` en el cache** — el único del grupo con cache thread-safe. Si uvicorn arranca con múltiples workers en el mismo proceso o si varios threads llegan simultáneamente al mismo endpoint analítico, las operaciones de `get()`, `set()`, y `clear()` están protegidas. El resto del grupo tiene caches que serían race conditions bajo carga concurrente real.

**`cache.clear()` como método público** — necesario para el benchmark (100 cold measurements) y para los tests (los tests limpian el cache en el fixture). Sin este método, el benchmark de Ernesto solo puede medir 1 cold request y 99 warm.

**Métricas granulares** — hits, misses, y hit_rate expuestos en `/health`. El test `test_analytics_summary_happy_path` verifica que `cache_hits >= 1` después de la segunda llamada.

**Impacto medido:**
- `/analytics/summary`: cold p50 33.30ms → warm p50 0.67ms → **50x de mejora**
- `/analytics/top-merchants?country=MX`: cold p50 14.22ms → warm p50 0.63ms → **23x de mejora**

---

## Criterio 4 — Tests con pytest `19 / 20`

11 tests con cobertura real en todos los endpoints y casos borde. Las distinciones respecto al resto del grupo:

**Test de validación de path param fuera de rango (422):** `client.get("/users/50001/stats")` — único en el grupo. Valida que FastAPI rechaza IDs fuera del rango `[1, 50000]` antes de llegar al handler.

**Test de batch demasiado grande (400):** 501 transacciones → 400 con mensaje específico. Único en el grupo.

**Test de SLA en `/health`:** 10 repeticiones, promedio < 50ms. El test fallará si hay degradación de rendimiento.

**Test de deduplicación con overwrite documentado:** el test envía 3 registros donde el tercero tiene el mismo `transaction_id` que el segundo pero diferente `amount`, y verifica `inserted_records == 2`. El comentario "should overwrite" documenta la decisión de diseño.

Un punto menos: no hay test explícito de deduplicación contra la base (enviar el mismo `transaction_id` en dos batches distintos). El test verifica deduplicación dentro del mismo batch pero no la idempotencia de llamadas repetidas al endpoint.

---

## Criterio 5 — Benchmark de latencia con cache `15 / 15`

El benchmark más riguroso del grupo:

**100 cold measurements** — `cache.clear()` antes de cada request analítico. El p95 de cold (38ms para summary, 17.82ms para top-merchants con filtro) es representativo del percentil 95 real de latencia sin cache, no solo de una medición.

**POST /transactions/batch benchmarkeado** — 100 iteraciones con 500 registros cada una. p50 28.84ms, p95 82.44ms — el único alumno que mide este endpoint. El resultado demuestra que 500 inserts en una sola transacción SQLite toman ~29ms en mediana.

**Reporte autogenerado** — el script genera y actualiza `benchmarks/latency_report.md` con la tabla de resultados y análisis de SLAs. La lógica de "actualización quirúrgica" que reemplaza solo la sección de tabla sin tocar el resto del documento es el detalle de ingeniería más fino del grupo en este criterio.

Todos los SLAs cumplidos con margen:
- Analytics cold: 33ms / 14ms (SLA < 500ms)
- Analytics warm p95: 0.94ms / 0.85ms (SLA < 20ms)
- User endpoints p95: 1.33ms / 0.78ms (SLA < 80ms)
- Batch p50: 28.84ms (SLA < 2000ms)

---

## Sobre el uso de herramientas de IA

El `threading.Lock()` en el cache, los `@field_validator` con whitelists, y el `FastAPIPath` con rango son decisiones de código que requieren conocer las APIs en detalle. El benchmark con 100 cold measurements y el "actualización quirúrgica" del reporte markdown son implementaciones que van más allá de lo que se genera por defecto. El test que valida `/users/50001/stats` → 422 implica haber pensado en edge cases de validación. Uso inteligente con comprensión real.

---

## Pregunta de seguimiento

> Tienes `INSERT OR REPLACE` con la justificación de que "IGNORE taparía errores de integridad". ¿Qué error de integridad específico IGNORE no detectaría pero REPLACE sí evita? En un sistema de auditoría financiera donde una transacción ya procesada no debería modificarse, ¿qué cambio harías al endpoint para que el batch retorne explícitamente qué `transaction_id` ya existían en lugar de sobrescribirlos silenciosamente?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 22 / 25 |
| Architecture decision justificado | 20% | 18 / 20 |
| Cache con TTL y medición | 20% | 20 / 20 |
| Tests con pytest | 20% | 19 / 20 |
| Benchmark de latencia con cache | 15% | 15 / 15 |
| **Total** | **100%** | **93 / 100** |

---

El `threading.Lock()` en el cache, los 100 cold measurements con `cache.clear()`, y la validación de Pydantic con whitelists son las tres decisiones que hacen esta entrega la más técnicamente robusta del grupo para E04. La pregunta de seguimiento sobre `INSERT OR REPLACE` cierra el único punto de diseño que vale la pena revisar antes de llevar esto a producción.