# Retroalimentación — Ejercicio 01: Formatos Bajo la Lupa

**Alumna:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 84 / 100

---

## Resumen general

El ejercicio está completo. Entregaste todos los archivos requeridos, el código corre, las mediciones son reales y el reporte cubre las tres escalas. Eso no es menor — muchos no llegan a ese punto. Dicho esto, hay diferencias importantes entre los criterios: el análisis escrito está bien desarrollado pero el rigor del benchmark tiene un error técnico que afecta la validez de parte de tus mediciones, y es el punto más importante a corregir para los siguientes ejercicios.

---

## Criterio 1 — CLI funcional y módulo organizado `18 / 20`

El CLI funciona correctamente. Acepta `--size` y `--formats` como argumentos, valida los valores posibles y la separación entre `generate_data.py`, `benchmark_cli.py` y `storage_benchmark/` es la correcta. El detalle más importante que sí hiciste bien: generas el DataFrame en memoria *antes* de entrar al loop de formatos, lo que significa que el tiempo de generación no contamina las mediciones de escritura. Eso demuestra que entendiste el punto.

El único problema es el `sys.path.append` manual al inicio de `benchmark_cli.py`. Eso indica que la configuración del paquete en `pyproject.toml` no está del todo resuelta. No es un error grave, pero en los siguientes ejercicios vamos a exigir que el proyecto esté correctamente instalado como paquete — que puedas hacer `from storage_benchmark.operations import ...` sin hacks de path.

---

## Criterio 2 — Rigor del benchmark `18 / 25`

Este es el criterio más importante técnicamente y donde hay el error más claro.

`measure_write_time` usa `runs=3` y reporta el promedio — correcto. Pero `measure_read_full`, `measure_read_selective` y `measure_memory_peak` se ejecutan una sola vez cada uno. Eso no es un benchmark, es una medición. Los tiempos de lectura tienen varianza significativa dependiendo del estado del cache del sistema operativo: la primera lectura es fría (el archivo no está en cache), las siguientes son calientes. Una sola corrida puede darte el número más optimista o el más pesimista sin que sepas cuál es.

Si vas a argumentar que Parquet Snappy es 8x más rápido que CSV, ese argumento necesita estar respaldado por mediciones repetidas. Con una sola corrida por lectura, no puedes hacer esa afirmación con rigor.

Para los siguientes ejercicios: **toda medición de rendimiento debe repetirse mínimo 3 veces y reportar promedio**. Sin excepciones.

---

## Criterio 3 — Análisis por escala `23 / 25`

Esta es la parte más sólida del ejercicio. Las tres secciones del reporte tienen observaciones distintas y no solo repiten los números de la tabla. La observación sobre JSON cruzando 1 GB de RAM en 500K registros, y la identificación del tradeoff CPU/disco de Gzip en 1M registros, son exactamente el tipo de conclusiones que se esperaban.

Lo que falta para llegar al puntaje completo es la explicación del mecanismo técnico detrás de la compresión de Parquet en columnas como `category`, `status` y `country_code`. Tu reporte describe el *efecto* — Parquet comprime mejor que CSV — pero no explica el *por qué* a nivel de cómo funciona Parquet internamente. Ese mecanismo tiene nombre. Investígalo para la siguiente entrega.

---

## Criterio 4 — Reporte y conclusiones `25 / 30`

El reporte está completo y bien estructurado. Las tablas cubren las tres escalas, las gráficas muestran tamaño en disco y tiempos de lectura, y la recomendación final tiene casos de uso específicos por formato — eso es lo que se pedía.

Tres puntos a mejorar:

**1. El pico de memoria no aparece en ninguna gráfica.** Es la métrica más dramática del ejercicio — JSON usando 2.3 GB de RAM para leer 188 MB en disco es el hallazgo más impactante que tienes. Ese número merecía una gráfica propia, no solo una columna en la tabla.

**2. El tono del reporte.** Frases como *"recomendación oficial para la arquitectura de datos en producción"* o *"equilibrio perfecto del mercado"* no son como escribe un ingeniero. Un ingeniero escribe: *"para este patrón de acceso, Parquet Snappy ofrece el mejor balance entre velocidad de escritura y tamaño en disco"*. La diferencia importa porque el tono absoluto hace que tus conclusiones parezcan menos fundadas, no más. En los siguientes ejercicios cuida el lenguaje técnico.

**3. Las gráficas están en inglés y el reporte en español.** Inconsistencia menor pero visible — elige uno y sé consistente.

---

## Sobre el uso de herramientas de IA

El código tiene huellas claras de trabajo propio: el bug de las lecturas sin repetición, el `sys.path.append`, el list comprehension para timestamps que es innecesariamente lento para 1M filas. Esas son decisiones de alguien que está resolviendo un problema real, no código generado completo.

El reporte escrito es otra historia. El tono, la estructura y el vocabulario son más propios de un asistente de IA que de un becario escribiendo sus conclusiones. Usar IA para estructurar ideas o revisar redacción está bien. Usarla para que escriba el análisis por ti no — porque en la revisión se nota, y más importante, porque te quitas la oportunidad de desarrollar el criterio técnico que es el punto del ejercicio.

Para los siguientes ejercicios: usa IA como herramienta de apoyo, no como autor. La diferencia práctica es simple: si no puedes explicar verbalmente lo que está escrito en tu reporte, no está listo para entregarse.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 2, responde esto en los comentarios de este archivo o en el canal:

> Tu reporte muestra que Parquet comprime mucho mejor que CSV en columnas como `category`, `status` y `country_code`, pero no tanto en `transaction_id` o `amount`. ¿Por qué ocurre esa diferencia? ¿Qué mecanismo interno de Parquet explica ese comportamiento?

No busques la respuesta en el reporte que ya entregaste — no está ahí. Investígala.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| CLI funcional y módulo organizado | 20% | 18 / 20 |
| Rigor del benchmark | 25% | 18 / 25 |
| Análisis por escala | 25% | 23 / 25 |
| Reporte y conclusiones | 30% | 25 / 30 |
| **Total** | **100%** | **84 / 100** |

---

Buen primer ejercicio. El siguiente tiene más piezas móviles — empiézalo con el benchmark bien diseñado desde el principio.