# Retroalimentación — Ejercicio 05: El Backend con Estructura

**Alumno:** Angel Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 88 / 100

---

## Resumen general

La entrega más arquitectónicamente completa del grupo después de Antonio. Cinco
decisiones la distinguen: `InMemoryCache` con `threading.Lock` en todas las
operaciones (incluido `metrics`), `ActiveRequestTracker` con contador de
requests activos expuesto en `/health`, real streaming de Parquet por row group
(no carga todo a memoria), `custom_exception_handler` → 422 registrado en
`settings.py`, y un test de latencia SLA sobre `/health` que ningún otro alumno
tiene. Los gaps concretos son dos: `update_or_create()` en loop para el batch
(N queries donde `bulk_create` haría 1), y el cache no se invalida después de
un batch insert — los endpoints analíticos sirven datos stale hasta que expira
el TTL.

---

## Criterio 1 — Modelos, migraciones e índices `22 / 25`

**Schema correcto en todos los tipos:**
- `user_id = IntegerField()`, `merchant_id = IntegerField()` ✓
- `transaction_id = CharField(max_length=36)` — UUID como string, correcto
- `db_table = 'transactions'` ✓
- `idx_user_timestamp` y `idx_country_user` con los nombres exactos del E3 y
  `fields=['user_id', '-timestamp']` con dirección descendente ✓

**`load_transactions` con streaming real por row group:**
```python
pf = pq.ParquetFile(parquet_path)
for i in range(pf.num_row_groups):
    table = pf.read_row_group(i)    # lee un grupo a la vez, no todo el archivo
    df = table.to_pandas()
```
`pq.ParquetFile.read_row_group()` lee un row group a la vez desde disco — no
carga el Parquet completo a memoria como hace Yaanit con `pd.read_parquet()`.
Para 1M filas esto es la diferencia entre usar ~500MB de RAM vs usar solo lo
que cabe en un row group típico (~10-50MB).

**Manejo de zona horaria con `USE_TZ = True`:**
```python
if df['timestamp'].dt.tz is None:
    df['timestamp'] = df['timestamp'].dt.tz_localize('UTC')
```
El Parquet del E1 tiene timestamps naive. Django con `USE_TZ=True` requiere
timestamps timezone-aware en los campos `DateTimeField`. Sin este paso, el ORM
lanza `RuntimeWarning` y potencialmente guarda datos incorrectos. Es el único
alumno que maneja este caso explícitamente.

**`itertuples()` es mejor que `iterrows()` pero no es óptimo.** Para 1M filas,
`itertuples()` es más rápido que `iterrows()` porque devuelve namedtuples en
lugar de objetos `Series`. Pero `df.to_dict('records')` seguido de list
comprehension sería más rápido aún — elimina el overhead de namedtuples y
accede directamente a dicts nativos de Python.

**WAL mode ausente.** `settings.py` no configura `connection_created`. La base
apunta a `../data/transactions.db` (la del E3), lo que es una decisión de
diseño válida para evitar duplicar 1M filas, pero el WAL del E3 no se
propaga automáticamente a las nuevas conexiones del E5.

**Sin `choices=` en los campos de dominio** — validación delegada completamente
al serializer.

---

## Criterio 2 — 6 endpoints funcionales con DRF `25 / 30`

**`InMemoryCache` thread-safe — el mejor del grupo:**
```python
def get(self, key):
    with self._lock:
        if key in self._cache:
            value, expire_at = self._cache[key]
            if time.time() < expire_at:
                self._hits += 1
                return value
            else:
                del self._cache[key]    # limpia la entrada expirada en el read
        self._misses += 1
        return None
```
Lock en `get`, `set`, `clear` y `metrics`. Limpia entradas expiradas en el
momento del read (en lugar de en un job separado). La propiedad `metrics`
también adquiere el lock — garantiza lectura consistente de `_hits` y
`_misses` bajo concurrencia.

**`ActiveRequestTracker` — único en el grupo:**
```python
class ActiveRequestTracker:
    def __enter__(self):
        with _active_requests_lock:
            _active_requests += 1
    def __exit__(self, ...):
        with _active_requests_lock:
            _active_requests -= 1
```
Cada endpoint lo envuelve con `with ActiveRequestTracker():`. El `/health`
expone `connections_active`. Esto permite observabilidad de concurrencia en
tiempo real — útil para detectar bottlenecks bajo carga.

**`/health` — el más completo del grupo:**
```python
"status": "healthy" if (sqlite_ok and duckdb_ok) else "degraded"
```
Verifica ambas conexiones activamente, reporta `cache_hit_rate`, `connections_active`
y `uptime_seconds`. "healthy" vs "degraded" es más informativo que solo "ok".

**DuckDB con double-checked locking y cursor por query:**
```python
if _duckdb_conn is None:
    with _duckdb_lock:
        if _duckdb_conn is None:   # segunda verificación
            _duckdb_conn = duckdb.connect(...)
return _duckdb_conn.cursor()
```
Retorna un nuevo `cursor()` por query desde la conexión compartida. Los
cursores DuckDB son independientes entre sí — esta es la forma correcta de
usar una conexión compartida en multi-thread, más limpia que el `with _lock:`
durante toda la query que usa Antonio.

**`user_id` validado en rango en la URL — único en el grupo:**
```python
if not (1 <= user_id <= 50000):
    raise ValidationError({"user_id": "User ID must be between 1 and 50000"})
```
Validación de rango en la URL antes de tocar el ORM, con 422 a través del
`custom_exception_handler`. Complementa la validación del serializer para
el batch.

**Paginación con `total_pages` y detección de out-of-range → 400:**
```python
if page > total_pages:
    return Response(
        {"detail": f"Page {page} is out of range. Total pages: {total_pages}."},
        status=HTTP_400_BAD_REQUEST
    )
```
El mensaje incluye el `total_pages` real — el cliente puede corregir sin
reintentar a ciegas.

**Gap principal — `update_or_create()` en loop para el batch:**
```python
for tx_data in unique_txs:
    Transaction.objects.update_or_create(
        transaction_id=tx_data['transaction_id'],
        defaults={...}
    )
```
Para 500 transacciones únicas, esto ejecuta 500 `SELECT + INSERT/UPDATE`
individuales. `bulk_create(ignore_conflicts=True)` haría un solo `INSERT` con
todos los valores. El overhead es real: 500 round-trips vs 1.

El comportamiento funcional es "upsert" (el test lo verifica con un ID enviado
dos veces con diferente `amount` — la segunda sobreescribe a la primera), lo
que es más potente que `bulk_create(ignore_conflicts=True)` que descarta el
duplicado. Pero si el requisito es "no duplicar", `bulk_create` es suficiente y
mucho más eficiente.

**Gap secundario — el cache no se invalida después de batch insert.** La vista
de batch no llama a `cache.clear()` ni a ningún `invalidate_prefix`. Si alguien
inserta 500 transacciones, `/analytics/summary` sigue sirviendo datos stale
hasta que expira el TTL. Antonio resolvía esto con `cache.invalidate_prefix('analytics:')`.

**Serializer completo — validaciones de todos los campos de dominio:**
```python
def validate_user_id(self, value):
    if not (1 <= value <= 50000): raise ...

def validate_merchant_id(self, value):
    if not (1 <= value <= 10000): raise ...

def validate_amount(self, value):
    if not (0.01 <= value <= 5000.00): raise ...

def validate_category(self, value):
    if value not in CATEGORIES: raise ...

def validate_country_code(self, value):
    if value not in COUNTRIES: raise ...

def validate_status(self, value):
    if value not in STATUSES: raise ...
```
El serializer más completo del grupo — todos los campos del schema tienen
reglas de negocio explícitas. El `test_post_batch_invalid_schema` lo
confirma enviando todas las validaciones en una sola transacción.

---

## Criterio 3 — Django Admin `18 / 20`

```python
list_display = (
    'transaction_id', 'timestamp', 'user_id', 'merchant_id',
    'amount', 'category', 'country_code', 'status',
)
list_filter  = ('status', 'country_code')
search_fields = ('transaction_id', 'user_id')
ordering = ('-timestamp',)
```

Todas las 8 columnas en `list_display` ✓, filtros requeridos ✓, búsqueda
requerida ✓, ordenamiento ✓. El admin está completo y navegable.

Un punto menos por `list_per_page` ausente — Django carga 100 filas por página
por default, lento para un modelo con 1M registros. Un `list_per_page = 50`
reduce el costo de cada carga de lista a la mitad. Es una línea que hace
diferencia observable.

Un punto menos por `readonly_fields` ausente — `transaction_id` es la PK de
negocio y no debería ser editable desde el admin.

---

## Criterio 4 — Tests `23 / 25`

11 tests. El segundo conjunto más sólido del grupo después de Antonio.

**`test_analytics_summary_happy_path` — verifica caching de forma indirecta:**
```python
# Primera llamada (cold)
response1 = client.get("/analytics/summary")
# Segunda llamada (warm — debería usar cache)
response2 = client.get("/analytics/summary")
assert data2 == data1

# Verificar cache_hits via /health
health_response = client.get("/health")
assert health_data["cache_hits"] >= 1
```
El test valida que el cache funciona sin acceder directamente a los internos —
observa el efecto externo a través del endpoint de salud. Elegante.

**`test_user_stats_happy_path` — contrato de datos con explicación del
tiebreaker:**
```python
assert data["country_code"] == "CO"  # CO y MX ambas freq=1, "CO" < "MX" alfabéticamente
```
El comment explica por qué `"CO"` y no `"MX"`. Esto confirma que el estudiante
entendió el `.order_by('-freq', 'country_code')` del view — el tiebreaker
alfabético es predecible y el test lo codifica explícitamente.

**`test_post_batch_happy_path` — valida el estado de la DB, no solo el response:**
```python
# El tercer item repite tx_id2 con amount diferente — prueba upsert
# Response: inserted_records=2 (únicos en el batch)
stats_resp = auth_client.get("/users/42/stats")
assert stats["transaction_count"] == 2
```
Después de la inserción, consulta `/users/42/stats` para verificar que la base
quedó con exactamente 2 registros. Test de estado final, no solo de respuesta.

**`test_health_sla_latency` — el único test de SLA del grupo:**
```python
for _ in range(10):
    start = time.perf_counter()
    response = client.get("/health")
    duration = (time.perf_counter() - start) * 1000
    latencies.append(duration)
avg_latency = sum(latencies) / len(latencies)
assert avg_latency < 50.0
```
10 requests, latencia promedio < 50ms. Metodológicamente correcto para el SLA
de un endpoint que no debe tocar backends pesados.

**`test_user_not_found_and_validation` — dos reglas en un test:**
404 para usuario sin transacciones Y 422 para `user_id=50001` (fuera de rango).
Verifica tanto la regla de negocio del ORM como la validación de rango del view.

**Gaps:**
Sin test de 401 para `POST /transactions/batch` — está para `/users/*` pero
no para el batch. Sin skip logic para los tests de analytics cuando el Parquet
no está disponible.

---

## Sobre el uso de herramientas de IA

El `ActiveRequestTracker` como context manager con su propio lock, el
`InMemoryCache` con lock en `metrics`, el manejo de timezone en
`load_transactions`, y el test que verifica el cache a través de los
`cache_hits` del health endpoint son decisiones que requieren comprensión
de concurrencia y diseño de observabilidad. El `test_user_stats_happy_path`
con el comment explicando el tiebreaker alfabético confirma que se entendió el
`.order_by('-freq', 'country_code')` del view. El gap del `update_or_create` en
loop es el tipo de decisión que funciona correctamente pero que un benchmark
revelaría rápidamente.

---

## Pregunta de seguimiento

> El batch usa `update_or_create()` en loop (N queries para N transacciones) y
> no invalida el cache de analytics después de insertar. Dos preguntas:
> (1) Reemplaza el loop por `bulk_create(ignore_conflicts=True)` y mide la
> diferencia de tiempo para un batch de 500 transacciones. ¿Qué trade-off
> pierdes al cambiar de `update_or_create` a `bulk_create`?
> (2) Agrega `cache.clear()` después de una inserción exitosa. ¿Es `clear()`
> la función correcta aquí, o necesitarías algo más granular? Considera qué
> pasa si un request llega a `/analytics/summary` justo mientras se está
> procesando el batch.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Modelos, migraciones e índices | 25% | 22 / 25 |
| 6 endpoints funcionales con DRF | 30% | 25 / 30 |
| Django Admin configurado | 20% | 18 / 20 |
| Tests (mínimo 6) | 25% | 23 / 25 |
| **Total** | **100%** | **88 / 100** |

---

El `InMemoryCache` thread-safe, el `ActiveRequestTracker`, y el test de
latencia SLA demuestran un nivel de atención a la observabilidad y concurrencia
que va más allá de hacer pasar la rúbrica. Los dos gaps concretos a corregir
antes de E06: `bulk_create` en lugar de `update_or_create` en loop, e
invalidación del cache después de batch insert.