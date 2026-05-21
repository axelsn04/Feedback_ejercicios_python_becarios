# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumno:** Antonio Jair Garcia Vargas
**Fecha de revisión:** Mayo 2026
**Calificación:** 99 / 100

---

## Resumen general

La entrega más rigurosa del grupo para este ejercicio por margen amplio. Tres decisiones la distinguen del resto: el benchmark usa 100 repeticiones con parámetros aleatorios desde seed fijo (no valores inventados ni un solo parámetro fijo), valida SLAs en p95 en lugar de mean, y la decisión de no agregar covering index para P4 está respaldada con datos medidos — no con opinión. El `schema_design.md` cierra explícitamente qué no se hizo y por qué. Es la única entrega del grupo que mide y reporta p50/p95/p99/stdev por escenario. El resultado: 5/5 SLAs cumplidos con margen mínimo de 19x.

---

## Criterio 1 — Schema design justificado `25 / 25`

El `schema_design.md` es el más completo del grupo. Cada decisión tiene evidencia empírica del benchmark, no solo razonamiento teórico.

**La decisión más valiosa — no agregar covering index a P4:** el documento analiza explícitamente agregar `amount` al índice compuesto para convertir P4 en un index-only scan. La conclusión: P4 cumple 625x bajo el SLA con el índice simple (0.08ms p95). Agregar el covering costaría ~8MB de almacenamiento y ralentizaría los INSERTs sin aportar nada al SLA. Esa decisión solo es defendible si se midió primero — y aquí se midió. El resto del grupo asumió la estructura correcta sin cuestionarla; este alumno la cuestionó y la respaldó con datos.

**`timestamp DESC` en el índice compuesto** — justificado correctamente con el EXPLAIN: con DESC declarado, P2 hace forward scan sobre el índice y devuelve las 20 entradas más recientes sin sort adicional. Sin DESC, SQLite haría backward scan o sort intermedio.

**Covering index para P5** — `(country_code, user_id)` permite que SQLite resuelva el GROUP BY sin tocar la tabla principal, confirmado por `USING COVERING INDEX` en el EXPLAIN. La explicación de por qué el orden de columnas es crítico (range scan sobre `country_code`, agrupamiento natural por `user_id` dentro de cada país) es correcta y precisa.

**Sección "Lo que NO se hizo"** — documenta índices descartados con justificación técnica. Es la única entrega del grupo con esta sección.

**Diseño de schema.sql:** los índices secundarios están comentados en el SQL y son creados por `benchmark_queries.py`. Esta decisión de diseño es correcta para este ejercicio: permite que la misma base esté en estado "sin índices" para la fase 1 del benchmark sin necesitar recrearla. El comentario en `schema.sql` documenta explícitamente que esta es la intención.

---

## Criterio 2 — Ingesta chunkeada eficiente `19 / 20`

**`pyarrow.parquet.ParquetFile.iter_batches(batch_size=chunk_size)`** — el chunking más real del grupo. PyArrow lee el Parquet en batches nativos a nivel de row group, sin cargar el archivo completo en memoria. El pico de RAM es proporcional al `chunk_size` desde el inicio.

**`PRAGMA wal_checkpoint(TRUNCATE)` al cerrar** — único en el grupo. Sin este paso, el archivo `.db-wal` queda con datos no mergeados y el tamaño reportado de la DB puede ser engañoso (el `.db` parece pequeño, el WAL acumula los cambios). `TRUNCATE` fusiona todo al `.db` principal antes de cerrar, lo que garantiza que `db_file_bytes()` mide el estado real de la base.

**`synchronous=NORMAL` con WAL, `synchronous=FULL` sin WAL** — la justificación es correcta: WAL con NORMAL solo pierde la última transacción no-fsynced ante crash del kernel (aceptable para ingesta que se puede reiniciar); DELETE mode con NORMAL sería arriesgado porque el journal podría quedar inconsistente.

**Append-mode en `results/ingest.json`** — cada corrida agrega una entrada al historial. El benchmark de WAL vs no-WAL vive en el mismo archivo, lo que facilita la comparación directa.

Un punto menos: no hay verificación de integridad al final — comparar `COUNT(*) DB` con el número de filas del Parquet confirmaría que no hubo filas perdidas. Para una ingesta de 1M filas que puede reiniciarse, ese check es barato y da certeza.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `30 / 30`

**100 repeticiones con parámetros aleatorios desde seed fijo** — el benchmark más estadísticamente válido del grupo. Los parámetros se muestrean del Parquet real, lo que garantiza que son valores que existen en la base. P3 usa ventanas aleatorias de 7 días dentro del rango del dataset; P4 usa cutoff = max_ts - 30 días (no `datetime.now()`) con user_ids variables; P5 muestrea country_codes con repetición. Seed=42 garantiza reproducibilidad.

**SLA validado en p95** — la decisión metodológica más importante del benchmark. El mean puede esconder latencias en la cola. Con N=100, p95 es la rep #95 ordenada — suficientemente robusta para detectar problemas reales sin ser tan extrema como p99. Ningún otro alumno del grupo hizo esta distinción.

**`ANALYZE` llamado dentro de `create_indexes()`** — el benchmark crea los índices y ejecuta ANALYZE como parte de la misma operación. Esto garantiza que el query planner tiene estadísticas actualizadas antes de medir la fase "con índices". Crítico: sin ANALYZE, SQLite puede ignorar los índices incluso cuando son la opción correcta.

**EXPLAIN guardado en archivo separado** (`results/explain_query_plan.txt`) con el plan sin y con índices para cada patrón — hace que el reporte pueda citarlo directamente sin tener que reejecutar el benchmark.

**DB size antes y después de crear índices** medido y reportado: +92 MB de overhead de índices documentado. Es el único alumno que cuantifica este costo.

**Diseño de fases: sin índices primero, con índices después** — 100 reps sin índices, luego crear índices, luego 100 reps con índices. Este orden garantiza que la fase "sin índices" no está contaminada por el page cache de la fase "con índices". Más limpio que el enfoque de dos conexiones separadas porque el orden temporal ya garantiza el aislamiento.

---

## Criterio 4 — Comparación SQLite vs DuckDB `25 / 25`

El análisis más completo del grupo para este criterio.

**P1 — 1,275x**: correctamente explicado que DuckDB hace un scan de ~1M valores de transaction_id porque los UUIDs no tienen localidad espacial en el Parquet (no hay clustering que permita descartar row groups). El predicate pushdown de DuckDB no ayuda aquí.

**P4 — DuckDB cumple el SLA (26ms < 50ms)**: la única entrega del grupo que identifica que DuckDB es competitivo en P4 y explica por qué — projection pushdown extremo (solo lee `amount`, `user_id`, `timestamp`), UNGROUPED_AGGREGATE vectorizado, y descarte de row groups por estadísticas de timestamp. SQLite todavía gana porque el B-tree lleva directamente a las ~30 filas relevantes, pero la diferencia (0.08ms vs 27ms) ya no es de tres órdenes de magnitud.

**P5 — 1.9x**: el análisis más honesto del grupo. DuckDB está muy cerca porque P5 es exactamente el patrón analítico donde DuckDB se diseñó para brillar. La observación de que agregar `SUM(amount)` o `AVG` probablemente empataría o daría ventaja a DuckDB es correcta.

**La tabla "lectura cualitativa"** — mapea tipos de query a ganador estructural, incluyendo los patrones no medidos (full table scan, joins de tablas grandes) donde DuckDB sería esperado. Es una guía de arquitectura útil.

**Conexión explícita con E4**: el reporte termina mapeando cada endpoint de E4 al engine correcto (SQLite vs DuckDB) con justificación basada en los datos medidos. Es la única entrega que cierra ese loop explícitamente.

---

## Sobre el uso de herramientas de IA

La sección §3 del `schema_design.md` — "Decisión sobre covering index (DATA-DRIVEN)" — no se genera por inercia. Requiere haber diseñado el covering alternativo, medido P4 sin él, observado que el margen es de 625x, y concluir que no vale la pena. El análisis de P4 vs DuckDB en el reporte (la única query donde DuckDB cumple el SLA) tampoco es genérico. El código tiene `iter_batches` de PyArrow y `wal_checkpoint(TRUNCATE)` — decisiones que requieren conocer la API a fondo. Uso inteligente con comprensión real.

---

## Pregunta de seguimiento

Para el Ejercicio 4:

> Tu schema_design.md predice que si P5 incluyera `SUM(amount)` o `AVG`, DuckDB empataría o ganaría porque agregar columnas en columnar es prácticamente gratis. Diseña esa query en DuckDB y mídela contra SQLite con el covering index apropiado `(country_code, user_id, amount)`. ¿La predicción se cumple?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 25 / 25 |
| Ingesta chunkeada eficiente | 20% | 19 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 30 / 30 |
| Comparación SQLite vs DuckDB | 25% | 25 / 25 |
| **Total** | **100%** | **99 / 100** |

---

La validación en p95 con 100 reps es la diferencia metodológica más importante del grupo. Para E04, ese mismo rigor aplicado al benchmark de latencia del API (p50/p95/p99 por endpoint) es exactamente lo que el ejercicio pide y lo que diferencia un benchmark válido de uno que solo parece serlo.