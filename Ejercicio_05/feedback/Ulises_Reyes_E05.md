# Retroalimentación — Ejercicio 05: El Backend con Estructura

**Alumno:** Ulises Reyes
**Fecha de revisión:** Mayo 2026
**Calificación:** 83 / 100

---

## Resumen general

La entrega más reflexiva del grupo en E05: el `decisions.md` voluntario —
pedido hasta el E8 — documenta 8 decisiones con lo que se eligió, lo que se
descartó, y en varios casos una crítica honesta del propio costo de la decisión
tomada. Eso refleja un nivel de comprensión genuina que va más allá de hacer
que el código corra. La arquitectura dual DuckDB/ORM está correctamente
replicada del E4, los índices tienen los nombres exactos del E3 con `-timestamp`
descendente, y el admin es el más elaborado del grupo. Los tres gaps concretos
son: el singleton de DuckDB no tiene lock durante la ejecución de queries
(solo en la inicialización), el batch no deduplica dentro del mismo lote (lo que
produce un conteo incorrecto en `inserted`), y los tests verifican estructura
pero no contrato de datos.

---

## Criterio 1 — Modelos, migraciones e índices `21 / 25`

**Índices con `-timestamp` descendente y nombres exactos del E3:**
```python
models.Index(fields=["user_id", "-timestamp"], name="idx_user_timestamp"),
models.Index(fields=["country_code", "user_id"], name="idx_country_user"),
```
La migración generada confirma ambos. `idx_user_timestamp` con dirección
descendente es el detalle que el resto del grupo no tiene — el índice sirve
los patrones P2/P3/P4 del E3 con un forward scan en lugar de backward.

**`timestamp = models.CharField(max_length=26)` — decisión documentada con
consecuencias concretas.** El `decisions.md` explica el razonamiento correctamente:
el orden lexicográfico de ISO8601 coincide con el cronológico, así que los
índices y los `ORDER BY` sobre strings funcionan igual. La crítica honesta
también está documentada: "Django no puede hacer queries de tipo
`timestamp__year=2025`". Lo que el documento no menciona es el acoplamiento
que crea en el batch view:
```python
timestamp = t["timestamp"].strftime("%Y-%m-%d %H:%M:%S"),
```
El serializer parsea el timestamp a `datetime` y la view lo reconvierte a string
para guardarlo como `CharField`. Ese ciclo `string → datetime → string` es
frágil: si el serializer cambia el tipo del campo, el `strftime` lanza
`AttributeError` silenciosamente en producción. La alternativa — `DateTimeField`
con `USE_TZ=False` y el comentario de por qué — habría evitado el acoplamiento
sin sacrificar compatibilidad con los datos del E1/E3.

**Sin `choices=` en `category`, `country_code`, `status`.** La migración
confirma que estos campos son `CharField` simples sin constrained values. Las
consecuencias son dos: primero, el Django Admin muestra text inputs en lugar de
dropdowns en el formulario de detalle — se pierde la restricción visual del
dominio. Segundo, el ORM acepta cualquier string en estos campos; la validación
queda exclusivamente en el serializer. Esto funciona, pero el modelo no refleja
las invariantes del negocio.

**WAL mode — ausente.** `settings.py` no fue entregado, pero la decisión
técnica no aparece en `decisions.md` (que documenta 8 decisiones otras). El E3
demostró que WAL mejora el throughput de escrituras concurrentes; para el batch
POST con `atomic()`, su ausencia tiene costo medible bajo carga.

**`max_length=36` en `transaction_id` — más preciso que 64.** Los UUIDs v4
tienen exactamente 36 caracteres con guiones. `max_length=64` (Antonio) es
correcto pero genérico; 36 codifica la invariante de que el campo es UUID.

---

## Criterio 2 — 6 endpoints funcionales con DRF `23 / 30`

**Singleton de DuckDB con double-checked locking — correcto para la
inicialización:**
```python
# services/duckdb.py
if _connection is not None:
    return _connection

with _lock:
    if _connection is not None:   # segunda verificación
        return _connection
    conn = duckdb.connect(database=":memory:")
    ...
    _connection = conn
```
El patrón previene doble inicialización en el primer acceso concurrente. El
`reset_connection()` para tests es una adición útil que ningún otro alumno
incluyó.

**Gap — el lock no protege la ejecución de queries.** `services/duckdb.py`
protege solo la inicialización. Las views llaman `get_duckdb_connection()` y
luego ejecutan queries directamente:
```python
conn = get_duckdb_connection()
totals = conn.execute("SELECT COUNT(*)...").fetchone()  # sin lock
by_country = conn.execute("SELECT ...").fetchall()      # sin lock
```
Con Django's threaded server, dos requests simultáneos a `/analytics/*` pueden
ejecutar queries concurrentes sobre la misma conexión `:memory:`. DuckDB's
in-memory connections no garantizan thread-safety para acceso concurrente. La
versión correcta es adquirir el lock durante la ejecución, como hace el
`_lock` del E4 o como Antonio lo implementó:
```python
with _lock:
    rows = conn.execute(sql, params).fetchall()
```

**404 en `UserTransactionsView` correcto en todas las páginas:**
```python
if not Transaction.objects.filter(user_id=user_id).exists():
    raise NotFound(...)
```
Esto ocurre antes de la paginación, sin la condición `if page == 1` que
causaba el bug de Ernesto. `/users/9999999/transactions?page=2` retorna 404.

**`PageNumberPagination` de DRF — idiomático.** Usar el paginador nativo de DRF
(`paginator.paginate_queryset`) produce el formato estándar `{count, next,
previous, results}` y maneja automáticamente `page_size_query_param` y
`max_page_size`. Diferente al enfoque manual de Antonio (calcula offset
explícitamente) pero igualmente correcto — y más integrado con el ecosistema DRF.

**Bug en el batch — intra-lote no deduplicado, conteo incorrecto:**

```python
ids = [t["transaction_id"] for t in validated]          # puede tener duplicados
existing = set(Transaction.objects.filter(...))
new_transactions = [t for t in validated if t["transaction_id"] not in existing]
duplicates = received - len(new_transactions)
```

Si el batch contiene `[A, B, A]` donde A y B son nuevos:
- `ids = [A, B, A]`
- `existing = {}` (ninguno en DB)
- `new_transactions = [A, B, A]` ← 3 elementos, A aparece dos veces
- `duplicates = 0`
- `bulk_create([A, B, A], ignore_conflicts=True)` → inserta A y B, ignora el segundo A
- **Response: `inserted=3` — incorrecto. Solo se insertaron 2.**

El test `test_batch_deduplication` no detecta esto porque prueba dos requests
secuenciales (primer insert, luego el mismo ID de nuevo), no un solo batch con
IDs repetidos. El comportamiento del DB es correcto; el conteo reportado no lo es.

**`HealthView` no verifica backends.** Retorna `"status": "ok"` basándose solo
en `time.monotonic()`. Si DuckDB falla al abrir el Parquet o SQLite está
corrupto, `/health` sigue reportando `"ok"`. Para cumplir el propósito del
endpoint de salud del E8, debería verificar al menos que las conexiones
responden.

**Batch sin `atomic()`.** Si el `bulk_create` falla a mitad de un lote grande,
no hay rollback. Las filas insertadas antes del fallo permanecen. Para 500
transacciones esto puede dejar un estado parcial en la base.

---

## Criterio 3 — Django Admin `19 / 20`

El admin más detallado del grupo:

```python
list_display = (
    "transaction_id_short",   # método custom: muestra los primeros 12 chars
    "user_id", "amount", "category", "country_code", "status", "timestamp",
)
list_filter     = ("status", "country_code")
search_fields   = ("transaction_id", "user_id")
ordering        = ("-timestamp",)
list_per_page   = 50
readonly_fields = ("transaction_id",)
fieldsets       = (...)
```

`fieldsets` organiza el formulario de detalle en cuatro secciones semánticas
(Identificación, Partes, Transacción, Geografía) — ningún otro alumno lo hizo.
`readonly_fields` protege la PK de negocio de edición accidental.

El método `transaction_id_short` muestra `{id[:12]}...` en la lista — mejora
la densidad visual de la tabla para UUIDs largos. El UUID completo es accesible
en el formulario de detalle.

El comment sobre el comportamiento de `search_fields` con LIKE y el potencial
de lentitud sin índice de texto es el tipo de nota que distingue a alguien que
probó el admin con datos reales de alguien que solo escribió el código.

Un punto menos por la consecuencia del modelo: sin `choices=` en los campos de
dominio, el formulario de detalle muestra text inputs para `category`,
`country_code` y `status` en lugar de dropdowns. El admin sigue siendo
navegable sin errores (el criterio se cumple), pero se pierde la validación
visual del dominio que los `choices` brindan gratuitamente.

---

## Criterio 4 — Tests `20 / 25`

22 tests. Cobertura amplia con un patrón estructurado: clase por recurso
(`HealthTests`, `AnalyticsTests`, `UserTests`, `BatchTests`, `AuthTests`).

**Lo que destaca:**

`test_obtain_token_invalid → 422` — el único alumno del grupo que prueba el
efecto del `custom_exception_handler` fuera del batch endpoint. `obtain_auth_token`
de DRF lanza `ValidationError` con credenciales incorrectas, que el handler
convierte a 422. Este test confirma que el handler aplica globalmente, no solo
a los serializers propios.

`test_batch_invalid_country` — valida que `country_code="ZZ"` retorna 422.
Ernesto no tenía este caso.

`test_user_stats_not_found` — verifica 404 para `/users/9999999/stats`, no
solo para `/transactions`. Cubre el segundo endpoint de usuario.

`setUp` inserta 5 transacciones con `status` alternado (`completed` si `i % 2 == 0`,
`failed` en caso contrario) — los datos de test tienen variedad intencional.

**Gaps respecto al máximo:**

Los tests de analytics verifican estructura, no contrato de datos:
```python
self.assertIn("total_transactions", response.data)
self.assertIn("by_country",         response.data)
```
El comment explica la razón: "el Parquet puede no estar disponible en CI".
Pero Antonio resuelve esto con fixtures del ORM para tests de usuario y
reserva el contrato de datos real (`n_transactions == 1_000_000`) para
analytics. Estos tests pasan incluso si DuckDB retorna los datos del parquet
equivocado.

`test_batch_ok` es demasiado permisivo:
```python
self.assertEqual(
    response.data["inserted"] + response.data["duplicates_skipped"], 3
)
```
Pasa aunque `inserted=0` y `duplicates=3` — no verifica que los 3 se insertaron
realmente. `self.assertEqual(response.data["inserted"], 3)` sería la assertion
correcta para datos frescos.

Sin test de paginación con valores exactos ni de `page=0 → 422` (límite de
parámetros). Sin test de intra-batch dedup (el único test de dedup es
secuencial).

---

## Sobre el `decisions.md` voluntario

Ningún otro alumno escribió este documento en E5 sin que se les pidiera. Ocho
decisiones documentadas con la estructura "qué elegí / qué descarté / por qué",
con críticas honestas propias ("En producción esto sería un problema de diseño —
dos bases con los mismos datos se desincronizarán", "138 segundos es aceptable
pero 3.4x más lento que SQL crudo del E3"). La cuantificación del overhead del
ORM con el número medido (7,249 vs 24,514 filas/s) conecta el análisis con
evidencia real del ejercicio anterior.

Este hábito —documentar decisiones con evidencia antes de que alguien lo pida—
es exactamente lo que se va a evaluar en el E8. Traerlo al E5 voluntariamente es
la mejor señal de comprensión genuina de la entrega.

---

## Sobre el uso de herramientas de IA

El double-checked locking en `services/duckdb.py` con el comment que explica
ambos niveles, la estructura de `decisions.md` con trade-offs cuantificados, el
`reset_connection()` para tests, y el comment de `admin.py` sobre el
comportamiento de LIKE con 1M registros son indicadores de comprensión real.
El bug de intra-dedup en el batch y los tests que aceptan `inserted=0` sin
quejarse son el tipo de error que se cuela cuando no se ejercitan los edge cases
adversariales — vale la pena revisar ese hábito antes de E06.

---

## Pregunta de seguimiento

> `services/duckdb.py` protege la inicialización con `_lock` pero no la
> ejecución de queries. Considera este escenario: dos requests llegan
> simultáneamente a `/analytics/summary` cuando la conexión ya está
> inicializada. Ambos llaman `get_duckdb_connection()` (que retorna inmediato
> sin lock), y ambos ejecutan `conn.execute(...)` en paralelo sobre la misma
> conexión `:memory:`. ¿Qué puede pasar? Agrega el lock a la ejecución en
> `views.py` o en `duckdb.py`, y explica por qué el lugar que elijas es mejor.
> Pista: las conexiones DuckDB creadas con `duckdb.connect(database="file.db",
> read_only=True)` sí soportan acceso concurrente — ¿por qué?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Modelos, migraciones e índices | 25% | 21 / 25 |
| 6 endpoints funcionales con DRF | 30% | 23 / 30 |
| Django Admin configurado | 20% | 19 / 20 |
| Tests (mínimo 6) | 25% | 20 / 25 |
| **Total** | **100%** | **83 / 100** |

---

El `decisions.md` voluntario y el admin con `fieldsets` y `readonly_fields`
demuestran comprensión del sistema más allá de hacer pasar la rúbrica. Los tres
gaps concretos a corregir antes de E06: el lock durante queries en DuckDB
(una línea en `views.py`), la deduplicación intra-lote en el batch (mismo
patrón del `seen: dict` de Antonio), y el `atomic()` en la inserción. La
pregunta de seguimiento lleva directamente a los dos primeros. 