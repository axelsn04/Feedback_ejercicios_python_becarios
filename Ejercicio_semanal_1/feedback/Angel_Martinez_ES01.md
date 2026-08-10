# Retroalimentación — Proyecto Integrador Semana 9
## Plataforma de Telemetría y Uso para un Servicio de Videollamadas

**Alumno:** Angel Martinez  
**Fecha de revisión:** Junio 2026  
**Calificación:** 86 / 100

---

## Calificación final: 86 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Modelo de datos y elección de motor(es) | 30% | **27 / 30** | Excelente |
| Aislamiento multi-tenant | 25% | **19 / 25** | Bueno |
| Sistema completo (API, pipeline, Docker) | 25% | **23 / 25** | Excelente |
| decisions.md | 20% | **17 / 20** | Excelente |
| **Total** | 100% | **86 / 100** | |

---

## 1. Modelo de datos y elección de motor(es) — 27/30

- La decisión de dos motores (SQLite para sesiones/facturación, DuckDB para telemetría) está bien fundamentada con cifras propias sobre el dataset real: SQLite tardando más de 12 segundos en agregaciones de calidad y superando 1.2 GB en disco, frente a DuckDB cargando los 10.6M de filas en menos de 3 segundos y resolviendo agregaciones complejas en menos de 45ms.
- Los índices (`idx_telemetry_org_ts`, `idx_telemetry_room_ts`, `idx_sessions_org_started`) están en `setup_db.py` y coinciden exactamente con los patrones de consulta reales de `main.py` (filtro por `org_id` + rango de tiempo para telemetría, `org_id` + `started_at` para facturación) — no son índices genéricos.
- La política de retención no solo describe el trade-off (pierdes detalle segundo a segundo después de 30 días) sino que lo cuantifica: más de 95% de reducción en el consumo de almacenamiento de DuckDB.

**Lo que impide el puntaje máximo:** el benchmark que justifica la elección de motor en la sección 1 de `decisions.md` se describe de forma narrativa (una medición por escenario), sin el mismo rigor metodológico que sí aplicaste en la sección 4 para el SLA de `/health` (50 corridas, percentil calculado con `statistics.quantiles`). Aplicar ese mismo estándar de repetición y percentil también a la comparación de motores lo llevaría al nivel más alto sin ambigüedad.

---

## 2. Aislamiento multi-tenant — 19/25

Hay bastante que reconocer aquí antes del punto central:

- El aislamiento no depende de que cada handler recuerde nada: `verify_tenant` es una dependencia de FastAPI (`Depends`) usada en todos los endpoints de negocio, así que un endpoint nuevo hereda la verificación automáticamente en vez de tener que acordarse de añadirla.
- `GET /rooms/{room_id}/history` no se conforma con filtrar — verifica activamente en la base de datos quién es el dueño real de la sala antes de responder, y devuelve `404` (no `403`) precisamente para no confirmarle a un atacante que esa sala existe. Es una decisión de seguridad más sofisticada de lo que pide la rúbrica.
- `POST /transactions/batch` y `POST /pipeline/ingest` verifican que los datos que un tenant intenta escribir realmente le pertenezcan, rechazando (o poniendo en cuarentena) cualquier evento con un `org_id` distinto al del encabezado — esto previene la "polución" de datos entre organizaciones, no solo la lectura cruzada.
- `tests/test_tenant_isolation.py` ataca activamente: pide el historial de la sala de ORG-02 con la cabecera de ORG-01 y exige el 404. No es una verificación indirecta.

**El punto central, y la razón de que esto no sea Excelente:** `X-Organization-ID` es un encabezado que declaras tú mismo — no hay ninguna verificación de que quien lo envía realmente representa a esa organización (no hay API key, token firmado, ni verificación contra un registro de credenciales). Confirmé que la palabra "credential"/"token"/"secret" no aparece en ningún lugar del código. Esto significa que, tal como está, cualquiera que sepa o adivine un `org_id` (siguiendo el patrón `ORG-01`...`ORG-40`, es enumerable) puede hacerse pasar por esa organización simplemente poniendo ese valor en el encabezado. Todo el trabajo cuidadoso de aislamiento que describí arriba protege correctamente una vez que el sistema confía en el encabezado — pero nada impide que ese encabezado mienta. Es la diferencia entre "el sistema aísla a los tenants entre sí" (que sí logras, y muy bien) y "el sistema verifica quién es cada tenant" (que todavía falta), y la garantía de negocio que pide el proyecto depende de la segunda.

---

## 3. Sistema completo (API, pipeline, Docker) — 23/25

- `docker compose up --build` levanta el sistema completo en un comando: `setup` corre con `Dockerfile.setup`, siembra ambas bases de forma idempotente (verificada: `setup_db.py` comprueba `COUNT(*)` antes de volver a insertar), y `api` espera a `service_completed_successfully` antes de arrancar.
- Endurecimiento real presente, no solo mencionado: usuario no-root (`appuser`) en ambos Dockerfiles, puerto expuesto solo en `127.0.0.1`.
- El pipeline de archivo (`/pipeline/ingest`) es sólido: extracción en streaming línea por línea (no carga el archivo completo en memoria), validación multi-error por registro, cuarentena estructurada con marca de tiempo, deduplicación tanto dentro del lote como contra lo ya existente en DuckDB, y aserciones de invariantes en tiempo de ejecución (`filas_validas + filas_rechazadas == filas_extraidas`) que detectarían un bug de conteo antes de que llegara a producción. Corrí la suite de tests localmente (con datos de prueba, no el dataset completo): 13/13 pasaron, incluyendo la prueba de idempotencia y la de p95.

**Lo que impide el puntaje máximo:** `POST /transactions/batch` usa un modelo Pydantic estricto para todo el lote — si un solo evento del lote no pasa la validación, la petición completa se rechaza con 422 (confirmado por tu propio test, que espera ese 422). Es el comportamiento correcto para una API de escritura validada, pero si este endpoint fuera el que recibe el flujo real de `eventos_nuevos.jsonl` en producción, un solo evento sucio tumbaría el lote entero en vez de aislarse. Por suerte, tu propio `/pipeline/ingest` sí está diseñado para ese escenario (streaming + cuarentena por registro) — vale la pena que el README deje explícito cuál de los dos endpoints es el destinado a la ingesta "sucia" en vivo, para que no quede ambigüedad sobre cuál cumple esa garantía.

---

## 4. decisions.md — 17/20

- Responde todas las preguntas obligatorias con cifras propias, no solo justificación cualitativa: rendimiento de motores, ahorro de almacenamiento por retención (>95%), plan concreto de migración a 10x (Parquet particionado por org, PostgreSQL para sesiones, cola de mensajería para desacoplar ingesta).
- La sección 4 ("Mitigación de Errores de Entregas Previas") es un ejercicio honesto de ingeniería iterativa: nombra puntos específicos corregidos (rigor de benchmark, idempotencia, manejo de memoria, concurrencia, hardening) en vez de solo afirmar que el sistema es robusto.
- El apartado de monitoreo de fugas entre tenants es concreto (umbral de más de 3 fallas de `org_id_mismatch` por minuto dispara alerta), no una promesa vaga.

**Lo que falta para el nivel más alto:** la sección de aislamiento se describe como una "garantía desde el diseño" sin nombrar la limitación explicada arriba — que `X-Organization-ID` no está respaldado por ninguna credencial verificable. El propio documento incluso da por hecho, al hablar de monitoreo, que existe "su API key", cuando en el código no hay ninguna. La rúbrica pide explícitamente trade-offs honestos incluso cuando no favorecen la propia decisión — y este es, de todo el documento, el más importante para nombrar.

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- El historial de commits del repositorio muestra progreso incremental real (configuración de Docker, luego endpoints y aislamiento, luego el pipeline, luego tests, luego documentación) en vez de una sola entrega monolítica — es exactamente el rastro que uno esperaría ver si el sistema se construyó por partes, probando cada una.
- Los números que aparecen en `decisions.md` coinciden con lo que el código realmente hace: los índices que describes existen en `setup_db.py` con los mismos nombres y columnas, y el test de p95 que mencionas en la sección 4 es código real que corre y pasa.
- La sección 4 completa —nombrar errores específicos de una entrega anterior y explicar cómo se corrigieron uno por uno— es el tipo de detalle que es difícil de sostener sin haber pasado realmente por esas correcciones.
- El punto que vale la pena poder explicar en una revisión en vivo, sin apoyarte en el documento: ¿por qué `decisions.md` describe el aislamiento como una garantía completa sin mencionar que `X-Organization-ID` no tiene respaldo criptográfico ni de credencial? Si se pensó que un header obligatorio con formato validado ya era suficiente, vale la pena repasar la diferencia entre "requerir un dato" y "autenticar a quien lo envía" — es la brecha de comprensión más importante para cerrar antes del siguiente proyecto.