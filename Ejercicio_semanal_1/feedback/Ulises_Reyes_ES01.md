# Retroalimentación — Proyecto Integrador Semana 9
## Plataforma de Telemetría y Uso para un Servicio de Videollamadas

**Alumno:** Ulises Josue Reyes Martinez  
**Fecha de revisión:** Junio 2026  
**Calificación:** 97 / 100
---

## Calificación final: 97 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Modelo de datos y elección de motor(es) | 30% | **29 / 30** | Excelente |
| Aislamiento multi-tenant | 25% | **25 / 25** | Excelente |
| Sistema completo (API, pipeline, Docker) | 25% | **23 / 25** | Excelente |
| decisions.md | 20% | **20 / 20** | Excelente |
| **Total** | 100% | **97 / 100** | |

---

## 1. Modelo de datos y elección de motor(es) — 29/30

La decisión entre motores no se tomó por default: está respaldada con benchmarks propios sobre los 10.6M eventos reales, con umbral p95 fijado *antes* de correr la prueba (B2, B3, B4), 20 corridas por prueba, y verificación de que ambos motores producen el mismo resultado de facturación al dígito — no solo que "corren rápido".

- **DuckDB descalificó a SQLite en B4** (8,753 ms vs. umbral de 2,000 ms) y **SQLite ganó B2/B3** por 3–7×. La división de motores (SQLite para servicio caliente, DuckDB/Parquet para analítico) es consecuencia directa de esa evidencia, no de preferencia.
- Los índices están atados explícitamente a una consulta de negocio nombrada (`salas_activas(org_id, ultimo_estado)` → "salas degradadas de mi org ahora"), y en la capa fría el "índice" es el orden físico de partición, correctamente entendido como una decisión distinta a un B-tree.
- La política de retención (30 días crudo, 13 meses agregado, indefinido para facturación) viene con su costo medido en MB/día y con lo que explícitamente se pierde pasado ese plazo — cumple exactamente lo que pide el nivel Excelente ("consecuencia medible", no solo una regla arbitraria).

**Lo único que no está del todo cerrado:** el "13 meses" de retención para `agg_sala_minuto` no tiene, a diferencia de todo lo demás en el documento, una cifra o umbral que lo derive (¿por qué 13 y no 12?). Es un detalle menor en un documento donde casi todo lo demás sí está anclado a un número medido — vale la pena que ese estándar sea parejo en todas las decisiones.

---

## 2. Aislamiento multi-tenant — 25/25

Este es el criterio más estricto de la rúbrica y se cumple de forma completa:

- El aislamiento está en el **modelo de datos**, no en la disciplina del handler: un archivo SQLite por organización y una partición Parquet por organización, resueltos en un único módulo (`tenancy.py`) a partir del token autenticado — una fuga requeriría un bug en esa resolución, no un `WHERE` olvidado.
- `tests/test_aislamiento.py` **ataca activamente**: pide datos de ORG-A con el token de ORG-B por tres vectores distintos (path, `room_id`, filtros cruzados), y exige 403 sin ninguna fila filtrada. El test estructural que recorre `app.routes` y aplica el ataque a **cualquier endpoint nuevo bajo `/orgs/`** automáticamente es exactamente el diseño que la rúbrica describe como el techo del criterio: el aislamiento no depende de que alguien se acuerde de probarlo.
- El monitoreo propuesto no es genérico: un canario periódico que exige `count(*) WHERE org_id != tenant` = 0 por cada ruta de tenant es una señal proactiva concreta de fuga, no una promesa de "lo revisaríamos".
- Como evidencia adicional (no como mecanismo del sistema entregado), se corrió un experimento real contra Postgres + Row-Level Security y se documentó un modo de falla legítimo y no trivial: RLS puede fallar **abierto** cuando una conexión reutilizada no resetea su variable de sesión entre requests de distinto tenant. Ese hallazgo refuerza con evidencia, en vez de solo afirmar, por qué el aislamiento físico es la elección más defendible aquí.

Esta suite corre y pasa de forma real (52 tests pasados, 0 fallos).

---

## 3. Sistema completo (API, pipeline, Docker) — 23/25

- La ingesta resiste exactamente lo que la rúbrica pide: eventos fuera de orden (upsert idempotente por `(room_id, timestamp)`), eventos malformados (dead-letter queue en vez de tumbar el proceso), y saturación (backpressure con 429 explícito). El bug de "un evento problemático tumbaba la ingesta de las 40 organizaciones" se documenta como encontrado y corregido, no como un caso hipotético.
- El sistema fue validado contra Docker real y contra el dataset completo (10.6M eventos), no solo contra una muestra de desarrollo — y ahí apareció y se corrigió un bug de memoria real que nunca se manifestó en desarrollo. Esa es precisamente la clase de validación que separa "funciona en mi máquina" de "funciona".
- Endurecimiento de producción presente y no solo mencionado: usuario no-root, límites de recursos, `restart:` policy, imagen reducida de 1.5GB a 350MB por una dependencia de test mal aislada.

**Por qué no es 25/25:** levantar el sistema de cero requiere tres pasos documentados — colocar los datos de origen, `docker compose up --build`, y luego `docker compose run --rm carga` para la carga histórica y la generación de tokens. Esto es exactamente lo que el nivel Excelente de este criterio describe como diferencia con el nivel Bueno: "sin pasos manuales adicionales" vs. "con algún paso manual menor documentado". El paso es simple, está documentado, y no cuestiona que el sistema funcione — pero tal como está redactada la rúbrica, no es un solo comando.

---

## 4. decisions.md — 20/20

Cumple el estándar más alto de este criterio en cada pregunta obligatoria:

- Cada decisión técnica trae una cifra propia (benchmarks con p95, ratios de disco, umbrales derivados de percentiles medidos del propio dataset), no una afirmación genérica.
- Los trade-offs se cuentan con honestidad, incluyendo los que no favorecen la decisión tomada (DuckDB pierde en B2/B3 y SQLite pierde en B4 — ambos se dicen, no solo el que ganó cada motor) y el sub-conteo de facturación que introduce la política de reconstrucción desde telemetría.
- La sección de monitoreo responde de forma concreta y no evasiva la pregunta más difícil del documento: cómo se detectaría una fuga entre tenants antes de que el cliente la reporte.
- Los bugs reales encontrados durante el desarrollo (carrera de arranque worker-vs-API, OOM a escala real, error de método en el propio harness de benchmarks) se documentan como post-mortems con causa raíz, no se ocultan ni se maquillan.

**Una observación, no una deducción:** el documento es considerablemente más largo que el mínimo pedido (bitácora cronológica completa de varios días de trabajo). El rigor es real y verificable, pero en un entorno de equipo, un documento de este tamaño compite con su propia legibilidad — vale la pena pensar, de cara al siguiente proyecto, en separar una síntesis ejecutiva corta (que aquí sí existe y está bien lograda) de la bitácora completa como un documento aparte.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero es relevante y vale la pena decirlo con la misma evidencia que el resto de este documento:

- El código y `decisions.md` están alineados, no son dos discursos separados: `tenancy.py` referencia secciones específicas de la bitácora (§4, §12) y lo que ahí se describe es exactamente lo que el código implementa.
- Los tests no solo existen: corren y pasan. Eso es evidencia de que el sistema descrito es el sistema que realmente está en el repositorio, no una narrativa construida aparte del código.
- Los bugs documentados (la carrera de arranque worker-vs-API, el OOM que solo aparece a los 10.6M eventos reales, el error de método en el propio harness de benchmarks) tienen causa raíz específica y explican por qué los intentos previos no funcionaron. Ese nivel de detalle es difícil de sostener sin haber depurado el problema de verdad.
- Un punto a tener en cuenta, no una acusación: el repositorio se entregó en un solo commit, así que el proceso incremental que la bitácora describe (día 1 a día 4) no queda verificable en el historial de Git. Esto no invalida nada de lo anterior, pero en una revisión en vivo sería razonable poder explicar, sin apoyarte en el documento, por qué se descartó el watermark original a favor de agregados mutables, o por qué DuckDB no sirve como buffer de escritura concurrente — son las dos decisiones más citadas en el proyecto y las más fáciles de verificar si el entendimiento es propio.