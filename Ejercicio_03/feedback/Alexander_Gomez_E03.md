# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumno:** Bryan Alexander Gomez Miranda
**Fecha de revisión:** Mayo 2026
**Calificación:** 84 / 100

---

## Resumen general

El reporte más honesto del grupo para este ejercicio: admite que P2, P3 y P4 no cumplen sus SLAs y analiza por qué. El schema design es sólido, la ingesta hace chunking real de I/O, y el análisis de los resultados demuestra comprensión real de cómo funciona el query planner de SQLite. El punto que baja más la calificación es la ausencia de `ANALYZE` antes del benchmark, que probablemente es la causa principal del fallo de SLA — no la distribución del dataset. La diferencia entre tu resultado (81ms en P2) y el de otros alumnos (0.154ms en P2 con el mismo dataset) apunta directamente a ese paso faltante.

---

## Criterio 1 — Schema design justificado `23 / 25`

El `schema_design.md` tiene el análisis más claro del grupo sobre por qué cada decisión es correcta, incluyendo lo que deliberadamente no se indexó.

**`timestamp` como TEXT ISO 8601** — la justificación es correcta y precisa: el orden lexicográfico de ese formato coincide con el cronológico, lo que permite usar comparaciones `<`, `>`, `BETWEEN` sin funciones de conversión. La nota de que usar `strftime()` en el WHERE rompería el uso del índice es exacta y es un error común.

**`idx_country_user (country_code, user_id)`** — covering index correcto para P5. El reporte confirma que funciona: `SEARCH transactions USING COVERING INDEX`.

**Lo que no se indexó y por qué** — la sección sobre decisiones negativas es la mejor del grupo. Documentar que `merchant_id`, `category`, `status` y `amount` no tienen índice, y explicar que cada índice innecesario añade overhead en escritura y puede confundir al optimizador, demuestra comprensión real del tradeoff.

**Dos puntos menos:** `idx_user_timestamp` no tiene `DESC` en timestamp. Para P2 (`ORDER BY timestamp DESC LIMIT 20`), sin `DESC` en el índice SQLite puede necesitar un backward scan en lugar del forward scan natural. Ulises añadió `DESC` y obtuvo 0.154ms en P2. El segundo punto: el `schema_design.md` no analiza el tradeoff `WITH ROWID` vs `WITHOUT ROWID`, que Ulises justificó explícitamente.

---

## Criterio 2 — Ingesta chunkeada eficiente `17 / 20`

**`pd.read_csv(chunksize=chunk_size)`** — chunking real de I/O. El lector de pandas consume el CSV en bloques, con pico de RAM proporcional al chunk en lugar del archivo completo. Correcto.

**`isolation_level=None` + `BEGIN` / `executemany` / `COMMIT` manual** — la forma más limpia del grupo para gestionar transacciones explícitas. `isolation_level=None` activa el modo autocommit de Python, lo que significa que los `BEGIN` y `COMMIT` explícitos tienen control total sin interferencia del driver.

**`INSERT OR IGNORE`** para deduplicación ✅. Progress reporting cada 5 chunks ✅.

**Tres puntos menos:** no hay verificación de integridad al final — comparar `COUNT(*) DB` vs conteo del CSV confirma que la ingesta fue completa sin filas perdidas. Los índices se crean en `schema.sql` antes de la ingesta, lo que significa que cada `executemany` actualiza los tres índices en tiempo real. Crearlos después sería ~20-30% más rápido. Tampoco hay limpieza de los archivos auxiliares WAL (`.db-wal`, `.db-shm`) al borrar la base.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `22 / 30`

P2, P3 y P4 fallan sus SLAs (81ms vs 50ms). El análisis del reporte es honesto sobre esto. Pero hay una causa que no identificaste: **la ausencia de `ANALYZE`**.

`ANALYZE` le dice al query planner de SQLite la distribución real de los datos — cuántas filas hay por `user_id`, cuántos valores distintos hay por columna. Sin `ANALYZE`, el planner trabaja con estimaciones por defecto y puede tomar decisiones subóptimas, como preferir full scan sobre index scan aunque el índice sea más eficiente.

La evidencia: Ulises corrió el mismo ejercicio con el mismo dataset, tiene el mismo índice `idx_user_timestamp`, y obtuvo **0.154ms en P2**. Tú obtuviste **81ms**. La diferencia de 500x entre dos benchmarks con el mismo índice sobre el mismo dataset no se explica por la distribución de datos — se explica por el estado del planner. Ulises corrió `ANALYZE` antes de medir; tú no.

Esto no invalida tu análisis del reporte — la observación de que la distribución uniforme del dataset hace que el índice tenga poca selectividad es correcta en principio. Pero el diagnóstico completo es: sin `ANALYZE`, el planner no sabe la distribución y hace full scan por defecto.

Lo que sí está bien: las mediciones tienen 3 repeticiones con promedio ✅, `cutoff_date` calculado desde el máximo del dataset con `datetime(?, '-30 days')` ✅ (no el bug de `datetime.now()` que tuvo Angel), usuario con más transacciones como parámetro de prueba ✅, EXPLAIN capturado para todos los patrones ✅, DuckDB comparado en los 5 ✅.

Sin `gc.collect()` entre mediciones — puede añadir ruido en mediciones de sub-millisecond.

---

## Criterio 4 — Comparación SQLite vs DuckDB `22 / 25`

El análisis del reporte es el más honesto del grupo y tiene observaciones genuinas.

**P1 2,280x más rápido en SQLite** — la explicación de que DuckDB debe abrir el Parquet, descomprimir bloques de columnas y escanear sin índice es correcta. El número concreto (98ms de overhead fijo de DuckDB) conecta con lo que Ulises también midió (~88ms).

**P4 y P5 ganan a DuckDB** — la explicación de la proyección columnar para P4 y la agregación vectorizada multi-thread para P5 son correctas. La observación de que P2 y P3 ganan a DuckDB por el page cache caliente después de P1 es genuina y precisa — en frío, DuckDB probablemente ganaría P2 y P3.

**La conclusión más valiosa del reporte**: "hay un resultado inesperado que vale la pena señalar: P2 y P3 las gana SQLite a pesar de hacer full scan, simplemente porque el page cache del sistema operativo tiene las páginas ya cargadas desde P1." Eso demuestra que leíste tus resultados con atención real.

**La conclusión de arquitectura** es clara y bien conectada con E04: SQLite para la capa transaccional, DuckDB para la analítica, pipeline de exportación entre ambas. La referencia a E04 como el lugar donde se construye esa arquitectura es correcta.

**Tres puntos menos:** el reporte no identifica `ANALYZE` como causa del fallo de SLA — esa conexión habría cerrado el análisis técnico completamente. Y la sección de comparación podría cuantificar más: "SQLite gana 1.3x en P2" es una diferencia pequeña que vale señalar como casi empate en condiciones reales.

---

## Sobre el uso de herramientas de IA

El reporte tiene voz propia y es el único del grupo que no esconde resultados negativos. La admisión de que P2, P3 y P4 fallan los SLAs, con análisis técnico de por qué, es el tipo de escritura que requiere haber ejecutado el código y leído los resultados. El `schema_design.md` tiene la sección "Lo que este schema no intenta hacer" que es una señal de pensamiento genuino. Uso asistido con comprensión real.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 4:

> Corre `ANALYZE` en tu base de datos y vuelve a ejecutar el benchmark de P2. El comando es simplemente `conn.execute("ANALYZE")` antes de las mediciones. ¿Cambia el EXPLAIN QUERY PLAN? ¿Cambia el tiempo? Si cambia, explica qué información le dio `ANALYZE` al planner que no tenía antes.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 23 / 25 |
| Ingesta chunkeada eficiente | 20% | 17 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 22 / 30 |
| Comparación SQLite vs DuckDB | 25% | 22 / 25 |
| **Total** | **100%** | **84 / 100** |

---

El análisis honesto de los fallos de SLA es el mejor del grupo. Para E04, la pregunta de seguimiento es concreta: corre `ANALYZE` y ve qué cambia. Esa respuesta va a cambiar cómo entiendes el tradeoff entre ingesta y consulta en sistemas transaccionales.