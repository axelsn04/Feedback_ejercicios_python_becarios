# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumna:** Yaanit Granados
**Fecha de revisión:** Mayo 2026
**Calificación:** 73 / 100

---

## Resumen general

El ejercicio está completo y los SLAs se cumplen: los 5 patrones responden dentro de sus límites con índices activos, el reporte tiene EXPLAIN para cada patrón, y la comparación DuckDB vs SQLite incluye análisis de cuándo cada engine gana. Las decisiones de schema tienen una elección interesante (`STRICT` mode) y un problema concreto (`user_id TEXT`). La ingesta sigue el patrón de cargar el archivo completo en memoria antes de iterar chunks — no es chunking real de I/O. El benchmark tiene fechas hardcodeadas en P4 que pueden medir un resultado vacío.

---

## Criterio 1 — Schema design justificado `20 / 25`

**`STRICT` mode** — decisión única en el grupo. SQLite por defecto tiene tipado dinámico: puedes insertar texto en una columna `INTEGER`. `STRICT` fuerza que los tipos declarados se respeten, lo que garantiza integridad desde la capa de base de datos. El `schema_design.md` lo justifica correctamente: es protección contra datos mal formados desde el pipeline.

**`idx_country_user ON (country_code, user_id)`** — covering index correcto para P5. El EXPLAIN lo confirma: `SEARCH USING COVERING INDEX`. P5 en 17ms vs 184ms sin índice es la diferencia más dramática y está bien documentada.

**`idx_user_timestamp ON (user_id, timestamp)`** — cubre P2, P3 y P4. No tiene `DESC` en timestamp, lo que significa que P2 (`ORDER BY timestamp DESC LIMIT 20`) requiere que SQLite haga un backward scan en el índice. Funciona, pero con `DESC` declarado el forward scan sería ligeramente más eficiente.

**`user_id TEXT NOT NULL`** — el problema principal del schema. El dataset del módulo define `user_id` como entero en el rango 1-50,000. Almacenar como TEXT implica que las comparaciones son lexicográficas: `"10" < "2"` en texto. Esto no rompe los índices ni los SLAs para los patrones de este ejercicio (el benchmark usa el mismo user_id que está almacenado en TEXT, así que las comparaciones son TEXT-TEXT y son consistentes), pero en E04 los endpoints reciben user_id por HTTP path como string y lo comparan con la base — lo que puede funcionar accidentalmente pero con riesgo de inconsistencias. El resto del grupo usó `INTEGER` por ser el tipo nativo para IDs numéricos.

---

## Criterio 2 — Ingesta chunkeada eficiente `12 / 20`

**El mismo problema de los ejercicios anteriores**: `df = pd.read_parquet(input_path)` carga el millón de filas completo en RAM antes de iterar. El loop sobre `df.iloc[start_idx:start_idx+chunk_size]` itera sobre slices de un DataFrame ya cargado. El pico de RAM es ~240MB desde el inicio, sin importar el `chunk_size`.

**Lo que sí está bien:** las transacciones explícitas con `BEGIN TRANSACTION` / `COMMIT` por chunk son correctas ✅. WAL vs no-WAL con flags `--wal` / `--no-wal` funciona ✅. Los resultados del reporte (51.93s con WAL vs 77.94s sin WAL) son datos reales y la justificación de por qué WAL es más rápido es correcta ✅.

**`list(chunk.itertuples(index=False, name=None))`** — retorna tuplas en el orden de las columnas del DataFrame. Si el Parquet tiene las columnas en un orden distinto al del `INSERT INTO transactions (col1, col2, ...)`, la ingesta insertaría valores en columnas incorrectas silenciosamente. El orden correcto se garantiza seleccionando columnas explícitamente antes de `itertuples()`, como hacen Ulises y Ernesto.

Sin verificación de integridad al final — no hay comparación de `COUNT(*) DB` vs filas del Parquet.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `19 / 30`

Los SLAs se cumplen en los cinco patrones y el EXPLAIN confirma que los índices se están usando. Eso es lo más importante.

**P4 con fecha hardcodeada:** `WHERE timestamp >= '2026-04-15'`. Si el dataset tiene transacciones hasta mayo 2026 (que es el caso según otros benchmarks del grupo), esta fecha sí está dentro del rango y P4 retorna filas reales. Pero si el dataset fuera de otro período, '2026-04-15' podría quedar fuera del rango y P4 mediría un resultado vacío — el mismo bug que tuvo Angel con `datetime.now()`. La forma correcta es `SELECT MAX(timestamp) FROM transactions` y calcular 30 días antes.

**P3 con rango `'2025-01-01' AND '2026-12-31'`**: cubre todo el año del dataset, lo que probablemente retorna todas las transacciones del usuario seleccionado. Es un rango demasiado amplio para representar el SLA real de "transacciones en un rango de fechas acotado".

**`cursor.fetchone()` con `LIMIT 1`** para obtener los parámetros de prueba: puede retornar un usuario con muy pocas transacciones (no el worst case del SLA). Para P2-P4, el peor caso es el usuario con más transacciones — como hace Ulises.

**Solo 3 repeticiones sin `gc.collect()`** — el GC de Python puede dispararse durante una medición. La varianza en mediciones de sub-millisecond puede ser significativa con solo 3 corridas.

**Sin `ANALYZE` antes del benchmark** — los resultados muestran que los índices SÍ se usan (los tiempos son correctos), lo que sugiere que SQLite tomó la decisión correcta de todas formas para este dataset específico.

---

## Criterio 4 — Comparación SQLite vs DuckDB `22 / 25`

El análisis es el más honesto del grupo en este criterio para E03.

**P1-P4: SQLite gana por órdenes de magnitud** — la explicación de "acceso directo de disco (random access I/O) leyendo una sola página de memoria" vs "DuckDB debe codificar bloques enteros del Parquet" es correcta.

**P5: "Convergencia de rendimiento"** (17ms vs 19.66ms) — la observación más valiosa del reporte: "Si el dataset creciera a 100 millones de registros, DuckDB superaría a SQLite, ya que su motor vectorizado puede sumar millones de valores con SIMD más rápido de lo que un motor basado en filas puede recorrer el árbol." Esa predicción es correcta y es el tipo de análisis que demuestra comprensión real del tradeoff OLTP vs OLAP.

---

## Sobre el uso de herramientas de IA

El reporte tiene análisis propio — la predicción de P5 a escala y la explicación del SIMD no son genéricas. El `STRICT` mode es una decisión que requiere conocer SQLite por encima de lo básico. Los bugs concretos (fechas hardcodeadas, carga completa en RAM) son señales de código que no fue revisado completamente contra las condiciones del ejercicio.

---

## Pregunta de seguimiento

> En `ingest.py`, el loop itera sobre `df.iloc[start_idx:start_idx+chunk_size]` sobre un DataFrame ya cargado en RAM. ¿Cómo cambiarías la ingesta para que realmente lea el Parquet en bloques de `chunk_size` filas sin cargar el archivo completo? Pista: `pyarrow.parquet.ParquetFile.iter_batches()` o `pd.read_parquet()` con columnas seleccionadas por batch.

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 20 / 25 |
| Ingesta chunkeada eficiente | 20% | 12 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 19 / 30 |
| Comparación SQLite vs DuckDB | 25% | 22 / 25 |
| **Total** | **100%** | **73 / 100** |

---

Los SLAs se cumplen y el análisis de P5 a escala es el más técnico del grupo para ese punto. Para E04 (ya revisado), el `user_id TEXT` no causó problemas visibles pero es el tipo de decisión de schema que genera bugs difíciles de rastrear cuando los tipos no coinciden entre capas.