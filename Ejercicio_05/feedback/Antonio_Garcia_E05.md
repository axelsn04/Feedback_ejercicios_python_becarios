# Retroalimentación — Ejercicio 05: El Backend con Estructura

**Alumno:** Antonio Garcia
**Fecha de revisión:** Mayo 2026
**Calificación:** 98 / 100

---

## Resumen general

La entrega más completa del grupo para E05. Cuatro decisiones la distinguen del resto: doble-checked locking en el singleton de DuckDB (el único alumno con la primitiva de thread-safety correcta para Django), `fields=["user_id", "-timestamp"]` con dirección descendente explícita en el índice compuesto (reproduce exactamente el comportamiento del E3), 18 tests con validación de contrato de datos real y fixtures determinísticos con valores verificables, y global `DEFAULT_PERMISSION_CLASSES = [IsAuthenticated]` con overrides explícitos a `AllowAny` en los endpoints públicos — el default más seguro del grupo. El único gap concreto: WAL mode ausente del settings.

---

## Criterio 1 — Modelos, migraciones e índices `24 / 25`

**`fields=["user_id", "-timestamp"]` — el único alumno con esto.**  
El índice compuesto para P2/P3/P4 está declarado con dirección descendente en `timestamp`:

```python
models.Index(
    fields=["user_id", "-timestamp"], name="idx_txns_user_timestamp"
)
```

Esto importa porque el patrón de acceso del E3 siempre ordena `ORDER BY timestamp DESC`. Un índice en `(user_id, timestamp ASC)` obliga a SQLite a recorrer el índice al revés para satisfacer `DESC` — incurre en un backward scan. Con la dirección correcta el scan es forward, que es lo que verifica el `EXPLAIN QUERY PLAN` del E3. El resto del grupo declaró el índice sin dirección, lo que genera `idx ASC` por defecto.

**`choices` en los tres campos de dominio fijo.** `CATEGORY_CHOICES`, `COUNTRY_CHOICES`, `STATUS_CHOICES` como constantes del módulo y pasados a `choices=`. Doble beneficio: el Django Admin muestra dropdowns en lugar de texto libre, y el ORM puede rechazar valores inválidos antes de llegar a SQLite. Los 10, 15 y 3 valores respectivos coinciden exactamente con el schema del módulo.

**`transaction_id = models.CharField(max_length=64, primary_key=True)` — PK natural.**  
No genera columna `id` autoincremental. La migración muestra `primary_key=True` sin columna `id` separada, lo que reproduce exactamente la constraint PRIMARY KEY del E3 sobre `transaction_id`. Esto también elimina el join implícito que Django haría si la PK fuera el autoincremental y `transaction_id` fuera un campo separado.

**`db_table = "transactions"` en `Meta`.**  
La tabla se llama igual que en el E3 (`transactions`), lo que es relevante si el E8 necesita usar ambas bases en paralelo sin renombrar tablas.

**La migración confirma todo lo anterior.** El `sqlmigrate` generará exactamente el `CREATE TABLE transactions (transaction_id VARCHAR(64) NOT NULL PRIMARY KEY, ...)` más los dos `CREATE INDEX` con los nombres originales del E3.

**Gap — WAL mode ausente.** `settings.py` no tiene la señal `connection_created` ni `OPTIONS: {'timeout': ...}` en `DATABASES`. El E3 demostró que WAL mejora el throughput de escrituras concurrentes. Para el `POST /transactions/batch`, que usa `atomic()`, esto es relevante bajo carga. La corrección es una línea en `settings.py`:

```python
from django.db.backends.signals import connection_created

def _set_wal(sender, connection, **kwargs):
    if connection.vendor == "sqlite":
        connection.cursor().execute("PRAGMA journal_mode=WAL;")

connection_created.connect(_set_wal)
```

---

## Criterio 2 — 6 endpoints funcionales con DRF `29 / 30`

**Double-checked locking en `analytics.py` — el único alumno con esto:**

```python
if _conn is None:
    with _lock:
        if _conn is None:   # segunda verificación dentro del lock
            conn = duckdb.connect(":memory:")
            ...
            _conn = conn
```

El patrón correcto para lazy initialization thread-safe. La verificación exterior evita el overhead del lock en todos los requests después de la primera inicialización. La verificación interior previene que dos threads que pasaron la verificación exterior simultáneamente initialicen la conexión dos veces. Sin la verificación interior, ambos threads ejecutarían `duckdb.connect()` y el segundo sobreescribiría `_conn` sin cerrar el primero — connection leak.

**`with _lock:` en todas las queries analíticas.** `summary()` y `top_merchants()` ambos adquieren el lock antes de ejecutar. Esto serializa los queries DuckDB, que es necesario porque `duckdb.DuckDBPyConnection` no es thread-safe para operaciones concurrentes sobre la misma conexión.

**`_positive_int()` helper centralizado:**

```python
def _positive_int(raw, default, *, name, lo, hi) -> int:
    ...
    raise ValidationError({name: f"debe estar entre {lo} y {hi}"})
```

Un único punto de validación para todos los query params enteros. El `raise ValidationError` fluye a través del `validation_to_422_handler` configurado en `REST_FRAMEWORK` → el test `test_users_transactions_pagination_invalid_422` pasa porque `page=0` llega a `_positive_int` con `lo=1` y levanta 422. Ningún otro alumno del grupo tiene validación de query params de paginación.

**404 en `UserTransactionsView` sin bug de página.** El `exists()` check está antes de cualquier paginación:

```python
if not Transaction.objects.filter(user_id=user_id).exists():
    return Response(..., status=404)
```

Esto corrije el bug de Ernesto donde `page=2` en un usuario inexistente retornaba 200 vacío. La comprobación ocurre independientemente del parámetro `page`.

**`authentication_classes: list = []` en `HealthView`** — va un paso más allá de `AllowAny`. `AllowAny` permite el acceso pero DRF aún intenta parsear el header `Authorization` si está presente, levantando potencialmente errores de token malformado. Vaciar `authentication_classes` elimina todo ese overhead — el `/health` no toca el token en ningún caso.

**`DEFAULT_PERMISSION_CLASSES = [IsAuthenticated]` globalmente.** Opuesto al enfoque de Ernesto (AllowAny global con overrides a IsAuthenticated). Este default es más seguro: cualquier endpoint nuevo que se agregue requiere autenticación a menos que explícitamente la deshabilite. Los endpoints públicos tienen `permission_classes = [AllowAny]` explícito — intención visible en el código.

**`db_transaction.atomic()` en batch insert.** Si falla algún insert dentro del bloque, Django hace rollback completo. Combinado con el `bulk_create()` (que agrupa todos los inserts en un solo statement), la atomicidad está garantizada.

**`status.HTTP_201_CREATED` en batch** — más semántico que 200. Los tests de Antonio ya esperan 201, lo que fuerza al implementador a elegir el código correcto.

**Punto menos — `_lock` serializa los reads de DuckDB.** `summary()` y `top_merchants()` compiten por el mismo lock. Si `summary()` tarda 300ms (cold DuckDB, sin cache explícito en esta implementación), todos los requests a `top_merchants()` esperan en cola. Para el ejercicio es correcto y seguro; en producción el siguiente paso sería un pool de conexiones DuckDB read-only o mover las queries a DuckDB's native multi-threading.

---

## Criterio 3 — Django Admin `20 / 20`

```python
list_display = (
    "transaction_id", "timestamp", "user_id", "merchant_id",
    "amount", "category", "country_code", "status",
)
list_filter = ("status", "country_code", "category")  # category es bonus
search_fields = ("transaction_id", "user_id")
ordering = ("-timestamp",)
list_per_page = 50
```

`list_per_page = 50` resuelve el problema de performance que tiene un admin sin este parámetro sobre 1M registros. Sin él Django intenta cargar 100 filas y aplica el `ordering = ("-timestamp",)` sobre toda la tabla — potencialmente sin índice útil en esa dirección dependiendo del motor. Con 50, el costo por página es controlado.

`list_filter` incluye `category` como tercer filtro, no requerido por la rúbrica pero natural para el caso de uso del E8 (monitoreo de transacciones por categoría).

`search_fields = ("transaction_id", "user_id")` — Django aplica LIKE en ambos campos. Para `user_id` (IntegerField) la búsqueda es `WHERE user_id::text LIKE '%query%'` que no usa índice. Si el caso de uso fuera búsqueda de usuarios en producción, se usaría `=user_id` para exact match. Para el ejercicio es aceptable.

---

## Criterio 4 — Tests `25 / 25`

18 tests. El conjunto más completo y técnicamente correcto del grupo.

**Contrato de datos real en analytics:**
```python
assert body["n_transactions"] == 1_000_000
assert len(body["by_country"]) == 15
assert len(body["by_category"]) == 10
```
No solo verifica que los campos existen — verifica que el Parquet del E1 se leyó correctamente. Si DuckDB apunta al archivo equivocado o el Parquet está corrupto, este test falla antes de llegar a producción.

**Validación de ordering:**
```python
assert body["transactions"][0]["transaction_id"] == "sample-2"
```
El fixture inserta `timestamp=datetime(2026, 1, 10+i)` para `i in range(3)`. `sample-2` tiene `2026-01-12`, la más reciente. Este assertion verifica que `ORDER BY -timestamp` funciona — no solo que el endpoint responde.

**Fixture determinístico con aritmética verificable:**
```python
# conftest.py
amount=10.0 * (i + 1)  # 10, 20, 30
category="Food" if i < 2 else "Travel"  # Food×2, Travel×1
```
```python
# test
assert body["total_amount"] == 60.0   # 10 + 20 + 30
assert body["most_frequent_category"] == "Food"  # 2 > 1
```
Los valores del test se derivan directamente del fixture — no son números mágicos. Si el fixture cambia, los assertions cambian predeciblemente.

**Deduplicación en dos fases con verificación de `duplicate_ids`:**
```python
def test_batch_dedupe_intra_lote(auth_client):
    txn = _valid_txn()
    payload = {"transactions": [txn, dict(txn)]}
    r = auth_client.post(...)
    assert body["inserted"] == 1
    assert body["duplicates_skipped"] == 1
    assert txn["transaction_id"] in body["duplicate_ids"]

def test_batch_dedupe_contra_base(auth_client):
    txn = _valid_txn()
    auth_client.post(...)           # primer insert
    r = auth_client.post(...)       # segundo insert
    assert body["inserted"] == 0
    assert body["duplicates_skipped"] == 1
```
Dos tests independientes para dos mecanismos de deduplicación distintos. El de `_intra_lote` verifica que el mismo `transaction_id` enviado dos veces en un batch cuenta una sola vez. El de `_contra_base` verifica idempotencia de la API: enviar el mismo payload dos veces no duplica datos.

**`page=0 → 422`** — el único test del grupo que valida límites de parámetros de paginación. `_positive_int` con `lo=1` rechaza 0 con 422 antes de llegar a la query del ORM.

**`test_batch_extra_field_422`** — serializer estricto que rechaza campos no declarados. Protege contra inyección de campos no esperados en el schema.

---

## Sobre el uso de herramientas de IA

El double-checked locking en `analytics.py`, el índice con `-timestamp`, el `_positive_int` helper con rangos, los 18 tests con fixtures cuya aritmética está verificada, y el `authentication_classes: list = []` en `/health` son decisiones que requieren comprensión de por qué importan. El comment en `analytics.py` que explica los dos niveles del locking ("la verificación exterior evita el lock overhead; la interior previene doble inicialización") y el comment en `models.py` que conecta cada índice con el patrón de acceso del E3 son consistentes con comprensión genuina del sistema.

---

## Pregunta de seguimiento

> `analytics.py` adquiere `_lock` durante la ejecución completa de cada query — `summary()` y `top_merchants()` comparten el mismo lock. Si llegan 10 requests concurrentes a `/analytics/*`, todos se serializan aunque DuckDB podría ejecutarlos en paralelo. ¿Cómo cambiarías `analytics.py` para permitir reads concurrentes sin abandonar la thread-safety? Considera que `duckdb.connect(":memory:")` crea una base que no puede compartirse entre threads — y que DuckDB tiene un modo de conexión específico para este caso.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Modelos, migraciones e índices | 25% | 24 / 25 |
| 6 endpoints funcionales con DRF | 30% | 29 / 30 |
| Django Admin configurado | 20% | 20 / 20 |
| Tests (mínimo 6) | 25% | 25 / 25 |
| **Total** | **100%** | **98 / 100** |

---

WAL mode en settings es el único gap concreto — una línea que conecta el `connection_created` signal con los PRAGMAs del E3. El lock que serializa los reads analíticos es correcto para este contexto pero la pregunta de seguimiento apunta al paso de producción que lo resuelve.