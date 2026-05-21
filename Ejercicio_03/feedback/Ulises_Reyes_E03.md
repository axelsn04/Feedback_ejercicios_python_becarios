# Retroalimentación — Ejercicio 03: La Capa Transaccional

**Alumno:** Ulises Josue Reyes Martinez
**Fecha de revisión:** Mayo 2026
**Calificación:** 97 / 100

---

## Resumen general

La entrega más completa del grupo para este ejercicio. El schema design es el más riguroso — cubre el tradeoff WITH ROWID vs WITHOUT ROWID, justifica el orden de columnas en cada índice con el principio de prefijo, y usa covering index para P5. La ingesta hace chunking real de I/O. El benchmark tiene `ANALYZE` antes de medir (crítico, único en el grupo) y dos conexiones separadas para evitar contaminación de page cache entre las fases con y sin índices. El reporte tiene el análisis más técnico del grupo con observaciones genuinas: la inicialización fija de ~88ms de DuckDB, el crossover de P4, y la nota de que P5 podría invertirse con Parquet clusterizado por country_code.

---

## Criterio 1 — Schema design justificado `25 / 25`

El `schema.sql` y el `schema_design.md` son los más completos del grupo.

**`(country_code, user_id)` como covering index para P5** — la decisión correcta. El índice contiene exactamente las dos columnas que P5 necesita, lo que confirma el EXPLAIN QUERY PLAN: `USING COVERING INDEX idx_country_user`. SQLite resuelve P5 sin tocar la tabla principal — un scan secuencial sobre las ~67k entradas del país contando cambios de `user_id`. El resultado: 8.59ms vs 50ms de un índice simple sobre solo `country_code`.

**`WITHOUT ROWID` analizado y descartado** con razonamiento correcto: con PK en UUID aleatorio, WITHOUT ROWID fragmenta el B-tree durante la ingesta porque SQLite debe insertar en posiciones arbitrarias en lugar de siempre al final del heap. La ganancia marginal en P1 no justifica el costo en velocidad de ingesta.

**`timestamp DESC` en el índice compuesto** — la observación de que P2 puede servirse sin sort porque las entradas del usuario ya están en orden DESC es exactamente el mecanismo. La aclaración de que P3 y P4 no se ven afectados negativamente (SQLite elige la dirección de scan según los predicados) es correcta.

**El principio de prefijo** explicado correctamente en `schema_design.md`: un índice compuesto `(A, B)` puede usarse para queries que filtran por A solo, o por A y B, pero no por B solo. Los tres patrones P2-P4 tienen `user_id` como primer predicado — todos usan el mismo índice.

**La vista `v_ingestion_summary`** como herramienta de verificación es un detalle que demuestra pensar en operabilidad, no solo en el ejercicio.

---

## Criterio 2 — Ingesta chunkeada eficiente `19 / 20`

**Chunking real de I/O** con `pd.read_csv(chunksize=chunk_size)`. Este es el único alumno del grupo que lo hace correctamente — el lector de pandas consume el CSV en bloques, con un pico de RAM de ~4MB por chunk en lugar de los ~240MB del archivo completo. La diferencia importa cuando el archivo es más grande que la RAM disponible.

**`executemany` con tuplas explícitas** en lugar de `to_sql()` — la justificación en el docstring es correcta: `to_sql()` tiene overhead de construcción de queries Python por fila; `executemany()` pasa las tuplas directamente al driver C de sqlite3.

**`INSERT OR IGNORE`** para deduplicación silenciosa — robusto para re-ingestas sin crash.

**Verificación de integridad** comparando conteos DB vs CSV, y consulta a `v_ingestion_summary` para confirmar distribución de datos.

**Progress reporting** cada 10 chunks con rows/s en tiempo real.

**Limpieza de archivos WAL** (`.db-wal`, `.db-shm`) al eliminar la base — detalle que la mayoría omite.

**Un punto menos:** los índices se crean en `schema.sql` que se ejecuta antes de la ingesta, lo que significa que cada INSERT actualiza los tres índices en paralelo. Crear los índices después de todos los inserts (como hace Ernesto) sería más eficiente para ingesta masiva — SQLite puede hacer un B-tree sort sobre los datos ya cargados, que es más eficiente que actualizar el índice fila por fila. La diferencia para 1M filas es de ~20-30% en tiempo de ingesta.

---

## Criterio 3 — Benchmark con EXPLAIN PLAN y SLAs `29 / 30`

**`ANALYZE` antes del benchmark** — el único alumno del grupo que lo hace. Sin ANALYZE, SQLite no tiene estadísticas de distribución de datos y puede ignorar los índices en tablas recién populadas, eligiendo full scan porque el planner no sabe cuántas filas hay por user_id. El resultado sin ANALYZE sería un benchmark que no mide el rendimiento real con índices. Que esté documentado en el docstring con la explicación correcta confirma que es una decisión consciente.

**Dos conexiones separadas** — conn_with se cierra antes de abrir conn_without para la fase sin índices. La justificación en el docstring es correcta: SQLite puede mantener páginas del índice anterior en su page cache, lo que haría que el benchmark "sin índices" sea artificialmente rápido. Con dos conexiones separadas, el page cache está frío al iniciar la segunda fase.

**`month_ago` calculado correctamente** con `datetime(max_ts, '-30 days')` desde el MAX del dataset — no desde `datetime.now()`. Este bug tuvo Angel y lo explica por qué su P4 midió probablemente 0 resultados.

**User con más transacciones como parámetro de prueba** — el peor caso del SLA. Un usuario con pocas transacciones terminaría en microsegundos y no mide el SLA real.

**5 repeticiones con `gc.collect()`**, avg/min/max y lista completa de runs guardada en JSON.

Un punto menos: el rango de fechas para P3 (`date_from` a `date_to`) cubre solo 2 días del primer mes del usuario y retorna 1 fila. El SLA de P3 es para "transacciones de un usuario en un rango de fechas" — un rango más representativo (como un trimestre con decenas de resultados) habría puesto a prueba el SLA de forma más rigurosa.

---

## Criterio 4 — Comparación SQLite vs DuckDB `24 / 25`

El análisis es el más técnico del grupo para este criterio.

**P1 — overhead de inicialización de DuckDB** (0.133ms vs 88.807ms): la observación de que DuckDB tiene un overhead fijo de ~88ms para cualquier query sobre Parquet — abrir el archivo, leer el footer de metadatos, localizar row groups — es correcta y es el insight más importante del ejercicio. Ese overhead no escala con el resultado: cuesta igual para 1 fila que para 10,000. Para lookups puntuales nunca se amortiza.

**P4 — el crossover**: DuckDB termina en 20ms, más rápido que SQLite sin índice (108ms), pero no puede alcanzar a SQLite con índice (0.158ms). La explicación — las transacciones del usuario están dispersas en múltiples row groups y DuckDB tiene que inspeccionarlos todos buscando el 0.004% del dataset — es correcta. El B-tree va directamente.

**P5 — covering index y la nota del Parquet clusterizado**: la observación de que con un Parquet clusterizado por `country_code`, DuckDB podría invertir el resultado es correcta. Si el Parquet estuviera particionado por país (como se hace en producción con Hive partitioning en S3), DuckDB solo leería la partición de MX — similar a lo que hace el covering index de SQLite.

**Conclusión de arquitectura**: directa y accionable. No hay un ganador universal — depende del patrón de acceso y del SLA.

Un punto menos porque el reporte se vuelve algo verbose al final y pierde precisión en la última sección. Las conclusiones más fuertes están bien articuladas al inicio de cada sección; al final el texto se dilata sin agregar contenido nuevo.

---

## Sobre el uso de herramientas de IA

El reporte tiene voz propia: "me llamó la atención", "creo que es una ley que aplica en todos los campos de la tecnología". Las observaciones técnicas — el overhead fijo de DuckDB, el crossover de P4, la nota del Parquet clusterizado — son genuinas y no genéricas. El código tiene docstrings que explican el razonamiento detrás de cada decisión, no solo lo que hace el código. La decisión de usar ANALYZE y dos conexiones separadas son ejemplos de conocimiento que va más allá del ejercicio básico. Uso de IA como apoyo, con comprensión real de fondo.

---

## Pregunta de seguimiento

Para el Ejercicio 4:

> Tu `ingest.py` crea los índices en `schema.sql` que se ejecuta antes de la ingesta. En el Ejercicio 4 vas a tener un endpoint `POST /transactions/batch` que inserta lotes de 500 filas en la base ya poblada. Para ese caso, ¿tener los índices creados antes de la ingesta es correcto, incorrecto, o depende? ¿Qué cambiaría si el endpoint recibe 10,000 batches de 500 filas a lo largo de un día?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| Schema design justificado | 25% | 25 / 25 |
| Ingesta chunkeada eficiente | 20% | 19 / 20 |
| Benchmark con EXPLAIN PLAN y SLAs | 30% | 29 / 30 |
| Comparación SQLite vs DuckDB | 25% | 24 / 25 |
| **Total** | **100%** | **97 / 100** |

---

El ANALYZE antes del benchmark y las dos conexiones separadas son los dos detalles que ningún otro alumno del grupo implementó y que hacen que estas mediciones sean las más confiables del grupo. Para E04 ese mismo nivel de rigor en el diseño del sistema — pensar en los edge cases antes de que fallen — es exactamente lo que se necesita.