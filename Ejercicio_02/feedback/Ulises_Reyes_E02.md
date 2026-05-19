# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumno:** Ulises Josue Reyes Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 95 / 100

---

## Resumen general

La entrega más técnicamente completa del grupo para este ejercicio. Las implementaciones de los tres engines resuelven problemas que otros alumnos dejaron abiertos: Q6 con desempate determinista en los tres engines, Q4 con las 24 horas garantizadas, Q8 con timestamps como strings para comparación directa. La observación sobre el cold start de pandas en Q1 es única. El único punto donde el trabajo muestra sus límites es en el nivel de AI del reporte escrito — más pronunciado que en E01.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `24 / 25`

Las implementaciones son las más cuidadas del grupo.

`pandas_engine.py` usa `_read_cols()` para column pruning en todas las queries que no necesitan todas las columnas. Eso replica en pandas la misma optimización que DuckDB y polars hacen automáticamente — y reduce el tiempo e I/O de pandas de forma medible, especialmente en Q1, Q3 y Q7. Los docstrings explican por qué Q5 no puede hacer column pruning (necesita todas las columnas para el SELECT *).

`polars_engine.py` resuelve el problema de Q6 correctamente: usa `sort + group_by(maintain_order=True) + first()` en lugar de `.over()`. Esto evita el riesgo de empate que tienen otras implementaciones del grupo, donde el `.filter(count == count.max().over(...))` devolvería múltiples filas si dos categorías empatan. La solución de sort + first es determinista y consistente con DuckDB.

`duckdb_engine.py` usa `RANK() OVER (ORDER BY tx_count DESC, category ASC)` con desempate alfabético explícito. Los tres engines tienen el mismo criterio de desempate en Q6 — lo que garantiza que la validación pase incluso cuando el dataset tiene empates reales.

Q4 garantiza las 24 horas en los tres engines: pandas con merge, polars con join sobre `all_hours`, DuckDB con `GENERATE_SERIES`. Q8 convierte timestamps a string `YYYY-MM-DD` en los tres engines antes de retornar, lo que evita problemas de comparación de tipo datetime entre pandas y polars.

Un punto menos: sys.path.insert y el benchmark usa promedio en lugar de mediana — el cold start queda incluido en el promedio. El reporte lo documenta, pero la metodología de Angel (5 iteraciones, descarta la primera, toma mediana) sería más rigurosa.

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `24 / 25`

Las tres interpretaciones son las más detalladas del grupo.

**Q3:** Identifica column pruning, la hash table de 50K grupos, y la heap mínima de TOP_N con complejidad O(n log 10). La comparación explícita con O(n log n) de un sort completo es exactamente el tipo de análisis que diferencia describir un plan de entenderlo.

**Q5:** Cuantifica exactamente: 73,704 filas materializadas de 1M — el 7.4% del archivo. Explica el dynamic filter cross-CTE y el row group filtering por estadísticas de min/max de Parquet. La diferencia de RAM entre pandas (65 MB), DuckDB (3.3 MB) y polars (0.7 MB) usada como confirmación del predicate pushdown es correcto.

**Q6:** La observación de que la hash table de 150 entradas cabe en la caché L2 del CPU es única en el grupo y correcta — operaciones de lookup sobre 150 entradas tienen latencia de ~1ns vs ~100ns si el acceso fuera a RAM. La conexión entre el doble criterio de ordenamiento en SQL y la consistencia de la validación (sin ese `category ASC`, los tres engines podrían retornar resultados distintos en empates) es el análisis más técnico del grupo en este criterio.

Un punto menos: el plan de Q6 muestra que DuckDB reemplazó internamente el `ROW_NUMBER()` con `arg_max_nulls_last`. Esa optimización — eliminar una window function y reemplazarla con una agregación de un solo paso — es la más interesante del plan y no aparece en la interpretación.

---

## Criterio 3 — Identificación y justificación de tradeoffs `24 / 25`

Los tres casos son empíricos y tienen la mejor cuantificación del grupo.

**Polars > pandas (Q7):** La explicación del pipeline lazy de polars — filter → group_by → filter ejecutado en un solo pase con predicate pushdown al Parquet — vs el proceso eager de pandas con copias intermedias, es correcta y específica.

**DuckDB > todos (Q8):** La mejor explicación del benchmark: pandas crea un objeto Python `datetime` por cada una de las 1M filas al ejecutar `dt.normalize()`, el proceso es secuencial porque Python no puede paralelizar la creación de objetos en el heap, y el resultado es 1.669s y 83 MB de RAM. DuckDB ejecuta `DATE_TRUNC` vectorizado en C++ sin objetos Python intermedios: 0.124s y 0.62 MB — una reducción de 134x en memoria. Conectar el mecanismo con los números es exactamente lo que se pide en este criterio.

**Comparable (Q5, ratio 1.3x):** Correctamente justificado como una query dominada por I/O donde los tres engines aplican predicate pushdown y el cuello de botella es la decodificación de bloques Parquet — similar para los tres porque todos usan Arrow internamente.

La observación sobre el cold start de pandas en Q1 — primera corrida 2x más lenta porque pyarrow inicializa el schema reader, las siguientes son calientes porque los metadatos quedan en caché — es la única en el grupo y demuestra lectura activa de los datos de repeticiones individuales.

---

## Criterio 4 — Reporte y recomendación de arquitectura `23 / 25`

El `generate_report.py` es el más sofisticado del grupo: genera las tablas, los ratios de speedup calculados dinámicamente desde el JSON, y las secciones de tradeoff con los números actualizados en cada ejecución. La sección de fondo técnico antes de los resultados — explicando cómo funciona cada engine internamente — ordena bien la lectura del reporte para alguien que no conoce los tres engines.

La recomendación final con bullet points diferenciados por caso de uso es accionable y técnicamente correcta.

Dos puntos que bajan la calificación. Primero, el reporte es muy extenso y el tono es uniformemente declarativo y técnico — más AI-asistido que E01, y más que el reporte de Ernesto de este ejercicio. Los typos en español ("profecional", "ams") sugieren revisión manual, pero la extensión y uniformidad del texto indica generación asistida sin mucha edición de fondo. Segundo, el reporte no cierra la pregunta pendiente de E01 sobre `tracemalloc` y DuckDB/polars — los valores de RAM de DuckDB y polars siguen sin explicación.

---

## Sobre el uso de herramientas de IA

El código es la parte más genuina de la entrega. Las decisiones de diseño — `_read_cols()` en pandas, `sort + maintain_order + first` en polars para Q6, `GENERATE_SERIES` en DuckDB para Q4, timestamps como string para comparación uniforme — requieren comprensión, no generación. Los docstrings que explican el razonamiento detrás de cada decisión son AI-asistidos en el estilo, pero las decisiones mismas son reales.

El reporte escrito es más AI-pesado que en E01. Úsalo como herramienta de estructura, no como autor — especialmente para las secciones de EXPLAIN ANALYZE que requieren que hayas visto el plan real.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> En Q6, el plan de DuckDB muestra que reemplazó internamente `ROW_NUMBER() OVER (...)` con `arg_max_nulls_last`. ¿Qué diferencia hay entre ambos en términos de complejidad algorítmica? ¿Por qué `arg_max` es más eficiente que una window function con ROW_NUMBER para este caso específico?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 24 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 24 / 25 |
| Identificación y justificación de tradeoffs | 25% | 24 / 25 |
| Reporte y recomendación de arquitectura | 25% | 23 / 25 |
| **Total** | **100%** | **95 / 100** |

---

Las implementaciones de los tres engines son las más correctas del grupo. La diferencia entre 95 y 100 está en el reporte — el análisis escrito no está a la altura del código. Para E03, ese mismo rigor de diseño que aplicaste en los engines (Q6 con desempate determinista, Q4 con 24 horas garantizadas) es exactamente el que necesitas para el schema de SQLite: diseñar para los patrones de acceso, no para lo que parece razonable a priori.