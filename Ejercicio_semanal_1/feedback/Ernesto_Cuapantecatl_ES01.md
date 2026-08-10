# Retroalimentación — Proyecto Integrador Semana 9
## Plataforma de Telemetría y Uso para un Servicio de Videollamadas

**Alumno:** Ernesto Cuapantecatl  
**Fecha de revisión:** Junio 2026  
**Calificación:** 93 / 100

---

## Calificación final: 93 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Modelo de datos y elección de motor(es) | 30% | **29 / 30** | Excelente |
| Aislamiento multi-tenant | 25% | **24 / 25** | Excelente |
| Sistema completo (API, pipeline, Docker) | 25% | **21 / 25** | Excelente / Bueno |
| decisions.md | 20% | **19 / 20** | Excelente |
| **Total** | 100% | **93 / 100** | |

---

## 1. Modelo de datos y elección de motor(es) — 29/30

- La comparación de motores no se queda en "DuckDB vs SQLite": también probaste Parquet monolítico contra particionado por día (30 iteraciones por query, según `data-model.md`), y la ganancia de 5-7x con latencia casi constante (~21ms) justificó particionar antes de justificar el motor. Es un nivel de evidencia más fino de lo que pide la rúbrica.
- La decisión de **no** crear el índice compuesto en `sesiones` es la parte que más vale la pena reconocer: en vez de añadirlo "por si acaso", mediste que con 2,833 filas no hay diferencia perceptible, dejaste el schema preparado, y calculaste cuándo sí importaría (~28,000 filas, ~7 meses). Es evidencia contra la tentación de sobre-optimizar, que es exactamente lo que la rúbrica pide premiar.
- La política de retención (HOT/WARM/COLD) trae su costo medido: 83% de reducción de volumen, implementada como código ejecutable con `--dry-run`, no solo como documento.

**Lo único que dejaría perfecto:** todo esto vive repartido en `findings.md` / `data-model.md` / `retention.md`, y `decisions.md` los referencia — es una organización que de hecho funciona muy bien para el lector, pero significa que alguien que solo abra `decisions.md` sin seguir los enlaces se pierde el detalle de las 30 iteraciones por query. Vale la pena un one-liner en `decisions.md` mismo dejándolo explícito.

---

## 2. Aislamiento multi-tenant — 24/25

Esto es lo más completo que he visto en esta ronda de revisiones:

- Autenticación real por token: `Authorization: Token <token>` se resuelve contra un mapeo `token → org_id` cargado del lado del servidor (`API_TOKENS`). El cliente nunca elige su `org_id` — lo determina el token que posee. Esto es una diferencia importante frente a simplemente pedir un encabezado con el `org_id`: aquí hace falta poseer el secreto, no solo declarar una identidad.
- El aislamiento no vive solo en la API: `QualityRepository` y `BillingRepository` exigen `org_id` como primer parámetro posicional obligatorio y lanzan `TenantIsolationError` si falta, es `None`, o no es string — así que incluso si la capa API tuviera un bug, la capa de datos igual rechazaría la llamada. Es aislamiento en dos capas independientes, no una sola.
- Las consultas por recurso (`room_id`) filtran por `org_id` Y `room_id` a la vez, así que pedir la sala de otra organización no da una fuga parcial ni un error revelador — da el mismo resultado vacío/404 que una sala inexistente, cerrando la puerta a enumeración.
- Probado en dos capas: 16 tests adversariales de repositorio (`test_isolation.py`, corrí la suite completa: 16/16 pasan) cubriendo 7 vectores (nombre de recurso ajeno, contaminación de resultados, org vacío, `None`, tipo incorrecto, inyección SQL), más pruebas end-to-end reales por HTTP con tokens (`test_e2e.py`) que verifican 401 sin token o con token inválido, y que dos tokens de distinta organización reciben resultados distintos.

**Lo que falta para el puntaje completo, y está bien que lo sepas:** el propio código de `POST /ingest/events` documenta honestamente que, aunque el token identifica una organización, el endpoint acepta eventos con **cualquier** `org_id` conocido en el archivo subido — no verifica que el `org_id` de cada evento coincida con el del token que autenticó la subida. Está explicado como una decisión de diseño (simular un servicio de ingesta centralizado) y no oculto, lo cual es honesto — pero significa que, tal como está, el token de una organización podría insertar eventos a nombre de otra organización conocida. Es la única grieta real en un diseño que, en todo lo demás, es el más completo que he revisado.

---

## 3. Sistema completo (API, pipeline, Docker) — 21/25

- El pipeline de ingesta resuelve, uno por uno, los cuatro problemas reales que documentaste en `findings.md`: desorden temporal (no asume orden en ningún punto), duplicados exactos (idempotencia por `(room_id, timestamp)` contra lo ya escrito en la partición), valores nulos o mal tipados en métricas (se sanean campo por campo a `None`, sin descartar el evento completo), y la organización desconocida ORG-99 (rechazada a cuarentena con motivo `unknown_org`, no silenciosamente ni de forma destructiva). Cada decisión de `findings.md` tiene su código correspondiente.
- Hardening real: usuario no-root en el Dockerfile, healthcheck configurado, y las pruebas de resiliencia pasan (`test_isolation.py`: 16/16 localmente).

**Por qué no es 25/25:** para llegar a `docker compose up --build` hace falta correr antes, fuera de Docker, `uv run python bench/prepare.py` (el paso que particiona el Parquet por día) — el propio `docker-compose.yml` solo copia esos archivos ya preparados, no los genera. Es un paso real y adicional fuera del contenedor, no solo colocar los datasets originales en una carpeta. Es exactamente la distinción entre "Excelente" y "Bueno" que marca la rúbrica en este criterio: un paso manual documentado, aunque sea simple, antes de que el sistema pueda arrancar solo con Docker.

---

## 4. decisions.md — 19/20

- Cada respuesta trae una cifra propia, no una afirmación general: 5-7x de mejora por particionar, 3-12x de ventaja de SQLite sobre DuckDB para facturación, 83% de reducción por retención, 0.6% de tasa de cuarentena observada (82 de 14,770) con el umbral de alerta puesto en 20%.
- El pivote a mitad de camino (usar `MAX(timestamp)` en vez de un reloj real, y descubrir que cada organización "apaga" su actividad en un momento distinto) es exactamente el tipo de historia honesta que pide la rúbrica: una decisión que cambió por evidencia del dataset, no por preferencia.
- El apartado de monitoreo de fuga entre tenants incluye la query SQL real (`GROUP BY room_id HAVING COUNT(DISTINCT org_id) > 1`) y el resultado actual (0 filas) como línea base — no es una promesa vaga.

**Lo único que le falta para el máximo:** la limitación de `/ingest/events` explicada arriba está honestamente comentada en el código, pero no se repite en la sección de aislamiento de `decisions.md`. Dado que ese documento es el que se lee sin entrar al código, valdría la pena que esa única grieta real del sistema también viviera ahí.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- El historial de commits (7) sigue el mismo orden que describe el propio README como el de lectura recomendada: exploración → benchmarks/modelo de datos → retención → endpoints/aislamiento/pipeline → tests → decisiones y documentación final. Es la secuencia que uno esperaría de alguien construyendo el sistema en el orden en que lo explica, no reconstruyéndola después.
- Los números no solo aparecen en el documento — están en el código y en los tests que corren: los 16 tests de `test_isolation.py` pasan tal cual, y el 0.6% de cuarentena que cita `decisions.md` es coherente con las reglas reales de `pipeline/transform.py`.
- Un detalle que dice más de lo que parece: `findings.md` corrige al propio material del curso ("el documento menciona '~15M', pero el volumen real es 10.6M") — cuestionar la fuente en vez de repetirla es una señal clara de exploración real, no de relleno.
- Si algo vale la pena profundizar en una conversación en vivo: la decisión de que `/ingest/events` no valide el `org_id` de cada evento contra el token que autentica la subida está bien documentada en el código, pero sería la pregunta más natural de una revisión — ¿qué tan lejos de "producción real" te parece esa simplificación, y qué cambiarías primero si tuvieras que cerrarla?