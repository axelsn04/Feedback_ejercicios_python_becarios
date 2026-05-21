# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumno:** Angel Martinez Aguilar
**Fecha de revisión:** Mayo 2026
**Calificación:** 64 / 100

---

## Resumen general

El ejercicio está completo estructuralmente: schema, ingesta, benchmark y comparación DuckDB. El análisis escrito en el README tiene el hallazgo más interesante del grupo para este ejercicio — P4 sin índices hace que DuckDB sea más rápido que SQLite, lo cual identifica correctamente el punto de cruce OLTP/OLAP. Pero hay tres problemas técnicos concretos que bajan significativamente la calificación: las métricas del benchmark son de una sola corrida sin repeticiones, el cutoff de P4 usa `datetime.now()` en lugar del máximo del dataset (probablemente retornando 0 filas), y el índice para P5 no es covering — lo que explica directamente por qué DuckDB gana ese patrón.

---

## Criterio 1 — Schema design justificado `17 / 25`

**Lo que está bien:**

`idx_user_timestamp ON (user_id, timestamp DESC)` es la decisión correcta para P2, P3 y P4. El `DESC` en timestamp es un detalle que la mayoría omite — garantiza que P2 recorra las entradas más recientes sin operación de sort adicional. La justificación en `schema_design.md` explica el mecanismo correctamente: todos los registros de un usuario están contiguos en el B-tree, ordenados por timestamp, lo que hace los range scans de P3 y P4 prácticamente gratuitos.

**El problema principal — P5:**

`idx_country_code ON (country_code)` es un índice simple, no un covering index. Para P5 — `SELECT user_id, COUNT(*) WHERE country_code = ? GROUP BY user_id HAVING cnt > 5` — SQLite necesita `country_code` para filtrar y `user_id` para agrupar. Con el índice simple, SQLite filtra las filas por país pero luego tiene que acceder a la tabla principal para leer `user_id` de cada fila — accesos aleatorios a disco por cada una de las ~67K filas de MX.

Si el índice fuera `(country_code, user_id)`, SQLite resolvería P5 completamente desde el índice sin tocar la tabla — COVERING INDEX. Ernesto usó ese índice y su P5 fue 9ms. El tuyo fue 50ms. La diferencia de 5x viene exactamente de esta decisión.

El `schema_design.md` explica el índice de `country_code` pero no identifica la oportunidad del covering index, lo que indica que no se investigó por qué P5 es más lento que los otros patrones.

**Detalle menor:** los índices se crean en `schema.sql` que se ejecuta antes de la ingesta. Eso significa que cada INSERT actualiza tres estructuras B-tree en lugar de una. Ernesto separó la creación de índices para hacerlos después de todos los inserts — más eficiente para ingesta masiva.

---

## Criterio 2 — Ingesta chunkeada eficiente `12 / 20`

Lee desde Parquet correctamente ✅. El `--wal` / `--no-wal` funciona ✅. `PRAGMA synchronous=OFF` es una optimización válida para ingesta masiva ✅.

**Problema 1: no es chunking real de I/O.** `df = pd.read_parquet(parquet_path)` carga el millón de filas completo en memoria antes de iterar. El loop sobre `df.iloc[i:i+chunk_size]` itera sobre slices de un DataFrame ya cargado. La memoria pico es ~240MB al inicio, sin importar el `chunk_size`. El beneficio del chunking — poder procesar archivos más grandes que la RAM disponible — no se cumple.

**Problema 2: conflicto entre `BEGIN TRANSACTION` y `to_sql`.** El código hace `cursor.execute("BEGIN TRANSACTION")` y luego `chunk.to_sql('transactions', conn, ...)`. `to_sql` de pandas gestiona sus propias transacciones internamente — no respeta el `BEGIN` abierto por el cursor. Esto puede causar comportamiento indefinido dependiendo de la versión de pandas y sqlite3. El resultado práctico es que la transacción explícita puede no estar funcionando como se intenta. La forma correcta es usar `executemany` directamente con `BEGIN` / `COMMIT` explícitos, como hace Ernesto.

**Progreso por chunk:** solo reporta cada 100K filas. El ejercicio pedía reporte por cada chunk.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `14 / 30`

**Problema crítico: una sola medición por query.** `run_sqlite_query` ejecuta la query una vez y retorna el tiempo de esa corrida. No hay loop, no hay promedio. Para tiempos de sub-milisegundo como estos (P1: 0.04ms, P4: 0.01ms), una sola medición tiene varianza enorme — el scheduler del SO, el estado del page cache, y el GC de Python pueden añadir decenas de microsegundos. Los resultados no son reproducibles ni estadísticamente válidos con una sola corrida.

**Problema grave en P4:** `last_month` se calcula como `datetime.now() - timedelta(days=30)`. El dataset tiene timestamps uniformes en el año anterior a la generación. Si el Parquet fue generado hace más de un mes, `datetime.now() - 30 días` cae fuera del rango del dataset y P4 retorna 0 filas. Medir P4 sobre un resultado vacío produce 0.01ms — que no es el tiempo de la query sobre datos reales, es el tiempo de confirmar que no hay nada que retornar. Esta es la misma razón por la que el README reporta que P4 sin índices tarda solo 36ms en SQLite — si hay 0 filas que matchean, el scan termina rápido.

**`ORDER BY RANDOM()` para los params de prueba:** los valores de `transaction_id`, `user_id` y `country_code` cambian en cada ejecución del benchmark. Eso hace los resultados no comparables entre corridas. Los valores de prueba deberían ser fijos o derivados de valores deterministas del dataset.

Lo que sí está bien: EXPLAIN QUERY PLAN capturado y guardado en JSON ✅, comparación con/sin índices ✅, DuckDB para los 5 patrones ✅, todos los SLAs técnicamente reportados como cumplidos ✅.

---

## Criterio 4 — Comparación SQLite vs DuckDB `21 / 25`

El análisis del README es el más honesto del grupo para este ejercicio. La identificación de que DuckDB gana P5 y la explicación correcta — procesamiento columnar paralelo para aggregaciones masivas — está bien. La observación de P4 como punto de cruce es genuina y técnicamente correcta: cuando la query es agregación sobre un usuario específico, el índice SQLite gana; cuando no hay índice y el scan es completo, DuckDB con su motor vectorizado es más rápido que un full scan de SQLite.

La conclusión de arquitectura híbrida — SQLite para transacciones en tiempo real, DuckDB para analytics — es correcta y conecta bien con E04.

Cuatro puntos menos porque la explicación de por qué DuckDB gana P5 no identifica la causa raíz: el índice `(country_code)` simple vs el covering index `(country_code, user_id)` que habría mantenido SQLite competitivo en P5. La diferencia entre 50ms y 9ms no es "DuckDB es mejor en analytics" — es una decisión de diseño de índice que se puede corregir.

---

## Sobre el uso de herramientas de IA

El README tiene el análisis más breve del grupo pero con observaciones genuinas — especialmente el hallazgo de P4 sin índices. El `schema_design.md` es más genérico y probablemente AI-asistido en el tono, pero las decisiones del schema son reales. Los bugs del benchmark (single measurement, datetime.now() en P4) son señales claras de código que no fue revisado completamente antes de entregar.

---

## Pregunta de seguimiento

Antes de comenzar el Ejercicio 4:

> Tu `idx_country_code` es un índice simple sobre `country_code`. Cámbialo por `CREATE INDEX idx_country_user ON transactions (country_code, user_id)`, regenera la base, y corre el benchmark de P5 de nuevo. ¿Cuánto mejora? ¿Aparece `COVERING INDEX` en el EXPLAIN QUERY PLAN? ¿Qué significa que aparezca esa etiqueta?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 17 / 25 |
| Ingesta chunkeada eficiente | 20% | 12 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 14 / 30 |
| Comparación SQLite vs DuckDB | 25% | 21 / 25 |
| **Total** | **100%** | **64 / 100** |

---

El hallazgo de P4 es el más interesante del grupo. Para E04, la pregunta de seguimiento cierra el punto más concreto de este ejercicio antes de construir el API.