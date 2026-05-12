# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumno:** Antonio Jair Garcia Vargas
**Fecha de revisión:** Mayo 2026
**Calificación:** 96 / 100

---

## Resumen general

Entrega sólida en todos los criterios. El código tiene arquitectura real — no solo funciona, está diseñado para crecer. El reporte demuestra comprensión técnica genuina: no solo reporta qué formato ganó, sino explica por qué a nivel de cómo funcionan los formatos internamente. Es la entrega más completa del grupo hasta ahora.

---

## Criterio 1 — CLI funcional y módulo organizado `20 / 20`

La separación entre `formats.py` y `benchmark.py` es la correcta. Cada formato está definido como un `@dataclass(frozen=True)` con tres operaciones uniformes — agregar un formato nuevo al módulo es una sola entrada en el diccionario `FORMATS`, sin tocar el benchmark. Eso es diseño extensible.

Detalles que demuestran criterio ingenieril: `gc.collect()` antes de cada medición para reducir ruido del garbage collector, `pathlib` en todo el código en lugar de strings de rutas, `--repetitions` y `--selective-columns` configurables desde el CLI, y los resultados guardados con metadata completa — seed, timestamp, columnas selectivas — lo que hace cada corrida reproducible y auditable.

Sin `sys.path.append`, sin hacks de importación. El proyecto está correctamente estructurado como paquete.

---

## Criterio 2 — Rigor del benchmark `22 / 25`

La escritura está bien: 3 repeticiones con el archivo eliminado entre runs y `gc.collect()` antes de cada una. La generación del dataset está correctamente excluida del timing.

El mismo punto débil que aparece frecuentemente: `read_full` y `read_selective` son mediciones únicas. Los tiempos de lectura tienen varianza real dependiendo del estado del page cache del sistema operativo — una sola corrida puede ser la más optimista o la más pesimista sin que sepas cuál es. Para el siguiente ejercicio, todas las mediciones deben repetirse, no solo la escritura.

Lo que sí está bien resuelto es la limitación de `tracemalloc`: la documentas explícitamente en el docstring del módulo y en el reporte, propones `psutil` como alternativa, y explicas por qué los números de Parquet reportan ~0 MB. Eso es honestidad técnica — es mejor documentar una limitación que ignorarla o reportar un número incorrecto como válido.

---

## Criterio 3 — Análisis por escala `25 / 25`

La sección de escalabilidad tiene exactamente lo que se busca. Observas que el tamaño en disco crece linealmente sin overhead fijo en los cinco formatos, que los tiempos también escalan linealmente pero con pendientes dos órdenes de magnitud distintas, y — el insight más importante — que la ventaja de Parquet sobre CSV no se cierra al escalar, al contrario se vuelve más relevante en términos absolutos.

El dato concreto lo dice todo: pasar de 100K a 1M agrega 1.3 segundos de lectura a CSV pero solo 0.08 segundos a Parquet+Snappy. Ese tipo de cuantificación es lo que convierte una observación en un argumento técnico.

La observación de que JSONL genera ~2.5 KB de overhead Python por fila durante la lectura, y que en 1M filas eso supera los 2.5 GB de RAM, también demuestra que corriste el experimento y pensaste en los números, no solo que los copiaste.

---

## Criterio 4 — Reporte y conclusiones `29 / 30`

Buenas decisiones de visualización: escala logarítmica con justificación escrita, y todas las escalas agrupadas en una sola gráfica por métrica. En escala lineal las barras de Parquet desaparecen visualmente — la decisión de usar log está justificada en el reporte, no solo aplicada.

El análisis explica correctamente los mecanismos internos: por qué Parquet no necesita convertir bytes ASCII a tipos numéricos, por qué JSONL genera ~8 millones de objetos Python intermedios al leer 1M filas, y por qué la compresión Gzip sale gratis en lectura — el ahorro de I/O es mayor que el costo de descomprimir. Esa última observación no es obvia y demuestra que entendiste el tradeoff real.

La recomendación está bien diferenciada: Snappy para producción, Gzip para cold storage, CSV solo como interfaz con sistemas legados o humanos. El lenguaje es técnico y apropiado — sin absolutos, sin marketing.

El único punto que falta: las tablas reportan ~0 MB de memoria para Parquet sin mostrar un número alternativo. Documentas la limitación de `tracemalloc` y mencionas `psutil`, pero el lector se queda sin un dato real de memoria RSS para Parquet. Una medición con `psutil.Process().memory_info().rss` antes y después de la lectura habría completado el análisis.

---

## Sobre el uso de herramientas de IA

El código tiene profundidad técnica que no viene de un prompt genérico. El comentario `UUID4 — uuid.uuid4() cuesta ~1us, para 1M son ~1s. No hay shortcut estándar` es el tipo de nota que escribe alguien que midió el problema. El `frozen=True` en el dataclass de `Format` es una decisión de diseño deliberada. La observación de que `tracemalloc` no ve los buffers de C de pyarrow requiere haber investigado activamente por qué los números de memoria para Parquet eran ~0 — eso no se descubre sin correr el código y cuestionarse el resultado.

Usaste herramientas de IA — el código es demasiado limpio y uniforme para no haberlo hecho — pero como apoyo, no como sustituto. La comprensión técnica detrás de las decisiones es tuya. Sigue así.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en el canal:

> Tu reporte documenta que `tracemalloc` no mide la memoria real de pyarrow porque asigna buffers fuera del heap de Python. Propones `psutil` como alternativa. Implementa esa medición en tu benchmark — una sola línea antes y después de `fmt.read_full(path)` — y reporta cuánto RAM usa realmente Parquet+Snappy al leer 1M filas. ¿Cambia tu conclusión sobre la ventaja de memoria de Parquet sobre CSV?

No es un ejercicio nuevo — es completar lo que dejaste pendiente en este.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 20 / 20 |
| Rigor del benchmark | 25% | 22 / 25 |
| Análisis por escala | 25% | 25 / 25 |
| Reporte y conclusiones | 30% | 29 / 30 |
| **Total** | **100%** | **96 / 100** |

---

Buen trabajo. El Ejercicio 2 tiene más piezas móviles — tres engines, ocho queries, validación de equivalencia. La misma mentalidad que aplicaste aquí te va a servir.