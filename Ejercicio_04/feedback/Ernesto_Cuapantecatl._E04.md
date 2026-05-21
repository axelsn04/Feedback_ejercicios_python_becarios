# Retroalimentación — Ejercicio 04: El Sistema Completo

**Alumno:** Ernesto Cuapantecatl
**Fecha de revisión:** Mayo 2026
**Calificación:** 98 / 100

---

## Resumen general

La entrega más completa del grupo para E04. Tres decisiones la distinguen del resto: el pre-calentamiento de DuckDB en el lifespan, el `/analytics/summary` que devuelve breakdown por país y categoría (no solo totales), y la deduplicación en dos fases con conteos detallados. El benchmark cubre los 6 endpoints con cold + 99 warm requests y genera el reporte automáticamente. Los 11 tests validan estructura, no solo status codes. El `architecture_decision.md` conecta explícitamente cada decisión con datos de E1, E2 y E3.

---

## Criterio 1 — 6 endpoints funcionales con datos reales `24 / 25`

**Pre-calentamiento de DuckDB en el lifespan** — único en el grupo. El lifespan ejecuta `SELECT COUNT(*) FROM transactions` al arrancar, lo que obliga a DuckDB a leer el Parquet a memoria una sola vez. El efecto medido: cold de 950ms cae a 53ms. Sin esta línea, la primera request del día absorbe todo el costo de I/O del archivo. Con ella, ese costo se paga al arrancar el servidor — el momento correcto.

**`/analytics/summary` con breakdown por país y categoría** — la implementación más completa del grupo. Tres queries DuckDB en una sola llamada: totales globales, agrupación por 15 países, agrupación por 10 categorías. El resultado es un payload rico que un dashboard puede usar directamente sin consultas adicionales.

**`/users/{user_id}/stats` con `top_category` y `country_code`** — también más completo que el resto del grupo. La mayoría solo retorna total_amount y transaction_count; esta implementación agrega las dos columnas de texto más útiles para un perfil de usuario.

**Deduplicación en dos fases con conteos detallados:**
```python
return {
    "received":           len(transactions),
    "duplicates_in_batch": len(transactions) - len(unique),
    "duplicates_in_db":   len(unique) - len(to_insert),
    "inserted":           len(to_insert),
}
```
Es el response más informativo del grupo para el batch insert — un cliente puede distinguir entre duplicados que ya venían repetidos en su propio request y duplicados que ya existían en la base.

**`sqlite3.Row` factory** — permite `dict(r)` en lugar de construcción manual de diccionarios columna por columna. Detalle de calidad de código que hace los endpoints de SQLite más limpios.

Un punto menos: `db.py` no establece `PRAGMA journal_mode=WAL` para SQLite. Los batch inserts que llegan mientras el servidor está corriendo usarán el journal mode por defecto (DELETE), lo que bloquea lectores durante cada commit. El `init_sqlite()` es el lugar correcto para configurar el pragma.

---

## Criterio 2 — Architecture decision justificado `20 / 20`

El `architecture_decision.md` es el más completo del grupo. Cada endpoint tiene su propia sección con justificación técnica, y la decisión de no usar cache en los endpoints de usuario está explícitamente razonada: "los datos de un usuario pueden cambiar con cada batch insert y la latencia ya cumple el SLA sin él". Eso es exactamente la consideración correcta — agregar cache aquí crearía inconsistencias sin beneficio medible.

La sección sobre `/health` sin consultar bases de datos ("solo se lee el timestamp de arranque, el hit rate del cache, y se verifica que las conexiones existen") es una decisión arquitectónica real que la mayoría del grupo no documenta.

El análisis del TTL de 60s es honesto: "un panel de control donde tener los datos al segundo no es crítico, 60s es aceptable. Si el negocio requiriera datos más actualizados, se podría activar una limpieza de memoria automática al recibir un paquete de datos nuevo." Esa observación sobre invalidar el cache al recibir un batch insert es la más madura del grupo — es el patrón correcto para resolver el trade-off.

La sección final que conecta E4 con E1/E2/E3 cierra el módulo de forma coherente: el Parquet de E1 como fuente de DuckDB, las conclusiones de E2 sobre cuándo usar cada engine, y los índices de E3 como la razón por la que los endpoints de usuario responden en 3ms.

---

## Criterio 3 — Cache con TTL y medición `19 / 20`

El impacto del cache está cuantificado con números reales:
- `/analytics/summary`: cold 53.87ms → warm p50 2.13ms (25x)
- `/analytics/top-merchants` (sin filtro): cold 347.36ms → warm p50 2.09ms (166x)
- Promedio analítico: 140.6ms cold → 2.1ms warm → **67x de mejora**

La observación sobre los 950ms sin pre-calentamiento vs 53ms con pre-calentamiento es el análisis más específico del grupo sobre el impacto del lifespan en la latencia cold.

El cache key para top-merchants incluye `limit` y `country`: `f"analytics:top-merchants:{limit}:{country}"`. Esto garantiza que combinaciones distintas de parámetros tienen sus propias entradas en cache — correcto.

Un punto menos: el `architecture_decision.md` menciona la posibilidad de invalidar el cache al recibir un batch insert, pero no está implementado. Es la extensión natural del diseño y habría completado el ciclo de consistencia.

---

## Criterio 4 — Tests con pytest `20 / 20`

11 tests, todos con validación de contenido:

- `/health`: valida `status == "ok"`, `uptime_s`, `cache_hit_rate`, y que ambas conexiones están activas ✅
- `/analytics/summary`: valida que `by_country` y `by_category` son listas y que `total_count > 0` ✅
- `/analytics/top-merchants`: valida que el default retorna exactamente 10 y que `merchant_id` y `total_volume` están presentes ✅
- `/users/1/transactions`: valida estructura con `transaction_id` ✅
- `404` para usuario inexistente ✅
- Paginación fuera de rango → lista vacía (no error) ✅
- `/users/1/stats`: valida los cuatro campos incluyendo `country_code` ✅
- Batch válido: `received==1`, `inserted==1` ✅
- Batch con schema inválido → 422 ✅
- SLA warm: segunda llamada < 20ms ✅

Es la única entrega del grupo que llega a 11 tests con cobertura real de todos los endpoints y casos borde.

---

## Criterio 5 — Benchmark de latencia con cache `15 / 15`

El `run_latency.py` cubre los 6 endpoints incluyendo `top-merchants` con y sin filtro de país — el único alumno del grupo que hace esta distinción (347ms sin filtro vs 20ms con filtro MX). `gc.collect()` antes de cada warm request. p50/p95/p99 con `np.percentile`. Genera `benchmarks/latency_report.md` automáticamente con análisis de impacto del cache y tabla de SLAs calculada programáticamente.

Todos los SLAs se cumplen con margen:
- Analytics cold: 53ms y 347ms (SLA < 500ms)
- Analytics warm p95: 2.45ms y 2.61ms (SLA < 20ms)
- User endpoints p95: 3.11ms y 2.68ms (SLA < 80ms)
- Health p95: 2.57ms (SLA < 50ms)

---

## Sobre el uso de herramientas de IA

El pre-calentamiento de DuckDB, el breakdown por país/categoría en summary, los conteos detallados de deduplicación, y la conexión explícita con E1/E2/E3 en el architecture_decision son decisiones que requieren comprensión del sistema completo. El análisis del TTL y la observación sobre invalidación de cache son genuinas. El typo en el reporte ("La busqueda es 0" en lugar de "O(log n)") sugiere texto que pasó por revisión manual incompleta.

---

## Pregunta de seguimiento

> Tu `architecture_decision.md` menciona invalidar el cache cuando llega un batch insert. ¿Cómo lo implementarías? ¿Invalidarías todas las claves del cache o solo las que podrían estar afectadas por las nuevas transacciones?

---

## Calificación por criterio

| Criterio | Peso | Puntaje |
|---|---|---|
| 6 endpoints funcionales con datos reales | 25% | 24 / 25 |
| Architecture decision justificado | 20% | 20 / 20 |
| Cache con TTL y medición | 20% | 19 / 20 |
| Tests con pytest | 20% | 20 / 20 |
| Benchmark de latencia con cache | 15% | 15 / 15 |
| **Total** | **100%** | **98 / 100** |

---

La entrega más completa del grupo para E04. El pre-calentamiento de DuckDB, los conteos detallados de deduplicación, y el benchmark que cubre todos los endpoints son los tres puntos que la diferencian. El único gap concreto es el WAL pragma en el init de SQLite.