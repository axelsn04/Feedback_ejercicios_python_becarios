# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumna:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 72 / 100

---

## Resumen general

El ejercicio está completo: las 8 queries corren en los tres engines, la validación pasa, hay resultados medidos y un reporte con los tres casos de tradeoff. El código funciona y los datos son reales. Hay tres problemas técnicos concretos que bajan la calificación: polars usa `read_parquet()` eager en lugar de `scan_parquet()` lazy, el EXPLAIN ANALYZE no está integrado al benchmark, y la explicación de por qué DuckDB gana Q1 es factualmente incorrecta.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `19 / 25`

Las 8 queries están implementadas y la validación con `assert_frame_equal` con `check_dtype=False` y `rtol=1e-3` es correcta. El benchmark pasa conexión de DuckDB y lee el Parquet directamente en cada query — cumple la restricción del enunciado.

**Problema principal: polars usa `pl.read_parquet()` en lugar de `pl.scan_parquet()`.** La diferencia no es menor. `read_parquet()` carga el archivo completo en un `DataFrame` en memoria de forma eager antes de ejecutar cualquier operación. `scan_parquet()` retorna un `LazyFrame` — polars construye el plan de ejecución y solo cuando llamas `.collect()` lee del disco, aplicando column pruning y predicate pushdown automáticamente. En este benchmark, polars tiene los datos en RAM al igual que pandas, lo que hace que la comparación entre polars y DuckDB sea "datos en RAM vs datos en disco" — no una comparación entre engines con las mismas condiciones.

**Sin `gc.collect()` entre mediciones.** El garbage collector de Python puede dispararse durante una medición y añadir tiempo que no corresponde al engine que se está midiendo. Es una línea de código que reduce el ruido.

Q4 no garantiza las 24 horas — si alguna hora no tiene transacciones fallidas no aparece en el resultado. Los tres engines son consistentes, así que la validación pasa, pero el spec dice "(0-23)".

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `18 / 25`

Las tres interpretaciones son correctas en sus observaciones técnicas pero tienen dos problemas.

**El EXPLAIN ANALYZE está en un archivo separado (`explain_queries.py`) que no está integrado al benchmark.** Los resultados no se capturan en el JSON de resultados ni se generan automáticamente al correr el benchmark. En las demás entregas del grupo, el benchmark captura los planes y los guarda junto con los tiempos — eso es lo correcto porque garantiza que el plan que se analiza corresponde exactamente al dataset y condiciones del benchmark.

**Las interpretaciones son correctas pero breves.** Q3 identifica column pruning y el TOP_N como heap de 10 — correcto. Q5 identifica predicate pushdown y cuantifica: 73,909 filas de 1M — correcto. Q6 identifica la reducción a 150 grupos y el tie-breaking en el ROW_NUMBER — correcto. Lo que falta en las tres es el mecanismo más profundo: por qué el HASH_GROUP_BY es el cuello de botella en Q3, cómo funciona el dynamic filter cross-CTE en Q5 (no es solo "predicado pushdown" genérico — DuckDB calcula el MAX primero y lo inyecta como filtro estático), y en Q6 por qué DuckDB reemplaza internamente el ROW_NUMBER con `arg_max_nulls_last`.

---

## Criterio 3 — Identificación y justificación de tradeoffs `17 / 25`

Los tres casos están identificados y hay datos en la tabla que los respaldan.

**Polars > pandas (Q8):** Correcto. 1.396s vs 0.064s. La explicación sobre extracción de fechas con objetos Python vs procesamiento paralelo de polars es correcta en dirección, aunque incompleta — el costo real de pandas en Q8 es que `dt.date` genera un objeto Python por cada una de las 1M filas, y la cardinalidad del output (~3,660 grupos) hace el GROUP BY más pesado.

**DuckDB ganador (Q1):** La observación de que DuckDB gana Q1 es correcta. Pero la explicación es factualmente incorrecta: el reporte dice "Pandas y Polars primero deben cargar todos los datos en la RAM para poder analizarlos" — pero en este benchmark, pandas y polars YA tienen los datos cargados en RAM desde el inicio. DuckDB lee desde disco para cada query. El hecho de que DuckDB gane Q1 (0.032s) a pesar de leer desde disco contra pandas (0.061s) y polars (0.038s) con datos en memoria es el insight real — demuestra que para aggregaciones simples de GROUP BY, el motor vectorizado de DuckDB en C++ puede superar a pandas sobre datos en RAM. Esa es la conclusión correcta, y es la opuesta a lo que dice el reporte.

**Comparable (Q2):** La justificación de que Q2 es comparable porque agrupa sobre solo 10 categorías es razonable y correcta.

---

## Criterio 4 — Reporte y recomendación de arquitectura `18 / 25`

El reporte tiene todos los elementos requeridos: tabla comparativa, interpretaciones de EXPLAIN, tres casos de tradeoff, y recomendación. La recomendación final en la sección 5 es correcta en sus conclusiones: DuckDB para datos en disco y analítica SQL, polars para pipelines de transformación, pandas para muestras pequeñas y ecosistema científico.

Lo que baja la calificación es la profundidad. El reporte no analiza el impacto de la decisión de polars eager vs lazy — que es el factor más importante del benchmark y explica casi todos los resultados. No hay discusión sobre tracemalloc y sus limitaciones para DuckDB y polars. Y el análisis de los tres casos de tradeoff es más breve que lo que el ejercicio requiere — el "caso DuckDB ganador" tiene la explicación incorrecta, lo que sugiere que el análisis fue escrito sin revisar las condiciones del experimento.

---

## Sobre el uso de herramientas de IA

El código funciona y la estructura es sólida. El error factual en la explicación de Q1 — que dice que pandas y polars tienen que cargar datos en RAM cuando ya los tienen cargados — es la señal más clara de análisis generado sin revisar contra las condiciones reales del benchmark. Un análisis genuino habría detectado esa inconsistencia al leer el código del benchmark antes de escribir el reporte.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> En tu benchmark, polars tiene los datos cargados en RAM con `pl.read_parquet()`. Si cambiaras a `pl.scan_parquet()` en todas las funciones del polars engine, ¿cómo crees que cambiarían los resultados de Q5 específicamente? ¿Por qué polars con `scan_parquet()` podría acercarse o superar a DuckDB en Q5 cuando DuckDB lee desde disco?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 19 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 18 / 25 |
| Identificación y justificación de tradeoffs | 25% | 17 / 25 |
| Reporte y recomendación de arquitectura | 25% | 18 / 25 |
| **Total** | **100%** | **72 / 100** |

---

El código corre y los datos son reales. El paso siguiente es cerrar la brecha entre lo que el código hace y lo que el reporte dice que hace — en E03 eso va a ser igual de importante para el schema y los benchmarks de indexing.