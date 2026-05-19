# Ejercicio 2 — El Motor de Consultas

**Módulo:** Python para Sistemas de Datos Modernos
**Duración estimada:** 5-6 horas

---

## El problema

El equipo de analytics necesita responder 8 preguntas de negocio sobre el histórico de transacciones. Tienes tres opciones: pandas (que ya conoces), DuckDB (que un colega recomienda) y polars (que está ganando terreno en la industria). Tu trabajo no es solo implementar las queries — es medir, comparar y entregar una recomendación de arquitectura respaldada por evidencia.

---

## Lo que construirás

Un sistema de benchmark de query engines que implementa las mismas 8 queries en tres engines distintos, valida que los resultados son equivalentes, y produce un reporte comparativo con recomendaciones técnicas.

---

## Prerequisito

Este ejercicio usa el archivo Parquet de 1M filas generado en el Ejercicio 1. Si no completaste el Ejercicio 1, solicita el archivo en el canal **#becarios** antes de empezar.

---

## Las 8 queries

Implementa exactamente estas queries en los tres engines. Los identificadores (Q1–Q8) son los que usarás en tu código y en tu reporte:

| ID | Query |
|---|---|
| Q1 | Conteo total de transacciones por `country_code`, ordenado de mayor a menor. |
| Q2 | Monto promedio, mínimo y máximo agrupado por `category`. |
| Q3 | Top 10 `user_id` por suma de `amount`, incluyendo su conteo de transacciones. |
| Q4 | Conteo de transacciones con `status='failed'` agrupado por hora del día (0–23). |
| Q5 | Transacciones con `amount > 500` en MX o CO, en los últimos 30 días del dataset. |
| Q6 | Por cada `country_code`, la `category` con más transacciones y su monto promedio. |
| Q7 | Usuarios con más de 5 transacciones fallidas. Retornar `user_id` y conteo. |
| Q8 | Monto promedio diario por `category` (un valor por día por categoría). |

---

## Paso 1 — Implementa los tres engines

Crea un directorio `engines/` con un archivo por engine:

- `engines/pandas_engine.py`
- `engines/duckdb_engine.py`
- `engines/polars_engine.py`

Cada archivo debe exponer una función por query con la misma firma en los tres engines. DuckDB debe leer el Parquet directamente — no está permitido cargarlo en pandas primero y pasárselo a DuckDB.

---

## Paso 2 — Benchmark y validación

Escribe un `benchmark.py` que:

- Ejecute las 8 queries en los 3 engines
- Valide que los resultados son **numéricamente equivalentes** entre engines antes de reportar tiempos
- Registre tiempo de ejecución y pico de memoria por query y engine
- Se ejecute así:

```bash
python benchmark.py --output results/
```

> Si pandas y DuckDB no producen el mismo resultado en una query, hay un error en tu implementación. No reportes tiempos sin validar la equivalencia primero.

---

## Paso 3 — Análisis con EXPLAIN ANALYZE

Para DuckDB, incluye el output de `EXPLAIN ANALYZE` para las queries **Q3, Q5 y Q6**. En tu reporte, interpreta cada plan: qué está haciendo DuckDB para ejecutar esa query y por qué es relevante para el rendimiento.

---

## Paso 4 — Reporte

Escribe un `report.md` con:

- Tabla comparativa de los 3 engines en las 8 queries (tiempo y memoria)
- Interpretación de `EXPLAIN ANALYZE` para Q3, Q5 y Q6
- Identificación de tres casos concretos:
  - Una query donde **polars** supera claramente a pandas
  - Una query donde **DuckDB** es el ganador claro
  - Una query donde los **tres son comparables**
- Justificación técnica de por qué ocurre en cada caso — no solo cuál fue más rápido
- Recomendación de qué engine usar según el patrón de acceso

---

## Entregables

```
ejercicio-02-consultas/
├── engines/
│   ├── pandas_engine.py
│   ├── duckdb_engine.py
│   └── polars_engine.py
├── benchmark.py
├── results/
│   └── query_results.json
└── report.md
```

---

## Cómo se evaluará

| Criterio | Peso |
|---|---|
| 8 queries en 3 engines, numéricamente validadas | 25% |
| Interpretación de EXPLAIN ANALYZE para Q3, Q5 y Q6 | 25% |
| Identificación y justificación de tradeoffs | 25% |
| Reporte y recomendación de arquitectura | 25% |

---

## Cómo entregar

Cuando termines, manda un mensaje en el canal **#becarios** de Discord con este formato:

```
Ejercicio 2 listo para revisión
Repo: [link]
Branch: [main u otro]
Nota (opcional): [algo que quieras comentar]
```