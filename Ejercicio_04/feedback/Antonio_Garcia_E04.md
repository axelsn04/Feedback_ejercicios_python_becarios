# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumno:** Antonio Jair Garcia Vargas
**Fecha de revisión:** Mayo 2026
**Calificación:** 99 / 100

---

## Resumen general

La entrega más completa del módulo. Cinco decisiones la distinguen de todas las demás: `Literal` types en Pydantic para validación de dominios (la más elegante del grupo), `threading.Lock` en **ambos** backends, `cached: bool` en las respuestas analytics, `duplicate_ids: list[str]` en el batch response, y el único benchmark del grupo que cubre los 6 endpoints con análisis por endpoint. El `architecture_decision.md` es también el único que documenta explícitamente el trade-off de staleness analítico y qué cambio haría si el requisito fuera "analytics ve cada escritura al instante".

---

## Criterio 1 — 6 endpoints funcionales con datos reales `25 / 25`

**`Literal` types para validación de dominios** — el más elegante del grupo:
```python
Category = Literal["Food", "Travel", "Electronics", ...]
CountryCode = Literal["MX", "CO", "BR", ...]
Status = Literal["completed", "failed", "pending"]
```
Cualquier valor fuera del set es rechazado por Pydantic con 422 sin necesidad de `@field_validator`. `model_config = {"extra": "forbid"}` rechaza campos desconocidos. En conjunto, la capa de validación más robusta del grupo.

**`threading.Lock` en ambos backends** — `Backends` dataclass tiene `duck_lock` y `sqlite_lock` independientes. Cada función de query los usa explícitamente:
```python
with b.duck_lock:
    rows = b.duck.execute(sql, params).fetchall()
with b.sqlite_lock:
    b.sqlite.execute("BEGIN"); b.sqlite.executemany(...)
```
El resto del grupo protege uno o ninguno. Para sync endpoints en el threadpool de uvicorn, `threading.Lock` es la primitiva correcta — uvicorn corre los endpoints síncronos en threads, no en coroutines.

**`cached: bool` en las respuestas de analytics** — el endpoint retorna `{**value, "cached": True/False}`. El cliente puede distinguir si recibió un resultado fresco de DuckDB o un hit del cache. Usado en el test `test_summary_cache_hit` para validar `assert second["cached"] is True`.

**`duplicate_ids: list[str]` en `BatchResult`** — el batch response devuelve los IDs que fueron saltados como duplicados, no solo el conteo. Es el response más informativo del grupo para el cliente — permite al upstream saber exactamente qué falló sin hacer otra query.

**No hay SQL injection** — todas las queries de DuckDB usan `[country, limit]` como parámetros posicionales, no interpolación de strings.

---

## Criterio 2 — Architecture decision justificado `20 / 20`

El `architecture_decision.md` hace una cosa que ningún otro alumno hace: documenta el trade-off de staleness y lo defiende con razonamiento técnico.

> "POST /transactions/batch inserta en SQLite. /analytics/* lee del Parquet vía DuckDB. Son dos fuentes distintas, así que una inserción no se refleja en los agregados analíticos. Esto es deliberado, no un bug."

Y luego documenta la alternativa que existiría si el requisito fuera "analytics debe ver cada escritura al instante":

> "Si el requisito fuera 'analytics debe ver cada escritura al instante', la alternativa sería que DuckDB consultara la propia SQLite (sqlite_scan), sacrificando la ventaja columnar."

Eso demuestra comprensión del sistema completo, no solo del ejercicio.

El documento conecta los números de E2 (DuckDB para agregados) y E3 (SQLite 1275× más rápido en P1) con las decisiones de E4. El argumento de por qué `/users/*` no se cachea ("ya están en sub-milisegundo por los índices, cachear agregaría complejidad de invalidación sin ganancia de latencia") es el más preciso del grupo.

La nota sobre concurrencia y WAL es la más técnica: "el modo WAL (heredado del E3) lo permite sin bloquear al escritor, que es justo el motivo por el que E3 dejó la base en WAL pensando en este ejercicio." Eso cierra el loop entre ejercicios.

---

## Criterio 3 — Cache con TTL y medición `20 / 20`

**`threading.Lock`** protege el dict y los contadores ✅. **`invalidate_prefix()`** ✅. **`clear()`** ✅. **`time.monotonic()`** ✅. **`ttl <= 0: return`** guard ✅ — si alguien pasa TTL de 0, el set es no-op en lugar de guardar un entry que siempre expira.

**`stats` property con `entries` count** — el `/health` expone `cache_entries: int`, el número de entradas actuales en el store. Útil para detectar cache que crece sin límite.

**`get()` retorna tupla `(hit, value)`** en lugar de `Optional[value]` — el endpoint puede hacer `hit, value = cache.get(key)` y no necesita checar `if result is None` (que podría confundirse con un hit válido de `None`).

**`cached: bool = False` en los response models** — el field está en el modelo y tiene default False, así que un endpoint que no cachea puede retornar el modelo sin incluir el campo y el cliente siempre obtiene un valor explícito.

El impacto medido en el benchmark:
- `/analytics/summary`: cold p50 45.59ms → warm p50 0.599ms → **76×**
- `/analytics/top-merchants`: cold p50 19.23ms → warm p50 0.668ms → **29×**

---

## Criterio 4 — Tests con pytest `20 / 20`

18 tests — empata con Ulises como el máximo del grupo. Cobertura real en cada endpoint.

**`test_summary_happy` valida el contrato del dataset:**
```python
assert body["n_transactions"] == 1_000_000
assert len(body["by_country"])  == 15
assert len(body["by_category"]) == 10
```
Si el Parquet cambia, el test falla.

**`test_top_merchants_happy` valida ordenamiento:**
```python
totals = [m["total_amount"] for m in body["merchants"]]
assert totals == sorted(totals, reverse=True)
```

**`test_user_transactions_happy` valida ordenamiento temporal:**
```python
ts = [t["timestamp"] for t in body["transactions"]]
assert ts == sorted(ts, reverse=True)
```
Es el único test del grupo que valida que las transacciones del usuario están ordenadas por timestamp DESC, no solo que existen.

**`test_summary_cache_hit` usa el campo `cached: bool`:**
```python
assert second["cached"] is True
```
El test no solo verifica que la segunda llamada sea más rápida — verifica que viene del cache. Más preciso que una medición de tiempo.

**`test_batch_dedupe` más completo del grupo:**
1. Envía dos transacciones con el mismo ID en un batch → `inserted=1, dups=1`
2. Reenvía el mismo ID en un segundo batch → `inserted=0, dups=1`
Valida tanto dedup intra-lote como contra la base.

**`test_pagination_invalid_params_422`** — 3 casos: page=0, page_size=0, page_size=1000.

**`test_health_sla_latency`** con p95 calculado sobre 50 mediciones usando `statistics.quantiles`.

---

## Criterio 5 — Benchmark de latencia con cache `15 / 15`

El benchmark cubre los 6 endpoints — el único del grupo con cobertura completa:

| Endpoint | Cold p50 | Warm p50 | SLA |
|---|---:|---:|---|
| GET /analytics/summary | 45.59ms | 0.599ms | <500ms cold / <20ms warm ✅ |
| GET /analytics/top-merchants | 19.23ms | 0.668ms | <500ms cold / <20ms warm ✅ |
| GET /users/*/transactions | — | 0.667ms | <80ms ✅ |
| GET /users/*/stats | — | 0.625ms | <80ms ✅ |
| GET /health | — | 0.641ms | <50ms ✅ |

El análisis del outlier de p99 en `top-merchants` cold (270ms en primer request por compilación del plan de DuckDB) es el más técnicamente preciso del grupo: "las siguientes 99 caen a ~20ms (de ahí que p50 y p95 sean ~19–22ms pero el máximo dispare el p99)". La metodología explica por qué TestClient in-process mide una cota inferior del costo real en producción, y por qué eso es correcto para validar el ratio cold/warm.

La conclusión del reporte conecta el diseño con los números: "la separación de backends se ve reflejada en los números: DuckDB absorbe los escaneos analíticos pesados, SQLite resuelve los lookups puntuales, y ningún endpoint paga por el backend equivocado."

---

## Sobre el uso de herramientas de IA

Los `Literal` types, el `duplicate_ids` en el response, el `cached: bool` en el modelo, el test de ordenamiento de timestamps, y la documentación del `sqlite_scan` como alternativa son decisiones que requieren pensar el sistema completo. El `architecture_decision.md` que documenta el trade-off de staleness y lo defiende es el tipo de análisis que no se genera por inercia. Uso inteligente con comprensión real de todos los ejercicios anteriores.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 25 / 25 |
| Architecture decision justificado | 20% | 20 / 20 |
| Cache con TTL y medición | 20% | 20 / 20 |
| Tests con pytest | 20% | 20 / 20 |
| Benchmark de latencia con cache | 15% | 15 / 15 |
| **Total** | **100%** | **99 / 100** |

---

El `Literal` type pattern, los threading.Lock en ambos backends, `duplicate_ids` en el batch response, y el benchmark completo de 6 endpoints hacen esta la entrega más completa del módulo. El único punto que baja de 100 es que el script de benchmark no fue compartido para verificar la metodología exacta. Los resultados en el reporte son coherentes y el análisis es sólido.