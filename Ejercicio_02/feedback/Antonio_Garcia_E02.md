# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumno:** Antonio Jair Garcia Vargas
**Fecha de revisión:** Mayo 2026
**Calificación:** 97 / 100

---

## Resumen general

La entrega más sofisticada del grupo para este ejercicio. El benchmark tiene la arquitectura más limpia: cada engine expone `load()` y `qN()` con la misma firma, `benchmark.py` no conoce los internos de ninguno, y la validación prueba los tres pares posibles. DuckDB reutiliza una conexión persistente con VIEW, polars trabaja con LazyFrame nativo, pandas no muta el DataFrame compartido. Tie-breaking explícito y consistente en los tres engines para todas las queries que lo necesitan. El reporte es el análisis más honesto del grupo — incluye el caveat de que DuckDB no gana ninguna query frente a polars y explica por qué estructuralmente, no como excusa.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `24 / 25`

La arquitectura de los engines es la mejor del grupo. El diseño `load() → handle, qN(handle) → result` hace que `benchmark.py` sea agnóstico al engine — agregar un cuarto engine (polars 2.x, ibis, etc.) es añadir un módulo sin tocar el runner. La validación prueba los tres pares `(pandas/duckdb, pandas/polars, duckdb/polars)` en lugar de solo pandas vs los demás, lo que detecta divergencias entre DuckDB y polars que la validación de dos pares no vería.

El tie-breaking es consistente en los tres engines en todas las queries que lo necesitan: Q1 secondary sort por `country_code`, Q3 por `user_id`, Q6 por `category ASC` en el ORDER BY de la window function, Q7 por `user_id`. Esto garantiza que los resultados son deterministas incluso con empates en el dataset. Ninguna otra entrega del grupo lo implementa de forma coordinada en los tres engines.

`pandas_engine.py` no muta el DataFrame compartido — `q4` usa `.loc[...].copy()`, `q8` usa `df.assign(day=day)` en lugar de `df["day"] = ...`. `polars_engine.py` usa `scan_parquet()` lazy correctamente y retorna `pl.DataFrame` nativo — la conversión a pandas ocurre en `benchmark.py` con `to_pandas()`, no dentro del engine.

Un punto menos: Q4 no garantiza las 24 horas. El spec dice "(0-23)", que implica que todas las horas deben aparecer aunque tengan conteo cero. Los tres engines son consistentes entre sí en este punto (todos omiten horas sin datos fallidos), así que la validación pasa, pero el resultado no cumple estrictamente la especificación. Ulises resolvió esto con un join sobre `all_hours = range(24)` en los tres engines.

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `25 / 25`

Las tres interpretaciones son correctas, tienen evidencia de haber leído los planes reales, y explican el mecanismo — no solo el resultado.

**Q3:** Identifica projection pushdown (solo `user_id` y `amount`), el HASH_GROUP_BY como cuello de botella con 50K cubetas, y el TOP_N como heap de tamaño 10 con complejidad O(n log 10) vs O(n log n) de un sort completo. La explicación de por qué polars gana a pesar de que DuckDB tenga el mismo TOP_N — polars tiene los datos en memoria Arrow sin el overhead de escanear el Parquet — es correcta y directa. La observación de que materializar con `CREATE TABLE txns AS SELECT * FROM read_parquet(...)` cerraría la brecha es exacta.

**Q5:** "El ejemplo más bonito de optimización dinámica de DuckDB" — y es correcto. Identifica predicate pushdown para `amount > 500` y `country_code IN (...)`, el dynamic filter cross-CTE que inyecta el `max(timestamp)` calculado en runtime como filtro estático al TABLE_SCAN, y cuantifica el resultado: 73,900 filas materializadas de 1M — el 7.4%. La explicación de por qué pandas gana Q5 a pesar de toda la optimización de DuckDB — pandas tiene el DataFrame en RAM, su filtro vectorizado sobre numpy es difícil de batir, y el costo de I/O que DuckDB paga en cada query (aunque sea reducido) supera el beneficio cuando la query retorna pocas filas — es la más precisa del grupo para ese caso.

**Q6:** Identifica correctamente el pipeline: HASH_GROUP_BY a 150 grupos → WINDOW → FILTER (rn = 1). La observación de que el hash de 150 entradas cabe en caché L2 también aparece aquí. La explicación de por qué pandas pierde — `groupby().agg()` sobre 1M filas con allocations intermedios — conecta el plan con los números medidos.

---

## Criterio 3 — Identificación y justificación de tradeoffs `25 / 25`

Los tres casos son empíricos, cuantificados, y tienen justificaciones técnicas precisas.

**Polars > pandas (Q7):** 6.8x más rápido. La explicación es la más completa: pandas ejecuta cuatro pasos secuenciales con allocations intermedios; polars compone toda la pipeline en el optimizador y la ejecuta paralelizada en Rust. La frase "el costo de Python como pegamento se paga 1 vez por grupo en pandas y 0 veces en polars" captura el mecanismo de forma precisa.

**DuckDB más cercano a ganar (Q6):** El análisis es honesto y técnicamente correcto: "en este benchmark no hay ninguna query donde DuckDB supere a polars". Identificar Q6 como el caso donde DuckDB se acerca más a polars (30% de diferencia vs 4-7x en otras queries) y explicar por qué — la window function es donde SQL tiene ventaja sobre el API de polars, y los 150 grupos caben en caché L2 — es más valioso que forzar un "DuckDB gana X" que no existe en los datos.

**Tres comparables (Q8):** La observación sobre Q8 es única en el grupo: pandas y DuckDB están dentro del 5% porque la cardinalidad del output es alta (3,660 grupos = 366 días × 10 categorías) y el tiempo queda dominado por construir y materializar el resultado, no por escanear o reducir. Cuando la salida es grande, las diferencias de engine se aplanan. Esa observación requiere haber pensado en qué determina el tiempo de cada query.

---

## Criterio 4 — Reporte y recomendación de arquitectura `23 / 25`

El reporte tiene el análisis de arquitectura más matizado del grupo. La sección de DuckDB explica su killer feature (leer Parquet con projection + predicate pushdown sin ETL previo, directamente contra archivos en disco/S3) y su costo (scan del Parquet en cada query al trabajar con VIEWs). La sección de polars explica correctamente que es el "reemplazo moderno de pandas en pipelines de producción" con la advertencia honesta de que el ecosistema circundante es menor. La recomendación final — DuckDB para SQL y datasets > RAM, polars para pipelines batch en Python, pandas como puente con el ecosistema científico — es la más accionable y diferenciada del grupo.

Dos puntos menores. Primero, el reporte no tiene tabla de pico de memoria separada por escala ni discusión sobre el tiempo de carga — que el benchmark sí mide (pandas 0.136s, DuckDB 0.008s, polars 0.000s) y que es un dato relevante para la recomendación de arquitectura: si se van a hacer 100 queries, el costo de carga de pandas se amortiza; si es 1 query, DuckDB y polars son claramente mejores. Segundo, la limitación de tracemalloc se menciona brevemente pero no se conecta con los números específicos de la tabla de memoria.

---

## Sobre el uso de herramientas de IA

El reporte tiene voz propia: "el ejemplo más bonito de optimización dinámica de DuckDB", el caveat honesto sobre DuckDB, la observación de Q8 sobre cardinalidad de output. Las decisiones de diseño en el código — abstracción de engines con `load()`, tie-breaking coordinado, `df.assign()` para evitar mutaciones — son de ingeniería, no de generación. Uso inteligente.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> Tu benchmark mide el tiempo de carga por separado (pandas: 0.136s, DuckDB: 0.008s, polars: 0.000s). Si un sistema de producción ejecuta 1 query por sesión, ¿cuál engine conviene? ¿Y si ejecuta 1,000 queries sobre el mismo dataset en la misma sesión? ¿En qué punto el costo de carga de pandas se amortiza completamente?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 24 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 25 / 25 |
| Identificación y justificación de tradeoffs | 25% | 25 / 25 |
| Reporte y recomendación de arquitectura | 25% | 23 / 25 |
| **Total** | **100%** | **97 / 100** |

---

El diseño del benchmark — con la capa de abstracción de engines, la carga separada, la validación de tres pares, y el tie-breaking coordinado — es el más cuidado del grupo. Para E03, ese mismo nivel de diseño aplicado al schema de SQLite y al script de ingesta va a ser la diferencia.