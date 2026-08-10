# Retroalimentación — Proyecto Integrador Semana 9
## Plataforma de Telemetría y Uso para un Servicio de Videollamadas

**Alumno:** Antonio García
**Fecha de revisión:** Junio 2026
**Calificación:** 68 / 100

---

## Calificación final: 68 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Modelo de datos y elección de motor(es) | 30% | **25 / 30** | Bueno |
| Aislamiento multi-tenant | 25% | **9 / 25** | Insuficiente |
| Sistema completo (API, pipeline, Docker) | 25% | **19 / 25** | Bueno |
| decisions.md | 20% | **15 / 20** | Bueno |
| **Total** | 100% | **68 / 100** | |

---

## 1. Modelo de datos y elección de motor(es) — 25/30

- La división DuckDB (telemetría, OLAP) / SQLite (facturación, OLTP) está bien razonada y es la decisión correcta: telemetría necesita scans agregados sobre 10.6M filas, facturación necesita exactitud transaccional sobre pocas filas. El benchmark propio (`benchmarks/benchmark_engines.py`) corre contra el dataset real y muestra el caso donde SQLite gana (lookup puntual de billing, 0.3ms) frente a donde DuckDB gana (todo lo demás).
- Los índices sí están ligados a un patrón de consulta real y verificado: `idx_sesiones_org ON sesiones (org_id, started_at)` en `setup_db.py` coincide exactamente con el `WHERE org_id = ? AND strftime('%Y-%m', started_at) = ?` de `monthly_usage()`, y `idx_sesiones_org_plan` con el desglose por plan. Esto es exactamente lo que el criterio pide: un índice diseñado para una consulta nombrada, no uno genérico.
- La política de retención (Parquet ~3 meses, sesiones de billing indefinido, buffer de eventos 7 días) es explícita y el trade-off del desfase OLTP/OLAP está bien identificado: un evento nuevo no aparece en `/rooms/active` hasta que el Parquet se regenera.

**Lo que falta para el nivel más alto:**
- El benchmark corre 5 veces por query (cold + promedio de 4 warm) sin un umbral fijado de antemano ni percentil — es evidencia real, pero menos rigurosa que fijar un p95 y un límite aceptable antes de medir.
- La retención está bien justificada en términos de comportamiento, pero no trae una cifra de costo (cuánto disco ahorra el snapshot de 3 meses frente a guardar todo, por ejemplo) — el criterio pide explícitamente que la retención tenga "una consecuencia medible", y aquí es cualitativa.

---

## 2. Aislamiento multi-tenant — 9/25

Este es el punto que más pesa en tu calificación y el más importante para corregir. Antes de los detalles técnicos, la razón central:

**No existe ningún mecanismo de autenticación en el sistema.** Revisé el proyecto completo (`app/`, `pipeline/`, `README.md`, `decisions.md`, `docker-compose.yml`) buscando token, API key, header de autorización o cualquier forma de verificar quién hace una petición, y no hay ninguna. Cada endpoint recibe el `org_id` como parte de la URL y confía en él sin verificarlo contra nada:

```
GET /orgs/ORG-01/rooms/active
GET /orgs/ORG-33/usage?month=2026-04
```

Cualquiera que pueda llamar a la API puede pedir los datos de cualquier organización simplemente cambiando el `org_id` en la ruta — y dado que los IDs siguen el patrón `ORG-01` a `ORG-40`, son triviales de enumerar. Esto es exactamente lo que la guía del proyecto llama "una cláusula de contrato": que los datos de una organización nunca sean visibles para otra. Tal como está, el sistema no lo garantiza frente a nadie externo — solo garantiza que, dado un `org_id`, la respuesta corresponde a ese `org_id`.

**Lo que sí está bien hecho, y vale la pena reconocerlo:**
- Todas las queries filtran por `org_id` de forma consistente y usan parámetros (`WHERE org_id = ?`), nunca concatenación de strings — eso cierra la puerta a inyección SQL, y los tests de `TestInjectionProtection` lo verifican.
- `audit_tenant_isolation()` en `db.py`, que detecta si un mismo `room_id` aparece bajo más de un `org_id` en los datos, es una buena idea de auditoría de datos.
- Los tests de `test_tenant_isolation.py` sí intentan cruzar la frontera activamente (pedir el timeline de una sala de ORG-33 usando la ruta de ORG-01) y no solo verifican que la respuesta trae el `org_id` correcto.

El problema no es la calidad del filtrado — es que filtrar correctamente por un `org_id` que nadie verifica no es aislamiento entre clientes reales, es solo consistencia interna de datos. Para que el aislamiento sea real, el `org_id` que importa tiene que salir de una credencial verificada (un token o API key por organización), no de lo que el llamante decida escribir en la URL.

---

## 3. Sistema completo (API, pipeline, Docker) — 19/25

- `docker compose up --build` levanta el sistema completo de cero en un solo comando: el servicio `setup` carga las sesiones y el servicio `api` espera a que termine (`condition: service_completed_successfully`) antes de arrancar. Esto cumple exactamente el estándar de "un comando, sin pasos manuales adicionales".
- El pipeline offline de ingesta (`pipeline/extract.py` → `transform.py` → `load.py`) maneja bien los problemas reales del dataset: JSON malformado no detiene la extracción, campos inválidos van a `quarantine` con una razón específica, y los timestamps fuera de orden se aceptan por diseño en vez de rechazarse.
- Healthcheck configurado en el Dockerfile y `restart: unless-stopped` en el servicio `api`.

**Lo que reduce la calificación:** el endpoint HTTP `POST /ingest` — el que en producción recibiría el flujo de eventos en vivo — no usa esta misma lógica de validación y cuarentena. Usa un modelo Pydantic con `extra="forbid"` y tipos estrictos, así que si **un solo evento** del lote llega con un campo inválido (por ejemplo, `connection_state: "broken"`), FastAPI rechaza el **lote completo** con un 422 — no solo el evento problemático. Esto está incluso confirmado por tu propio test (`test_ingest_invalid_state_422`, que espera ese 422 como comportamiento correcto). Es lo opuesto de lo que pide la rúbrica: "un evento mal formado... no debería tumbar la ingesta" — aquí sí puede tumbar el resto del lote que sí era válido. La lógica de cuarentena que construiste en `pipeline/` es buena; el endpoint en vivo simplemente no la usa.

---

## 4. decisions.md — 15/20

- Responde las preguntas obligatorias del documento (elección de motores, aislamiento, retención, escenario ×10, monitoreo, detección de fuga) y supera el mínimo de 700 palabras.
- Los hallazgos del dataset (§5) son concretos y específicos: `org_id` es VARCHAR y no INTEGER como se asumió al inicio, ORG-99 existe solo en el stream y no en el histórico, 50% de los timestamps del JSONL llegan desordenados. Ese nivel de detalle indica exploración real del dataset, no una descripción genérica.
- La sección de detección de fuga (§8) es concreta: auditoría de datos, monitor de respuestas, y tests en CI — no es solo "lo revisaríamos".

**Lo que falta para el nivel más alto:** la sección de aislamiento (§2) describe el mecanismo como si fuera una garantía completa ("la arquitectura lo impone") pero en ningún momento reconoce que no hay autenticación — que es, como se explica arriba, la limitación más importante de todo el proyecto. Un documento de decisiones al nivel más alto de la rúbrica pide trade-offs honestos incluso — especialmente — cuando no favorecen al propio diseño. Vale la pena volver a esta sección y nombrar explícitamente esa limitación, igual que sí se nombran otras (como el desfase OLTP/OLAP en §3).

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- Los hallazgos del dataset descritos en `decisions.md` (el tipo de `org_id`, la ausencia de `ORG-99` en el histórico, el porcentaje de timestamps desordenados) coinciden con lo que el código realmente hace con esos casos — no son afirmaciones sueltas de un documento aparte del proyecto.
- El pipeline de ingesta (`pipeline/`) tiene tests propios que corren y pasan, y su diseño (extract → transform → load, cuarentena por razón) es coherente y deliberado, no genérico.
- El punto que vale la pena que puedas explicar en una revisión en vivo, sin apoyarte en el documento: ¿por qué el diseño de aislamiento en `decisions.md` no menciona la ausencia de autenticación como limitación? Si al escribir esa sección se pensó que el filtrado por `org_id` era suficiente por sí solo, es la brecha de comprensión más importante para cerrar antes del siguiente proyecto; si se pensó en ello y se decidió no mencionarlo, vale más la pena decirlo directamente la próxima vez — la rúbrica premia explícitamente los trade-offs que no favorecen la propia decisión.