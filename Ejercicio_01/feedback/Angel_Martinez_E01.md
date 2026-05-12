# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumno:** Angel Martinez Aguilar
**Fecha de revisión:** Mayo 2026
**Calificación:** 80 / 100

---

## Resumen general

Entrega completa con una decisión de arquitectura que se distingue del resto del grupo: separar la generación del reporte (`generate_report.py`) de la ejecución del benchmark. Las tablas y gráficas se generan automáticamente desde los JSONs de resultados, y las conclusiones escritas se preservan entre corridas. Eso es pensar el sistema, no solo escribir código que funciona. El análisis escrito también es de los más sólidos del grupo — hay observaciones originales con números concretos. Lo que baja la calificación es el rigor del benchmark: las lecturas no se repiten y hay detalles en las escrituras que afectan la validez de las mediciones.

---

## Criterio 1 — CLI funcional y módulo organizado `16 / 20`

La estructura de un archivo por formato (`csv_benchmark.py`, `jsonl_benchmark.py`, `parquet_benchmark.py`) es una decisión válida — cada formato encapsula su propia lógica de medición. `gc.collect()` se llama entre formatos en el CLI, lo que reduce la contaminación cruzada entre benchmarks. El `generate_report.py` como script separado que auto-genera tablas y gráficas desde los JSONs, y preserva las conclusiones escritas entre corridas, es la decisión de diseño más inteligente del grupo.

Dos puntos que afectan la calificación. Primero, `generate_data.py` no tiene parámetro `--seed` — los resultados no son reproducibles: dos corridas del mismo benchmark pueden dar datasets distintos y resultados ligeramente diferentes. Para un benchmark que pretende ser evidencia técnica, la reproducibilidad importa. Segundo, las escrituras en `parquet_benchmark.py` se repiten 3 veces pero sin borrar el archivo entre runs y sin `gc.collect()` entre cada repetición — la segunda y tercera escritura operan sobre un archivo que el sistema operativo puede tener en cache, lo que sesga el promedio a la baja.

---

## Criterio 2 — Rigor del benchmark `17 / 25`

La escritura tiene 3 repeticiones, lo que es correcto en intención. Los problemas de implementación ya mencionados — sin borrado de archivo entre runs, sin `gc.collect()` entre repeticiones de escritura — afectan la validez de esos promedios.

El problema más serio: **la lectura completa y la lectura selectiva son mediciones únicas**. Una sola corrida de lectura puede ser fría (archivo no en cache del SO) o caliente (archivo en cache de una corrida anterior) dependiendo del orden de ejecución, y no hay forma de saber cuál fue. El tiempo de JSONL en 1M de filas reportado es 24.23 segundos — los otros alumnos del grupo obtuvieron entre 3 y 6 segundos para el mismo dataset. Ese número puede ser real si el archivo estaba frío en un disco lento, pero con una sola medición no hay forma de distinguir un outlier de la medición real.

Adicionalmente, la memoria solo se mide en lectura completa — no en lectura selectiva. Para Parquet, donde la lectura selectiva es uno de los casos de uso más relevantes, no tener ese dato es una omisión.

---

## Criterio 3 — Análisis por escala `22 / 25`

El análisis escrito es el más original del grupo. Tres observaciones que se distinguen:

La sección de escalabilidad extrapola a 10 millones de filas: si JSONL necesita ~2.45 GB para 1M de registros, a 10M necesitaría ~25 GB — inviable en hardware estándar. Esa proyección no es un cálculo complejo, pero requiere haber pensado en el dato, no solo reportarlo.

La observación al final del análisis — que el crecimiento en memoria y almacenamiento es proporcional al crecimiento de datos (x10), pero que el tiempo de Parquet crece menos que esa proporción — es el tipo de insight que demuestra que leíste tus propios resultados con atención. Parquet Snappy pasa de 0.01s a 0.12s de lectura al ir de 100K a 1M filas: eso es un factor 12x para 10x de datos, pero en el orden de milisegundos. CSV pasa de 0.17s a 1.78s: factor 10.5x, pero en el orden de segundos. La diferencia importa en producción.

Lo que falta para el puntaje completo: los typos en el texto ("reelevantes", "a pensar de") muestran que fue escrito por ti, lo cual es positivo, pero el reporte tiene secciones donde la profundidad técnica varía — las primeras dos secciones explican bien los mecanismos, la sección de compresión es más superficial y no explica por qué Snappy es más rápido que Gzip a nivel de diseño del algoritmo.

---

## Criterio 4 — Reporte y conclusiones `25 / 30`

El `generate_report.py` que auto-genera tablas y gráficas desde los JSONs y preserva las conclusiones escritas es la mejor decisión de diseño del ejercicio. El reporte siempre está sincronizado con los datos sin tener que reescribir manualmente las tablas.

Las gráficas usan escala lineal. Con datos donde Parquet y JSONL difieren dos órdenes de magnitud, en escala lineal las barras de Parquet son prácticamente invisibles en algunas gráficas. Escala logarítmica con una nota explicando por qué es la decisión correcta para este tipo de comparaciones.

Los valores de memoria para Parquet son prácticamente idénticos al tamaño del archivo en disco — el mismo patrón que en otras entregas del grupo. El reporte no lo investiga ni lo menciona. Esa coincidencia es una señal de que `tracemalloc` no está midiendo la memoria real de pyarrow, y reportar esos números como válidos sin ninguna advertencia es un problema técnico.

La recomendación final es correcta y tiene justificación cuantitativa, que es lo que se pide.

---

## Sobre el uso de herramientas de IA

Los typos en el reporte ("reelevantes", "a pensar de") son una señal clara de escritura propia — los modelos de lenguaje rara vez producen esos errores específicos en español. La extrapolación a 10M de filas y la observación sobre la proporción de crecimiento tampoco son el tipo de contenido que aparece en un reporte generado sin revisión. El código tiene asistencia de IA evidente en la estructura y los docstrings, pero las decisiones de diseño — un archivo por formato, `generate_report.py` separado — muestran criterio propio. Uso de IA como herramienta, no como autor.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en el canal:

> En tu benchmark, las lecturas son mediciones únicas. El tiempo de JSONL en 1M de filas fue 24.23s — otros alumnos obtuvieron entre 3 y 6 segundos para el mismo dataset. ¿Cómo sabes si ese número es representativo o fue una medición fría atípica? ¿Qué cambiarías en la implementación para tener certeza?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 16 / 20 |
| Rigor del benchmark | 25% | 17 / 25 |
| Análisis por escala | 25% | 22 / 25 |
| Reporte y conclusiones | 30% | 25 / 30 |
| **Total** | **100%** | **80 / 100** |

---

El análisis es sólido y el `generate_report.py` es la mejor decisión de diseño del grupo. Para el siguiente ejercicio enfócate en el rigor de las mediciones — todas las métricas deben repetirse, no solo la escritura.