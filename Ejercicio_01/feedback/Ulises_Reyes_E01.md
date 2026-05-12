# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumno:** Ulises Josue Reyes Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 69 / 100

---

## Resumen general

El ejercicio está completo en cuanto a entregables: tienes tablas para las tres escalas, gráficas, código funcional y un reporte con conclusiones. El benchmark corre y los datos son reales. Dicho esto, hay dos problemas concretos que bajan la calificación de forma significativa: el análisis por escala no analiza realmente cómo cambia el comportamiento al escalar, y hay una anomalía en los valores de memoria de Parquet que quedó sin investigar. Ambos son solucionables y son exactamente el tipo de cosa que el siguiente ejercicio te va a volver a pedir.

---

## Criterio 1 — CLI funcional y módulo organizado `15 / 20`

La estructura tiene más archivos que cualquier otro alumno del grupo: `constants.py`, `data_generator.py`, `formats.py`, `metrics.py`, `runner.py`, `report.py`. Eso no es un problema en sí — más separación puede ser buena arquitectura. El problema es que algunas de esas abstracciones no agregan valor real.

`FormatHandler` es una clase donde todos los métodos son `@staticmethod`. Una clase con todos sus métodos estáticos es básicamente un módulo con nombre de clase — no hay estado, no hay herencia, no hay razón para que sea una clase. Las funciones sueltas en un módulo habrían hecho lo mismo con menos ruido.

El problema más concreto: `repeat_measurement` está definida en `metrics.py` pero **nunca se llama**. `runner.py` hace su propio loop con `measure_time` directamente. Tener una función definida que no se usa indica que el módulo y el runner no se diseñaron juntos — uno fue escrito sin saber qué iba a hacer el otro.

Otros detalles: `benchmark_cli.py` tiene `sys.path.insert(0, ...)` en lugar de un proyecto correctamente configurado como paquete, y `get_writer` usa lambdas mientras `get_reader_full` usa referencias directas sin razón para la inconsistencia. No hay `gc.collect()` antes de las mediciones.

---

## Criterio 2 — Rigor del benchmark `20 / 25`

El punto más sólido del ejercicio: **todas las mediciones se repiten 3 veces** — escritura, lectura completa, lectura selectiva y memoria — y se reporta el promedio. Eso es correcto y más riguroso que algunos otros alumnos del grupo.

Hay dos cosas que afectan el rigor. Primera: no hay `gc.collect()` antes de las mediciones. El garbage collector de Python puede dispararse en cualquier momento durante una medición y añadir tiempo que no corresponde al formato que estás midiendo. Agregarle `gc.collect()` antes de cada medición reduce ese ruido.

Segunda, y más importante: los valores de memoria para Parquet son prácticamente idénticos al tamaño del archivo en disco.

| Formato | Tamaño en disco | Memoria reportada |
|---|---|---|
| parquet | 6.59 MB | 6.60 MB |
| parquet_snappy | 5.76 MB | 5.77 MB |
| parquet_gzip | 4.03 MB | 4.04 MB |

Esa coincidencia exacta es una señal de que `tracemalloc` no está midiendo lo mismo para Parquet que para CSV o JSON. No es un error de tu código necesariamente — es una limitación de `tracemalloc` con los buffers de C de pyarrow. Pero el reporte no lo menciona, no lo investiga, y reporta esos números como válidos junto a los de CSV sin ninguna advertencia. Un número sin contexto puede ser peor que no tener el número.

---

## Criterio 3 — Análisis por escala `13 / 25`

Este es el criterio más importante del ejercicio y donde hay el problema más serio.

La siguiente frase aparece textualmente en las tres secciones de escala — 100k, 500k y 1m — con solo el número cambiado:

> *"Con X filas, el patrón ya es suficientemente claro para tomar decisiones de producción: el costo de texto plano en CSV/JSON empieza a dominar sobre la simplicidad del formato."*

Cambiar el número no es analizar por escala. El punto del ejercicio es observar **cómo cambia el comportamiento** cuando los datos crecen: que la ventaja de Parquet en lectura selectiva no se cierra al escalar sino que se amplía en términos absolutos, que JSONL crece en memoria de forma que a 1M de filas ya no es viable en hardware estándar, que la curva de escritura de Gzip tiene una pendiente diferente a la de Snappy a medida que escalan las filas. Esas observaciones no están en el reporte.

La sección "Lectura transversal" lista ganadores por categoría, pero eso es un resumen de la tabla — no un análisis. Analizar es explicar por qué el ganador gana y qué significa ese resultado para una decisión de arquitectura real.

---

## Criterio 4 — Reporte y conclusiones `21 / 30`

Las tablas están completas y los datos son correctos. Las conclusiones llegan a la recomendación correcta — Parquet+Snappy para producción — y el razonamiento general sobre el tradeoff Snappy vs Gzip está bien.

Dos cosas concretas a mejorar:

**Las gráficas usan escala lineal.** Con datos donde Parquet y JSON difieren dos órdenes de magnitud, en escala lineal las barras de Parquet desaparecen visualmente. En la gráfica de 100k prácticamente no se distinguen. Escala logarítmica con una nota explicando por qué la usas es la decisión correcta para este tipo de datos.

**Las conclusiones no cuantifican.** "Parquet reduce significativamente el tiempo de lectura" no es una conclusión técnica — es una descripción. "Parquet+Snappy reduce el tiempo de lectura selectiva en 93% frente a CSV a 1M de filas, con un archivo 47% más pequeño" sí lo es. Para el siguiente ejercicio, cuando hagas recomendaciones, apoya cada una con el número que la justifica.

---

## Sobre el uso de herramientas de IA

El código tiene evidencia de trabajo propio: el `BenchmarkRunner` como clase, la separación en `constants.py`, la decisión de repetir memoria también. El reporte es donde el uso de IA se nota más. La repetición verbatim de la misma frase en las tres secciones de escala es una señal clara de que el análisis fue generado con una plantilla o prompt sin revisar el output — un modelo de lenguaje tiende a repetir estructuras cuando se le pide que repita la misma tarea tres veces. 

Usar IA para estructurar o revisar está bien. Usarla para generar el análisis sin leer lo que produjo no — porque el objetivo del ejercicio es que desarrolles el criterio para leer datos y sacar conclusiones, no que tengas un reporte con el formato correcto.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en el canal:

> Los valores de memoria que `tracemalloc` reportó para Parquet son prácticamente idénticos al tamaño del archivo en disco. ¿Por qué ocurre eso? ¿Qué está midiendo realmente `tracemalloc` en ese caso, y qué herramienta usarías para medir la memoria RSS real del proceso?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 15 / 20 |
| Rigor del benchmark | 25% | 20 / 25 |
| Análisis por escala | 25% | 13 / 25 |
| Reporte y conclusiones | 30% | 21 / 30 |
| **Total** | **100%** | **69 / 100** |

---

El código funciona y los datos son reales — esa es la base. Lo que falta es la capa de análisis: leer tus propios resultados con ojo crítico y extraer conclusiones que no estaban obvias antes de correr el benchmark. Eso es lo que practica el siguiente ejercicio.