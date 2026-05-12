# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 88 / 100

---

## Resumen general

Entrega sólida en todos los criterios. El benchmark tiene el rigor correcto: lecturas repetidas 3 veces con promedio, escritura también promediada, separación clara entre módulos de escritura y lectura. El reporte es el más natural del grupo — tiene voz propia, usa números concretos para justificar cada argumento, y la sección de escalabilidad incluye el insight más original que vi en este ejercicio. Lo que falta es menor en comparación con lo que está bien.

---

## Criterio 1 — CLI funcional y módulo organizado `17 / 20`

La separación entre `writers.py` y `readers.py` es limpia y funcional. El CLI genera el dataset una sola vez antes del loop de formatos, lo cual es correcto. `parse_size()` como función separada que maneja el formato flexible de entrada (100k, 500k, 1m) es una buena decisión — cualquier otro script puede reutilizarla. Sin `sys.path` hacks, con `pathlib` en todo el código.

Tres puntos menores. Primero, `generate_data.py` no tiene parámetro `--seed`: los resultados no son reproducibles entre corridas — dos ejecuciones del mismo benchmark pueden dar tiempos ligeramente distintos por variabilidad en los datos generados. Segundo, la generación de timestamps usa un list comprehension que itera fila por fila — para 1M de filas hay alternativas vectorizadas con `pd.to_timedelta` que son significativamente más rápidas. Tercero, la memoria solo se mide en lectura completa, no en lectura selectiva.

---

## Criterio 2 — Rigor del benchmark `21 / 25`

Este es el benchmark más riguroso del grupo junto con Antonio. Lecturas completas: 3 repeticiones con promedio. Lecturas selectivas: 3 repeticiones con promedio. Para la memoria, tomas el máximo entre las repeticiones en lugar del promedio — eso es más conservador y más correcto: el pico real es el peor caso, no el caso promedio.

Dos detalles que faltan. Primero, no hay `gc.collect()` antes de las mediciones — el garbage collector puede dispararse durante una lectura y añadir tiempo que no corresponde al formato que se está midiendo. Es una línea de código que reduce el ruido. Segundo, la memoria solo se captura en lectura completa. Para Parquet, donde la lectura selectiva es uno de los casos de uso más relevantes, no medir su consumo de RAM es un dato que queda sin evidencia.

---

## Criterio 3 — Análisis por escala `24 / 25`

La sección de escalabilidad tiene el mejor insight del grupo en este ejercicio:

> *"En lectura selectiva, el tiempo solo subió unas 2.5 veces para manejar 10 veces más datos. Esto sugiere que gran parte del tiempo se va en 'abrir el archivo' y leer la configuración inicial; una vez que arranca, procesar más volumen no le cuesta tanto trabajo extra."*

Eso es correcto y no es una observación obvia. Parquet tiene overhead de inicialización — leer el footer de metadatos, inicializar el motor de pyarrow — que es fijo independientemente del tamaño del archivo. Una vez que ese overhead está pagado, procesar más filas cuesta proporcionalmente menos. CSV no tiene ese overhead, pero tampoco tiene la eficiencia de column pruning, por eso escala linealmente.

La observación sobre JSONL — que pandas crea objetos intermedios durante el parsing que saturan la memoria — también está bien argumentada con los números.

Un punto menos porque la sección de compresión (Gzip vs Snappy) se queda en describir el tradeoff sin explicar el mecanismo: por qué Gzip es más lento para escribir. El algoritmo LZ77 de Gzip hace múltiples pasadas buscando secuencias repetidas en el buffer — es secuencial y CPU-bound por diseño. Snappy sacrifica ratio de compresión por velocidad de throughput. Con ese detalle la sección habría estado completa.

---

## Criterio 4 — Reporte y conclusiones `26 / 30`

Las dos gráficas agrupan las tres escalas en una sola visualización por métrica — esa es la decisión correcta para mostrar comportamiento de escalabilidad. El análisis escrito tiene voz propia: "Lo curioso es Parquet", "Tengo entendido que pandas tiene que crear muchos objetos intermedios", "yo recomendaría" — eso indica que el análisis fue escrito y revisado, no generado. Los números están usados correctamente como evidencia: "12 veces más rápido que el CSV y más de 160 veces más rápido que el JSONL" — ese tipo de cuantificación es lo que se espera.

Dos puntos que afectan la calificación. Primero, los valores de memoria para Parquet son prácticamente iguales al tamaño del archivo en disco — el reporte no lo investiga ni lo menciona. Esa coincidencia es una señal de limitación de `tracemalloc` con los buffers de C de pyarrow, y reportar esos números sin contexto puede llevar a conclusiones incorrectas sobre el consumo real de memoria. Segundo, no se puede ver si las gráficas usan escala logarítmica — con diferencias de dos órdenes de magnitud entre JSONL y Parquet, en escala lineal las barras de Parquet son prácticamente invisibles.

---

## Sobre el uso de herramientas de IA

El reporte es el más genuino del grupo. Frases como "Lo curioso es Parquet" o "Tengo entendido que pandas tiene que crear muchos objetos intermedios" tienen el tono de alguien que está procesando los resultados en tiempo real, no de un texto generado. Los números están usados como argumentos propios, no como decoración. El código tiene asistencia de IA evidente en la limpieza y estructura, pero las decisiones de diseño — `parse_size()` flexible, `max` en lugar de promedio para el pico de memoria — son decisiones que requieren criterio. Uso inteligente.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en el canal:

> En tu análisis mencionas que Parquet tiene overhead de inicialización que explica por qué escala mejor que linealmente en lecturas selectivas. Ese overhead viene de leer el **footer** del archivo Parquet. ¿Qué información guarda ese footer y por qué es fundamental para que funcione el column pruning?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 17 / 20 |
| Rigor del benchmark | 25% | 21 / 25 |
| Análisis por escala | 25% | 24 / 25 |
| Reporte y conclusiones | 30% | 26 / 30 |
| **Total** | **100%** | **88 / 100** |

---

Buen trabajo. El insight de escalabilidad de Parquet es exactamente el tipo de observación que se busca en este módulo. Para el Ejercicio 2, esa misma mentalidad — buscar el mecanismo detrás del número — te va a servir para entender predicate pushdown en DuckDB.