# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumno:** Angel Martinez Aguilar
**Fecha de revisión:** Mayo 2026
**Calificación:** 85 / 100

---

## Resumen general

Entrega completa con el benchmark más riguroso del grupo en términos de metodología de medición: 5 iteraciones, descarta el cold start, usa la mediana de las corridas calientes, y llama a `gc.collect()` antes de cada medición. El `create_report.py` mantiene la consistencia con el Ejercicio 1 — auto-genera las tablas desde el JSON y preserva el análisis escrito. Los tres casos de tradeoff están identificados y las interpretaciones de EXPLAIN ANALYZE son correctas. Lo que baja la calificación es la profundidad del análisis y un detalle de diseño en los engines que vale la pena entender.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `21 / 25`

Las 8 queries están implementadas en los tres engines y la validación usa `assert_frame_equal` con `atol=1e-3` — robusta. El benchmark descarta la primera corrida (cold start) y toma la mediana de las restantes. Eso es mejor metodología que cualquier otra entrega del grupo.

Hay un detalle de diseño que afecta la interpretabilidad del benchmark: los tres engines leen desde el archivo Parquet en cada llamada de query. Pandas carga el dataset completo a un DataFrame, polars hace un `scan_parquet` lazy, y DuckDB lee directamente. La comparación es consistente entre engines — todos miden I/O incluido — pero el resultado no es comparable con benchmarks donde pandas pre-carga los datos. La consecuencia práctica es que los tiempos de pandas (0.09–0.71s) incluyen el costo de leer y parsear el archivo Parquet completo cada vez, mientras que polars y DuckDB aprovechan column pruning durante la lectura. Eso amplifica artificialmente la desventaja de pandas.

Un segundo punto: `polars_engine.py` convierte todos los resultados a pandas con `.to_pandas()` al final de cada función. Eso añade overhead de conversión al timing de polars. Para una comparación limpia, polars debería retornar su DataFrame nativo y la validación debería normalizar ambos formatos.

El mismo riesgo de Q6 que en otras entregas: el filtro de polars `.filter(pl.col('count') == pl.col('count').max().over('country_code'))` devuelve múltiples filas en caso de empate, mientras que `ROW_NUMBER() OVER` de DuckDB siempre elige una. La validación puede fallar en ese caso.

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `21 / 25`

Las tres interpretaciones son correctas pero más breves que lo que el ejercicio busca.

**Q3:** Identifica el column pruning en el TABLE_SCAN, el HASH_GROUP_BY para los 50,000 grupos, y el TOP_N con heap de 10 elementos. Correcto. Falta la observación más importante: por qué DuckDB es más lento que pandas en Q3 si los datos están "en disco". La respuesta es que cuando el sistema operativo tiene el archivo en page cache (caliente), la ventaja de I/O de DuckDB desaparece y su overhead de inicialización del pipeline se vuelve visible. Esa observación eleva el análisis de descriptivo a explicativo.

**Q5:** Identifica correctamente el dynamic filter inyectado desde la subquery de `MAX(timestamp)` y el predicate pushdown que reduce de 1M a 73K filas durante la lectura del Parquet. La mención de que los row groups del Parquet se descartan antes de llegar al pipeline es correcta.

**Q6:** Identifica el `arg_max` que reemplaza al `ROW_NUMBER()` y la reducción de 150 a 15 filas. Correcto pero muy breve — no explica el mecanismo del struct_pack que DuckDB usa para empaquetar `{category, avg_amount, count}` antes de encontrar el máximo por país.

---

## Criterio 3 — Identificación y justificación de tradeoffs `22 / 25`

Los tres casos están identificados y son empíricos — vienen de los datos medidos, no de argumentos teóricos.

**Polars > pandas (Q8):** Correcto. Q8 es el caso más claro del benchmark: polars 16x más rápido que pandas. La explicación sobre multi-hilo en Rust y la API lazy para fechas es correcta.

**DuckDB ganador claro (Q1):** Válido. DuckDB gana Q1 con 0.011s vs 0.023s de polars. La explicación sobre ejecución vectorial por bloques en lugar de filas es correcta. Un punto extra habría sido notar que Q1 es un full scan + GROUP BY simple donde DuckDB no necesita columnas extra — projection pushdown al máximo.

**Tres comparables (Q5):** El caso más débil. Polars 0.066s, DuckDB 0.079s, pandas 0.091s — pandas es el doble de lento que polars, lo que difícilmente es "comparable". La justificación de alta selectividad del filtro es correcta, pero Q5 no es el caso más limpio para ilustrar igualdad entre engines. Una mejor opción habría sido Q4 donde DuckDB (0.013s) y polars (0.014s) son prácticamente idénticos.

---

## Criterio 4 — Reporte y recomendación de arquitectura `21 / 25`

El `create_report.py` que sincroniza tablas desde el JSON manteniendo el análisis escrito es consistente con el Ejercicio 1 — buen criterio de diseño que se repite. La tabla comparativa es clara y todas las queries tienen validación ✅.

Dos puntos que faltan. Primero, el reporte no tiene ninguna discusión sobre las limitaciones de `tracemalloc` para DuckDB y polars. Los valores de RAM de DuckDB (0.01 MB en la mayoría de queries) y polars (0.01–0.02 MB) son producto de que `tracemalloc` solo ve el heap de Python, no los buffers de C++ ni de Rust. Reportar esos números sin contexto puede llevar a conclusiones incorrectas sobre el consumo real de memoria. Ernesto lo investigó en E01 — era una pregunta pendiente que aquí habría cerrado.

Segundo, la recomendación dice "usar DuckDB para ranking y filtros dinámicos" pero en este benchmark polars ganó 7 de las 8 queries. La recomendación debería reflejar ese resultado con más precisión: DuckDB brilla cuando los datos están en disco y no caben en RAM, no cuando compite con polars sobre datos ya cacheados por el SO.

---

## Sobre el uso de herramientas de IA

El reporte es directo y técnicamente correcto. No tiene la voz personal de E01 ("Tengo entendido que", "me surge una duda") — es más declarativo. El `create_report.py` y el diseño del benchmark de 5 iteraciones con descarte de cold start son decisiones de ingeniería que muestran criterio propio. Uso asistido, sin señales de que el análisis fue generado sin revisión.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> En tu benchmark, los tres engines leen desde el archivo Parquet en cada query call. Ernesto pre-cargó pandas en memoria antes del benchmark. ¿Cuál de los dos diseños es más representativo de un caso de producción real? ¿Cambia tu recomendación de arquitectura dependiendo de cuál escenario asumas?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 21 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 21 / 25 |
| Identificación y justificación de tradeoffs | 25% | 22 / 25 |
| Reporte y recomendación de arquitectura | 25% | 21 / 25 |
| **Total** | **100%** | **85 / 100** |

---

La metodología de medición es la más rigurosa del grupo — 5 iteraciones, descarte de cold start, mediana. Eso es exactamente el rigor que faltó en E01. Para E03, ese mismo nivel de cuidado en las mediciones es lo que va a separar un benchmark válido de uno que solo parece serlo.