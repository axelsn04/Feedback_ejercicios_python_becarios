# Retroalimentación — Ejercicio 05: El Backend con Estructura

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 85 / 100

---

## Resumen general

La entrega replica correctamente la arquitectura dual del E04 dentro de Django: DuckDB para analytics, ORM para consultas transaccionales. Tres decisiones sólidas: WAL mode configurado en el lugar correcto (`connection_created` en `settings.py`, no en un management command), `bulk_create(ignore_conflicts=True)` para ingesta idempotente, e `invalidate_prefix('analytics:')` en el batch insert para consistencia de caché. El reporte conecta las decisiones con evidencia de E2 y E3. Los gaps son concretos: un bug de 404 en paginación, riesgo de thread-safety en el singleton de DuckDB, y tests que aceptan 200 o 404 indistintamente en dos endpoints críticos.

---

## Criterio 1 — Modelos, migraciones e índices `22 / 25`

**WAL mode en el lugar correcto.** La señal `connection_created` en `settings.py` aplica `PRAGMA journal_mode=WAL` en cada nueva conexión, lo que garantiza que cualquier apertura de conexión a SQLite — desde el ORM, desde tests, desde el management command — activa el modo correcto. Un `PRAGMA` dentro de `ingest.py` o del management command solo cubriría esa conexión específica. Esta es la implementación correcta para Django.

**`bulk_create(ignore_conflicts=True)` para idempotencia.** Corrige directamente el gap señalado en el feedback de E4. Ejecutar `load_transactions` más de una vez no duplica registros — el `INSERT OR IGNORE` implícito lo maneja SQLite a nivel de constraint, no Python a nivel de verificación previa (lo que sería más lento y propenso a race conditions).

**Índices del E3 replicados correctamente con `Meta.indexes`:**
```python
models.Index(fields=['user_id', 'timestamp'], name='idx_user_timestamp'),
models.Index(fields=['country_code', 'user_id'], name='idx_country_user'),
```
Estos cubren exactamente los patrones P2/P3/P4 del E3. La migración generada por Django incluirá los `CREATE INDEX` correspondientes — versionados y reproducibles.

**`USE_TZ = False` en settings — punto de atención.** El dataset usa timestamps en UTC. Con `USE_TZ = False`, Django no aplica conversión de zona horaria en el ORM, lo que es internamente consistente — pero si en algún momento se activa `USE_TZ = True` (que es el default de Django 4.0+), todos los comparadores de timestamp del ORM (`filter(timestamp__gte=...)`) empezarán a fallar silenciosamente porque Django asumirá que los valores en la base son aware. Mejor explicitarlo en el `schema_design.md` como decisión consciente.

Un punto menos por no tener acceso al `models.py` ni a las migraciones generadas para verificar el schema completo — el reporte los describe correctamente pero no están entre los archivos entregados.

---

## Criterio 2 — 6 endpoints funcionales con DRF `26 / 30`

Los 6 endpoints están implementados, con backend correcto por endpoint y autenticación dividida apropiadamente.

**DuckDB con lazy init y pre-calentamiento:**
```python
_duckdb_conn.execute("SELECT COUNT(*) FROM transactions").fetchone()
```
Fuerza la lectura del Parquet a memoria en el primer request, no en cada uno. Misma estrategia que el E04 — correcta.

**`invalidate_prefix('analytics:')` en batch insert** — invalida tanto `analytics:summary` como todas las variantes de `analytics:top-merchants:*` en una sola llamada. Más robusto que invalidar por clave específica.

**Deduplicación en dos fases en el batch:**
```python
# Fase 1: deduplicar dentro del mismo batch
seen = set(); unique = []

# Fase 2: verificar contra la base
existing_ids = set(Transaction.objects.filter(...).values_list('transaction_id', flat=True))
```
El reporte del response distingue `duplicates_in_batch` y `duplicates_in_db`, que es más informativo que un solo contador de descartados.

**Bug en `UserTransactionsView` — 404 solo en page=1:**
```python
if not results and page == 1:
    exists = Transaction.objects.filter(user_id=user_id).exists()
    if not exists:
        return Response(..., status=404)
```
Si un `user_id` no existe y se pide `/users/99999/transactions?page=2`, el endpoint retorna `[]` con HTTP 200 en lugar de 404. El test `test_user_not_found_404` no detecta esto porque siempre usa la URL sin `?page=`. En producción, un cliente que itera páginas trataría el 200 vacío como "última página" en lugar de "usuario no existe".

**Riesgo de thread-safety en `_duckdb_conn`:**
```python
_duckdb_conn = None

def get_duckdb():
    global _duckdb_conn
    if _duckdb_conn is None:
        _duckdb_conn = duckdb.connect()
        ...
```
Django por defecto corre con `runserver` en un solo thread, pero en producción con gunicorn + workers, múltiples threads pueden ejecutar `get_duckdb()` simultáneamente cuando `_duckdb_conn` es `None`, inicializando múltiples conexiones. La segunda escritura sobre `_duckdb_conn` "gana", pero el `CREATE VIEW` sobre la primera conexión se pierde. La solución es un `threading.Lock` en el bloque de inicialización — diferente a `asyncio.Lock` porque Django es síncrono.

---

## Criterio 3 — Django Admin `17 / 20`

El reporte describe la configuración con precisión: `list_display` con las 8 columnas del schema, `list_filter` por `status` y `country_code`, `search_fields` por `transaction_id` y `user_id`, `ordering` por timestamp descendente. Todo lo que requiere el criterio está listado.

Un punto menos porque `admin.py` no está entre los archivos entregados — no es posible verificar que las 8 columnas están bien nombradas (`transaction_id`, no `id`) ni que el `search_fields` usa la sintaxis correcta (`=transaction_id` para búsqueda exacta vs `transaction_id` para LIKE).

Un punto menos adicional por no incluir `date_hierarchy` o `list_per_page` — no son requeridos pero son los controles estándar de admin para un modelo con 1M de registros. Sin `list_per_page` configurado, el admin intenta cargar 100 filas por página sobre SQLite con 1M de registros, lo que puede ser lento sin índice en la columna de ordering.

---

## Criterio 4 — Tests `20 / 25`

10 tests, todos en `BaseTestCase` con setup compartido limpio. Supera el mínimo requerido (6).

**Lo que funciona bien:**

`test_batch_valid` verifica el contrato de respuesta con números exactos:
```python
self.assertEqual(resp.data['received'], 1)
self.assertEqual(resp.data['inserted'], 1)
```
No solo que el endpoint retorna 200, sino que el conteo es correcto.

`test_user_not_found_404` usa `user_id=99999999`, que es más robusto que `user_id=0` — en teoría podría existir un usuario 0 si el ORM lo permite.

`test_batch_invalid_schema_422` envía un payload deliberadamente roto (`amount: 'not_a_number'`), que valida que el serializer rechaza tipos incorrectos.

**Gaps:**

`test_transactions_with_token` y `test_stats_with_token` aceptan 200 o 404 indistintamente:
```python
self.assertIn(resp.status_code, [200, 404])
```
Esto siempre pasa — no valida nada. Si el endpoint retorna 500, el test falla; si retorna cualquier otro código, también falla. Pero 200 con un body vacío o 404 con body correcto pasan igual. Para que estos tests sean útiles necesitan un `user_id` conocido del dataset (como el `user_id=2076` verificado en E3 que usó Ulises) y verificar la estructura del response.

No hay test de deduplicación — insertar la misma transacción dos veces y verificar `inserted=0` en el segundo request. El batch insert tiene lógica de deduplicación pero ningún test la ejercita.

No hay test de paginación ni de límites de parámetros (`?limit=999` en top-merchants debería retornar 422).

---

## Sobre el reporte técnico

El `architecture_decision.md` (visible en el documento de reporte) es el más completo del E5 del grupo visible hasta ahora. Tres puntos que lo distinguen:

La justificación de DuckDB para analytics conecta con evidencia del E2: "DuckDB con projection pushdown es significativamente más rápido que el ORM para operaciones de `GROUP BY` sobre 1M de filas". No es una afirmación genérica — es la misma observación medida en E2.

La tabla de mejoras aplicadas desde el feedback de E4 demuestra que el ciclo de retroalimentación funciona: WAL mode, invalidación de caché en batch, variables de entorno — los tres gaps señalados en E4 aparecen resueltos aquí.

La comparación FastAPI vs DRF tiene la respuesta correcta a la pregunta de fondo: no hay un ganador absoluto, hay criterios de selección. "Si el proyecto va a crecer y necesita estructura, Django te evita decisiones de diseño que FastAPI dejaría abiertas" es la conclusión técnicamente honesta.

---

## Sobre el uso de herramientas de IA

La conexión explícita entre las decisiones de E5 y los benchmarks de E2/E3, la señal `connection_created` en el lugar correcto (no en el lugar obvio), y la corrección aplicada de los tres gaps de E4 son señales de trabajo genuino. El bug de 404 en page>1 y los tests que aceptan 200 o 404 son el tipo de error que se cuela cuando no se ejecutan los tests contra escenarios adversariales reales — vale la pena revisar ese hábito antes de E06 y E07.

---

## Pregunta de seguimiento

Antes de continuar con el Ejercicio 6:

> `get_duckdb()` inicializa `_duckdb_conn` sin ningún lock. En Django con gunicorn multi-thread, ¿qué pasa si dos requests llegan simultáneamente cuando `_duckdb_conn` es `None`? Agrega un `threading.Lock` que proteja la inicialización y explica por qué `threading.Lock` es la primitiva correcta aquí y no `asyncio.Lock` — aunque en el E04 de FastAPI la primitiva correcta era exactamente la opuesta.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Modelos, migraciones e índices | 25% | 22 / 25 |
| 6 endpoints funcionales con DRF | 30% | 26 / 30 |
| Django Admin configurado | 20% | 17 / 20 |
| Tests (mínimo 6) | 25% | 20 / 25 |
| **Total** | **100%** | **85 / 100** |

---

La arquitectura es sólida y el reporte demuestra comprensión real del trade-off FastAPI vs DRF. El gap más importante a corregir antes de E06 es el patrón de tests que aceptan múltiples códigos HTTP — un test que siempre pasa no detecta regresiones. El bug de paginación y el thread-safety en DuckDB son rápidos de arreglar y la pregunta de seguimiento te lleva directo a ambos.