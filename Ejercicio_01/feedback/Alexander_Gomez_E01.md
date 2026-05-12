# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumno:** Bryan Alexander Gomez Miranda
**Fecha de revisión:** Mayo 2026
**Calificación:** 85 / 100

---

## Resumen general

Entrega con el reporte técnico más detallado del grupo y una arquitectura de código que muestra criterio real de ingeniería: manejo de errores, modo headless para matplotlib, `FORMAT_REGISTRY` como estructura extensible, y la documentación del OOM Killer para JSONL en 1M de filas como un hallazgo de sistema. Hay un problema concreto que baja la calificación de forma significativa: el reporte afirma que todas las métricas se miden con 3 repeticiones y promedio, pero el código hace lo contrario — solo las escrituras se repiten. Esa inconsistencia entre lo que el reporte dice y lo que el código hace es el punto más importante de esta revisión.

---

## Criterio 1 — CLI funcional y módulo organizado `18 / 20`

`FORMAT_REGISTRY` en `runner.py` es la mejor implementación de este patrón en el grupo: cada formato está registrado con su writer, reader completo, reader selectivo y extensión en una sola entrada de diccionario. Agregar un nuevo formato es una sola línea. El manejo de errores en `benchmark_all` — que captura `MemoryError` y excepciones generales, registra el error en el resultado y continúa con los demás formatos — es exactamente el comportamiento correcto para un benchmark que puede fallar en condiciones de estrés. El backend `Agg` para matplotlib en modo headless es un detalle que la mayoría ignora y que evita errores en servidores sin display.

Dos puntos menores. El `sys.path.insert(0, str(PROJECT_DIR))` en `benchmark_cli.py` es el mismo hack de importación que aparece en varias entregas del grupo — el proyecto debería estar configurado como paquete instalable con `uv` o `poetry`. Segundo, la generación de timestamps usa un list comprehension que itera fila por fila para 1M de filas; hay alternativas vectorizadas que son más rápidas, aunque para este ejercicio no afecta los resultados del benchmark ya que la generación se excluye del timing.

---

## Criterio 2 — Rigor del benchmark `18 / 25`

Este criterio tiene el problema más importante de la entrega, y es necesario ser directo al respecto.

El footer del reporte dice:

> *"Mediciones realizadas en Python 3.14 con `time.perf_counter()` (3 repeticiones, promedio reportado)"*

`runner.py` dice otra cosa. `_measure_write` repite 3 veces — correcto. `_measure_read` y `_measure_read_selective` son mediciones únicas. No hay loop, no hay promedio, no hay repetición. El reporte describe un benchmark más riguroso que el que fue implementado.

Esto importa por dos razones. Primera, técnica: una sola medición de lectura puede ser fría o caliente dependiendo del estado del page cache del sistema operativo, y sin repetición no hay forma de saber cuál fue. Segunda, de integridad: un reporte técnico que describe las condiciones del experimento tiene que describir lo que realmente ocurrió. Si el reporte dice 3 repeticiones y el código hace 1, hay una inconsistencia que invalida parte de la evidencia presentada.

Lo que sí está bien: el manejo del OOM para JSONL en 1M de filas es la mejor respuesta del grupo a ese problema. En lugar de que el script crashee, el error es capturado, registrado con el tipo de excepción, y el benchmark continúa con los demás formatos. Los resultados muestran `—` para JSONL en 1M con una nota explicativa en el reporte. Eso es comportamiento correcto de ingeniería.

---

## Criterio 3 — Análisis por escala `24 / 25`

El análisis de escalabilidad es sólido. La observación sobre JSONL — que fue el formato **más rápido en escritura** con 100k filas (0.23s vs 0.37s de CSV) pero el más lento en lectura por un factor de 11x — es el tipo de asimetría que demuestra lectura activa de los datos. Muchos alumnos solo reportan que JSONL es lento; esta entrega señala el tradeoff asimétrico entre serializar y deserializar.

La sección 3.4 tiene la mejor observación de escalabilidad del grupo: que la compresión de Parquet se vuelve **más eficiente con más datos** porque el dictionary encoding trabaja mejor cuando hay más repeticiones de los mismos valores en columnas de baja cardinalidad. Con 10k apariciones de "Food" la tabla de diccionario se amortiza mejor que con 1k. Esa observación conecta directamente con el diseño del schema del dataset.

El único punto que falta: la sección de escritura menciona que Gzip hace "múltiples pasadas" pero no explica el mecanismo (LZ77 construye un árbol de frecuencias), lo que habría completado el análisis.

---

## Criterio 4 — Reporte y conclusiones `25 / 30`

El reporte tiene la mayor profundidad técnica del grupo: predicate pushdown con metadata de footer, proyección a nivel de bloques físicos antes de que los datos lleguen a memoria, recomendaciones diferenciadas por caso de uso (cold storage vs pipelines frecuentes vs intercambio con sistemas externos), y una mención a la compatibilidad con DuckDB que conecta directamente con el siguiente ejercicio. La documentación del OOM Killer como hallazgo, con la proyección que lo explica, es un ejemplo de cómo reportar un fallo de sistema de forma informativa.

Dos puntos que afectan la calificación. Primero, las gráficas usan escala lineal y son separadas por escala — con diferencias de dos órdenes de magnitud, en escala lineal las barras de Parquet son casi invisibles. Agrupar las tres escalas en una sola gráfica por métrica, como hicieron Antonio y Ernesto, también facilita ver el comportamiento de escalabilidad. Segundo, y conectado con el Criterio 2: el footer afirma condiciones del experimento que el código no cumple. Un reporte técnico es tan válido como la metodología que describe.

---

## Sobre el uso de herramientas de IA

El reporte usa vocabulario muy preciso: "OOM Killer", "predicate pushdown", "proyección a nivel de bloques físicos", "asimétrico por diseño". Ese nivel de precisión técnica combinado con la ausencia de voz personal — no hay un "Lo curioso es" o "Tengo entendido que", todo es declarativo y en tercera persona técnica — sugiere uso significativo de IA en la redacción.

El indicador más claro es la inconsistencia entre el footer del reporte y el código. El footer describe el benchmark ideal — 3 repeticiones para todo — sin verificar lo que el código realmente hace. Eso es característico de un texto generado que describe las mejores prácticas del dominio sin inspeccionar la implementación concreta. El código tiene evidencia de trabajo propio (el manejo de OOM, el FORMAT_REGISTRY), pero el reporte parece describir un experimento diferente al que fue ejecutado.

Para los siguientes ejercicios: el reporte debe describir exactamente lo que hiciste, no lo que debería haberse hecho. Si las lecturas no se repiten, el reporte debe decirlo — y puede incluir la justificación de por qué o señalarlo como limitación.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en el canal:

> El footer de tu reporte dice "3 repeticiones, promedio reportado" para todas las métricas, pero `_measure_read` en `runner.py` es una medición única. ¿Cuál de los dos describe lo que realmente mediste? Corrige uno de los dos — el código o el reporte — y explica qué impacto tendría repetir las lecturas 3 veces en los números que obtuviste para JSONL.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 18 / 20 |
| Rigor del benchmark | 25% | 18 / 25 |
| Análisis por escala | 25% | 24 / 25 |
| Reporte y conclusiones | 30% | 25 / 30 |
| **Total** | **100%** | **85 / 100** |

---

El manejo del OOM y la profundidad del análisis muestran que entendiste el ejercicio. Lo que falta es alinear lo que el reporte describe con lo que el código hace — en ingeniería, esa consistencia no es opcional.