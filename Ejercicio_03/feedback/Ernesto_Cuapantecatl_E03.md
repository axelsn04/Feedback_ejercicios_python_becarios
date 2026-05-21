# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 89 / 100

---

## Resumen general

El ejercicio está muy bien ejecutado en schema design y benchmark. Los índices están correctamente elegidos para los 5 patrones, la justificación en `schema_design.md` explica el mecanismo técnico de cada decisión, los SLAs se cumplen con margen amplio, y el análisis de EXPLAIN QUERY PLAN es el más detallado del grupo para este ejercicio. La ingesta tiene dos problemas concretos que bajan la calificación: lee desde CSV en lugar de Parquet y carga el archivo completo en memoria antes de iterar chunks, lo que no es chunking real de I/O.

---

## Criterio 1 — Schema design justificado `24 / 25`

El schema es correcto y la justificación es genuina. Las decisiones más importantes están bien razonadas:

**`timestamp TEXT` en formato ISO 8601** — la observación de que las comparaciones `>=` y `BETWEEN` producen el mismo resultado que las cronológicas porque ISO 8601 ordena alfabéticamente igual que temporalmente es correcta y no obvia. Es exactamente la razón por la que esta decisión funciona para range scans en SQLite.

**`idx_user_timestamp ON (user_id, timestamp)`** — el razonamiento del orden de columnas es correcto: `user_id` primero porque todas las queries P2-P4 filtran por usuario, y `timestamp` segundo para que el range scan sea directo dentro de cada usuario sin necesidad de ordenación posterior.

**`idx_country_user ON (country_code, user_id)`** — la observación de que este es un covering index para P5 es correcta: SQLite resuelve la query completa sin tocar la tabla principal porque `country_code` y `user_id` están en el índice, y el `COUNT(*)` se calcula contando entradas del índice. Esto se confirma en el EXPLAIN del reporte: `USING COVERING INDEX`.

**Índices que no se crearon** — la justificación explícita de por qué `status`, `category`, `merchant_id` y `amount` no tienen índices individuales demuestra comprensión real: crear índices sobre columnas de baja cardinalidad (status tiene 3 valores) no reduce significativamente el espacio de búsqueda y sí añade overhead en cada INSERT.

Un punto menos: `amount REAL` almacena floats de punto flotante, y la nota al margen de que en producción se usaría entero en centavos para evitar errores de redondeo es correcta — pero podría haber ido un paso más y evaluado si afecta a este ejercicio. Para sumas de montos (`SUM(amount)`), la acumulación de errores de punto flotante en 1M filas puede ser visible. No es un error crítico aquí pero es un tradeoff que vale documentar.

---

## Criterio 2 — Ingesta chunkeada eficiente `13 / 20`

**Problema 1: lee desde CSV, no desde Parquet.** El ejercicio dice explícitamente "cargar el Parquet en chunks del tamaño especificado". `ingest.py` usa `pd.read_csv()`. El resultado funcional es el mismo, pero el ejercicio pide específicamente el Parquet de E1 como fuente de verdad del módulo.

**Problema 2: no es chunking real de I/O.** El código hace `df = pd.read_csv(csv_path)` — carga el archivo completo en memoria (~240 MB de RAM para 1M filas) — y luego itera sobre slices del DataFrame. Esto no es chunking: el costo de memoria se paga al inicio. El chunking real sería leer el archivo en bloques con `pd.read_parquet()` + `pyarrow` usando `batch_size`, o con `pd.read_csv(chunksize=N)` que sí lee en bloques. La diferencia importa cuando el archivo es más grande que la RAM disponible — que es exactamente el caso de uso que justifica el chunking.

Lo que sí está bien: las transacciones explícitas con `with conn:` funcionan correctamente — cada bloque de `executemany` se commit de forma independiente. Los índices se crean después de todos los inserts ✅. El `--wal` / `--no-wal` funciona y el reporte compara ambos correctamente ✅. La ingesta completa en menos de 3 minutos ✅.

Sin reporte de progreso por chunk — el ejercicio pedía algo como "Chunk 5/40 insertado en 1.2s". El código solo imprime el resultado final.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `28 / 30`

El benchmark es sólido. 10 repeticiones con promedio, EXPLAIN QUERY PLAN integrado al JSON, comparación con/sin índices, y DuckDB para los 5 patrones. Los SLAs se cumplen todos con margen:

- P1: 0.036ms (SLA: 10ms) — **278x de margen**
- P2: 0.079ms (SLA: 50ms) — **633x de margen**
- P3: 0.025ms (SLA: 50ms) — **2,000x de margen**
- P4: 0.013ms (SLA: 50ms) — **3,846x de margen**
- P5: 9.173ms (SLA: 200ms) — **22x de margen**

Las interpretaciones de EXPLAIN son las mejores del grupo para este ejercicio. La observación de que P2 sin índice muestra `USE TEMP B-TREE FOR ORDER BY` — SQLite tiene que construir un B-tree temporal en memoria para ordenar por timestamp después de escanear toda la tabla — y que con el índice compuesto ese paso desaparece porque las filas ya están ordenadas por timestamp dentro de cada user_id, es exactamente el mecanismo correcto. La observación equivalente para P5 con `USE TEMP B-TREE FOR GROUP BY` también es correcta.

Sin `gc.collect()` entre repeticiones — punto menor, pero el ruido del GC puede sesgar mediciones de sub-millisecond como estas.

---

## Criterio 4 — Comparación SQLite vs DuckDB `24 / 25`

El análisis es correcto y bien razonado. Los factores son dramáticos (SQLite 1,730x más rápido en P1, 303x en P2) y la explicación del mecanismo es precisa: DuckDB no tiene índice sobre `transaction_id` en el Parquet y tiene que escanear row groups completos para encontrar un UUID, mientras SQLite navega el B-tree en ~20 comparaciones.

La observación sobre P5 como "punto de cruce" es la más interesante del reporte: cuando la query se parece más a analytics que a transaccional, la ventaja de SQLite se reduce y DuckDB se acerca. La pregunta al margen — si con un dataset más grande DuckDB podría superar a SQLite en P5 — es exactamente la pregunta correcta. La respuesta es sí, porque P5 es esencialmente un GROUP BY sobre un subconjunto grande de filas, que es el patrón donde DuckDB brilla. La ventaja del covering index de SQLite en P5 sería cada vez más estrecha a medida que crece el número de países y el número de usuarios por país.

La conclusión — usar ambos en producción: DuckDB para analytics y SQLite para la capa transaccional — conecta directamente con E04 y es la arquitectura correcta.

---

## Sobre el uso de herramientas de IA

El reporte es genuino. El "Registro de Tiempos" al final es el tipo de sección que se agrega de forma orgánica, no generada. Las observaciones técnicas usan lenguaje hedgeado ("considero", "tengo entendido", "me surge la duda") que es consistente con las entregas anteriores. Los 7-9 horas reportados son creíbles dado la complejidad del ejercicio — y la honestidad de reportarlo por encima de las 5-6 estimadas habla bien del alumno.

---

## Pregunta de seguimiento

Para el Ejercicio 4:

> Tu `ingest.py` carga el CSV completo en memoria antes de chunkar. En E04 vas a necesitar ingesta incremental (el endpoint `POST /transactions/batch` del API). ¿Cómo cambiarías la ingesta para que nunca cargue más de `chunk_size` filas en memoria al mismo tiempo, leyendo desde Parquet?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 24 / 25 |
| Ingesta chunkeada eficiente | 20% | 13 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 28 / 30 |
| Comparación SQLite vs DuckDB | 25% | 24 / 25 |
| **Total** | **100%** | **89 / 100** |

---

El schema design y el benchmark están al nivel de las mejores entregas del módulo. Para E04, la pregunta de seguimiento es concreta: corrige el chunking real de I/O antes de construir el endpoint de ingesta.