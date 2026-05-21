# Ejercicio 3 — La Capa Transaccional

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

El equipo de producto necesita consultas por usuario individual que respondan en menos de 50ms. DuckDB es excelente para analytics pero no fue diseñado para ese patrón de acceso. Tu trabajo es diseñar la capa correcta en SQLite, demostrar con números que funcple los requerimientos, y comparar contra DuckDB para entender cuándo cada uno gana.

---

## Lo que construirás

Una base de datos SQLite optimizada para consultas transaccionales, un pipeline de ingesta eficiente desde el Parquet del Ejercicio 1, y un benchmark que demuestra el impacto real del indexing.

---

## Prerequisito

Este ejercicio usa el Parquet de 1M filas generado en el Ejercicio 1. Si no lo tienes disponible, solicítalo en el canal **#becarios** antes de empezar.

---

## Los 5 patrones de acceso que debes cumplir

Tu schema y tus índices deben estar diseñados para estos patrones específicamente:

| ID | Patrón | SLA |
|---|---|---|
| P1 | Buscar una transacción por `transaction_id` exacto. | < 10ms |
| P2 | Obtener las últimas 20 transacciones de un `user_id`, ordenadas por `timestamp`. | < 50ms |
| P3 | Todas las transacciones de un `user_id` en un rango de fechas dado. | < 50ms |
| P4 | Suma de `amount` de un `user_id` en el último mes. | < 50ms |
| P5 | Todos los `user_id` de un `country_code` con más de N transacciones (N parametrizable). | < 200ms |

---

## Paso 1 — Diseña el schema

Escribe un archivo `schema.sql` con el DDL completo. Decide qué índices crear, de qué tipo, y sobre qué columnas. Documenta cada decisión en un archivo `schema_design.md`: por qué ese índice, por qué ese tipo de dato, por qué esa estructura.

> No crees índices en todas las columnas porque "más es mejor". Cada índice tiene overhead en escritura y ocupa espacio. Justifica cada uno con el patrón de acceso que optimiza.

---

## Paso 2 — Ingesta chunkeada

Escribe `ingest.py` con `--chunk-size` como argumento CLI. La ingesta debe:

- Cargar el Parquet en chunks del tamaño especificado (no cargarlo completo en memoria)
- Usar transacciones explícitas: un `BEGIN` / `COMMIT` por chunk, no por fila
- Aceptar un flag `--wal` para activar `PRAGMA journal_mode=WAL` y medir el impacto
- Reportar progreso durante la ingesta (ej. `Chunk 5/40 insertado en 1.2s`)
- La ingesta de 1M registros debe completarse en menos de 3 minutos

> Un `INSERT` sin transacción explícita hace un commit por fila en SQLite. Eso puede ser 50-100x más lento que un commit por chunk. Mídelo.

---

## Paso 3 — Benchmark de queries e indexing

Escribe `benchmark_queries.py` que ejecute los 5 patrones **con y sin índices**. Para cada caso:

- Mide el tiempo con `time.perf_counter()` (mínimo 3 repeticiones, reporta promedio)
- Captura el output de `EXPLAIN QUERY PLAN` para verificar que el índice se está usando
- Demuestra que con tus índices los patrones P1-P4 cumplen su SLA

Agrega una comparación: para cada patrón, SQLite con índice vs DuckDB sobre el Parquet de E1. ¿Cuál gana y por qué?

---

## Paso 4 — Reporte

Escribe un `report.md` con:

- Justificación técnica de cada decisión de schema (tipos, índices, estructura)
- Tabla de impacto del indexing: tiempo con índice vs sin índice para los 5 patrones
- Resultado de ingesta: tiempo total con y sin WAL mode, tamaño final de la base
- Comparación SQLite vs DuckDB por patrón de acceso con conclusión fundada
- El archivo `.db` **no va en el repositorio** — incluye instrucciones para regenerarlo desde cero con un solo comando

---

## Entregables

```
ejercicio-03-sqlite/
├── schema.sql              ← DDL completo con comentarios
├── schema_design.md        ← justificación de cada decisión
├── ingest.py               ← CLI con --chunk-size y --wal / --no-wal
├── benchmark_queries.py    ← 5 patrones con/sin índices + comparación vs DuckDB
├── results/
│   ├── ingestion_report.json
│   └── query_benchmark.json
└── report.md
```

> El archivo `.db` va en `.gitignore`. Tu `README` debe incluir el comando exacto para regenerar la base desde cero.

---

## Cómo se evaluará

| Criterio | Peso |
|---|---|
| Schema design justificado | 25% |
| Ingesta chunkeada eficiente | 20% |
| Benchmark con EXPLAIN PLAN y SLAs | 30% |
| Comparación SQLite vs DuckDB | 25% |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 3 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```