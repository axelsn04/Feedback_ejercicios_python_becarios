# Retroalimentación — Ejercicio 02: El Motor de Consultas

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 93 / 100

---

## Resumen general

Entrega sólida en todos los criterios. El benchmark tiene validación numérica robusta, DuckDB lee desde Parquet sin cargarlo a memoria, y el análisis de EXPLAIN ANALYZE muestra lectura activa de los planes — no descripción genérica. El insight más valioso del reporte es la pregunta que te haces sobre Q3: por qué DuckDB es más lento que pandas y polars si el benchmark corre sobre datos en RAM. Que hayas identificado eso y lo hayas explicado correctamente dice más sobre tu comprensión del ejercicio que cualquier tabla de resultados.

---

## Criterio 1 — 8 queries en 3 engines, numéricamente validadas `23 / 25`

Las 8 queries están implementadas en los tres engines. La función `validate()` en `benchmark.py` es la más robusta del grupo: convierte todo a pandas, ordena por todas las columnas antes de comparar para eliminar diferencias de orden, y usa `np.allclose` con tolerancia `rtol=1e-3` para columnas numéricas. Eso maneja correctamente las diferencias de punto flotante que aparecen al comparar resultados de engines distintos. DuckDB opera vía `CREATE VIEW` sobre el Parquet sin cargarlo a memoria — cumple la restricción del ejercicio.

El único punto de riesgo está en `polars_engine.py` para Q6:

```python
.filter(
    pl.col('count') == pl.col('count').max().over('country_code')
)
```

Si dos categorías tienen exactamente el mismo conteo para un país, este filtro devuelve ambas filas. `ROW_NUMBER() OVER` en DuckDB siempre elige una. Si el dataset tiene esos empates, la validación de Q6 fallará — no por un error de lógica sino por una diferencia en el desempate. Es un caso de borde pero en un dataset de 1M filas con distribución uniforme puede ocurrir.

---

## Criterio 2 — Interpretación de EXPLAIN ANALYZE (Q3, Q5, Q6) `24 / 25`

Las tres interpretaciones son correctas y tienen evidencia de lectura real del plan.

**Q3:** Identificas el projection pushdown (solo `user_id` y `amount` leídos del Parquet), el `HASH_GROUP_BY` como cuello de botella con 50,000 grupos, y el `TOP_N` con heap de 10 elementos. La pregunta que te haces — por qué DuckDB es más lento que pandas y polars en esta query — y la respuesta que das son exactamente correctas: cuando los datos ya están en RAM, el overhead de inicialización del pipeline de DuckDB es visible y su ventaja de I/O desaparece. DuckDB está optimizado para cuando los datos viven en disco.

**Q5:** Identificas el predicate pushdown con los filtros aplicados durante la lectura del Parquet, el dynamic filter cross-CTE que inyecta la fecha de corte calculada antes de leer el archivo, y cuantificas el resultado: DuckDB procesó el 7.4% del archivo. La observación sobre el `NESTED_LOOP_JOIN` de 1 fila siendo esencialmente un filtro disfrazado de join es correcta — con un solo elemento en el lado derecho el costo es mínimo.

**Q6:** Detectas la optimización más avanzada de las tres: DuckDB eliminó el `ROW_NUMBER() OVER` que escribiste en el SQL y lo reemplazó internamente con `arg_max_nulls_last`. Esa observación requiere haber comparado el SQL original con el plan de ejecución. El mecanismo que describes — empaquetar el struct, encontrar el máximo por grupo, desempaquetar — es correcto.

Un punto menos porque en la sección de memoria no llegaste a identificar la herramienta correcta para medir RAM en DuckDB y Polars. La respuesta es `psutil.Process().memory_info().rss` tomado antes y después de la operación.

---

## Criterio 3 — Identificación y justificación de tradeoffs `22 / 25`

El ejercicio pide identificar tres casos empíricos: una query donde polars gana, una donde DuckDB gana claramente, y una donde los tres son comparables. El problema es que polars ganó las 8 queries de este benchmark, así que los dos últimos casos no aparecen en los datos.

El reporte lo maneja con honestidad: explica que DuckDB no ganó porque los datos ya estaban en RAM y documenta sus ventajas en el escenario correcto. Eso es razonamiento correcto, pero el ejercicio pedía evidencia del benchmark, no argumentos sobre escenarios alternativos.

La forma de haberlo resuelto sería diseñar una prueba adicional donde DuckDB leyera el Parquet en frío — sin haberlo cargado con `pl.read_parquet()` primero — para mostrar su ventaja real en Q5 sobre datos en disco. Esa prueba habría completado el análisis empírico que faltó.

---

## Criterio 4 — Reporte y recomendación de arquitectura `24 / 25`

La nota sobre `tracemalloc` es la mejor explicación de este módulo hasta ahora: pandas construye en el heap de Python (visible para tracemalloc), DuckDB procesa en C++ (invisible), polars usa Rust fuera del heap de Python (invisible). Correcta, bien argumentada, y honesta sobre no conocer la herramienta correcta para medir los otros dos — que es `psutil`.

La tabla de decisión al final es práctica: cubre los casos relevantes de producción y los conecta con el engine correcto sin sobre-simplificar. "Datos en disco, no caben en RAM → DuckDB" y "Integración con scikit-learn → Pandas" son las dos entradas más importantes y están correctas.

Un punto menos porque el reporte no incluye ninguna gráfica. Con datos de 8 queries y 3 engines, una visualización habría hecho la comparación más clara — especialmente para mostrar que Q8 es el outlier de pandas (0.7s vs 0.04s de DuckDB y polars).

---

## Sobre el uso de herramientas de IA

El reporte tiene voz propia: "Aquí me surge una duda", "Si no estoy mal", "Honestamente, no estoy 100% seguro", "En mi opinión". Las interpretaciones de EXPLAIN ANALYZE conectan con observaciones específicas del plan, no con descripciones genéricas del tema. El razonamiento sobre Q3 — identificar el problema, formularlo como pregunta, y responderlo con el mecanismo correcto — es el tipo de análisis que no se genera por inercia. Uso inteligente.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 3, responde esto en el canal:

> En Q5, DuckDB procesó el 7.4% del archivo gracias al predicate pushdown y el dynamic filter. Pero en este benchmark los datos se leyeron desde un Parquet que ya estaba en disco caliente (el sistema operativo tenía las páginas en cache). ¿Cómo cambiaría ese 7.4% si el Parquet estuviera en S3 con latencia de red? ¿Y si el archivo tuviera 10 millones de filas en lugar de 1 millón?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 8 queries en 3 engines, validadas | 25% | 23 / 25 |
| Interpretación de EXPLAIN ANALYZE | 25% | 24 / 25 |
| Identificación y justificación de tradeoffs | 25% | 22 / 25 |
| Reporte y recomendación de arquitectura | 25% | 24 / 25 |
| **Total** | **100%** | **93 / 100** |

---

El análisis de Q3 — identificar por qué DuckDB pierde cuando los datos están en RAM y explicar el mecanismo correctamente — es exactamente el tipo de pensamiento que desarrolla este módulo. Para el Ejercicio 3 vas a necesitar ese mismo instinto: los índices de SQLite tienen overhead en escritura, y entender cuándo ese costo vale la pena requiere el mismo razonamiento que aplicaste aquí.