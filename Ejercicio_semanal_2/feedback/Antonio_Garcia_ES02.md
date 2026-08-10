# Retroalimentación — Proyecto Integrador II, Semana 10
## Red de Logística Distribuida

**Alumno:** Antonio García
**Fecha de revisión:** Julio 2026
**Calificación:** 63 / 100

---

## Calificación final: 63 / 100

| Criterio | Peso | Puntos obtenidos | Nivel |
|---|---|---|---|
| Transferencia atómica y modelo de consistencia | 30% | **29 / 30** | Excelente |
| Reconciliación offline | 25% | **7 / 25** | Insuficiente |
| Sistema completo (API, analítica, Docker) | 25% | **17 / 25** | Bueno |
| decisions.md | 20% | **10 / 20** | Regular |
| **Total** | 100% | **63 / 100** | |

---

## 1. Transferencia atómica y modelo de consistencia — 29/30

Esta es la parte más fuerte del proyecto, y cumple el nivel más alto de la rúbrica sin ambigüedad:

- El estado intermedio "en tránsito" está modelado explícitamente: `nodo_actual = NULL` junto con un registro en la tabla `transferencias` — no es un estado implícito que haya que inferir.
- `test_rollback_on_failure_after_insert` es exactamente el test que pide la rúbrica: usa un trigger de SQLite que fuerza un `ABORT` después de que el `INSERT` en `transferencias` ya se ejecutó pero antes de que el `UPDATE` en `paquetes` corra, y verifica que ambas operaciones se revierten — el paquete queda en su nodo original, sin registro de transferencia huérfano. Corrí la suite completa localmente: pasa.
- `transferencias_atascadas()` detecta transferencias cuyo `started_at + sla_horas` ya venció — el mecanismo de detección que la rúbrica pide explícitamente además del rollback.
- El orden de validaciones (primero "¿ya tiene transferencia activa?", después "¿está en el nodo correcto?") está razonado en `decisions.md` con el caso concreto que evita un mensaje de error engañoso — es un detalle fino que muestra que pensaste en la experiencia del operador, no solo en la mecánica.

No tengo una crítica sustancial para este criterio.

---

## 2. Reconciliación offline — 7/25

Aquí está el hallazgo más importante de esta revisión, y vale la pena que lo veas con el mismo detalle con que lo encontré.

`decisions.md` presenta una tabla con 287 conflictos clasificados en 4 categorías (106 `paquete_id_duplicado`, 70 `nodo_dice_entregado_sistema_dice_devuelto`, 58 `nodo_dice_en_transito_sistema_dice_entregado`, 53 `timestamp_imposible`) como evidencia de que el lote de reconciliación offline se procesó y resolvió. Para verificarlo, busqué en todo el repositorio dónde se calcula esa clasificación — y no la encontré:

- `pipeline/reconcile.py` solo hace `event.get("conflicto_tipo")`: **lee** un campo que asume que ya existe en el evento.
- `pipeline/transform.py` únicamente valida formato (campos requeridos, timestamp parseable, tipo de evento válido) — no compara nada contra el estado actual del paquete.
- `pipeline/extract.py` y `benchmarks/explore_dataset.py` tampoco calculan ni derivan `conflicto_tipo` en ningún punto.
- Los únicos lugares donde `conflicto_tipo` tiene un valor son los tests, que lo inyectan directamente como dato de entrada sintético (`_event(conflicto_tipo="paquete_id_duplicado")`).

En otras palabras: el sistema, tal como está, **no determina** si un evento offline es un duplicado o contradice un estado terminal — solo sabe hacerlo si alguien más ya se lo dice de antemano en el campo `conflicto_tipo`. El archivo real `eventos_offline.jsonl` de la guía del proyecto no trae ese campo (los campos documentados son `paquete_id`, `nodo_id`, `timestamp`, `tipo_evento`, `operador_id`, `metadata`). Contra el dataset real, todo evento entraría por la rama "sin conflicto" de `reconcile.py`, y el único filtro real que quedaría en pie es la comparación de timestamps en `actualizar_paquete_offline` — es decir, el sistema terminaría aplicando **last-write-wins puro**, exactamente la política que la rúbrica describe como el piso de "Regular" ("se resuelven con last-write-wins sin documentar esa elección"), pero presentada en `decisions.md` como si fuera una política de tres niveles (`node-authority con excepciones`) ya implementada y verificada.

Esto no es una crítica al diseño de la política — node-authority con excepciones para estados terminales es una buena decisión de negocio, bien argumentada en el texto. El problema es que el mecanismo que la ejecuta contra datos reales (no contra datos ya etiquetados por los tests) no existe todavía en el código.

---

## 3. Sistema completo (API, analítica, Docker) — 17/25

- `docker compose up --build` levanta todo de cero en un comando: `setup` corre antes que `api` vía `service_completed_successfully`, sin pasos manuales adicionales. Esto sí cumple el nivel más alto de este aspecto puntual.
- Los 9 endpoints de negocio existen y cubren las capacidades pedidas: estado de paquete, historial, transferencia (con confirmación), inventario por nodo, operaciones de red, SLA en riesgo, y paquetes perdidos.

**Dos problemas concretos que bajan la nota:**
- El endpoint `POST /reconciliacion` hereda el problema de la sección anterior: existe, responde, y no revienta — pero contra el lote real no clasificaría los conflictos como pretende `decisions.md`. Es una de las capacidades de negocio explícitamente exigidas ("ingerir el lote de reconciliación offline y resolver los conflictos"), y no funciona como se describe.
- El campo `frescura_segundos` en `GET /paquetes/{id}/estado` está fijado a `0.0` en el código, sin importar si la respuesta viene del caché o no (`app/main.py`, línea 94). `decisions.md` lo describe como el mecanismo para que "el consumidor sepa la edad del dato" — pero un consumidor que reciba un dato con 14 segundos de antigüedad de caché vería `frescura_segundos: 0.0` igual, que es información incorrecta, no solo incompleta.

---

## 4. decisions.md — 10/20

El documento está bien estructurado y responde todas las preguntas obligatorias del punto 6 con un formato claro (tabla de benchmarks, tabla de políticas de reconciliación, garantías por endpoint). El problema no es de estructura — es que la sección más citada como evidencia (la tabla de 287 conflictos) no se puede reproducir con el código entregado, tal como se explica en el punto 2. En un curso que ha insistido en "evidencia medible, no promesa" desde la semana de telemetría, una cifra que no se puede regenerar corriendo el propio código es exactamente el tipo de afirmación que la rúbrica quiere que evites, incluso si el resto del documento sí tiene evidencia real (los benchmarks de motor sí están respaldados y son verificables).

---

## Comprensión y apropiación del trabajo

Esto no forma parte de los 100 puntos de la rúbrica, pero vale la pena decirlo con la misma evidencia que el resto del documento:

- El test de rollback con trigger de SQLite (sección 1) es un diseño genuinamente sofisticado — no es algo que se copie sin entender qué prueba ni por qué un trigger `BEFORE UPDATE` simula bien una falla a mitad de transacción.
- El punto que vale la pena resolver antes de la próxima entrega, y que sería la primera pregunta en una revisión en vivo: ¿dónde está — o dónde estaba pensado que estuviera — el código que compara un evento offline contra el estado actual del paquete para producir `conflicto_tipo`? Si existió en algún momento y se perdió, vale la pena recuperarlo; si nunca se escribió, es la pieza más importante que falta para que el proyecto haga lo que `decisions.md` dice que hace.