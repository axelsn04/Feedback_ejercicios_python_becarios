# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumno:** Ulises Josue Reyes Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 97 / 100

---

## Resumen general

La entrega más avanzada del grupo para E04. Cuatro decisiones la distinguen del resto: `asyncio.Lock` para serializar escrituras concurrentes (el único alumno con la primitiva de concurrencia correcta para FastAPI), 18 tests con validación de contrato de datos real, benchmark con 3 corridas y análisis comparativo de varianza entre corridas, y arquitectura limpia donde todas las queries están en `db.py` y `main.py` solo orquesta. El único gap: el benchmark cubre solo los 3 endpoints analíticos, no los de usuario ni el batch POST.

---

## Criterio 1 — 6 endpoints funcionales con datos reales `25 / 25`

**`asyncio.Lock` para escrituras en SQLite** — el único alumno del grupo con esta protección. `insert_batch()` en `db.py` es `async` y usa `async with _sqlite_write_lock:` para serializar escrituras concurrentes:

```python
async with _sqlite_write_lock:
    # verificar duplicados
    # insertar nuevos
```

Por qué esto importa: FastAPI procesa requests concurrentemente. Sin el lock, dos `POST /transactions/batch` simultáneos pueden pasar la verificación de duplicados al mismo tiempo, ver el mismo conjunto de IDs "no existentes", e intentar insertar las mismas filas — generando un conflicto de PRIMARY KEY o peor, datos inconsistentes. Angel tiene `threading.Lock` en el cache, que bloquea el event loop; `asyncio.Lock` libera el event loop mientras espera, que es el comportamiento correcto en código async.

**Cache con `invalidate_prefix("analytics:")`** después de batch insert. `invalidate_prefix()` elimina todas las claves que empiezan con "analytics:", lo que invalida tanto `analytics:summary` como todos los `analytics:top-merchants:*` en una sola llamada. Es más robusto que `invalidate("summary")` — si se agregara un nuevo endpoint analítico, se invalida automáticamente.

**`/dev/cache/clear` endpoint** — permite al benchmark forzar condición cold en cada request sin reiniciar el servidor. Sin este endpoint, el benchmark solo puede medir 1 cold request (el primero) y 99 warm. Con él, se miden 100 cold requests independientes.

**Separación de responsabilidades** — todas las queries están en `db.py`, `main.py` es un orquestador delgado. Ventaja: si se cambia el engine de SQLite a PostgreSQL, se cambia solo `db.py`. Los endpoints no saben cómo se implementa cada query.

**No hay SQL injection** — todos los queries de DuckDB usan parámetros vinculados `[country.upper(), limit]` y `[limit]`. Ninguna interpolación de strings en SQL.

---

## Criterio 2 — Architecture decision justificado `20 / 20`

El `architecture_decision.md` documenta cada decisión con razonamiento técnico y conecta con los ejercicios anteriores. Cuatro puntos que lo distinguen:

El análisis del overhead de apertura de Parquet cuantificado: "abrir la conexión a DuckDB sobre Parquet cuesta ~88ms (medido en E3). Si cada request abre su propia conexión, ese costo se paga en cada request". Es el mismo número que aparece en los benchmarks de E3 — no es un número inventado.

La justificación del `asyncio.Lock`: "FastAPI puede recibir dos batch requests simultáneos. Sin lock, ambos podrían pasar la verificación de duplicados al mismo tiempo". La explicación del race condition es precisa.

La justificación de por qué `/health` nunca toca la base de datos es la más explícita del grupo: "Si viola esta regla, /health puede tardar más de 50ms cuando la DB está bajo carga, lo que hace que el monitor de salud sea inútil precisamente cuando más lo necesitas".

La invalidación con prefijo está documentada con el razonamiento correcto: "Después de un insert exitoso invalida el cache de analytics para que los próximos requests a /analytics/* reflejen los datos actualizados."

---

## Criterio 3 — Cache con TTL y medición `19 / 20`

**`invalidate_prefix(prefix)`** — elimina todas las entradas cuya clave empieza con el prefijo dado. Más robusto que invalidar por clave individual: si se agrega `/analytics/revenue` mañana, el batch POST lo invalida automáticamente sin cambiar el código.

**`make_key(*parts)` helper estático** — construye cache keys canónicas: `make_key("analytics", "top-merchants", "limit=10", "country=MX")` → `"analytics:top-merchants:limit=10:country=MX"`. Elimina f-strings ad-hoc en los endpoints y garantiza formato consistente.

**`CacheEntry` y `CacheStats` como dataclasses** — estructura más explícita que tuplas. `CacheEntry(value, expires_at)` es más legible que `(value, expires_at)` y el typechecker puede validarla.

**`time.monotonic()`** — correcto para mediciones de TTL. `time.time()` puede retroceder si el reloj del sistema se ajusta (NTP); `monotonic()` es garantizado no-decreasing.

Un punto menos: el cache no tiene lock. Bajo carga concurrente, operaciones de `get()`, `set()` e `invalidate_prefix()` sobre el mismo dict Python pueden tener race conditions. Angel implementó `threading.Lock`; aquí sería `asyncio.Lock` para ser consistente con el resto del código async. Para producción con uvicorn multi-worker esto importa.

---

## Criterio 4 — Tests con pytest `20 / 20`

18 tests — el máximo del grupo. La cobertura es la más completa:

**`test_analytics_summary_ok` valida el contrato de datos:**
```python
assert body["total_transactions"] == 1_000_000
assert len(body["by_country"])  == 15
assert len(body["by_category"]) == 10
```
Este es el único test del grupo que valida los números reales del dataset — no solo que los campos existen sino que tienen los valores correctos para el dataset de 1M transacciones con 15 países y 10 categorías. Si el Parquet cambia, el test falla antes de que el bug llegue a producción.

**`test_analytics_top_merchants_ok` valida ordenamiento:**
```python
amounts = [m["total_amount"] for m in body["merchants"]]
assert amounts == sorted(amounts, reverse=True)
```
Valida que el endpoint no solo retorna merchants sino que están correctamente ordenados.

**`test_analytics_top_merchants_invalid_limit`** — 422 para limit=0 y limit=999. El único test del grupo que valida límites de parámetros query.

**`test_batch_invalid_schema`** cubre 3 casos en un test: amount negativo, category inválida ("Gambling" no está en el whitelist), y campo faltante (sin amount). La granularidad de validación es la más completa del grupo.

**`test_batch_empty_list` y `test_batch_over_limit`** — casos borde del batch que ningún otro alumno prueba en los dos extremos.

**Fixture `valid_transaction`** parametrizable — permite `{**valid_transaction, "transaction_id": str(uuid.uuid4())}` en los tests de batch para generar IDs únicos por run.

**Pytest skip automático** cuando los datos no existen — comportamiento profesional que evita fallos confusos en CI.

**user_id=2076** hardcodeado en los tests con el comentario "existe en el dataset (verificado en E3)". Mismo principio que el `valid_user_id` fixture de Bryan — usar datos reales conocidos en lugar de user_id=1 que podría o no existir.

---

## Criterio 5 — Benchmark de latencia con cache `13 / 15`

El benchmark más sofisticado del grupo en diseño, pero con cobertura limitada a endpoints analíticos.

**3 corridas con reporte comparativo de varianza** — el único alumno del grupo que mide esto. El `build_comparative_report()` calcula p50/p99 promedio, mínimo y máximo de p99 entre corridas, y porcentaje de corridas que pasan el SLA. Eso es evidencia estadística real, no un número puntual.

**`/dev/cache/clear` antes de cada request cold** — permite medir 100 requests cold independientes, no uno solo. El análisis de varianza entre corridas solo es válido si cada corrida mide condiciones comparables.

**Resultados de 3 corridas (summary):**
- Cold p50 promedio: 39.61ms (máx p99: 51.41ms) — 3/3 PASS
- Warm p50 promedio: 0.70ms (máx p99: 1.39ms) — 3/3 PASS

**Dos puntos menos:** el benchmark no incluye los endpoints de usuario ni `POST /transactions/batch`. Los endpoints de SQLite tienen su propio SLA (<80ms) que no está verificado con datos estadísticos. El README es honesto sobre esto ("endpoints analíticos"), pero para un sistema completo faltan la mitad de los endpoints.

---

## Sobre el uso de herramientas de IA

El `asyncio.Lock` en `insert_batch`, el `invalidate_prefix()`, el `make_key()` helper, los 18 tests con validación de contrato de datos, y el benchmark con múltiples corridas y análisis comparativo son decisiones que requieren comprensión del sistema. El docstring de `build_comparative_report()` que explica por qué el p99 máximo es más relevante que el promedio para validar SLAs es el tipo de análisis que no se genera por defecto. Los inline comments en `db.py` y `cache.py` que explican el razonamiento de cada pragma y cada decisión de diseño son genuinos.

---

## Pregunta de seguimiento

> El cache no tiene `asyncio.Lock`. Si llegan dos requests simultáneos a `/analytics/summary` cuando el cache está vacío, ambos pueden pasar el `cache.get()` y obtener `None`, ejecutar la query de DuckDB en paralelo, y hacer `cache.set()` dos veces con el mismo resultado. ¿Es eso un bug? ¿Cuándo sí importaría y cuándo no?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 25 / 25 |
| Architecture decision justificado | 20% | 20 / 20 |
| Cache con TTL y medición | 20% | 19 / 20 |
| Tests con pytest | 20% | 20 / 20 |
| Benchmark de latencia con cache | 15% | 13 / 15 |
| **Total** | **100%** | **97 / 100** |

---

El `asyncio.Lock` para escrituras concurrentes y los 18 tests con validación de contrato de datos son las dos decisiones que hacen esta entrega la más técnicamente correcta del grupo en E04. El benchmark de 3 corridas con análisis comparativo de varianza es la evidencia estadística más sólida del grupo para demostrar que los SLAs se cumplen consistentemente, no por suerte en una sola medición.