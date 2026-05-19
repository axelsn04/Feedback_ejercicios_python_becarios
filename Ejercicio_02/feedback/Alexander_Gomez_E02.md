# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumno:** Bryan Alexander Gomez Miranda
**Fecha de revisión:** Mayo 2026
**Calificación:** 86 / 100

---

## Resumen general

Entrega completa con resultados interesantes y análisis honesto. El benchmark produce el patrón más distinto del grupo: pandas gana Q3 y Q5 porque el DataFrame está pre-cargado en RAM, mientras DuckDB y polars leen desde disco en cada llamada. El reporte lo identifica y lo explica correctamente — eso es más valioso que tener resultados donde DuckDB gana todo sin entender por qué. Hay dos problemas técnicos en el código que vale la pena corregir antes del siguiente ejercicio.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `19 / 25`

Las 8 queries están implementadas en los tres engines y la validación reporta ✅ 8/8.

**Problema 1: pandas muta el DataFrame compartido.** `q5` ejecuta `df["timestamp"] = pd.to_datetime(df["timestamp"])` sobre el DataFrame que recibe como argumento — el mismo que comparten todas las queries del benchmark. `q8` agrega una columna `"date"` al mismo DataFrame. Estas operaciones modifican el estado del objeto en memoria de forma persistente: después de que `q5` corre por primera vez, el DataFrame ya tiene `timestamp` como datetime64 en lugar de string/timestamp original. Esto significa que las mediciones de tiempo para `q5` en la segunda y tercera repetición son distintas a la primera — la primera incluye la conversión, las siguientes no. El promedio de 3 runs no mide lo mismo en los tres runs. La corrección es trabajar sobre una copia: `tmp = df.copy()` o `pd.to_datetime(df["timestamp"])` sin reasignación.

**Problema 2: polars usa `read_parquet()` en lugar de `scan_parquet()`.** `pl.read_parquet()` carga el archivo completo en memoria de forma eager antes de ejecutar cualquier operación. `pl.scan_parquet()` es lazy — construye el plan de ejecución y solo lee las columnas y filas necesarias cuando llamas `.collect()`. La consecuencia es que polars pierde su principal ventaja en este benchmark: predicate pushdown y column pruning al leer. El reporte lo reconoce explícitamente, lo cual está bien.

**Riesgo en polars Q6:** El código hace `.sort("transaction_count", descending=True).group_by("country_code").agg([pl.col("category").first()...])`. En polars, `group_by()` sin `maintain_order=True` no garantiza que las filas dentro de cada grupo estén en el orden del sort previo. El `.first()` puede retornar una fila arbitraria, no necesariamente la de mayor conteo. Si la validación pasó es por coincidencia del dataset o por la tolerancia de la función de normalización. El fix es `group_by("country_code", maintain_order=True)` después del sort, o usar la estrategia que usó Ulises.

**Punto positivo:** el diseño asimétrico — pandas pre-cargado vs DuckDB/polars desde disco — produce el caso más didáctico del grupo: pandas puede superar a DuckDB cuando los datos ya están en RAM. El reporte lo capitaliza bien.

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `22 / 25`

Las tres interpretaciones son correctas y tienen la observación más importante de este ejercicio.

**Q3:** Identifica column pruning, HASH_GROUP_BY para 50K grupos, TOP_N como heap de 10 elementos. La explicación de por qué pandas gana Q3 — el DataFrame ya está en RAM, el `groupby` sobre arrays numpy es más barato que el overhead de abrir el archivo Parquet, leerlo y ejecutar el plan — es la más clara del grupo para este caso específico. Mencionar que DuckDB está optimizado para datos en disco y que su ventaja desaparece cuando los datos ya están cacheados es exactamente el punto.

**Q5:** Identifica predicate pushdown con `amount > 500`, el dynamic filter cross-CTE para el timestamp cutoff, y cuantifica: 73,704 de 1M filas materializadas. La explicación adicional de por qué pandas gana a pesar de la optimización de DuckDB — costo de materializar y transferir 9,883 filas de vuelta a pandas — es un nivel de detalle que no aparece en otras entregas. Correcto.

**Q6:** Identifica los dos niveles de HASH_GROUP_BY, la función `arg_max_nulls_last` que DuckDB usa para reemplazar ROW_NUMBER, y la reducción de 150 a 15 filas. Correcto pero más breve que las otras dos interpretaciones — falta explicar qué significa `arg_max` como mecanismo (opera en un solo pase sobre los 150 grupos sin construir la tabla de rankings completa).

---

## Criterio 3 — Identificación y justificación de tradeoffs `23 / 25`

Los tres casos son empíricos y el razonamiento es sólido.

**Pandas > DuckDB (Q3):** Correcto y bien argumentado. Pandas tiene el DataFrame en RAM, el groupby sobre numpy es O(n) sin overhead de I/O. DuckDB paga el costo de abrir el archivo en cada llamada. La observación de que con una sola medición el page cache podría haber favorecido a DuckDB es correcta — por eso el promedio de 3 runs es importante.

**DuckDB ganador claro (Q8):** Correcto. 9x más rápido que pandas. La explicación sobre `dt.date` generando objetos Python `datetime.date` por cada fila (costosos en tiempo y memoria) vs DuckDB ejecutando `CAST(timestamp AS DATE)` vectorizado en C++ sin objetos Python es precisa. Los números lo confirman: 0.701s y 117 MB para pandas, 0.078s y 0.42 MB para DuckDB.

**Tres comparables (Q2):** Razonable. Pandas 0.049s, DuckDB 0.047s — prácticamente idénticos. La justificación de que Q2 opera sobre 10 grupos con una sola columna numérica — el caso más simple para cualquier motor de agregación — es correcta. La observación de que polars es más lento porque recarga el Parquet desde disco en lugar de usar el cache del DataFrame en RAM también es correcta.

---

## Criterio 4 — Reporte y recomendación de arquitectura `22 / 25`

El reporte tiene voz propia y es honesto sobre las limitaciones del diseño del benchmark. La sección de polars explica correctamente que `scan_parquet()` habría sido la implementación correcta — el alumno identificó el problema y lo documentó en lugar de ignorarlo. La recomendación diferencia bien los tres casos: DuckDB para archivos en disco, pandas cuando el DataFrame ya está en RAM, polars para pipelines lazy con múltiples pasos.

Dos puntos que faltan. Primero, el footer del reporte dice "*El pico de memoria se registró en la primera ejecución con tracemalloc*" — lo cual es honesto, pero el código tiene un problema adicional: hay una línea de código muerta en `measure()`. El `return` correcto está seguido de otro `return` inalcanzable. No afecta la ejecución pero indica que el código no fue revisado completamente antes de entregar. Segundo, no hay discusión sobre las limitaciones de `tracemalloc` para DuckDB y polars — los valores de RAM de DuckDB (0.42 MB en Q8, por ejemplo) no representan el uso real de memoria del proceso.

---

## Sobre el uso de herramientas de IA

El benchmark produce resultados distintos al resto del grupo (pandas gana Q3 y Q5) y el reporte los explica correctamente. Esa explicación — que depende de entender la diferencia entre datos en RAM vs datos en disco — no es genérica. El bug de mutación del DataFrame en pandas también sugiere código escrito sin revisión completa, lo que es una señal de trabajo propio. El tono del reporte es más declarativo que conversacional, con alguna asistencia de IA en la prosa, pero las observaciones técnicas son genuinas.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> En `pandas_engine.py`, `q5` y `q8` modifican el DataFrame compartido que reciben como argumento. Si en el benchmark `q5` corre antes que `q8`, ¿qué columna extra tiene el DataFrame cuando `q8` empieza a ejecutarse? ¿Cómo afecta eso a las mediciones de tiempo del run 2 y 3 de `q8`?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 19 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 22 / 25 |
| Identificación y justificación de tradeoffs | 25% | 23 / 25 |
| Reporte y recomendación de arquitectura | 25% | 22 / 25 |
| **Total** | **100%** | **86 / 100** |

---

El análisis de por qué pandas gana Q3 y Q5 es el más claro del grupo para ese caso. Para E03, ese mismo instinto — entender el costo real de cada operación según dónde viven los datos — es exactamente lo que vas a necesitar para diseñar los índices de SQLite.